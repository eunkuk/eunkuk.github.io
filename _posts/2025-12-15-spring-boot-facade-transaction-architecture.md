---
title: "Spring Boot DDD — Facade는 왜 Domain이 아닌 Application 레이어에 있는가"
date: 2025-12-15 22:00:00 +0900
categories: [Dev, Architecture]
tags: [spring-boot, ddd, facade, transaction, clean-architecture, layered-architecture, retrospective]
description: "6모듈 모노레포에서 Facade 패턴의 위치, 트랜잭션 경계, Entity 생성 패턴에 대한 아키텍처 결정 과정을 기록한다. Claude V2 vs Gemini V3 논쟁부터 최종 V6 결정까지."
---

> 6모듈 모노레포에서 Facade 패턴의 위치, 트랜잭션 경계, Entity 생성 패턴에 대한 아키텍처 결정 과정을 기록한다. Claude V2 vs Gemini V3 논쟁부터 최종 V6 결정까지.

---

## 1. 상황

> 이 글은 [모노레포 + 6모듈 구조를 선택한 경위](/posts/monorepo-multi-module-retrospective/)의 연장이다. 이전 글에서 **왜** 이 구조를 택했는지를 다뤘다면, 여기서는 그 구조 위에서 Facade를 어디에 두고, 트랜잭션 경계를 어떻게 긋고, 엔티티를 어떤 규칙으로 만들 것인지를 기록한다.

서비스를 모노레포 + 6모듈 구조로 운영하고 있다. 개발자는 나 하나다.

6개 모듈의 역할은 이렇다.

| 모듈 | 타입 | 역할 |
|------|------|------|
| `admin-api` | Application | 관리자 API (Thymeleaf, 엑셀, 데이터 관리) |
| `service-api` | Application | 사용자 API (순수 REST) |
| `app-common` | Library | Facade, 공통 DTO (유스케이스 조율) |
| `domain` | Library | 엔티티, Repository, DomainService (핵심 도메인) |
| `infra` | Library | 보안, JWT, 캐시, 메일/PDF, FCM, 에러 핸들링 |
| `common` | Library | 유틸리티, 예외 기본 클래스 (Spring 의존 없음) |

의존성 그래프는 아래와 같다.

```
admin-api / service-api     (Application Layer)
        |  depends on
        v
app-common                       (Shared Application Layer - Facades)
        |  depends on
        |----------+----------+
        v          v          v
domain  infra  common
```

핵심 제약이 하나 있다. **`domain`은 `infra`에 의존하지 않는다.** 도메인 코어가 인프라를 모르게 만드는 것이 헥사고날 아키텍처의 기본 원칙이고, Gradle 모듈 경계가 이를 컴파일 타임에 강제한다. `domain`에서 `infra`의 클래스를 import하면 빌드가 깨진다.

Admin과 Service는 같은 도메인 로직을 공유해야 한다. 주문 생성, 결제 처리, 이력 조회 — 이 흐름은 양쪽에서 모두 필요하다. 문제는 이 공유 로직을 어디에 둘 것인가였다.

---

## 2. Facade 패턴 3가지 유형

프로젝트에서 Facade는 세 가지 유형으로 분류된다. 각각 역할과 트랜잭션 특성이 다르다.

### 유형 1: Pure Read (QueryFacade)

통계, 검색, 목록 조회처럼 부수 효과(side effect)가 없는 읽기 전용 Facade다.

```java
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

`@Transactional(readOnly = true)`를 클래스 레벨에 선언한다. 여러 DomainService를 조합하지만 데이터를 변경하지 않는다.

> 이 패턴이 CQRS와 어떤 관계인지는 [풀 CQRS 없이 Command/Query를 분리한 방법](/posts/lightweight-cqrs-facade-separation/)에서 자세히 다룬다.

### 유형 2: CRUD + Workflow

생성·수정과 함께 캐시 evict, 보안 스냅샷 갱신 같은 부수 효과가 따라오는 Facade다.

```java
@Service
@RequiredArgsConstructor
public class MemberFacade {

    private final MemberDomainService memberDomainService;
    private final MemberAuthSnapshotService memberAuthSnapshotService; // infra (캐시)
    private final PasswordEncoder passwordEncoder;                     // infra (보안)

