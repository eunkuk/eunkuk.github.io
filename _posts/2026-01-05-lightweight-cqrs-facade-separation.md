---
title: "풀 CQRS 없이 Command/Query를 분리한 방법 — Facade 패턴으로 경량 CQRS"
date: 2026-01-05 22:00:00 +0900
categories: [Dev, Architecture]
tags: [cqrs, facade, spring-boot, transaction, ddd, read-model, query-separation, retrospective]
description: "이벤트 소싱 없이도 Command와 Query를 분리할 수 있다. Facade 레벨에서 경량 CQRS를 적용한 과정과 readOnly 트랜잭션, Cross-Domain 읽기 전용 조합 패턴을 기록한다."
---

> 이벤트 소싱 없이도 Command와 Query를 분리할 수 있다. Facade 레벨에서 경량 CQRS를 적용한 과정과 readOnly 트랜잭션, Cross-Domain 읽기 전용 조합 패턴을 기록한다.

---

## 1. 상황

> 이 글은 [모노레포 + 6모듈 구조를 선택한 경위](/posts/monorepo-multi-module-retrospective/)와 [Facade의 위치와 트랜잭션 경계](/posts/spring-boot-facade-transaction-architecture/)의 연장이다. 이전 글에서 **왜** 이 구조를 택했고, Facade를 **어디에** 두는지를 다뤘다면, 여기서는 Command와 Query를 **어떻게** 분리했는지를 기록한다.

서비스가 커지면서 하나의 Facade에 조회와 변경 로직이 섞이기 시작했다.

```java
@Service
@RequiredArgsConstructor
public class MemberFacade {

    // Command — 데이터를 변경한다
    public Member createMember(MemberCreateRequest request) { ... }
    public void updatePassword(Long memberId, PasswordUpdateRequest request) { ... }

    // Query — 데이터를 읽기만 한다
    public MemberDetailResponse getMemberDetail(Long memberId) { ... }
    public Page<MemberListResponse> searchMembers(MemberSearchCondition condition, Pageable pageable) { ... }
}
```

`updatePassword()`와 `searchMembers()`가 같은 클래스에 있다. 읽기와 쓰기의 트랜잭션 특성이 다른데 한 곳에 공존하면 문제가 생긴다.

- `updatePassword()`에는 `@Transactional`이 필요하다 — dirty checking, 커밋, 롤백.
- `searchMembers()`에는 `@Transactional(readOnly = true)`가 적합하다 — 스냅샷 비교 생략, read replica 라우팅.

클래스 레벨에서 `@Transactional`을 선언하면 조회 메서드까지 쓰기 트랜잭션으로 동작한다. `readOnly = true`를 클래스 레벨에 선언하면 변경 메서드에서 일일이 오버라이드해야 한다. 어느 쪽이든 실수 여지가 생긴다.

---

## 2. CQRS란 무엇인가

CQRS(Command Query Responsibility Segregation)는 Greg Young이 Bertrand Meyer의 CQS(Command-Query Separation) 원칙을 아키텍처 레벨로 확장한 개념이다. 핵심은 단순하다 — **데이터를 변경하는 책임과 데이터를 읽는 책임을 분리한다.**

CQS는 메서드 레벨의 원칙이다. "하나의 메서드는 상태를 변경하거나(Command), 값을 반환하거나(Query), 둘 중 하나만 해야 한다." CQRS는 이 원칙을 서비스 또는 모델 레벨로 끌어올린 것이다.

### 풀 CQRS

풀 CQRS는 Write Model과 Read Model을 완전히 분리한다.

```
[Command] → Write Model → Event Store → Event Bus → Projection → Read Model → [Query]
```

- 별도의 Write Model과 Read Model
- 이벤트 소싱 — 상태 변경을 이벤트로 저장
- Eventual Consistency — Write와 Read 사이의 동기화 지연 허용
- 별도의 Read DB (Elasticsearch, Redis, 별도 RDBMS 등)

### 경량 CQRS

경량 CQRS는 같은 DB, 같은 모델을 사용하되, **진입점(서비스 또는 Facade)을 분리**한다.

```
[Command] → CommandFacade → DomainService → 같은 DB
[Query]   → QueryFacade   → DomainService → 같은 DB
```

- 같은 DB, 같은 엔티티
- 이벤트 소싱 없음, Strong Consistency 유지
- 진입점만 분리 — Command용 Facade와 Query용 Facade

### 스펙트럼으로서의 CQRS

CQRS는 all-or-nothing이 아니다. 스펙트럼이다.

```
0 ──────────── 3 ──────────── 6 ──────────── 10
분리 없음     Facade 분리     별도 Read Model    이벤트 소싱
(단일 Service)  (같은 DB)      (별도 DB)         (Event Store)
              ↑
              우리의 위치
```

