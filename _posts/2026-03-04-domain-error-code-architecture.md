---
title: "DomainException에서 ApiResponse까지 — 에러 코드 아키텍처를 설계한 과정"
date: 2026-03-04 22:00:00 +0900
categories: [Dev, Architecture]
tags: [spring-boot, error-handling, exception, ddd, aop, clean-architecture, retrospective]
description: "도메인 계층은 HTTP를 모르고, API 계층은 비즈니스 규칙을 모른다. 두 세계를 잇는 에러 코드 아키텍처를 설계한 과정을 기록한다."
---

> 도메인 계층은 HTTP를 모르고, API 계층은 비즈니스 규칙을 모른다. 두 세계를 잇는 에러 코드 아키텍처를 설계한 과정을 기록한다.

---

## 1. 상황

> 이 글은 [DomainService vs Facade 경계](/posts/domain-service-vs-facade-boundary/)의 연장이다. DomainService가 순수 비즈니스 로직을 담당한다고 했는데, 그 DomainService에서 던지는 예외는 어떻게 처리되는가? 도메인 계층이 HTTP를 모르면, 에러를 어떻게 API 응답으로 바꾸는가? 그 과정을 기록한다.

초기에는 에러 처리가 단순했다. 아니, 단순한 게 아니라 체계가 없었다.

```java
// 초기: 구분 불가능한 에러
if (voucher == null) {
    throw new RuntimeException("이용권을 찾을 수 없습니다.");
}

if (voucher.isExpired()) {
    throw new RuntimeException("이용권이 만료되었습니다.");
}
```

`RuntimeException`에 한국어 메시지를 넣고 있었다. 클라이언트는 에러 종류를 구분할 수 없었다. "이용권을 찾을 수 없습니다"와 "이용권이 만료되었습니다"가 모두 HTTP 500으로 내려갔다. 프론트엔드에서 에러별로 다른 UI를 보여주고 싶어도 방법이 없었다.

두 가지를 해결해야 했다. 에러를 **코드로 구분**할 수 있어야 하고, 도메인 계층이 **HTTP를 몰라도** 동작해야 한다.

---

## 2. 왜 에러 코드가 두 벌인가

에러 코드를 하나만 만들면 안 되는가? 처음에는 하나로 만들려고 했다. 에러 코드에 HTTP 상태와 메시지를 모두 담으면 편하지 않을까?

```java
// 이렇게 하나로 합치면 편하지 않을까?
public enum ErrorCode {
    VOUCHER_EXPIRED("6102", "이용권이 만료되었습니다.", HttpStatus.BAD_REQUEST),
    NOT_FOUND_VOUCHER("3010", "이용권을 찾을 수 없습니다.", HttpStatus.NOT_FOUND);
}
```

편하다. 하지만 문제가 있다. 이 `ErrorCode`는 `HttpStatus`를 가지고 있다. `HttpStatus`는 Spring Web의 클래스다. 이걸 도메인 계층에서 사용하려면 도메인 모듈이 Spring Web에 의존해야 한다.

프로젝트의 모듈 구조는 이렇다.

```
app-domain   → 엔티티, DomainService, Repository (순수 비즈니스)
app-infra    → 보안, Swagger, 에러 응답, 캐시 (인프라)
app-admin    → 관리자 API (Controller)
```

Gradle이 모듈 간 의존 방향을 강제한다. `app-domain`은 `app-infra`에 의존하지 않는다. 도메인 모듈에서 `HttpStatus`를 사용할 수 없다. 그래서 에러 코드가 두 벌이 되었다.

### DomainErrorCode — 도메인 계층

```java
// app-domain 모듈
public enum DomainErrorCode {

    // ── Voucher ──
    NOT_FOUND_VOUCHER("이용권을 찾을 수 없습니다."),
    VOUCHER_EXPIRED("이용권이 만료되었습니다."),
    INVALID_VOUCHER_TRANSITION("유효하지 않은 이용권 상태 전환입니다."),
    VOUCHER_ALREADY_ASSIGNED("이미 할당된 이용권입니다."),

    // ── Member ──
    NOT_FOUND_MEMBER("회원을 찾을 수 없습니다."),
    MEMBER_INACTIVE("비활성화된 회원입니다."),
    PASSWORD_MISMATCH("비밀번호가 일치하지 않습니다."),
    DUPLICATE_USER_EMAIL("이미 사용 중인 이메일입니다."),

    // ── Post ──
    NOT_FOUND_POST("게시글을 찾을 수 없습니다."),
    POST_ALREADY_DELETED("이미 삭제된 게시글입니다.");

    private final String message;
}
```