    public Member createMember(MemberCreateRequest request) {
        String encodedPassword = passwordEncoder.encode(request.getPassword());
        Member member = memberDomainService.createAndSave(request, encodedPassword);
        memberAuthSnapshotService.refreshSnapshot(member.getId());
        return member;
    }

    public void updateMember(Long memberId, MemberUpdateRequest request) {
        memberDomainService.update(memberId, request);
        memberAuthSnapshotService.evictSnapshot(memberId);
    }
}
```

`MemberAuthSnapshotService`(Redis 캐시)와 `PasswordEncoder`(Spring Security)는 둘 다 `infra`에 속한다. 이 Facade가 도메인 모듈에 있을 수 없는 이유가 바로 이 의존성이다.

### 유형 3: Cross-Domain Aggregation

여러 도메인의 데이터를 조합해서 하나의 응답을 만드는 Facade다.

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class OrderDetailFacade {

    private final OrderDomainService orderDomainService;
    private final PaymentDomainService paymentDomainService;
    private final ProductDomainService productDomainService;
    private final ShippingDomainService shippingDomainService;

    public OrderDetailResponse getOrderDetail(Long orderId) {
        Order order = orderDomainService.getById(orderId);
        Payment payment = paymentDomainService.getByOrderId(orderId);
        List<Product> products = productDomainService.getByOrderId(orderId);
        Shipping shipping = shippingDomainService.getByOrderId(orderId);
        return OrderDetailResponse.of(order, payment, products, shipping);
    }
}
```

주문 → 결제 → 상품 → 배송. 네 개의 DomainService를 조합해서 하나의 응답을 만든다. 각 DomainService는 자기 Aggregate만 안다. 이 탐색 경로를 아는 건 Facade뿐이다.

### Facade 생성 판단 플로차트

새로운 비즈니스 로직이 생겼을 때 Facade를 만들어야 하는지 판단하는 기준이다.

```
[Q1] Admin과 Service 양쪽에서 사용?
    YES --> Facade (app-common)
    NO  --> [Q2] 여러 DomainService 조합?
        YES --> [Q3] Side Effect (캐시, 보안, 외부 연동)?
            YES --> Facade (app-common)
            NO  --> Application Service에서 조합
        NO  --> DomainService 직접 사용
```

Q1에서 YES이면 무조건 Facade다. Q2에서 YES이고 Side Effect가 있으면 Facade다. Side Effect가 없으면 Application Service에서 직접 조합해도 된다.

---

## 3. 왜 Facade를 app-common에 뒀는가

"Facade는 여러 DomainService를 조합하니까 도메인 계층에 두는 게 자연스럽지 않나?"

처음에는 그렇게 생각했다. Facade가 DomainService만 의존한다면 `domain`에 둘 수 있었을 것이다. 하지만 실제 코드를 보면 Facade는 인프라에 의존한다.

### 인프라 의존의 실제 증거

```java
MemberFacade
  +-- MemberDomainService         // domain
  +-- MemberAuthSnapshotService   // infra (Redis 캐시)
  +-- PasswordEncoder             // infra (Spring Security)

PostFileFacade
  +-- PostDomainService           // domain
  +-- FileStorageService          // infra (S3/로컬 스토리지)

AuthFacade
  +-- MemberDomainService         // domain
  +-- JwtTokenProvider            // infra (JWT)
  +-- RefreshTokenService         // infra (Redis)
```

세 Facade 모두 인프라 컴포넌트를 주입받는다. 비밀번호 인코딩, 파일 저장, JWT 발급 — 이것들은 비즈니스 규칙이 아니라 애플리케이션 서비스 수준의 조율이다.

### 순환 의존성 문제

만약 Facade를 `domain`에 넣으면 어떻게 되는가.

```
domain
  +-- MemberFacade
  |     +-- PasswordEncoder (infra에 있음)
  |
  --> domain이 infra에 의존해야 함

infra
  +-- SecurityConfig
  |     +-- MemberDomainService (domain에 있음)
  |
  --> infra가 domain에 의존해야 함

결과: domain <--> infra 순환 의존
      Gradle 빌드 실패
```

`domain → infra` 의존성이 생기는 순간, 핵심 제약이 깨진다. 도메인이 인프라를 모른다는 원칙이 무너지고, Gradle이 순환 의존으로 빌드를 거부한다.