0은 하나의 서비스에 Command와 Query가 전부 섞여 있는 상태. 10은 이벤트 소싱 + 별도 Read DB + Eventual Consistency까지 갖춘 풀 CQRS. 자기 프로젝트의 규모와 복잡도에 맞는 위치를 정하면 된다.

---

## 3. 우리가 선택한 위치: Facade 레벨 분리

스펙트럼에서 3 정도의 위치를 택했다. Facade를 Command와 Query로 분리한다.

### 네이밍 규칙

| Facade 유형 | 네이밍 | 역할 |
|------------|--------|------|
| Command | `{Domain}Facade` | 생성, 수정, 삭제 + side effect |
| Query | `{Domain}QueryFacade` | 조회, 검색, 통계 |

Command Facade는 기본 이름을 가진다. Query Facade에 `Query` 접미사를 붙인다. 이름만 보고 이 클래스가 데이터를 변경하는지, 읽기만 하는지 알 수 있다.

### 분리 전 vs 분리 후

**분리 전** — 하나의 Facade에 Command와 Query가 공존:

```java
@Service
@RequiredArgsConstructor
public class MemberFacade {

    public Member createMember(MemberCreateRequest request) { ... }
    public void updatePassword(Long memberId, PasswordUpdateRequest request) { ... }
    public MemberDetailResponse getMemberDetail(Long memberId) { ... }
    public Page<MemberListResponse> searchMembers(MemberSearchCondition condition, Pageable pageable) { ... }
}
```

**분리 후** — Command와 Query를 각각의 Facade로:

```java
// Command — 데이터를 변경한다
@Service
@RequiredArgsConstructor
public class MemberFacade {

    private final MemberDomainService memberDomainService;
    private final MemberAuthSnapshotService memberAuthSnapshotService;
    private final PasswordEncoder passwordEncoder;

    public Member createMember(MemberCreateRequest request) {
        String encodedPassword = passwordEncoder.encode(request.getPassword());
        Member member = memberDomainService.createAndSave(request, encodedPassword);
        memberAuthSnapshotService.refreshSnapshot(member.getId());
        return member;
    }

    public void updatePassword(Long memberId, PasswordUpdateRequest request) {
        memberDomainService.updatePassword(memberId, request);
        memberAuthSnapshotService.evictSnapshot(memberId);
    }
}
```

```java
// Query — 데이터를 읽기만 한다
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class MemberQueryFacade {

    private final MemberDomainService memberDomainService;
    private final VoucherDomainService voucherDomainService;

    public MemberDetailResponse getMemberDetail(Long memberId) {
        Member member = memberDomainService.getById(memberId);
        List<Voucher> vouchers = voucherDomainService.getByMemberId(memberId);
        return MemberDetailResponse.of(member, vouchers);
    }

    public Page<MemberListResponse> searchMembers(MemberSearchCondition condition, Pageable pageable) {
        return memberDomainService.search(condition, pageable)
                .map(MemberListResponse::from);
    }
}
```

차이가 보이는가.

- `MemberFacade`는 `PasswordEncoder`, `MemberAuthSnapshotService` 같은 인프라 의존성이 있다. 데이터를 변경하고 side effect를 발생시킨다.
- `MemberQueryFacade`는 DomainService만 의존한다. 데이터를 읽기만 하고, 클래스 레벨에 `@Transactional(readOnly = true)`를 선언한다.

### 분리 대상 판단 기준

모든 Facade를 분리할 필요는 없다. 판단 기준은 이렇다.

| 조건 | 분리 여부 |
|------|----------|
| 조회 메서드가 2개 이상이고 독립된 조회 패턴이 있다 | 분리 |
| Command와 Query의 의존성이 다르다 | 분리 |
| 조회 메서드가 1개이고 Command와 의존성이 동일하다 | 유지 |

`PostFacade`에 `createPostMultipart()`와 `getPost()`가 함께 있다면, `getPost()`가 단순 조회이고 `PostQueryFacade`로 분리하는 것이 자연스러운 경우가 분리 대상이다.

---

## 4. readOnly 트랜잭션의 실질적 이점

