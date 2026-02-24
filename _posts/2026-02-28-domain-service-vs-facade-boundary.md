---
title: "DomainService vs Facade — 이 로직은 어디에 둬야 하는가"
date: 2026-02-28 22:00:00 +0900
categories: [Dev, Architecture]
tags: [spring-boot, ddd, domain-service, facade, clean-architecture, layered-architecture, retrospective]
description: "Facade에 넣을지 DomainService에 넣을지 매번 고민된다. 순수 비즈니스 로직과 유스케이스 조율의 경계를 정한 과정을 기록한다."
---

> Facade에 넣을지 DomainService에 넣을지 매번 고민된다. 순수 비즈니스 로직과 유스케이스 조율의 경계를 정한 과정을 기록한다.

---

## 1. 상황

> 이 글은 [Facade의 위치와 트랜잭션 경계](/posts/spring-boot-facade-transaction-architecture/), [경량 CQRS — Command/Query 분리](/posts/lightweight-cqrs-facade-separation/), [Application-level Join의 발견](/posts/application-level-join-facade-reusability/)의 연장이다. 이전 글에서 Facade를 **어디에** 두고, Command와 Query를 **어떻게** 분리하며, 쿼리 전략이 **어떻게 바뀌었는지**를 다뤘다면, 여기서는 로직을 DomainService에 둘지 Facade에 둘지 **경계를 어떻게 정했는지**를 기록한다.

초기에는 Facade에 거의 모든 로직을 넣었다. DomainService는 Repository를 감싸는 wrapper 수준이었다. `getById()`, `save()` 정도만 있었다.

```java
// 초기: DomainService가 repository wrapper 수준
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class OrderDomainService {

    private final OrderRepository orderRepository;

    public Order getById(Long id) {
        return orderRepository.findById(id)
            .orElseThrow(() -> new DomainException(DomainErrorCode.NOT_FOUND_ORDER));
    }

    @Transactional
    public void save(Order order) {
        orderRepository.save(order);
    }
}
```

쿠폰 코드 형식 검증, 만료 체크, 상태 전이 검증 — 이런 것들이 전부 Facade에 있었다. Facade가 비즈니스 규칙도 알고, 인프라 서비스도 호출하고, DTO도 만들었다. 코드가 늘어나면서 반복되는 질문이 생겼다.

"이 검증 로직이 Facade에 있는 게 맞나?"

같은 검증이 Admin Facade와 Service Facade에서 각각 구현되어 있었다. 쿠폰 만료 체크를 Facade마다 하고 있었다. DRY 원칙을 위반하고 있었고, 비즈니스 규칙이 Facade에 분산되어 있어서 "이 규칙이 어디에 정의되어 있지?"를 찾기 어려웠다.

---

## 2. DomainService의 역할

판단 기준은 하나였다. **"이 로직이 Spring 없이도 의미가 있는가?"**

HTTP 요청, 캐시, 파일 업로드, 외부 API — 이런 것들이 없어도 그 자체로 비즈니스 의미가 있는 로직이면 DomainService에 둔다.