### Clean Architecture 관점

Clean Architecture의 동심원에 대입하면 각 계층의 위치가 명확해진다.

```
+-------------------------------------------------------+
|  Frameworks & Drivers                                  |
|  (admin-api, service-api)                  |
|                                                        |
|  +---------------------------------------------------+ |
|  |  Interface Adapters                                | |
|  |  (app-common -- Facades, DTOs)              | |
|  |                                                    | |
|  |  +-----------------------------------------------+ | |
|  |  |  Application Business Rules                   | | |
|  |  |  (infra -- 캐시, 보안, 외부 연동)       | | |
|  |  |                                               | | |
|  |  |  +-------------------------------------------+| | |
|  |  |  |  Enterprise Business Rules                || | |
|  |  |  |  (domain -- Entity, DomainService) || | |
|  |  |  +-------------------------------------------+| | |
|  |  +-----------------------------------------------+ | |
|  +---------------------------------------------------+ |
+-------------------------------------------------------+
```

DomainService는 Enterprise Business Rules다. "주문의 상태가 CREATED일 때만 결제할 수 있다"는 규칙. Facade는 Interface Adapters다. "주문을 생성하고, 캐시를 갱신하고, 알림을 보낸다"는 유스케이스 조율. 이 둘은 다른 계층에 속한다.

> Facade를 domain에 넣고 싶은 유혹은 "Facade가 DomainService를 조합하니까 도메인의 일부"라는 생각에서 온다. 하지만 Facade의 본질은 조합이 아니라 조율이다. 조합은 비즈니스 규칙이고, 조율은 애플리케이션 로직이다.

---

## 4. 트랜잭션 전략: Application Service가 유일한 경계

트랜잭션을 어디서 시작할 것인가. 이 질문에 세 가지 선택지가 있었다.

### 선택지 비교

| 위치 | 장점 | 단점 | 판정 |
|------|------|------|------|
| Application Service | 가이드 준수, 테스트 용이, 경계 명확 | 위임 boilerplate | 채택 |
| Facade | 간결, 위임 레이어 제거 | 3개 가이드 문서 위반, 통합 테스트 필수 | 거부 |
| Controller | Service 레이어 생략 가능 | WebMvcTest 불가, OSIV 의존, 계층 오염 | 절대 거부 |

### 왜 Application Service인가

Application Service는 `admin-api`나 `service-api`에 있는 `@Service` 클래스다. Controller와 Facade 사이에 위치한다.

```java
// admin-api의 Application Service
@Service
@RequiredArgsConstructor
public class PostAdminService {

    private final PostFileFacade postFileFacade;

    @Transactional
    public PostResponse createPost(PostCreateRequest request, MultipartFile thumbnail) {
        return postFileFacade.createPostMultipart(request, thumbnail);
    }

    @Transactional
    public void updatePost(Long postId, PostUpdateRequest request) {
        postFileFacade.updatePost(postId, request);
    }

    @Transactional(readOnly = true)
    public PostDetailResponse getPostDetail(Long postId) {
        return postFileFacade.getPostDetail(postId);
    }
}
```

`@Transactional`은 이 Application Service에만 선언한다. Facade에는 `@Transactional`을 붙이지 않는다.

### 트랜잭션 전파 흐름

```
Controller: createPost()
  |
  v
Application Service: createPost() @Transactional  <-- TX 경계 시작
  +-- TX1 시작
  |   |
  |   v
  |   Facade: createPostMultipart()  --> TX1 참여 (REQUIRED 전파)
  |     |
  |     +-- DomainService: createPostAndSave()  --> TX1 참여
  |     |
  |     +-- DomainService: updatePostTags()     --> TX1 참여
  |     |
  |     +-- FileStorageService: upload()        --> TX1 참여
  |
  +-- TX1 커밋 (전체 원자적)
```

Spring의 `@Transactional`은 기본 전파 속성이 `REQUIRED`다. 이미 트랜잭션이 존재하면 그 트랜잭션에 참여한다. Facade에 별도의 `@Transactional`을 선언하지 않아도 Application Service의 트랜잭션에 자동 참여한다. 이것이 핵심이다.

### 왜 Facade에 @Transactional을 붙이면 안 되는가