QueryFacade에 `@Transactional(readOnly = true)`를 클래스 레벨에 선언하는 이유가 있다.

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)  // 클래스 레벨 선언
public class MemberQueryFacade {
    // 이 클래스의 모든 메서드는 readOnly 트랜잭션으로 동작한다
}
```

### 1. Hibernate dirty checking 비활성화

Hibernate는 트랜잭션이 끝날 때 영속성 컨텍스트에 있는 모든 엔티티의 스냅샷을 비교해서 변경된 필드를 찾는다. 이것이 dirty checking이다.

`readOnly = true`이면 Hibernate는 스냅샷 자체를 저장하지 않는다. 엔티티를 읽어오되, 변경 감지를 위한 복사본을 만들지 않는다. flush도 하지 않는다.

조회 Facade에서 수십 개의 엔티티를 읽어올 때, 이 스냅샷 비교가 생략되는 것은 실질적인 메모리와 CPU 절감이다.

### 2. DB 레플리케이션 환경에서 read replica 라우팅

Spring의 `AbstractRoutingDataSource`를 사용하면 트랜잭션의 `readOnly` 속성에 따라 DataSource를 분기할 수 있다.

```
readOnly = true  → Read Replica (SELECT 전용)
readOnly = false → Primary (INSERT, UPDATE, DELETE)
```

현재 프로젝트에서 레플리케이션을 쓰고 있지는 않다. 하지만 QueryFacade에 `readOnly = true`를 선언해두면, 나중에 레플리케이션을 도입할 때 코드 변경 없이 라우팅이 가능하다.

### 3. 의도 명시

가장 중요한 이점이다.

`@Transactional(readOnly = true)`가 클래스 레벨에 있으면, 이 클래스를 여는 누구든(미래의 나 포함) **"이 클래스는 데이터를 변경하지 않는다"**는 것을 즉시 안다. 여기에 `update()` 메서드를 추가하려는 유혹이 생기면 클래스 이름(`QueryFacade`)과 어노테이션이 경고를 준다.

코드가 의도를 선언하는 것. 1인 개발에서 이것이 곧 자기 문서화다.

---

## 5. Cross-Domain Aggregation — 읽기 전용 조합 Facade

CQRS에서 Read Model은 여러 Aggregate의 데이터를 조합해서 클라이언트가 필요한 형태로 제공하는 역할을 한다. 풀 CQRS에서는 이벤트를 프로젝션해서 별도의 Read DB에 저장한다.

우리 프로젝트에서는 이벤트 소싱 없이, 같은 DB에서 여러 DomainService를 조합하는 **읽기 전용 Facade**가 이 역할을 한다.

### GenePlanFacade — 실제 예시

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class GenePlanFacade {

    private final GeneDomainService geneDomainService;
    private final GeneResultDomainService geneResultDomainService;
    private final VoucherKitDomainService voucherKitDomainService;
    private final VoucherPlanDomainService voucherPlanDomainService;

    public GeneResultResponse getGeneByVoucherId(Long voucherId) {
        VoucherKit kit = voucherKitDomainService.getLatestKit(voucherId);
        Gene gene = geneDomainService.getByBarcode(kit.getBarcode());
        Map<String, String> wgrsScores = geneResultDomainService.getResultMap(gene.getId());
        List<Plan> plans = voucherPlanDomainService.getPlansByVoucherId(voucherId);
        return GeneResultResponse.of(gene, wgrsScores, plans);
    }
}
```

이 Facade는 네 개의 DomainService를 조합한다.

```
Voucher → VoucherKit(barcode) → Gene → GeneResult(W-GRS 점수)
                                     → Plan(검사 플랜)
```

이용권으로 키트를 찾고, 키트의 바코드로 유전자 데이터를 찾고, 유전자 데이터의 결과와 검사 플랜을 조합해서 하나의 응답을 만든다. 각 DomainService는 자기 Aggregate만 알고 있다. 이 탐색 경로를 아는 것은 GenePlanFacade뿐이다.

### 풀 CQRS의 Read Model과의 대응

| 풀 CQRS | 우리 프로젝트 |
|---------|-------------|
| Event Store → Projection → Read DB | DomainService 조합 → 같은 DB |
| 별도의 Read Model 클래스 | `GeneResultResponse` DTO |
| Eventual Consistency | Strong Consistency |
| 이벤트 핸들러가 프로젝션 유지 | Facade가 호출 시점에 조합 |

같은 역할(여러 Aggregate를 조합해서 조회 최적화된 응답을 제공)을 하지만, 인프라 복잡도가 다르다.

### 네이밍 규칙: Cross-Domain Facade

```
{Domain1}{Domain2}Facade  — 이름부터 cross-domain임을 명시
```

`GenePlanFacade`는 Gene과 Plan(Voucher를 통한) 도메인을 조합한다. 이름만 보고 단일 도메인이 아닌 cross-domain 조합임을 알 수 있다. `OrderDetailFacade`도 마찬가지다 — Order, Payment, Product, Shipping을 조합한다.

### Application Service에서 조합하는 패턴과의 차이

"Facade를 안 쓰고 Application Service에서 여러 DomainService를 직접 조합하면 안 되나?"

된다. 하지만 프로젝트에서는 두 가지 기준으로 구분한다.