쿠폰 코드 형식 검증을 예로 들면 이렇다.

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class CouponValidationDomainService {

    private static final Pattern CODE_PATTERN = Pattern.compile("^[A-Z0-9]{4}-[A-Z0-9]{4}-[A-Z0-9]{4}$");

    private final CouponRepository couponRepository;

    /**
     * 코드 형식이 유효한지 검증한다.
     */
    public boolean isValidFormat(String code) {
        return code != null && CODE_PATTERN.matcher(code).matches();
    }

    /**
     * 쿠폰 코드를 검증하고 결과를 반환한다.
     * 형식 → 존재 여부 → 상태 → 만료 순서로 검증한다.
     */
    public CouponValidationResult validateCode(String code) {
        if (!isValidFormat(code)) {
            return CouponValidationResult.invalidFormat();
        }

        Optional<Coupon> couponOpt = couponRepository.findByCode(code);
        if (couponOpt.isEmpty()) {
            return CouponValidationResult.notFound();
        }

        return validateState(couponOpt.get());
    }

    /**
     * 쿠폰의 상태와 만료 여부를 검증한다.
     */
    public CouponValidationResult validateState(Coupon coupon) {
        if (!coupon.canBeUsed()) {
            return CouponValidationResult.invalidState(coupon.getStatusEnum());
        }

        if (coupon.isExpired()) {
            return CouponValidationResult.expired();
        }

        return CouponValidationResult.valid(coupon);
    }
}
```

이 코드에서 Spring이 하는 일은 빈 등록과 트랜잭션 관리뿐이다. 핵심 로직 — 정규식 검증, 상태 확인, 만료 체크 — 은 프레임워크와 무관하다. 이런 로직이 DomainService의 영역이다.

DomainService에 적합한 로직의 패턴을 정리하면 이렇다.

- **형식 검증**: 코드 포맷, 이메일 형식, 전화번호 형식
- **상태 전이 검증**: `canBeUsed()`, `canTransitionTo()`, `isExpired()`
- **비즈니스 규칙 판단**: 중복 체크, 권한 확인, 수량 제한
- **엔티티 조회 + Not Found 예외**: `getById()` — 없으면 `DomainException`

---

## 3. Facade의 역할

Facade는 유스케이스 조율자다. 여러 DomainService를 조합하거나, 인프라 서비스를 끼워 넣거나, 다른 Aggregate의 데이터를 모아서 하나의 응답을 만든다.

실제 코드에서 관찰한 Facade는 크게 세 가지 유형이었다.

### Thin Facade — 거의 1:1 위임

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ArticleReactionFacade {

    private final ArticleReactionDomainService articleReactionDomainService;
    private final ArticleBookmarkDomainService articleBookmarkDomainService;

    @Transactional
    public long toggleLike(Long memberId, Long articleId, String ip) {
        return articleReactionDomainService.toggleLike(memberId, articleId, ip);
    }

    public boolean isLikedBy(Long memberId, Long articleId) {
        return articleReactionDomainService.isLikedBy(memberId, articleId);
    }

    @Transactional
    public void bookmark(Long memberId, Long articleId) {
        articleBookmarkDomainService.bookmark(memberId, articleId);
    }

    @Transactional
    public void unbookmark(Long memberId, Long articleId) {
        articleBookmarkDomainService.unbookmark(memberId, articleId);
    }
}
```

45줄. 4개의 메서드가 전부 DomainService에 그대로 위임한다. 변환도, 조합도 없다. "이게 Facade가 필요한가?"라는 의문이 든다. 이 질문은 뒤에서 다시 다룬다.

### Orchestrator Facade — 여러 DomainService 조합

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class OrderFacade {

    private final OrderDomainService orderDomainService;
    private final PaymentDomainService paymentDomainService;
    private final ShippingDomainService shippingDomainService;
    private final MemberDomainService memberDomainService;
    private final OrderMapper orderMapper;

    public OrderDetailResponse getOrderDetailForAdmin(Long orderId) {
        Order order = orderDomainService.getById(orderId);
        Payment payment = paymentDomainService.findByOrderId(orderId).orElse(null);
        Shipping shipping = shippingDomainService.findByOrderId(orderId).orElse(null);

        String memberName = Optional.ofNullable(order.getMemberId())
            .map(memberDomainService::getById)
            .map(Member::getUsername)
            .orElse(null);

        return orderMapper.toDetailResponse(order, payment, shipping, memberName);
    }
}
```

4개의 DomainService를 호출하고, 결과를 모아서 하나의 응답을 만든다. 각 DomainService는 자기 Aggregate만 알고, Facade가 이들을 조합한다. 이것이 [Application-level Join](/posts/application-level-join-facade-reusability/)에서 다룬 패턴이다.

### Cross-Domain Facade — 다른 Aggregate 데이터로 계산

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class PricingCalculationFacade {

    private final ProductDomainService productDomainService;
    private final CouponDomainService couponDomainService;
    private final MemberGradeDomainService memberGradeDomainService;
    private final DiscountPolicyDomainService discountPolicyDomainService;

    public PriceCalculationResult calculate(Long productId, Long memberId, String couponCode) {
        Product product = productDomainService.getById(productId);
        MemberGrade grade = memberGradeDomainService.getByMemberId(memberId);
        Coupon coupon = couponDomainService.findByCode(couponCode).orElse(null);

        DiscountPolicy policy = discountPolicyDomainService.resolve(product, grade);
        int finalPrice = policy.apply(product.getPrice(), coupon);

        return PriceCalculationResult.of(product, finalPrice, policy, coupon);
    }
}
```

