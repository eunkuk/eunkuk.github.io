---
title: "6모듈 멀티모듈에서 인프라를 다시 세우다 — 마이그레이션 회고 (4) 인증, 보안, 인프라"
date: 2026-02-21 22:00:00 +0900
categories: [Dev, Migration]
tags: [spring-boot, jwt, security, fcm, caffeine, bucket4j, undertow, infrastructure]
description: "PHP 세션 기반 인증에서 JWT 3종 토큰으로, 중복된 알림 로직에서 추상화된 알림 체계로, 수동 히스토리에서 Hibernate Envers로. 마이그레이션에서 보이지 않는 인프라 비용을 기록한다."
---

> PHP 세션 기반 인증에서 JWT 3종 토큰으로, 중복된 알림 로직에서 추상화된 알림 체계로, 수동 히스토리에서 Hibernate Envers로. 마이그레이션에서 보이지 않는 인프라 비용을 기록한다.

---

## 1. PHP의 인증

PHP 프로젝트의 인증은 세션 + DB 토큰 혼합 방식이었다.

### Admin: 세션 기반

관리자 페이지는 PHP 세션을 사용했다. 로그인하면 `$_SESSION`에 사용자 정보를 넣고, 매 요청마다 세션 유효성을 확인한다.

```php
// Admin 인증 — 세션 체크
class CB_Admin_Controller extends CB_Controller
{
    public function __construct()
    {
        parent::__construct();
        if (!$this->session->userdata('admin_id')) {
            redirect('/admin/login');
        }
    }
}
```

세션 만료 시간이 고정이고, 갱신 메커니즘이 없다. 관리자가 긴 작업을 하다가 세션이 만료되면 입력하던 데이터를 잃는다.

### API: DB 토큰 + exit 패턴

모바일 앱용 API는 DB에 저장된 토큰으로 인증한다.

```php
// API 인증 — DB 토큰
class Api_voucher extends CB_Controller
{
    public function __construct()
    {
        parent::__construct();
        $token = $this->input->get_request_header('Authorization');
        $member = $this->Member_model->get_by_token($token);
        if (!$member) {
            echo json_encode(['status' => 'error', 'message' => '인증 실패']);
            exit;
        }
        $this->member = $member;
    }
}
```

문제가 여러 가지다.

| 문제 | 영향 |
|------|------|
| 토큰 갱신 없음 | 만료되면 재로그인 필요 |
| 매 요청 DB 조회 | `get_by_token()` → 매 API 호출마다 SELECT |
| exit 패턴 | 미들웨어 없이 각 컨트롤러에서 처리 |
| 토큰 무효화 불가 | 로그아웃해도 토큰이 유효 |
| 12개 컨트롤러에 중복 | 인증 로직 수정 시 12곳 변경 |

---

## 2. Spring Boot JWT 3종 토큰

Spring Boot에서는 JWT 기반 인증으로 전환했다. 토큰을 3종류로 분리한다.

| 토큰 | 유효시간 | 용도 |
|------|---------|------|
| Access Token | 30분 | API 요청 인증. 서버에 상태를 저장하지 않아 매 요청 DB 조회가 없다. 유효시간이 짧아 탈취 시 영향 범위를 제한한다 |
| Refresh Token | 14일 | Access Token 만료 시 재발급. PHP에는 이 개념이 없어 만료되면 재로그인해야 했다 |
| Register Token | 30분 | KCP 본인인증 후 회원가입까지의 중간 상태. 인증 결과(CI, 이름, 전화번호)를 토큰에 담아 세션이나 DB 저장 없이 전달한다 |

`tokenType` 클레임으로 토큰 종류를 구분한다. Refresh Token을 Authorization 헤더에 넣어도 필터에서 차단된다.

### JWT 로그아웃 무효화

JWT는 stateless라 로그아웃해도 토큰이 살아있는 문제가 있다. Redis 없이 `lastLogoutDatetime` + Caffeine 캐시로 해결했다. 자세한 구현은 [JWT 로그아웃 무효화](/posts/jwt-invalidation-without-redis/)에서 다뤘다.

---

## 3. 서비스 계층 부재와 중복