HTTP를 전혀 모른다. 숫자 코드도 없다. enum 이름과 한국어 메시지만 있다. 도메인 계층에서 이 코드를 던지면 "비즈니스적으로 무엇이 잘못되었는가"를 표현한다.

### ErrorCode — API 계층

```java
// app-infra 모듈
public enum ErrorCode {

    // ── 1000: 요청 오류 ──
    INVALID_REQUEST("1000", "잘못된 요청입니다.", HttpStatus.BAD_REQUEST),
    VALIDATION_ERROR("1001", "입력값 검증에 실패했습니다.", HttpStatus.BAD_REQUEST),
    TYPE_MISMATCH("1002", "파라미터 타입이 올바르지 않습니다.", HttpStatus.BAD_REQUEST),

    // ── 2000: 인증/인가 ──
    UNAUTHORIZED("2000", "인증이 필요합니다.", HttpStatus.UNAUTHORIZED),
    FORBIDDEN("2001", "접근 권한이 없습니다.", HttpStatus.FORBIDDEN),
    LOGIN_FAILED("2010", "로그인 정보를 확인해주세요.", HttpStatus.UNAUTHORIZED),
    ACCOUNT_DISABLED("2011", "비활성화된 계정입니다.", HttpStatus.FORBIDDEN),

    // ── 3000: Not Found ──
    NOT_FOUND_USER("3001", "회원을 찾을 수 없습니다.", HttpStatus.NOT_FOUND),
    NOT_FOUND_VOUCHER("3010", "이용권을 찾을 수 없습니다.", HttpStatus.NOT_FOUND),
    NOT_FOUND_POST("3020", "게시글을 찾을 수 없습니다.", HttpStatus.NOT_FOUND),

    // ── 4000: 서버 내부 오류 ──
    INTERNAL_ERROR("4000", "서버 내부 오류입니다.", HttpStatus.INTERNAL_SERVER_ERROR),

    // ── 6000: 비즈니스 규칙 위반 ──
    INVALID_VOUCHER_STATE("6100", "이용권 상태가 유효하지 않습니다.", HttpStatus.BAD_REQUEST),
    VOUCHER_EXPIRED("6102", "이미 만료된 이용권입니다.", HttpStatus.BAD_REQUEST),
    VOUCHER_ALREADY_ASSIGNED("6105", "이미 할당된 이용권입니다.", HttpStatus.CONFLICT),
    PASSWORD_MISMATCH_ERROR("6132", "비밀번호가 일치하지 않습니다.", HttpStatus.BAD_REQUEST),

    // ── 9000: 알 수 없는 오류 ──
    UNKNOWN_ERROR("9000", "알 수 없는 오류입니다.", HttpStatus.INTERNAL_SERVER_ERROR);

    private final String code;
    private final String message;
    private final HttpStatus httpStatus;
}
```

숫자 코드, 메시지, HTTP 상태 코드를 모두 가지고 있다. API 응답에 이 정보가 포함된다.

---

## 3. 번호 체계

에러 코드에 의미를 담았다. 번호만 보고 에러의 성격을 파악할 수 있다.

| 범위 | 성격 | 예시 |
|------|------|------|
| 1000~1999 | 요청 오류 (Validation) | 잘못된 입력, 타입 불일치, 누락된 파라미터 |
| 2000~2999 | 인증/인가 | 미인증, 권한 없음, 로그인 실패, 토큰 만료 |
| 3000~3999 | 리소스 없음 | 회원·이용권·게시글 Not Found |
| 4000~4999 | 서버 내부 오류 | 데이터 처리 실패, 파일 오류 |
| 5000~5999 | 외부 API 오류 | 외부 서비스 통신 실패, 타임아웃 |
| 6000~6999 | 비즈니스 규칙 위반 | 만료, 상태 전이 불가, 중복 |
| 9000 | 알 수 없는 오류 | catch-all |

