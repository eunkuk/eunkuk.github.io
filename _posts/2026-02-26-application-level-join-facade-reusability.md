---
title: "Facade로 공용 로직을 뽑았더니 JOIN이 사라졌다 — Application-level Join의 발견"
date: 2026-02-26 22:00:00 +0900
categories: [Dev, Architecture]
tags: [spring-boot, facade, join, query-optimization, ddd, aggregate, application-level-join, retrospective]
description: "Facade에 공용 로직을 모으면서 발견한 것 — Repository에서 JOIN이 필요 없어진다. Application-level Join과 RDBMS Join의 트레이드오프, Facade 재활용성이 쿼리 전략까지 바꾸는 과정을 기록한다."
---

> Facade에 공용 로직을 모으면서 발견한 것 — Repository에서 JOIN이 필요 없어진다. Application-level Join과 RDBMS Join의 트레이드오프, Facade 재활용성이 쿼리 전략까지 바꾸는 과정을 기록한다.

---

## 1. 상황

> 이 글은 [Facade의 위치와 트랜잭션 경계](/posts/spring-boot-facade-transaction-architecture/)와 [경량 CQRS — Command/Query 분리](/posts/lightweight-cqrs-facade-separation/)의 연장이다. 이전 글에서 Facade를 **어디에** 두고, Command와 Query를 **어떻게** 분리하는지를 다뤘다면, 여기서는 Facade 재활용 과정에서 쿼리 전략이 **어떻게 바뀌었는지**를 기록한다.

Facade에 공용 로직을 뽑는 리팩토링을 하고 있었다.

Admin과 Service에서 같은 조회를 하고 있었다. 예를 들어 주문 상세를 조회할 때, 주문 정보와 함께 회원 이름, 결제 정보가 필요하다. 기존에는 이걸 Repository에서 한 방 JOIN으로 해결하고 있었다.

```java
// Repository: 3테이블 JOIN으로 한 방 조회
public OrderDetailDto findOrderDetail(Long orderId) {
    return queryFactory.select(...)
        .from(order)
        .join(member).on(member.id.eq(order.memberId))
        .join(payment).on(payment.orderId.eq(order.id))
        .where(order.id.eq(orderId))
        .fetchFirst();
}
```

한 번의 쿼리로 주문 + 회원 + 결제를 모두 가져온다. 나름 효율적이다. 이 쿼리 자체에 문제가 있었던 건 아니다.

질문은 Facade를 만들면서 생겼다. 이 조회 로직을 공용 Facade로 뽑으려는데, **이 JOIN 쿼리까지 그대로 공용으로 가져가는 게 맞는가?**

Facade는 여러 DomainService를 조합하는 계층이다. `OrderDomainService`와 `MemberDomainService`와 `PaymentDomainService`를 각각 호출한다. 그러면 Facade가 이미 각 Aggregate의 데이터를 개별적으로 가져올 수 있는데, Repository에서 굳이 3테이블을 JOIN할 필요가 있을까?

---

## 2. 두 가지 쿼리 전략

같은 결과를 얻는 두 가지 방법이 있다.

### RDBMS Join

DB가 한 방에 해결한다. 애플리케이션은 조건만 넘기고, DB가 여러 테이블을 JOIN해서 결과를 돌려준다.

```sql
SELECT o.*, m.name, p.amount, p.status
FROM order_info o
    JOIN member m ON m.id = o.member_id
    JOIN payment p ON p.order_id = o.id
WHERE o.id = ?
```

한 번의 쿼리로 끝난다. 하지만 JOIN이 필요하고, 테이블이 늘어나면 쿼리가 복잡해진다.

### Application-level Join

애플리케이션에서 단계별로 가져온다. 각 Aggregate를 독립적으로 조회하고, 애플리케이션 코드에서 조합한다.

```sql
-- Step 1: 주문 조회
SELECT * FROM order_info WHERE id = ?

-- Step 2: 회원 조회
SELECT * FROM member WHERE id = ?

-- Step 3: 결제 조회
SELECT * FROM payment WHERE order_id = ?
```

쿼리가 3번 나간다. 하지만 각 쿼리는 단순하고, JOIN이 없다.

### 비교

| 기준 | RDBMS Join | Application-level Join |
|------|-----------|----------------------|
| 쿼리 수 | 1회 | 2회 이상 |
| JOIN 복잡도 | 테이블 수에 비례 | 없음 |
| 각 쿼리의 단순도 | 테이블이 많으면 복잡 | 항상 단순 |
| 캐싱 가능성 | JOIN 결과 전체를 캐싱 | 각 단계별 캐싱 가능 |
| Aggregate 독립성 | 테이블 간 결합 | 각 Aggregate 독립 |
| 적합한 상황 | 목록/검색/필터/페이징 | 단건 조회, ID 확보된 경우 |

둘 다 정답이 될 수 있다. 상황에 따라 다르다.

---

## 3. Facade가 만든 변화

Facade를 공용 계층으로 뽑으면서 자연스럽게 질문이 이어졌다. 기존 JOIN 쿼리를 그대로 가져가면 어떤 모양이 되는가, 그리고 Facade의 관점에서 다시 보면 어떻게 달라지는가.