Product, MemberGrade, Coupon, DiscountPolicy — 서로 다른 Aggregate의 데이터를 가져와서 최종 가격을 계산한다. 어떤 DomainService 하나에 넣기 어렵다. Product가 MemberGrade를 알 필요도 없고, Coupon이 DiscountPolicy를 알 필요도 없다. Facade가 이들을 모아서 계산을 조율한다.

---

## 4. 경계가 모호한 케이스들

판단이 쉬운 경우만 있으면 좋겠지만, 실제로는 애매한 경우가 많았다.

### Case 1: 검증 로직 → DomainService

쿠폰 코드가 유효한지 검증하는 로직이 Facade에 있었다. 정규식 매칭, 상태 확인, 만료 체크 — 전부 순수 비즈니스 규칙이다. HTTP나 캐시가 없어도 동작한다. DomainService로 옮겼다.

### Case 2: 캐시 evict + 도메인 로직 → Facade

회원 정보를 수정한 뒤 인증 캐시를 무효화해야 하는 경우가 있었다.

```java
// Facade: 도메인 로직 + 인프라 side effect
@Transactional
public void updateMember(Long memberId, MemberUpdateRequest request) {
    Member member = memberDomainService.getById(memberId);
    member.updateProfile(request.getEmail(), request.getNickname());

    if (request.getPassword() != null) {
        String encoded = passwordEncoder.encode(request.getPassword());
        member.changePassword(encoded);
    }

    authCacheService.evict(memberId);  // 캐시 무효화
}
```

`member.updateProfile()`과 `member.changePassword()`는 엔티티의 도메인 메서드다. 하지만 `passwordEncoder`는 인프라 서비스이고, `evict()`는 캐시 무효화다. 이 조합은 Facade에서 해야 한다. DomainService가 PasswordEncoder를 알 필요가 없다.

### Case 3: 복잡한 쿼리 → DomainService에 두되 Repository에 위임

동적 검색 쿼리는 QueryDSL Repository에 구현한다. 이 Repository를 호출하는 건 DomainService다. Facade가 Repository를 직접 호출하지 않는다.

```
Facade → DomainService → QueryDSL Repository
                          (복잡한 쿼리는 여기)
```

DomainService가 쿼리의 "무엇을"을 정의하고, Repository가 "어떻게"를 구현한다.

### 판단 플로차트

```
로직이 중복되는가?
  ├─ YES → 공통 계층으로 뽑아야 한다
  │         ↓
  │       순수 비즈니스 로직인가?
  │         ├─ YES → DomainService
  │         └─ NO  → Facade (app-common)
  └─ NO  → 현재 위치 유지
```

"순수 비즈니스 로직인가?"의 판단 기준은 단순하다. **Spring 컨텍스트, HTTP, 캐시, 파일, 외부 API — 이런 것들이 없어도 동작하는가?** 그렇다면 DomainService, 아니라면 Facade.

---

## 5. Thin Facade 논쟁

ArticleReactionFacade처럼 거의 1:1로 위임만 하는 Facade가 정당한가? 코드 리뷰에서 이 질문이 나왔다.

"단순 위임 Facade는 불필요한 계층이다. 제거해야 한다."

일리 있는 지적이다. 단순 위임이면 Facade가 아무 가치를 더하지 않는다. Controller → Facade → DomainService 세 계층을 거치는데, 중간 계층이 `return domainService.doSomething(args);` 만 한다면 사라져도 된다.

하지만 반론도 있었다.

