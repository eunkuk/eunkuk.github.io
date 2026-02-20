---
title: "JWT 로그아웃했는데 토큰이 살아있다 — Redis 없이 무효화하기"
date: 2025-12-25 22:00:00 +0900
categories: [Dev, Backend]
tags: [jwt, spring-boot, security, caffeine, cache, logout, token, redis, java]
description: "JWT는 stateless라 로그아웃해도 토큰이 유효하다. Redis 없이 lastLogoutDatetime + Caffeine 캐시로 해결한 과정을 정리한다."
---

## 1. 문제: 로그아웃해도 토큰이 유효하다

Spring Boot + JWT 기반 인증을 쓰고 있었다. 로그인하면 access token과 refresh token을 발급하고, 클라이언트가 Authorization 헤더에 access token을 담아 요청한다. 흔한 구성이다.

문제는 **로그아웃**이었다.

```java
// AuthService.java - logout
public void logout(String userId) {
    // TODO: Redis/DB 기반 블랙리스트 사용 시, 여기서 refreshToken 무효화 처리
    // tokenBlacklistService.blacklist(refreshToken, tokenExpiry);
}
```

TODO만 달아놓고 실제 무효화 로직이 없었다. 사용자가 로그아웃을 해도 기존에 발급된 access token은 만료 시간까지 그대로 살아있다.

JWT는 서버에 상태를 저장하지 않는 것이 원칙이다. 토큰 자체에 만료 시간이 박혀 있으니, 서버가 "이 토큰은 무효"라고 기억하려면 어딘가에 상태를 저장해야 한다. 그게 JWT의 stateless 원칙과 충돌한다.

그래서 보통 Redis에 블랙리스트를 두는데, 이 프로젝트에는 **Redis가 없었다**.

---

## 2. 선택지 검토

여섯 가지 방법을 검토했다.

### Redis 블랙리스트

```
키: bl:refresh:{sha256(token)}
값: (없음)
TTL: 토큰 남은 유효시간
```

가장 정석적인 방법이다. 로그아웃 시 토큰 해시를 Redis에 넣고, 필터에서 매 요청마다 블랙리스트를 확인한다. TTL이 지나면 자동 삭제되니까 관리도 편하다.

문제는 **Redis 의존성이 없다**는 것이었다. 토큰 무효화 하나 때문에 Redis를 새로 올리는 건 과했다.

### DB 블랙리스트 테이블

```sql
CREATE TABLE revoked_refresh_token (
    token_hash VARCHAR(64) PRIMARY KEY,
    expires_at TIMESTAMP,
    created_at TIMESTAMP
);
```

Redis 대신 DB 테이블에 블랙리스트를 저장하는 방법. 스케줄러로 만료된 행을 주기적으로 정리해야 한다. refresh 엔드포인트에서만 확인하면 되니까 매 요청마다 조회할 필요는 없다.

가능하지만, 테이블 하나 더 만드는 게 싫었다.

### refreshTokenHash를 Member에 저장

Member 엔티티에 `SHA-256(refreshToken)` 값을 저장하고, refresh 요청 시 비교하는 방법. 로그아웃하면 null로 밀면 된다.

단점이 명확했다. **한 사용자 = 한 기기**가 된다. 다중 기기 로그인을 지원하려면 별도 테이블이 필요하고, 그러면 DB 블랙리스트와 다를 게 없다.

### lastLogoutDatetime 타임스탬프 (선택)

Member 테이블에 `lastLogoutDatetime` 컬럼 하나만 추가한다. 토큰의 `iat`(발급 시각)과 비교해서, 로그아웃 이후에 발급된 토큰만 허용한다.

```
iat > lastLogoutDatetime → 유효
iat ≤ lastLogoutDatetime → 무효
```

테이블 추가 없이 컬럼 하나로 끝난다. **이걸로 갔다.**

### 극단적 선택: access token 유효시간을 5분으로