### 그대로 가져가면: Repository가 여러 Aggregate를 알아야 한다

기존 방식을 Facade로 감싸면 이렇게 된다.

```java
// Facade: Repository의 JOIN 쿼리를 그대로 위임
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class OrderQueryFacade {

    private final OrderRepository orderRepository;

    public OrderDetailResponse getOrderDetail(Long orderId) {
        OrderDetailDto dto = orderRepository.findOrderDetail(orderId);  // 3테이블 JOIN
        return OrderDetailResponse.from(dto);
    }
}
```

```java
// Repository: 3테이블 JOIN
public OrderDetailDto findOrderDetail(Long orderId) {
    return queryFactory.select(...)
        .from(order)
        .join(member).on(member.id.eq(order.memberId))
        .join(payment).on(payment.orderId.eq(order.id))
        .where(order.id.eq(orderId))
        .fetchFirst();
}
```

동작은 한다. 하지만 어색한 점이 있다. Facade를 도입한 이유가 여러 DomainService를 조합하기 위해서인데, 정작 조합은 Repository의 JOIN이 하고 있다. Facade는 껍데기만 씌운 셈이다. 그리고 Order Repository가 member 테이블과 payment 테이블의 구조를 알아야 한다 — Aggregate 간 경계가 무너진다.

### Facade 관점에서 다시 보면: 각 DomainService가 자기 Aggregate를 조회한다

Facade가 본래 하는 일 — 여러 DomainService를 조합 — 을 그대로 살리면 이렇게 바뀐다.

```java
// Facade: 각 DomainService를 조합
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class OrderQueryFacade {

    private final OrderDomainService orderDomainService;
    private final MemberDomainService memberDomainService;
    private final PaymentDomainService paymentDomainService;

    public OrderDetailResponse getOrderDetail(Long orderId) {
        Order order = orderDomainService.getById(orderId);
        Member member = memberDomainService.getById(order.getMemberId());
        Payment payment = paymentDomainService.getByOrderId(orderId);
        return OrderDetailResponse.of(order, member, payment);
    }
}
```

```java
// Order Repository: order 테이블만 조회, JOIN 없음
public Order findById(Long orderId) {
    return orderRepository.findById(orderId).orElseThrow();
}

// Member Repository: member 테이블만 조회
public Member findById(Long memberId) {
    return memberRepository.findById(memberId).orElseThrow();
}

// Payment Repository: payment 테이블만 조회
public Payment findByOrderId(Long orderId) {
    return paymentRepository.findByOrderId(orderId).orElseThrow();
}
```

달라진 점을 정리하면 이렇다.

1. **3테이블 JOIN이 사라졌다.** 각 Repository는 자기 테이블만 조회한다. member 테이블의 구조가 바뀌어도 Order Repository는 영향이 없다.
2. **Facade가 실질적인 조합을 한다.** 껍데기가 아니라, 실제로 여러 Aggregate를 가져와서 하나의 응답을 만든다.
3. **각 DomainService가 독립적으로 재사용된다.** `memberDomainService.getById()`는 주문 상세뿐 아니라 다른 Facade에서도 쓸 수 있다.

결과적으로, Facade를 공용 계층으로 뽑으면서 "이 Facade의 역할이 뭔가?"를 고민한 것이 쿼리 전략까지 바꾼 셈이다. Facade가 조합을 담당하니, Repository는 자기 Aggregate만 알면 된다.

### 실제 효과: 4테이블 JOIN → 단일 테이블 쿼리들

단순한 예시에서는 차이가 작아 보인다. 하지만 실제 코드에서는 복합 조회가 있었다.

```java
// Before: 4테이블 JOIN
queryFactory.select(...)
    .from(order)
    .join(member).on(member.id.eq(order.memberId))
    .join(payment).on(payment.orderId.eq(order.id))
    .join(product).on(product.id.eq(order.productId))
    .where(order.id.eq(orderId))
    .fetchFirst();
```

```java
// After: Facade에서 조합
Order order = orderDomainService.getById(orderId);
Member member = memberDomainService.getById(order.getMemberId());
Payment payment = paymentDomainService.getByOrderId(orderId);
Product product = productDomainService.getById(order.getProductId());
return OrderDetailResponse.of(order, member, payment, product);
```

4테이블 JOIN 하나가 단일 테이블 쿼리 4개로 바뀌었다. 쿼리 수는 늘었지만 각 쿼리는 PK 또는 인덱스 기반의 단순 조회다. 실행 계획이 예측 가능하고, 각 DomainService를 독립적으로 캐싱할 수 있다.

---

## 4. 왜 모든 쿼리를 바꾸지 않았는가

Application-level Join이 항상 좋은 건 아니다. 하이브리드 전략을 쓴다.

### Application-level Join이 적합한 경우

- **단건 조회**: 하나의 주문, 하나의 회원 상세 — ID가 이미 확보된 상태에서 조회
- **Facade 경유**: Facade가 여러 DomainService를 조합하는 구조에서 자연스럽다
- **캐싱 대상**: 각 Aggregate를 독립적으로 캐싱할 수 있다. 회원 정보가 바뀌어도 주문 캐시는 유효하다