6000번대는 도메인별로 다시 세분화했다.

| 범위 | 도메인 |
|------|--------|
| 6100~6109 | 주문 (Order) |
| 6110~6119 | 결제 (Payment) |
| 6130~6139 | 회원 (Member) |
| … | 도메인이 늘어나면 범위를 추가 |

로그에서 `6102`가 보이면 "주문 도메인의 비즈니스 규칙 위반"이라는 걸 바로 알 수 있다. `3001`이면 "회원 Not Found"다. 번호 체계가 에러의 성격을 알려준다.

---

## 4. 변환 흐름

DomainException이 던져지면 어떻게 API 응답이 되는가? 5단계를 거친다.

```
Entity / DomainService
    ↓ throw DomainException(DomainErrorCode.VOUCHER_EXPIRED)
DomainExceptionTranslator (AOP @Around)
    ↓ DomainErrorCodeMapper.toErrorCode()
throw ApiException(ErrorCode.VOUCHER_EXPIRED)
    ↓
GlobalExceptionHandler (@RestControllerAdvice)
    ↓
ApiResponse { "code": "6102", "message": "이미 만료된 이용권입니다." }
```

각 단계를 코드로 보면 이렇다.

### Step 1: 도메인에서 예외를 던진다

```java
// Entity 또는 DomainService
public void markAsUsed(Long memberId) {
    if (!canBeUsed()) {
        throw new DomainException(DomainErrorCode.INVALID_VOUCHER_TRANSITION);
    }
    if (isExpired()) {
        throw new DomainException(DomainErrorCode.VOUCHER_EXPIRED);
    }
    this.state = VoucherState.USED.getCode();
    this.memberId = memberId;
}
```

DomainException은 DomainErrorCode만 가지고 있다. HTTP 상태 코드가 뭔지 모른다. "이용권이 만료되었다"는 비즈니스 사실만 표현한다.

### Step 2: AOP가 가로챈다

```java
@Aspect
@Component
@RequiredArgsConstructor
public class DomainExceptionTranslator {

    @Around("execution(* com.example.myapp..*Controller.*(..))")
    public Object translate(ProceedingJoinPoint joinPoint) throws Throwable {
        try {
            return joinPoint.proceed();
        } catch (DomainException ex) {
            log.warn("[DomainException] code={}, message='{}'",
                ex.getErrorCode(), ex.getMessage());
            ErrorCode errorCode = DomainErrorCodeMapper.toErrorCode(ex.getErrorCode());
            throw new ApiException(errorCode, ex.getMessage());
        }
    }
}
```

모든 Controller 메서드를 `@Around`로 감싸고 있다. DomainException이 발생하면 자동으로 ApiException으로 변환한다. Controller에 try-catch를 쓸 필요가 없다. DomainService도 try-catch 없이 깔끔하게 예외를 던지면 된다.

### Step 3: 매핑 테이블

```java
public class DomainErrorCodeMapper {

    private static final Map<DomainErrorCode, ErrorCode> MAPPING = Map.ofEntries(
        // 정확한 1:1 매핑
        Map.entry(DomainErrorCode.NOT_FOUND_VOUCHER, ErrorCode.NOT_FOUND_VOUCHER),
        Map.entry(DomainErrorCode.VOUCHER_EXPIRED, ErrorCode.VOUCHER_EXPIRED),
        Map.entry(DomainErrorCode.NOT_FOUND_MEMBER, ErrorCode.NOT_FOUND_USER),

        // 보안을 위한 일반화 매핑
        Map.entry(DomainErrorCode.PASSWORD_MISMATCH, ErrorCode.LOGIN_FAILED),
        Map.entry(DomainErrorCode.MEMBER_INACTIVE, ErrorCode.ACCOUNT_DISABLED),

        // 내부 오류로 매핑
        Map.entry(DomainErrorCode.PASSWORD_ENCODER_REQUIRED, ErrorCode.INTERNAL_ERROR)
    );

    public static ErrorCode toErrorCode(DomainErrorCode domainErrorCode) {
        return MAPPING.getOrDefault(domainErrorCode, ErrorCode.UNKNOWN_ERROR);
    }
}
```