access token을 5분으로 극단적으로 줄이면 로그아웃 후 최대 5분만 버티면 된다. DB 조회도 필요 없다.

잠깐 고려했지만 바로 접었다. 5분마다 refresh 요청이 들어오면 그것도 비용이다.

---

## 3. 구현: lastLogoutDatetime

### DB 마이그레이션

```sql
ALTER TABLE member ADD COLUMN mem_lastlogout_datetime TIMESTAMP NULL;
```

### Member 엔티티

```java
@Column(name = "mem_lastlogout_datetime")
private LocalDateTime lastLogoutDatetime;
```

### 로그아웃 시 타임스탬프 갱신

```java
public void logout(String userId) {
    memberRepository.findByUserId(userId)
        .ifPresent(m -> m.setLastLogoutDatetime(LocalDateTime.now()));
}
```

이 한 줄이 핵심이다. 로그아웃하면 현재 시각을 찍는다. 이후 이 시각 이전에 발급된 토큰은 전부 무효가 된다.

### Refresh 시 검증

```java
Date issuedAt = jwtTokenProvider.getClaimsEvenIfExpired(refreshToken).getIssuedAt();

if (issuedAt != null && member.getLastLogoutDatetime() != null) {
    LocalDateTime tokenIat = LocalDateTime.ofInstant(
        issuedAt.toInstant(), ZoneId.systemDefault());

    if (!tokenIat.isAfter(member.getLastLogoutDatetime())) {
        log.warn("Refresh failed - token issued before last logout. userId={}", userId);
        throw new ApiException(ErrorCode.LOGIN_FAILED);
    }
}
```

refresh token의 `iat`가 `lastLogoutDatetime` 이전이면 거부한다. 로그아웃 후 탈취된 refresh token으로 재발급하려는 시도를 막는다.

### JwtAuthenticationFilter에서 매 요청 검증

```java
Claims claims = jwtTokenProvider.getClaims(token);
Date issuedAt = claims.getIssuedAt();
String userId = claims.getSubject();

Member member = memberRepository.findByUserId(userId)
    .orElseThrow(() -> new ApiException(ErrorCode.UNAUTHORIZED));

if (member.getLastLogoutDatetime() != null) {
    LocalDateTime tokenIat = issuedAt.toInstant()
        .atZone(ZoneId.systemDefault()).toLocalDateTime();
    if (!tokenIat.isAfter(member.getLastLogoutDatetime())) {
        throw new ApiException(ErrorCode.INVALID_TOKEN);
    }
}
```

여기서 문제가 생겼다. **매 요청마다 DB를 조회한다.**

---

## 4. 문제: 매 요청마다 DB 조회

`JwtAuthenticationFilter`는 인증이 필요한 모든 요청에서 실행된다. 요청 하나당 `memberRepository.findByUserId()` 쿼리 하나가 날아간다.

트래픽이 적으면 괜찮지만, API 요청이 늘어나면 DB 부하가 직결된다. `lastLogoutDatetime` 하나 확인하자고 매번 Member를 풀로 조회하는 건 비용이 너무 컸다.

### 해결: Caffeine 캐시

인증에 필요한 최소한의 정보만 로컬 캐시에 올렸다.

```java
public record MemberAuthSnapshot(
    Long id,
    Integer state,
    LocalDateTime lastLogoutDatetime
) {}
```

Member 전체가 아니라 인증 판단에 필요한 세 필드만 담는 record다.

```java
@Component
@RequiredArgsConstructor
public class MemberAuthSnapshotService {
    private final MemberRepository memberRepository;
    private final Cache<String, MemberAuthSnapshot> memberAuthCache;

    public MemberAuthSnapshot getByUserId(String userId) {
        return memberAuthCache.get(userId, k ->
            memberRepository.findByUserId(k)
                .map(m -> new MemberAuthSnapshot(
                    m.getId(), m.getState(), m.getLastLogoutDatetime()))
                .orElse(null));
    }

    public void evict(String userId) {
        if (userId != null) memberAuthCache.invalidate(userId);
    }
}
```