```java
// 잘못된 예: Facade에 @Transactional 선언
@Service
@Transactional  // <-- 이러면 안 된다
public class MemberFacade {

    public Member createMember(MemberCreateRequest request) {
        // ...
    }
}
```

Facade에 `@Transactional`을 붙이면 세 가지 문제가 생긴다.

1. **테스트가 무거워진다.** Facade를 단위 테스트하려면 트랜잭션 매니저, DataSource가 전부 필요하다. Application Service에 `@Transactional`이 있으면 Facade는 순수한 자바 객체로 테스트할 수 있다.
2. **트랜잭션 경계가 중복된다.** Application Service에도 있고 Facade에도 있으면, 어디가 진짜 경계인지 모호해진다.
3. **아키텍처 가이드를 위반한다.** "트랜잭션은 Application Service에서만 선언한다"고 명시했다. 1인 개발에서 일관성이 무너지면 가이드의 존재 의의가 사라진다.

### 왜 Controller에서 @Transactional은 절대 거부인가

```java
// 절대 하면 안 되는 예
@RestController
@Transactional  // <-- 절대 거부
public class PostController {

    @PostMapping("/posts")
    public ResponseEntity<PostResponse> createPost(@RequestBody PostCreateRequest request) {
        // ...
    }
}
```

1. **WebMvcTest 불가.** `@WebMvcTest`는 웹 계층만 로드한다. Controller에 `@Transactional`이 있으면 DataSource가 필요한데, `@WebMvcTest`는 DataSource를 로드하지 않는다.
2. **OSIV 의존.** Controller에서 트랜잭션이 시작되면 Open Session In View와 엮인다. Lazy Loading이 트랜잭션 안에서 동작하면서 N+1 쿼리의 원인이 된다.
3. **계층 오염.** Controller는 HTTP 요청을 받아서 Service에 위임하는 역할이다. 트랜잭션 관리까지 맡기면 단일 책임 원칙 위반이다.

---

## 5. V2-V5 아키텍처 논쟁 기록

이 아키텍처를 확정하기까지 Claude와 Gemini 간의 논쟁이 있었다. 같은 코드를 보여주고 리뷰를 요청했을 때, 정반대의 견해가 나왔다.

### 논쟁 요약

| 버전 | 주체 | 핵심 주장 | 판정 |
|------|------|----------|------|
| V2 | Claude | 현재 구조 건전, Service 조합 패턴 우수 | 기준 채택 |
| V3 | Gemini | Facade에 `@Transactional`, Service 불필요 | 거부 |
| V4 | Claude | V2 기반 + Controller TX 조건부 허용 | 부분 철회 |
| V5 | Gemini | Controller TX 위험 경고 + V3 재주장 | 부분 수용 |
| V6 | 최종 | V2 기반 + V5 지적 수용 + 실용적 개선 | 최종안 |

### V2: Claude — "현재 구조가 건전하다"

Application Service에 트랜잭션 경계를 두는 것이 정석이고, Facade가 `app-common`에 있는 것은 인프라 의존성 때문에 자연스럽다는 결론. 이 견해를 기준안으로 채택했다.

### V3: Gemini — "Facade에 @Transactional을 붙여라"

> "Facade가 이미 유스케이스를 조율하고 있다. Facade에 `@Transactional`을 선언하면 Application Service가 불필요한 위임 레이어가 된다. 제거하고 Controller → Facade로 직접 호출하라."

핵심 지적은 "Application Service가 Facade를 단순 위임만 한다면 존재 가치가 없다"는 것이었다. 실제로 단순 위임인 Service가 꽤 있었다.

```java
// V3가 지적한 "단순 위임" 패턴
@Service
public class VoucherAdminService {

    @Transactional
    public VoucherResponse createVoucher(VoucherCreateRequest request) {
        return voucherFacade.createVoucher(request);  // 위임만 한다
    }
}
```

하지만 V3를 거부한 이유가 있다. Facade에 `@Transactional`을 붙이면 `app-common` 모듈이 트랜잭션 매니저에 의존한다. 이 모듈은 Admin과 Service 양쪽에서 공유되는데, 트랜잭션 설정이 양쪽에서 동일하다는 보장이 없다. 트랜잭션 경계를 각 Application 모듈에서 관리하는 게 안전하다.

### V4: Claude — "Controller TX를 조건부로 허용하자"