| 기준 | Application Service | Cross-Domain Facade |
|------|-------------------|-------------------|
| 사용처 | 한쪽(Admin 또는 Service)에서만 사용 | 양쪽에서 공유 |
| 모듈 위치 | `admin-api` 또는 `service-api` | `app-common` |

Admin에서만 필요한 조합이면 Application Service에서 직접 한다. 양쪽에서 공유해야 하면 `app-common`의 Facade로 올린다.

---

## 6. 풀 CQRS까지 가지 않은 이유

경량 CQRS를 택한 것은 의도적인 선택이었다.

### 조회 패턴이 단순하다

유전자검사 결과 조회는 복잡한 집계가 아니다. 이용권 → 키트 → 유전자 데이터 → 결과로 이어지는 탐색 경로가 정해져 있고, 대부분의 조회가 단일 엔티티 또는 소수의 엔티티 조합이다. Elasticsearch 같은 별도의 검색 엔진이 필요한 수준의 조회 복잡도가 아니다.

### 이벤트 소싱의 복잡도

풀 CQRS를 도입하면 따라오는 것들이 있다.

- **Event Store**: 상태 변경을 이벤트로 저장하는 별도의 저장소
- **Projection**: 이벤트를 Read Model로 변환하는 프로세서
- **Eventual Consistency**: Write와 Read 사이의 동기화 지연. "방금 수정했는데 조회하면 이전 데이터가 나온다"를 사용자에게 설명해야 한다
- **이벤트 버전 관리**: 이벤트 스키마가 변경되면 기존 이벤트와의 호환성 처리

이것들은 각각이 상당한 설계와 운영 복잡도를 가진다.

### 1인 개발에서의 인프라 오버헤드

풀 CQRS를 운영하려면 인프라가 따라와야 한다.

| 필요한 인프라 | 용도 |
|-------------|------|
| 메시지 브로커 (Kafka, RabbitMQ) | 이벤트 전달 |
| 별도 Read DB (Elasticsearch, Redis) | 조회 최적화 |
| 프로젝션 프로세서 | 이벤트 → Read Model 변환 |
| 모니터링 | Projection lag, dead letter queue |

1인 개발에서 이 인프라를 구축하고 유지하는 비용이 CQRS가 주는 이점보다 크다.

### Facade 분리만으로 충분한 이점

현재 규모에서 Facade 분리만으로 이미 핵심 이점을 확보하고 있다.

| 이점 | 풀 CQRS | Facade 분리 |
|------|---------|------------|
| Command/Query 책임 분리 | O | O |
| 트랜잭션 최적화 (readOnly) | O | O |
| 의도 명시 (클래스 이름, 어노테이션) | O | O |
| 조회 성능 독립 확장 | O | X |
| Eventual Consistency 허용 | O | X |

첫 세 가지가 현재 프로젝트에서 필요한 이점이다. 나머지 두 가지는 현재 규모에서 필요하지 않다.

---

## 7. 돌아보며

CQRS는 all-or-nothing이 아니다.

"이벤트 소싱을 안 쓰면 CQRS가 아니다"라는 건 오해다. Greg Young 본인도 CQRS가 이벤트 소싱을 전제하지 않는다고 말했다. Command와 Query의 책임을 분리하는 것 자체가 CQRS의 본질이고, 그 분리의 깊이는 프로젝트의 요구에 맞추면 된다.

Facade 레벨 분리는 스펙트럼에서 가장 적은 비용으로 핵심 이점을 가져가는 방법이다.

> 1. **트랜잭션 최적화** — QueryFacade에 `readOnly = true` 클래스 레벨 선언. dirty checking 비활성화, 미래의 read replica 라우팅 준비.
> 2. **의도 명시** — 클래스 이름과 어노테이션이 "이 코드가 데이터를 변경하는가, 읽기만 하는가"를 선언한다.
> 3. **책임 분리** — Command Facade는 인프라 의존성(캐시, 보안)을 가진다. Query Facade는 DomainService만 의존한다. 의존성 구조 자체가 다르다.

스펙트럼에서 자기 위치를 정하고, 그 위치에서 최대한의 이점을 가져가는 것. 서비스가 성장해서 조회 패턴이 복잡해지면 그때 한 칸 더 옮기면 된다. Facade 분리가 되어 있으면 Query 쪽만 별도의 Read Model로 교체하는 것도 상대적으로 수월하다 — 이미 진입점이 분리되어 있으니까.

---

**시리즈 연결:**
- [모노레포 + 6모듈 구조를 선택한 경위](/posts/monorepo-multi-module-retrospective/) — 6모듈 구조의 **왜**
- [Facade는 왜 Domain이 아닌 Application 레이어에 있는가](/posts/spring-boot-facade-transaction-architecture/) — Facade 위치, 트랜잭션 경계의 **규칙**
- 이 글 — Command/Query 분리의 **어떻게** (CQRS 관점)