알림이 대표적인 사례다. PHP에서는 서비스 계층이 없으니 SMS 발송 로직이 여러 컨트롤러에 중복되어 있었다. 문자를 보내야 하는 곳마다 SMS API 호출 코드를 직접 작성했다. 이건 알림만의 문제가 아니라, 서비스 계층이 없는 구조의 본질적인 문제다. 공통 로직을 모을 곳이 없으니 컨트롤러마다 복사해서 쓴다. 기술을 전환하려면(예: SMS → FCM) 중복된 모든 곳을 찾아서 고쳐야 한다.

Spring Boot에서는 서비스 계층이 이 문제를 해결한다. 알림이든, 검증이든, 상태 변경이든 로직이 한 곳에 있으니 수단을 바꿔도 호출하는 쪽은 영향이 없다.

---

## 4. PHP에 없던 것들 — 캐시와 Rate Limit

인프라 전환에서 가장 영향이 큰 것은 PHP에 아예 없던 것들이었다. 특히 **캐시**와 **Rate Limit**이다.

### Caffeine 캐시

PHP에서는 캐시가 없었다. 매 요청마다 DB를 조회한다. Spring Boot에서는 인증 정보를 Caffeine 로컬 캐시(5분 TTL, 최대 10만건)에 올려 매 요청 DB 조회를 제거했다. 설정값, 자주 조회되는 데이터도 캐시에 올렸다.

### Bucket4j Rate Limit

PHP에는 Rate Limit이 없었다. 로그인 시도를 무한히 할 수 있었다. Spring Boot에서는 Bucket4j로 엔드포인트별 요청 제한을 건다.

---

## 5. Hibernate Envers 감사 추적

PHP에서는 게시글 수정 이력을 수동으로 관리했다.

```php
// PHP — 수동 히스토리 저장
$this->Board_post_history_model->insert([
    'post_id' => $post_id,
    'post_title' => $old_title,
    'post_content' => $old_content,
    'modified_at' => date('Y-m-d H:i:s'),
    'modified_by' => $admin_id
]);
```

이걸 모든 수정 가능한 엔티티에 대해 일일이 해야 한다. 실수로 빠뜨리면 이력이 남지 않는다.

Spring Boot에서는 Hibernate Envers로 자동 감사 추적한다.

```java
@Entity
@Audited  // 이 한 줄이면 모든 변경 이력이 자동 기록된다
@Table(name = "post")
public class Post {
    // ... 필드 정의
}
```

`@Audited` 어노테이션을 붙이면 Envers가 자동으로 `post_AUD` 테이블을 만들고, INSERT/UPDATE/DELETE 시점에 변경 전 상태를 자동 기록한다.

| 항목 | PHP 수동 히스토리 | Hibernate Envers |
|------|-----------------|-----------------|
| 대상 추가 | 모델 + 컨트롤러 코드 추가 | `@Audited` 한 줄 |
| 누락 위험 | 높음 (개발자가 잊으면 끝) | 없음 (자동) |
| 조회 | 히스토리 모델 직접 쿼리 | AuditReader API |
| 롤백 | 수동 구현 | 리비전 번호로 복원 가능 |
| 별도 모델 | 엔티티마다 히스토리 모델 필요 | 불필요 (자동 생성) |

PHP에서는 히스토리가 필요한 엔티티마다 별도 모델(`Board_post_history_model`)을 만들어야 했다. Envers에서는 `@Audited` 하나면 된다.

---

## 6. 에러 핸들링

PHP의 에러 처리 패턴이다.

```php
// PHP — echo + exit 에러 처리
if (!$voucher) {
    echo json_encode([
        'status' => 'error',
        'message' => '이용권을 찾을 수 없습니다.'
    ]);
    exit;
}
```

에러 형식이 통일되지 않았다. 어떤 API는 `status: error`, 어떤 API는 `result: fail`, 어떤 API는 HTTP 상태 코드를 200으로 보내면서 바디에 에러를 담는다.

`@ControllerAdvice`로 전역 에러 핸들러를 두고, `DomainException`(비즈니스 규칙 위반)과 `ApiException`(인프라 수준 문제)을 분리했다. 모든 에러가 일관된 `ErrorResponse` 형식으로 반환된다.

---

## 7. 메뉴 기반 권한 시스템

PHP의 권한 관리는 하드코딩이었다.

```php
// PHP — 하드코딩 권한 체크
if ($this->member['mem_level'] < 9) {
    echo json_encode(['status' => 'error', 'message' => '권한 없음']);
    exit;
}
```

`mem_level`이라는 정수값으로 권한을 구분한다. 9 이상이면 관리자. 이 매직넘버가 여러 컨트롤러에 흩어져 있다.