V3의 지적을 Claude에게 보여주고 재평가를 요청했다. 기본 입장(V2)을 유지하면서도 "단순 조회 Controller에서 `@Transactional(readOnly = true)`는 허용 가능하다"고 양보했다. 이 양보를 나중에 부분 철회했다.

### V5: Gemini — "Controller TX는 위험하다"

> "Controller에 `@Transactional`을 허용하면, 한 번 허용한 경계가 무너진다. readOnly든 아니든 Controller에 트랜잭션을 두지 마라."

이 지적은 수용했다. 1인 개발에서 "조건부 허용"은 결국 "전면 허용"으로 흘러간다. 다만 V5에서 Gemini가 V3(Facade에 `@Transactional`)을 다시 주장한 부분은 거부했다.

### V6: 최종 결정

V2를 기반으로 하되, V5의 지적을 수용해서 최종안을 확정했다.

**Three Golden Rules:**

> **1. 트랜잭션은 Application Service에서만 선언한다.**
> Controller, Facade, DomainService에는 `@Transactional`을 붙이지 않는다. 유일한 예외는 QueryFacade의 `@Transactional(readOnly = true)`.
>
> **2. Facade는 app-common에 둔다.**
> 인프라 의존성이 있든 없든, Admin과 Service에서 공유되는 유스케이스 조율은 app-common이 담당한다.
>
> **3. Application Service는 진정한 가치가 있을 때만 유지한다.**
> Facade를 단순 위임만 하는 Service는 제거를 검토한다. 하지만 트랜잭션 경계, 권한 검증, 입력 변환 등 부가 로직이 하나라도 있으면 유지한다.

Rule 3은 V3(Gemini)의 지적을 실용적으로 수용한 것이다. 가치가 있으면 유지하고, 순수 위임만이면 제거를 검토한다.

---

## 6. Entity 스타일: 정적 팩토리 > Builder

Entity를 어떻게 생성할 것인가. 두 가지 패턴을 비교했다.

### Builder 패턴의 문제점

```java
// Builder -- 잘못된 상태의 객체 생성 가능
Voucher voucher = Voucher.builder()
    .code("ABC123")
    .state(1)               // 매직 넘버, 검증 없음
    .build();

// 이런 것도 가능하다
Voucher invalid = Voucher.builder()
    .state(99)              // 존재하지 않는 상태
    .build();               // code가 null인 Voucher 생성
```

Builder는 모든 필드를 선택적으로 설정할 수 있다. 필수 필드를 빼먹어도 컴파일 에러가 나지 않는다. `state`에 매직 넘버를 넣어도 검증이 없다. 잘못된 상태의 객체가 만들어지는 것을 막을 수 없다.

Lombok의 `@Builder`를 쓰면 상황이 더 나빠진다. 모든 필드에 대한 setter가 열리기 때문이다. 도메인 모델의 불변성이 깨진다.

### 정적 팩토리 메서드의 장점

```java
// 정적 팩토리 -- 생성 시 비즈니스 규칙 검증 강제
public static Voucher create(String code, LocalDateTime validUntil, SaleChannel channel) {
    if (code == null || code.length() != 10) {
        throw new DomainException(DomainErrorCode.INVALID_INPUT);
    }
    if (validUntil.isBefore(LocalDateTime.now())) {
        throw new DomainException(DomainErrorCode.EXPIRED_VOUCHER);
    }
    Voucher voucher = new Voucher();
    voucher.code = code;
    voucher.state = VoucherState.ISSUED.getCode();
    voucher.validUntil = validUntil;
    voucher.saleChannel = channel;
    voucher.issuedAt = LocalDateTime.now();
    return voucher;
}
```

정적 팩토리 메서드는 생성 시점에 비즈니스 규칙을 강제한다. `code`가 null이거나 10자가 아니면 예외. `validUntil`이 과거면 예외. `state`는 항상 `ISSUED`로 시작하며 호출자가 임의로 설정할 수 없다. 잘못된 상태의 객체가 생성되는 것 자체를 불가능하게 만든다.

### Rich Domain Model vs Anemic Domain Model

Martin Fowler는 2003년에 Anemic Domain Model을 "anti-pattern"으로 지목했다. Anemic Domain Model은 엔티티가 getter/setter만 가진 데이터 홀더이고, 모든 비즈니스 로직이 Service에 흩어져 있는 구조다.