### RDBMS Join이 적합한 경우

- **목록 검색 + 필터**: "이름에 '김'이 포함된 회원의 주문 목록" — member 테이블의 조건으로 order를 필터링
- **페이징이 있는 목록 조회**: 페이징은 DB에서 하는 게 효율적이다. Application에서 전체를 가져와서 잘라내는 건 비효율
- **여러 테이블의 조건 조합**: "결제 완료이고, 배송 전이고, VIP 회원인 주문" — 여러 테이블의 조건을 동시에 걸어야 한다
- **집계/통계**: COUNT, SUM, GROUP BY — DB가 최적화된 영역

### 판단 기준

| 기준 | Application-level Join | RDBMS Join |
|------|----------------------|-----------|
| 조회 대상 | 단건 | 목록 |
| 식별자 확보 | 이미 있음 (Facade 경유) | 없음 (검색 조건) |
| 필터 조건 | 단순 (ID 기반) | 복합 (여러 테이블) |
| 페이징 | 불필요 | 필요 |
| 결과 크기 | 소량 | 대량 가능 |

간단히 말하면 — **ID로 단건을 찾으면 Application-level, 조건으로 목록을 검색하면 RDBMS Join.**

---

## 5. 장단점

Application-level Join은 만능이 아니다.

### 장점

**쿼리 단순화.** JOIN이 사라지면 쿼리가 단순해진다. 단순한 쿼리는 읽기 쉽고, 디버깅하기 쉽고, 실행 계획이 예측 가능하다. `WHERE id = ?`는 PK 인덱스만 있으면 성능이 보장된다.

**Aggregate 독립성.** DDD에서 Aggregate 간의 참조는 ID로 한다. Application-level Join은 이 원칙에 자연스럽게 부합한다. Order Repository는 order 테이블만 알면 된다. member 테이블의 구조가 바뀌어도 Order Repository는 영향이 없다.

**캐싱 용이.** RDBMS Join은 결과 전체를 캐싱해야 하지만, Application-level Join은 각 Aggregate를 독립적으로 캐싱할 수 있다. 회원 정보가 바뀌면 회원 캐시만 무효화하면 된다. 주문 캐시, 결제 캐시는 그대로 유효하다.

**재사용성.** `memberDomainService.getById()`는 주문 상세에서도, 이용권 상세에서도, 배송 상세에서도 재사용된다. JOIN 쿼리 안에 묶여 있으면 재사용이 안 된다.

### 단점

**네트워크 라운드트립 증가.** 1번의 쿼리가 3~4번이 된다. 같은 DB 서버 안에서 JOIN하는 것보다 느릴 수 있다. 다만 단건 조회에서 이 차이는 대부분 미미하다.

**N+1 위험.** 목록 조회에서 Application-level Join을 쓰면 위험하다. 주문 100건을 가져온 뒤 각 주문의 회원 정보를 개별 조회하면 쿼리가 101번 나간다. 이것이 목록 조회에서 RDBMS Join을 유지하는 이유다.

**벌크 조회 부적합.** "전체 회원의 최근 주문" 같은 조회에는 적합하지 않다. 이 경우 DB가 한 번에 처리하는 게 압도적으로 효율적이다.

---

## 6. 돌아보며

처음부터 "Application-level Join을 적용하자"고 시작한 게 아니다. Facade를 공용 계층으로 뽑으면서 "기존 JOIN 쿼리를 그대로 가져가는 게 맞나?"라는 질문이 생겼고, Facade의 역할을 제대로 살리다 보니 쿼리 전략까지 바뀐 것이다.

> **1. Facade를 공용으로 뽑으면 쿼리 전략을 재고할 기회가 생긴다.** "이 JOIN까지 공용으로 가져가야 하나?"라는 질문이 자연스럽게 나온다. Facade가 여러 DomainService를 조합하는 계층이라면, Repository에서 여러 테이블을 JOIN할 이유가 줄어든다.

> **2. 쿼리 전략은 all-or-nothing이 아니라 하이브리드다.** 단건 조회에서 ID가 이미 있으면 Application-level Join. 목록 검색에서 필터 조건이 여러 테이블에 걸쳐 있으면 RDBMS Join. 둘 중 하나를 선택하는 게 아니라 상황에 맞게 섞는다.

> **3. 아키텍처 결정은 한 번에 완성되지 않는다.** Facade를 도입한 건 Admin과 Service에서 로직을 공유하기 위해서였다. 쿼리 전략 개선은 의도하지 않았지만, 구조를 정리하다 보니 따라온 것이다. 좋은 구조는 의도하지 않은 이점을 만들어낸다.

---

**시리즈 연결:**
- [Facade는 왜 Domain이 아닌 Application 레이어에 있는가](/posts/spring-boot-facade-transaction-architecture/) — Facade **어디에**
- [풀 CQRS 없이 Command/Query를 분리한 방법](/posts/lightweight-cqrs-facade-separation/) — Command/Query **분리**
- 이 글 — Facade 재활용 과정에서 쿼리 전략이 **어떻게 바뀌었는지**