`Cache.get(key, loader)` — 캐시에 있으면 바로 반환, 없으면 DB 조회 후 캐시에 넣는다.

```java
@Configuration
public class CacheConfig {
    @Bean
    public Cache<String, MemberAuthSnapshot> memberAuthCache() {
        return Caffeine.newBuilder()
            .maximumSize(100_000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .build();
    }
}
```

TTL 5분, 최대 10만 건. 로그아웃하면 `evict()`로 해당 사용자 캐시를 날린다.

이제 필터는 이렇게 바뀐다.

```java
MemberAuthSnapshot snapshot = memberAuthSnapshotService.getByUserId(userId);

if (snapshot == null)
    throw new ApiException(ErrorCode.UNAUTHORIZED);
if (snapshot.state() != null && snapshot.state() != 0)
    throw new ApiException(ErrorCode.ACCOUNT_DISABLED);
if (snapshot.lastLogoutDatetime() != null) {
    LocalDateTime tokenIat = issuedAt.toInstant()
        .atZone(ZoneId.systemDefault()).toLocalDateTime();
    if (!tokenIat.isAfter(snapshot.lastLogoutDatetime()))
        throw new ApiException(ErrorCode.INVALID_TOKEN, "로그아웃 이후 발급된 토큰만 허용됩니다.");
}
```

DB 조회가 캐시 조회로 바뀌었다. 캐시 미스가 나야 DB를 한 번 타고, 이후 5분간은 메모리에서 바로 꺼낸다.

---

## 5. 덤: tokenType 클레임

구현하다 보니 하나 더 발견했다. **refresh token을 Authorization 헤더에 넣어도 필터를 통과하는** 문제였다.

access token과 refresh token 모두 같은 secret key로 서명하니까, 필터의 `validateToken()`은 둘 다 통과시킨다. refresh token이 Authorization 헤더에 들어오면 캐시 조회까지 다 하고 나서야 인가 단계에서 실패하거나, 심하면 그냥 통과한다.

토큰 생성 시 `tokenType` 클레임을 추가했다.

```java
// access token 생성
.claim("tokenType", "access")

// refresh token 생성
.claim("tokenType", "refresh")
```

필터에서 캐시 조회 **전에** 먼저 확인한다.

```java
String tokenType = claims.get("tokenType", String.class);
if (!"access".equals(tokenType)) {
    throw new ApiException(ErrorCode.INVALID_TOKEN, "access 토큰만 허용됩니다.");
}
```

refresh token이 헤더에 들어오면 DB/캐시 조회 없이 바로 거부된다.

---

## 6. 돌아보며

1. **JWT 무효화에 Redis가 필수는 아니다.** `lastLogoutDatetime` 컬럼 하나로 "로그아웃 이후 발급된 토큰만 허용"이라는 규칙을 구현할 수 있다.
2. **대신 per-user 무효화다.** 특정 토큰 하나만 골라서 무효화할 수는 없다. 로그아웃하면 해당 사용자의 모든 토큰이 무효가 된다. 단일 세션 모델이라면 충분하다.
3. **매 요청 DB 조회는 반드시 캐싱하자.** 인증 필터는 모든 요청에서 실행된다. Caffeine 같은 로컬 캐시로 DB 부하를 줄일 수 있다.
4. **Caffeine은 단일 인스턴스 한정이다.** 서버가 여러 대면 캐시 eviction이 전파되지 않는다. 다중 인스턴스가 필요해지면 그때 Redis를 붙이면 된다. 지금 필요 없는 인프라를 미리 올릴 이유는 없다.
5. **tokenType 클레임은 생각보다 중요하다.** access와 refresh를 같은 키로 서명하면 필터가 구분하지 못한다. 클레임 하나 추가하는 것만으로 불필요한 처리를 막을 수 있다.