```java
// Anemic Domain Model — 엔티티는 데이터만, 로직은 Service에
@Entity
@Getter @Setter  // 모든 필드가 외부에 열려 있다
public class Order {
    private Long id;
    private String status;
    private BigDecimal totalAmount;
    private LocalDateTime paidAt;
}

@Service
public class OrderService {

    public void pay(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        if (!"CREATED".equals(order.getStatus())) {  // 비즈니스 규칙이 Service에 있다
            throw new IllegalStateException("결제 불가 상태");
        }
        order.setStatus("PAID");              // setter로 상태를 직접 변경
        order.setPaidAt(LocalDateTime.now());
        orderRepository.save(order);
    }
}
```

이 구조의 문제는 세 가지다.

1. **비즈니스 규칙이 분산된다.** "CREATED일 때만 결제 가능"이라는 규칙이 `OrderService`에만 있다. 다른 Service에서 `order.setStatus("PAID")`를 호출하면 검증 없이 상태가 바뀐다.
2. **불변식(invariant)이 깨진다.** `@Setter`가 열려 있으므로 어디서든 `order.setStatus("INVALID")`를 호출할 수 있다. 컴파일 타임에 이를 막을 방법이 없다.
3. **중복이 퍼진다.** 같은 상태 검증 로직이 여러 Service에 복사된다. 규칙이 변경되면 모든 복사본을 찾아서 수정해야 한다.

Rich Domain Model은 이 로직을 엔티티 안으로 가져온다.

```java
// Rich Domain Model — 엔티티가 자기 상태를 관리한다
@Entity
@Getter  // Getter만, Setter 없음
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Order {
    private Long id;
    private String status;
    private BigDecimal totalAmount;
    private LocalDateTime paidAt;

    public static Order create(BigDecimal totalAmount) {
        Order order = new Order();
        order.status = "CREATED";
        order.totalAmount = totalAmount;
        return order;
    }

    public void pay() {
        if (!"CREATED".equals(this.status)) {
            throw new DomainException("결제 불가 상태");
        }
        this.status = "PAID";
        this.paidAt = LocalDateTime.now();
    }

    public void cancel() {
        if ("SHIPPED".equals(this.status)) {
            throw new DomainException("배송 후 취소 불가");
        }
        this.status = "CANCELLED";
    }
}
```

`pay()`, `cancel()` — 상태 전이 로직이 엔티티 안에 있다. Service는 `order.pay()`를 호출할 뿐, 상태 변경의 세부 규칙을 알 필요가 없다. `@Setter`가 없으므로 외부에서 `order.setStatus("PAID")`를 호출하는 것 자체가 불가능하다.

정적 팩토리 메서드는 Rich Domain Model의 입구다. 객체 생성 시점부터 불변식을 강제하고, 이후 상태 변경은 도메인 메서드를 통해서만 가능하게 만든다. `@Setter`와 `@Builder`를 금지하는 이유가 여기에 있다 — 이것들은 불변식을 우회하는 통로를 만든다.

### Entity 구조 규칙

Entity 클래스의 구조를 일관되게 유지하기 위한 규칙을 정했다.

```java
@Entity                                          // 1. JPA 어노테이션
@Table(name = "voucher")                         // 2. 테이블 매핑
@Getter                                          // 3. Getter만 허용
@NoArgsConstructor(access = AccessLevel.PROTECTED)  // 4. JPA용 기본 생성자 (외부 차단)
public class Voucher {

    // === PK ===
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "vou_id")
    private Long id;

    // === FK ===
    @Column(name = "member_id")
    private Long memberId;

    @Column(name = "plan_id")
    private Long planId;

    // === Business Fields ===
    @Column(name = "vou_code", nullable = false, length = 10)
    private String code;

    @Column(name = "vou_valid_until")
    private LocalDateTime validUntil;

    // === Status ===
    @Column(name = "vou_state", nullable = false)
    private Integer state;

    // === Audit ===
    @CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // === Factory Methods ===
    public static Voucher create(String code, LocalDateTime validUntil, SaleChannel channel) {
        // ... 검증 + 생성
    }

    // === Enum Methods ===
    public VoucherState getVoucherState() {
        return VoucherState.fromCode(this.state);
    }

    // === Domain Logic ===
    public void activate(Long memberId) {
        if (this.getVoucherState() != VoucherState.ISSUED) {
            throw new DomainException(DomainErrorCode.INVALID_STATE_TRANSITION);
        }
        this.memberId = memberId;
        this.state = VoucherState.ACTIVATED.getCode();
    }
}
```