핵심은 **운영 친화성**이다. 하드코딩된 매직넘버를 DB로 옮겨서, 관리자가 코드 배포 없이 권한을 관리할 수 있게 만드는 것.

Spring Boot에서는 `@PreAuthorize` 어노테이션 방식 대신 **메뉴 기반 URL 경로 매칭 + Interceptor** 방식을 택했다. 세 테이블(Menu, Role, RoleMenu)로 구성하고, `MenuAuthorizationInterceptor`가 매 요청을 자동으로 체크한다. 정확 매칭 우선, 없으면 와일드카드 탐색. **Default Deny** — 등록되지 않은 경로는 무조건 거부된다.

| 항목 | PHP 하드코딩 | 메뉴 기반 DB 관리 |
|------|-------------|------------------|
| 권한 변경 | 코드 수정 + 배포 | 관리자 페이지에서 즉시 반영 |
| 누락 위험 | 조건문 빠뜨리면 열림 | 등록 안 하면 닫힘 (Default Deny) |
| 역할 추가 | 코드 전체에 조건 추가 | DB에 Role + RoleMenu 행 추가 |
| 관리 주체 | 개발자만 가능 | 관리자가 직접 가능 |

1인 개발에서 "역할 A에 유전자 관리 권한을 추가한다"가 코드 배포 없이 관리자 페이지에서 가능하다는 건 큰 차이다.

---

## 8. 돌아보며

인프라는 마이그레이션의 보이지 않는 비용이다.

"PHP를 Spring Boot로 옮긴다"고 할 때 보통 떠올리는 건 컨트롤러와 모델의 전환이다. 하지만 실제 작업 시간의 상당 부분은 인프라에 들어갔다. JWT 인증 설계, Caffeine 캐시 전략, Envers 감사 추적, 에러 핸들링 통일, Spring Security RBAC 구성 — 이것들이 기능이 아닌 인프라이기 때문에 외부에서 보이지 않는다.

PHP에서 이 인프라가 없었던 게 아니다. 세션, exit 패턴, 수동 히스토리, 매직넘버 권한 — PHP 나름의 인프라가 있었다. 문제는 그 인프라가 **암묵적**이었다는 것이다. 인증 로직이 12개 컨트롤러 생성자에 흩어져 있고, 권한 체크가 매직넘버로 하드코딩되어 있으면, 그건 인프라가 아니라 그냥 코드다.

Spring Boot로 전환하면서 이 암묵적 인프라를 **명시적 인프라**로 바꿨다. JWT 필터, `@PreAuthorize`, `@ControllerAdvice`, `@Audited` — 한 곳에서 선언하고, 시스템 전체에 적용된다.

정리하면 세 가지였다.

> 1. **인증은 마이그레이션에서 가장 먼저 해결해야 하는 문제다.** 다른 모든 기능이 인증 위에서 동작한다. JWT 설계(3종 토큰, 갱신 전략, 무효화 방법)를 초기에 확정하지 않으면 후속 작업이 전부 흔들린다.
>
> 2. **캐시와 Rate Limit은 보안이다.** PHP에 캐시가 없다는 건 매 요청 DB 조회라는 뜻이고, Rate Limit이 없다는 건 무제한 로그인 시도가 가능하다는 뜻이다. 성능 최적화가 아니라 보안 조치다.
>
> 3. **인프라 전환 비용을 과소평가하지 말아야 한다.** 도메인 로직 마이그레이션만 생각하면 안 된다. 인증, 보안, 캐시, 감사 추적, 에러 핸들링 — 이 "보이지 않는" 작업이 전체 일정의 상당 부분을 차지한다. 기획서에는 "인증을 JWT로 바꾼다"라고 한 줄이지만, 실제 구현은 3종 토큰 설계 + 필터 체인 + 캐시 전략 + 로그아웃 무효화다.

---

**시리즈:**
- [Part 1](/posts/php-codeigniter-spring-boot-migration-why/) — 왜 다시 만들었나
- [Part 2](/posts/php-to-spring-boot-domain-redesign/) — 도메인 재설계
- [Part 3](/posts/admin-controller-migration-legacy-compatibility/) — 레거시 호환과 신규 API
- **Part 4 (이 글)** — 인증, 보안, 인프라
- [Part 5](/posts/migration-lessons-learned-retrospective/) — 교훈과 수치