정적 Map으로 컴파일 시점에 만들어진다. 런타임 비용이 없다. 매핑이 없는 DomainErrorCode는 `UNKNOWN_ERROR`로 빠진다.

switch문으로도 같은 일을 할 수 있다.

```java
public class DomainErrorCodeMapper {

    public static ErrorCode toErrorCode(DomainErrorCode domainErrorCode) {
        return switch (domainErrorCode) {
            case NOT_FOUND_ORDER -> ErrorCode.NOT_FOUND_ORDER;
            case ORDER_EXPIRED -> ErrorCode.ORDER_EXPIRED;
            case NOT_FOUND_MEMBER -> ErrorCode.NOT_FOUND_USER;
            case PASSWORD_MISMATCH -> ErrorCode.LOGIN_FAILED;
            case MEMBER_INACTIVE -> ErrorCode.ACCOUNT_DISABLED;
            default -> ErrorCode.UNKNOWN_ERROR;
        };
    }
}
```

두 방식의 차이는 크지 않다.

| 기준 | Map | switch |
|------|-----|--------|
| 매핑 누락 감지 | 런타임 (`getOrDefault`) | 컴파일 타임 (sealed + exhaustive switch) |
| 가독성 | 선언적, 한 눈에 전체 매핑 파악 | 절차적, 분기가 많으면 길어짐 |
| 확장성 | entry 한 줄 추가 | case 한 줄 추가 |

우리는 Map 방식을 선택했다. DomainErrorCode가 100개를 넘으면서 switch보다 선언적인 Map이 읽기 편했고, `Map.ofEntries()`의 불변 보장이 안전했다. 에러 코드가 적은 프로젝트라면 switch가 더 간결할 수 있다.