**금지 목록:**
- `@Setter` — 외부에서 필드를 임의로 변경하는 것을 막는다
- `@Builder` — 잘못된 상태의 객체 생성을 허용한다
- `@AllArgsConstructor` — 모든 필드를 외부에 노출한다

**섹션 순서:**

| 순서 | 섹션 | 설명 |
|------|------|------|
| 1 | PK | `@Id`, `@GeneratedValue` |
| 2 | FK | 연관 엔티티의 ID (직접 참조보다 ID 참조 선호) |
| 3 | Business Fields | 도메인 고유 필드 |
| 4 | Status | 상태 코드 (`Integer`로 저장, Enum 변환 메서드 제공) |
| 5 | Audit | `createdAt`, `updatedAt` |
| 6 | Factory Methods | `create()`, `createForImport()` 등 |
| 7 | Enum Methods | 상태 코드 → Enum 변환 |
| 8 | Domain Logic | 상태 전이, 비즈니스 행위 |
| 9 | Validation | `isActivatable()`, `isExpired()` 등 |

이 구조가 앞에서 설명한 Rich Domain Model이다. `activate()` 메서드를 보자. Voucher를 활성화하려면 현재 상태가 `ISSUED`여야 한다는 규칙이 엔티티 안에 있다. Service에서 `if (voucher.getState() == 0)` 같은 조건을 체크하는 게 아니라, 엔티티 자신이 상태 전이 가능 여부를 판단한다. 정적 팩토리로 생성을 통제하고, 도메인 메서드로 변경을 통제한다. 외부에서 엔티티의 상태를 임의로 조작할 수 있는 경로가 없다.

---

## 7. 돌아보며

1인 개발에서 아키텍처 가이드를 15개 이상 문서화한 것이 과했을까.

처음에는 그렇게 생각했다. 혼자 개발하는데 누가 이 가이드를 읽겠나. 하지만 시간이 지나면서 가이드의 독자가 "미래의 나"라는 걸 깨달았다. 3개월 전에 왜 이런 결정을 했는지 기억이 안 날 때, 가이드 문서가 그 이유를 알려준다.

AI(Claude, Gemini) 간의 아키텍처 논쟁이 의외로 유용했다. 혼자 개발하면 자기 결정에 대한 반론을 들을 기회가 없다. 두 AI가 정반대의 견해를 제시하면, 어느 쪽이 맞는지 직접 판단해야 한다. 그 판단 과정에서 아키텍처에 대한 이해가 깊어진다. 전부 수용하거나 전부 거부하는 게 아니라, 각 주장의 근거를 따져서 부분적으로 수용하는 것이 핵심이다.

결국 세 가지로 정리된다.

> 1. **완벽한 아키텍처보다 일관된 아키텍처가 중요하다.** 트랜잭션을 Application Service에서 관리하든 Facade에서 관리하든, 한 가지로 통일하는 것이 핵심이다. 절반은 Service에서, 절반은 Facade에서 관리하는 게 최악이다.
>
> 2. **규칙은 단순해야 지켜진다.** "Controller에서 `@Transactional`은 readOnly이고, 단순 조회이고, Service를 거치지 않는 경우에만 허용"이라는 규칙은 너무 복잡하다. "Controller에서 `@Transactional` 금지"가 지켜지는 규칙이다.
>
> 3. **문서화된 결정은 자산이다.** V2부터 V6까지의 논쟁 기록이 남아있기 때문에, 나중에 같은 질문이 나왔을 때 처음부터 고민하지 않아도 된다. "왜 Facade를 domain에 안 넣었지?"라는 질문에 V3 논쟁 기록을 보여주면 된다.

아키텍처는 정답이 없다. 하지만 정답이 없다고 아무렇게나 해도 된다는 뜻은 아니다. 제약 조건 안에서 일관된 결정을 내리고, 그 결정의 이유를 기록하는 것. 그게 1인 개발에서 아키텍처가 하는 일이다.