**Admin과 Service 양쪽 진입점을 통일한다.** ArticleReactionFacade는 공용 모듈에 있다. Admin Controller와 Service Controller 양쪽에서 이 Facade를 사용한다. 지금은 단순 위임이지만, 나중에 한쪽에서 "좋아요 시 알림을 보내야 한다"는 요구가 생기면 Facade에 로직을 추가하면 된다. DomainService에는 순수 비즈니스 로직만 남기고.

**판단 기준을 정했다.**

| 상황 | 판단 |
|------|------|
| Admin과 Service 양쪽에서 사용 | Thin이어도 유지 |
| 한쪽에서만 사용 + 인프라 로직 없음 | 제거 검토 |
| 지금은 Thin이지만 인프라 로직 추가 예정 | 유지 |

ArticleReactionFacade는 양쪽에서 사용하니 유지했다. 나중에 알림이나 로깅 같은 인프라 로직이 들어올 때 Facade가 그 자리를 해준다.

---

## 6. Facade가 Repository를 직접 호출하면

경계가 무너졌다는 신호가 하나 있다. Facade가 Repository를 직접 호출하는 경우다.

```java
// 경계가 무너진 코드
@Service
@RequiredArgsConstructor
public class OrderFacade {

    private final OrderRepository orderRepository;        // Repository 직접 의존
    private final MemberDomainService memberDomainService;

    public OrderDetailResponse getOrderDetail(Long orderId) {
        Order order = orderRepository.findById(orderId)   // DomainService를 거치지 않음
            .orElseThrow(() -> new ApiException(ErrorCode.NOT_FOUND_ORDER));
        // ...
    }
}
```

두 가지 문제가 보인다.

1. **예외 타입이 잘못됐다.** Facade는 `DomainException`이 아니라 `ApiException`을 직접 던지고 있다. `Not Found` 예외는 비즈니스 규칙이다. DomainService에서 `DomainException`을 던져야 하고, AOP가 이를 `ApiException`으로 변환해야 한다.
2. **DomainService의 역할이 빠졌다.** DomainService가 "주문이 존재하는가"를 판단하는 게 자연스럽다. Facade가 Repository를 직접 호출하면 DomainService를 우회하는 통로가 생긴다.

올바른 흐름은 이렇다.

```
Facade → DomainService → Repository
                ↓
         DomainException (Not Found)
                ↓
         AOP → ApiException → GlobalExceptionHandler
```

---

## 7. 돌아보며

DomainService와 Facade의 경계가 처음부터 명확했던 건 아니다. 초기에는 DomainService가 Repository wrapper였고, 비즈니스 로직이 Facade에 흩어져 있었다. 리팩토링하면서 점진적으로 경계가 잡혔다.

> **1. "Spring 없이도 의미가 있는 로직"이면 DomainService, 아니면 Facade.** 정규식 검증, 상태 전이 확인, 만료 체크 — 이런 순수 비즈니스 로직은 프레임워크와 무관하게 동작해야 한다. 캐시 무효화, 비밀번호 인코딩, 파일 업로드 — 인프라가 필요한 조합은 Facade가 맡는다.

> **2. Facade가 Repository를 직접 호출하면 경계가 무너진 신호다.** Facade → DomainService → Repository 흐름을 유지해야 한다. Facade가 Repository를 직접 호출하면 DomainService를 우회하는 통로가 생기고, 예외 처리 흐름도 깨진다.

> **3. 처음부터 완벽하게 나눌 수 없다.** 코드를 작성하면서 "이 로직이 여기 있는 게 맞나?" 질문이 반복된다. 그 질문이 경계를 정리하는 과정이다. 리팩토링하면서 경계가 명확해지는 것이지, 처음부터 완벽한 설계를 할 수 있는 게 아니다.

---

**시리즈 연결:**
- [Facade는 왜 Domain이 아닌 Application 레이어에 있는가](/posts/spring-boot-facade-transaction-architecture/) — Facade **위치**
- [풀 CQRS 없이 Command/Query를 분리한 방법](/posts/lightweight-cqrs-facade-separation/) — Command/Query **분리**
- [Facade로 공용 로직을 뽑았더니 JOIN이 사라졌다](/posts/application-level-join-facade-reusability/) — Application-level **Join**
- 이 글 — DomainService와 Facade의 **경계**
