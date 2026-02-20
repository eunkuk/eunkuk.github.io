---
title: "96개 PHP 어드민 컨트롤러를 42개로 재구성하다 — 마이그레이션 회고 (3) 컨트롤러 전환과 신규 API"
date: 2026-02-19 22:00:00 +0900
categories: [Dev, Architecture]
tags: [spring-boot, rest-api, legacy, migration, admin, controller, swagger]
description: "PHP 96개 어드민 컨트롤러를 Spring Boot 42개로 재구성하면서, 28개 전환 컨트롤러와 14개 신규 REST API를 설계한 과정을 기록한다."
---

> PHP 96개 어드민 컨트롤러를 Spring Boot 42개로 재구성하면서, 28개 전환 컨트롤러와 14개 신규 REST API를 설계한 과정을 기록한다.

---

## 1. PHP 어드민 컨트롤러 구조

PHP 관리자 시스템은 96개 컨트롤러가 흩어져 있었다. CI Board 기반이라 게시판, 쇼핑몰, 포인트 등 실제로 쓰지 않는 기능의 컨트롤러도 상당수 포함되어 있었고, 같은 도메인이 여러 컨트롤러로 나뉘어 있는 구조였다. 예를 들어 회원 관련만 해도 Members, Membergroup, Dormant, Loginlog 등 5개 파일이다.

---

## 2. 전환 전략

관리자 웹은 Thymeleaf로 새로 만들었다. PHP 시절의 CI Board 기반 관리자와는 완전히 다른 시스템이고 접속 URL 자체가 다르기 때문에, 기존 PHP와의 호환을 신경 쓸 필요 없이 처음부터 깨끗하게 설계할 수 있었다.

다만 PHP 96개 컨트롤러의 기능을 새 구조로 옮기면서, 기존 응답 패턴을 Spring Boot 표준으로 정리하는 작업이 필요했다.

```php
// PHP의 Admin Ajax 응답 패턴
echo json_encode([
    'result' => 'success',
    'data' => [
        'list' => $members,
        'total_count' => $total,
        'page' => $page,
        'per_page' => $per_page
    ]
]);
```

Spring Boot의 표준 REST 응답과는 구조가 다르다. 페이지네이션 형식도, 에러 응답도, 필드명도 다르다.

### 전환 컨트롤러 28개

PHP 96개 Admin 컨트롤러 중 실제로 사용되는 기능만 추려서 28개 컨트롤러로 재구성했다. 같은 도메인에 흩어져 있던 컨트롤러를 하나로 통합하고, 더 이상 쓰지 않는 기능은 제거했다.

### 전환의 핵심 패턴

재구성 과정에서 몇 가지 공통 패턴이 있었다.

**1. 도메인 단위 통합**

PHP에서 Members, Membergroup, Dormant, Loginlog 등으로 흩어져 있던 컨트롤러를 도메인 단위로 하나로 합쳤다.

**2. 응답 구조 표준화**

PHP의 응답 형식을 Spring Boot 표준으로 정리했다.

| 항목 | PHP 응답 형식 | Spring Boot 표준 |
|------|-------------|-----------------|
| 성공 키 | `result: "success"` | HTTP 200 + JSON body |
| 목록 | `data.list` | `content` (Page 객체) |
| 총 개수 | `data.total_count` | `totalElements` |
| 페이지 번호 | `data.page` (1부터) | `number` (0부터) |
| 페이지 크기 | `data.per_page` | `size` |

내부 서비스 계층은 처음부터 Spring Boot 표준 DTO만 사용한다. PHP 시절의 응답 형식을 끌고 가지 않는다.

**3. 불필요한 기능 제거**

CI Board에서 기본 제공하던 게시판, 쇼핑몰, 포인트 등 실제로 사용하지 않는 기능의 컨트롤러를 제거했다.

---

## 3. 신규 어드민 컨트롤러

레거시 호환 컨트롤러 외에, 순수 REST API 컨트롤러를 새로 만들었다. 대부분은 PHP에 아예 없던 기능이다 — 역할/메뉴 권한 관리, 활동 로그, 설문, 푸시 관리 등. 마이그레이션이 단순 전환이 아니라 신규 개발을 동반한다는 점을 보여준다. 나머지는 PHP의 여러 컨트롤러에 흩어져 있던 기능을 하나로 모은 것이다.

---

## 4. 서비스 API 매핑