### Step 4-5: GlobalExceptionHandler가 응답을 만든다

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ApiException.class)
    public ResponseEntity<ApiResponse<Object>> handleApiException(
            ApiException ex, HttpServletRequest request) {

        ErrorCode errorCode = ex.getErrorCode();

        if (isClientError(errorCode)) {
            log.warn("[ApiException] code={}, message='{}', requestId={}",
                errorCode.getCode(), ex.getMessage(), MDC.get("requestId"));
        } else {
            log.error("[ApiException] code={}, message='{}', requestId={}",
                errorCode.getCode(), ex.getMessage(), MDC.get("requestId"));
        }

        if (shouldNotifyError(errorCode)) {
            sendAlert(errorCode, ex.getMessage(), request);
        }

        return ResponseEntity
            .status(errorCode.getHttpStatus())
            .body(ApiResponse.fail(errorCode.getCode(), ex.getMessage()));
    }

    private boolean isClientError(ErrorCode errorCode) {
        String code = errorCode.getCode();
        return code.startsWith("1") || code.startsWith("2");
    }

    private boolean shouldNotifyError(ErrorCode errorCode) {
        String code = errorCode.getCode();
        return !code.startsWith("1") && !code.startsWith("2");
    }
}
```

번호 체계를 활용해서 로깅 레벨과 알림 여부를 결정한다. 1000~2999번대(클라이언트 실수)는 WARN으로 로깅하고 알림을 보내지 않는다. 3000번대 이상(서버가 대응해야 하는 에러)은 ERROR로 로깅하고 알림을 보낸다.

최종 응답은 이렇다.

```json
{
    "code": "6102",
    "message": "이미 만료된 이용권입니다.",
    "data": null,
    "requestId": "req-abc123"
}
```

---

## 5. 설계에서 고민한 것들

### 보안: 구체적 이유를 노출하지 않는다

비밀번호가 틀렸을 때 "비밀번호가 틀렸습니다"라고 응답하면, 공격자가 "이 아이디는 존재하는구나"를 알 수 있다. 도메인에서는 `PASSWORD_MISMATCH`로 정확한 원인을 던지지만, API 응답에서는 `LOGIN_FAILED` — "로그인 정보를 확인해주세요"로 일반화한다.

```java
// 도메인: 정확한 원인
Map.entry(DomainErrorCode.PASSWORD_MISMATCH, ErrorCode.LOGIN_FAILED),
Map.entry(DomainErrorCode.MEMBER_INACTIVE, ErrorCode.ACCOUNT_DISABLED),
```

내부 로그에는 `PASSWORD_MISMATCH`가 기록되니 디버깅에 문제없다. 외부 응답만 일반화한다.

### 매핑 누락 대응

새로운 DomainErrorCode를 추가하고 매핑을 빠뜨리면 어떻게 되는가?

```java
public static ErrorCode toErrorCode(DomainErrorCode domainErrorCode) {
    return MAPPING.getOrDefault(domainErrorCode, ErrorCode.UNKNOWN_ERROR);
}
```

`getOrDefault`가 `UNKNOWN_ERROR`를 반환한다. 에러 코드 `9000`이다. 3000번 이상이니 알림이 발생하고, 로그에 기록된다. 매핑 누락을 빠르게 발견할 수 있다.

실제로는 코드 리뷰에서 매핑 누락을 잡는 게 가장 효과적이다. PR 체크리스트에 "새 에러 코드는 DomainErrorCode → ErrorCode → DomainErrorCodeMapper에 반영"을 포함시켰다.

### 왜 AOP인가

DomainExceptionTranslator가 없으면 어떻게 되는가? Controller마다 try-catch를 써야 한다.

```java
// AOP 없이: 모든 Controller에 try-catch
@PostMapping("/vouchers/{id}/use")
public ApiResponse<Void> useVoucher(@PathVariable Long id) {
    try {
        voucherFacade.useVoucher(id);
        return ApiResponse.ok();
    } catch (DomainException ex) {
        ErrorCode errorCode = DomainErrorCodeMapper.toErrorCode(ex.getErrorCode());
        throw new ApiException(errorCode, ex.getMessage());
    }
}
```

Controller가 10개, 메서드가 50개면 try-catch가 50번 반복된다. AOP로 한 번만 정의하면 모든 Controller에 자동 적용된다. DomainService는 try-catch 없이 예외를 던지고, Controller는 try-catch 없이 Facade를 호출한다.

---

## 6. 돌아보며

에러 코드 아키텍처는 처음에 "왜 이렇게 복잡하게 해야 하지?"라고 생각했다. DomainErrorCode, ErrorCode, Mapper, AOP Translator, GlobalExceptionHandler — 다섯 개의 구성 요소가 연결되어 있다. 하지만 각 구성 요소가 하나의 책임을 맡고 있고, 계층 간 독립성을 유지하기 위해 필요한 구조다.

> **1. 에러 코드 분리는 계층 독립성을 지키기 위한 필수 비용이다.** DomainErrorCode와 ErrorCode를 하나로 합치면 도메인 모듈이 Spring Web에 의존해야 한다. 도메인의 순수성을 지키려면 에러 코드도 분리해야 한다. 번거롭지만 필수적인 비용이다.

> **2. 번호 체계에 의미를 담으면 로그만 보고도 에러의 성격을 파악할 수 있다.** `6102`를 보면 "6000번대 = 비즈니스 규칙 위반, 61xx = 이용권 도메인"이라는 걸 바로 안다. 번호가 무작위면 매번 코드를 찾아봐야 한다. 로깅 레벨과 알림 여부도 번호 범위로 자동 결정된다.

> **3. AOP 변환으로 도메인 코드의 순수성을 지킬 수 있다.** DomainService에 try-catch가 없다. Controller에도 try-catch가 없다. 예외 변환은 AOP가 자동으로 해준다. 비즈니스 로직을 작성하는 코드와 에러를 변환하는 코드가 분리되어 있어서, 각각을 독립적으로 수정할 수 있다.

---

**시리즈 연결:**
- [DomainService vs Facade — 경계를 정한 과정](/posts/domain-service-vs-facade-boundary/) — 로직의 **경계**
- [쿼리 복잡도에 따른 Repository 전략](/posts/repository-query-tier-selection/) — Repository **쿼리**
- 이 글 — DomainException에서 ApiResponse까지 **에러 흐름**