관리자 API뿐 아니라 서비스(사용자향) API도 재구성했다. 불필요해진 것을 제거하고, 새 비즈니스에 필요한 것을 추가했다. 마이그레이션은 단순히 코드를 옮기는 작업이 아니라, 불필요한 것을 버리고 필요한 것을 추가하는 기회다.

---

## 5. Swagger 문서화

PHP에는 API 문서가 없었다. 프론트엔드 개발자가 API 스펙을 알려면 PHP 코드를 읽거나 나에게 물어봐야 했다.

Spring Boot에서는 SpringDoc OpenAPI를 적용해 API 문서를 자동 생성했다. 컨트롤러에 `@Tag`, `@Operation`, `@Parameter` 어노테이션을 붙이면 Swagger UI에서 엔드포인트 목록, 요청/응답 스키마, 즉시 테스트가 가능하다.

| 항목 | PHP | Spring Boot |
|------|-----|-------------|
| API 문서 | 없음 | Swagger UI 자동 생성 |
| 엔드포인트 목록 | PHP 코드 직접 탐색 | `/swagger-ui.html`에서 확인 |
| 요청/응답 스키마 | 구두 전달 | DTO 기반 자동 스키마 |
| API 테스트 | Postman 수동 | Swagger UI에서 즉시 테스트 |
| API 변경 추적 | 없음 | DTO 변경 시 문서 자동 갱신 |

프론트엔드 개발자와의 소통이 눈에 띄게 개선됐다. "이 API 파라미터가 뭔가요?"라는 질문이 "Swagger 확인했는데 이 필드 의미가 뭔가요?"로 바뀌었다. 질문의 수준이 달라진다.

---

## 6. 전체 컨트롤러 매핑 요약

Admin 컨트롤러는 크게 줄었다. 중복되거나 사용되지 않는 기능을 제거한 결과다. 반면 Service API는 오히려 늘었다 — 새 비즈니스(플랜, 설문, 결제)를 추가했기 때문이다. 마이그레이션은 줄이는 작업이 아니라, 필요한 것과 불필요한 것을 판단하는 작업이다.

---

## 7. 돌아보며

96개를 42개로 줄인 건 리팩토링이 아니라 판단의 결과다.

관리자 웹을 Thymeleaf로 새로 만들었기 때문에 기존 PHP와의 호환을 고려할 필요가 없었다. 접속 URL 자체가 다른 완전히 새로운 시스템이다. 덕분에 처음부터 Spring Boot 표준 구조로 설계할 수 있었다.

정리하면 세 가지였다.

> 1. **컨트롤러 수가 줄어든 건 실제로 사용되는 기능만 남긴 것이다.** PHP 시절 상당수의 컨트롤러는 CI Board가 기본 제공하는 것일 뿐 실제 비즈니스에서 쓰이지 않았다. 마이그레이션은 불필요한 것을 버리는 기회이기도 하다.
>
> 2. **새로 만드는 시스템이라 처음부터 깨끗하게 갈 수 있었다.** 기존 PHP와 접속 URL 자체가 다르기 때문에, 호환을 고려할 필요 없이 도메인 단위 통합과 응답 구조 표준화를 바로 적용했다.
>
> 3. **API 문서화의 가치는 과소평가된다.** PHP에서 Spring Boot로 전환하면서 기술적으로 가장 즉각적인 효과를 체감한 건 Swagger였다. 코드가 곧 문서가 되니 FE 개발자와의 소통 비용이 크게 줄었다.

그리고 어드민에서 유전자 데이터 업로드 기능을 만들면서 한 가지 더 생각하게 된 게 있다. 업로드를 받는 쪽은 새로 만들었는데, 업로드하는 쪽 — 연구원이 분석 파이프라인을 돌리고, 결과 파일을 직접 올리는 과정 — 은 여전히 수동이었다. 어드민만 잘 만드는 게 아니라 그 앞단의 분석 과정까지 자동화하면 완전한 워크플로우가 되겠다는 생각이 이때 처음 들었다. 이 생각은 나중에 [GeneTitan Agent](/posts/genetitan-agent-retrospective/)로 이어진다.

---

**시리즈:**
- [Part 1](/posts/php-codeigniter-spring-boot-migration-why/) — 왜 다시 만들었나
- [Part 2](/posts/php-to-spring-boot-domain-redesign/) — 도메인 재설계
- **Part 3 (이 글)** — 레거시 호환과 신규 API
- [Part 4](/posts/spring-boot-infrastructure-auth-security/) — 인증, 보안, 인프라
- [Part 5](/posts/migration-lessons-learned-retrospective/) — 교훈과 수치
