---
title: "Rich Domain Model을 실제로 적용하면서 타협한 것들"
date: 2026-03-06 22:00:00 +0900
categories: [Dev, Architecture]
tags: [spring-boot, ddd, entity, rich-domain-model, jpa, retrospective]
description: "@Setter 금지, @Builder 금지, 정적 팩토리 필수 — 원칙은 알겠는데 실전에서는 타협이 필요했다. Rich Domain Model을 적용하면서 지킨 것과 양보한 것을 기록한다."
---

> @Setter 금지, @Builder 금지, 정적 팩토리 필수 — 원칙은 알겠는데 실전에서는 타협이 필요했다. Rich Domain Model을 적용하면서 지킨 것과 양보한 것을 기록한다.

---

## 1. 상황

> 이 글은 [DomainService vs Facade 경계](/posts/domain-service-vs-facade-boundary/)와 [에러 코드 아키텍처](/posts/domain-error-code-architecture/)의 연장이다. DomainService에 순수 비즈니스 로직을 두고, 도메인 예외를 분리해서 던진다고 했다. 그러면 엔티티 자체는 어떻게 설계하는가? Rich Domain Model을 적용하면서 **지킨 것과 타협한 것**을 기록한다.

마이그레이션 전의 레거시 시스템은 PHP CodeIgniter였다. 모델이라는 개념이 있었지만, 실질적으로는 DB wrapper였다. getter/setter 없이 연관 배열(associative array)로 데이터를 전달했다. 비즈니스 로직은 Controller에 있었다.

```php
// PHP 레거시: 모델 = DB wrapper
$order = $this->order_model->get($id);
$order['status'] = 3;  // 매직 넘버로 상태 변경
$this->order_model->update($id, $order);
```

`status = 3`이 무슨 의미인지 코드만 봐서는 알 수 없었다. 상태 전이 규칙도 없었다. 아무 값이나 넣을 수 있었다.

Java로 마이그레이션하면서 결정해야 할 것이 있었다. 엔티티를 어떤 수준으로 설계할 것인가. Anemic Model(데이터만 담는 구조체)과 Rich Model(비즈니스 로직을 포함하는 객체) 사이에서 위치를 정해야 했다.

```
Anemic                                              Rich
  |──────────────|──────────────|──────────────|──────|
  getter/setter    검증 포함       상태 전이        DDD
  only             팩토리         도메인 메서드    Aggregate
                                                  Root
                                  ↑
                              우리 위치
```

교과서적인 DDD Aggregate Root까지 가지는 않았다. 하지만 Anemic Model도 아니다. 정적 팩토리, 상태 전이 메서드, 비즈니스 검증 — 이 정도를 엔티티에 넣는 수준으로 결정했다.

---

## 2. 금지 목록

타협 없이 지킨 것이 있다. `@Setter`, `@Builder`, `@AllArgsConstructor` — 이 세 가지는 금지했다.

### @Setter — 불변식을 우회하는 통로

```java
// @Setter가 있으면
order.setStatus(99);  // 아무 값이나 넣을 수 있다
order.setDeadline(null);  // 마감일을 null로 만들 수 있다
```

`setStatus(99)` — 99가 유효한 상태 코드인지 아닌지 검증이 없다. 레거시 PHP에서 `$order['status'] = 3;`과 본질적으로 같다. 외부에서 엔티티의 내부 상태를 마음대로 바꿀 수 있으면, 비즈니스 규칙을 엔티티에 넣는 의미가 없어진다.

### @Builder — 필수 필드 누락을 컴파일러가 못 잡는다

```java
// @Builder가 있으면
Order order = Order.builder()
    .title("주문 A")
    // deadline 빠짐 — 컴파일 에러 없음
    .build();
```

마감일 없는 주문이 만들어진다. Builder는 모든 필드가 선택적이다. 필수 필드를 빠뜨려도 컴파일 에러가 나지 않는다. 런타임에 NPE로 터지거나, 더 나쁘면 잘못된 데이터가 DB에 들어간다.

### @AllArgsConstructor — 모든 필드를 외부에 노출

```java
// @AllArgsConstructor가 있으면
Order order = new Order(null, memberId, "주문 A", deadline,
    confirmedAt, shippedAt, shippingPhone, channel, status, reviewStatus,
    createdAt, updatedAt);
```

12개 필드를 순서대로 넣어야 한다. `status`와 `reviewStatus`의 순서가 바뀌어도 둘 다 Integer니까 컴파일 에러가 나지 않는다. 그리고 `createdAt`, `updatedAt` 같은 Audit 필드를 외부에서 설정할 수 있게 된다.

---

## 3. 정적 팩토리 메서드

`@Builder`와 `@AllArgsConstructor` 대신 정적 팩토리 메서드를 쓴다.

### 기본: create()

```java
@Entity
@Table(name = "orders")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Order {

    // ==================== Primary Key ====================
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // ==================== Foreign Key ====================
    @Column(name = "member_id")
    private Long memberId;

    // ==================== Business Fields ====================
    private String title;
    private LocalDateTime deadline;
    private LocalDateTime confirmedAt;
    private LocalDateTime shippedAt;
    private String shippingPhone;
    private Integer channel;

    // ==================== Status ====================
    private Integer status;
    private Integer reviewStatus;

    // ==================== Audit ====================
    @CreationTimestamp
    private LocalDateTime createdAt;
    @UpdateTimestamp
    private LocalDateTime updatedAt;

    // ==================== Factory Methods ====================
    public static Order create(String title, LocalDateTime deadline, Integer channel) {
        Order order = new Order();
        order.title = title;
        order.deadline = deadline;
        order.channel = channel;
        order.status = OrderStatus.CREATED.getCode();
        order.reviewStatus = ReviewStatus.NONE.getCode();
        return order;
    }
}
```

`create()` 안에서 초기 상태를 설정한다. `status`는 항상 `CREATED`로 시작하고, `reviewStatus`는 `NONE`으로 시작한다. 외부에서 이 초기값을 바꿀 수 없다.

"왜 생성자 대신 팩토리인가?" — 이름이 있어서 의도가 명확하다. `new Order(title, deadline, channel)`보다 `Order.create(title, deadline, channel)`이 의도를 잘 표현한다.

### 상황별 팩토리

회원 엔티티는 생성 경로가 여러 가지일 수 있다.

```java
public class Member {

    // OAuth 소셜 로그인으로 가입 (외부 인증 정보만)
    public static Member createFromOAuth(String provider, String providerId,
                                          String email, String name) {
        Member member = new Member();
        member.provider = provider;
        member.providerId = providerId;
        member.email = email;
        member.username = name;
        member.state = MemberState.PENDING.getCode();  // 프로필 입력 대기
        return member;
    }

    // 표준 회원가입 (이메일 + 비밀번호)
    public static Member create(String email, String nickname,
                                 String encodedPassword) {
        Member member = new Member();
        member.email = email;
        member.nickname = nickname;
        member.username = nickname;
        member.password = encodedPassword;
        member.state = MemberState.ACTIVE.getCode();  // 바로 활성
        return member;
    }

    // 관리자 생성
    public static Member createAdmin(String email, String password,
                                      String username, String nickname) {
        Member member = new Member();
        member.email = email;
        member.password = password;
        member.username = username;
        member.nickname = nickname;
        member.state = MemberState.ACTIVE.getCode();
        return member;
    }
}
```

`createFromOAuth()` — 소셜 로그인을 통한 가입. 상태가 `PENDING`이다. 아직 비밀번호가 없다.
`create()` — 표준 가입 완료. 상태가 `ACTIVE`다.
`createAdmin()` — 관리자가 직접 회원을 만든다.

팩토리 이름만 보면 어떤 경로로 회원이 생성되는지 알 수 있다.

---

## 4. 상태 전이 메서드

엔티티가 자기 상태를 관리한다. 외부에서 `setStatus(3)` 하는 게 아니라, `confirm()`을 호출한다.

```java
public class Order {

    // ==================== State Transition ====================
    public void confirm(Long memberId) {
        if (!canBeConfirmed()) {
            throw new DomainException(DomainErrorCode.INVALID_ORDER_TRANSITION);
        }
        if (isExpired()) {
            throw new DomainException(DomainErrorCode.ORDER_EXPIRED);
        }
        this.status = OrderStatus.CONFIRMED.getCode();
        this.memberId = memberId;
        this.confirmedAt = LocalDateTime.now();
    }

    public void ship(LocalDateTime shippedAt, String shippingPhone) {
        if (!getStatusEnum().canTransitionTo(OrderStatus.SHIPPED)) {
            throw new DomainException(DomainErrorCode.INVALID_ORDER_TRANSITION);
        }
        this.status = OrderStatus.SHIPPED.getCode();
        this.shippedAt = shippedAt;
        this.shippingPhone = PhoneNumberFormatter.format(shippingPhone);
    }

    public void cancel() {
        this.status = OrderStatus.CANCELLED.getCode();
    }

    // ==================== Domain Logic ====================
    public boolean canBeConfirmed() {
        return getStatusEnum() == OrderStatus.CREATED && !isExpired();
    }

    public boolean isExpired() {
        return deadline != null && deadline.isBefore(LocalDateTime.now());
    }

    public boolean isActive() {
        return getStatusEnum() == OrderStatus.CONFIRMED && !isExpired();
    }
}
```

`confirm()`을 호출하면 엔티티가 스스로 전이 가능 여부를 검증한다. 확인 가능한 상태(`CREATED`)가 아니거나 만료되었으면 DomainException을 던진다. 외부에서 상태 값을 직접 넣는 게 아니라, 엔티티에게 "확인되었다"라고 알려주면 엔티티가 알아서 검증하고 상태를 바꾼다.

### Enum 패턴: Integer로 저장, Enum으로 변환

```java
public class Order {

    private Integer status = 0;  // DB에는 Integer로 저장

    // ==================== Enum Conversion ====================
    public OrderStatus getStatusEnum() {
        return OrderStatus.fromCode(this.status);
    }

    public OrderChannel getChannelEnum() {
        return OrderChannel.fromCode(this.channel);
    }
}
```

왜 `@Enumerated(EnumType.STRING)`을 쓰지 않는가? 레거시 DB가 Integer로 상태를 저장하고 있었기 때문이다. 이것은 뒤에서 다시 다룬다.

---

## 5. 타협한 것들

교과서적인 Rich Domain Model을 그대로 적용하지는 못했다. 레거시 시스템과의 호환성, 현실적인 제약 때문에 타협한 것들이 있다.

### 타협 1: Integer 상태 코드

`@Enumerated(EnumType.STRING)`으로 저장하면 DB에 `"CONFIRMED"`, `"SHIPPED"` 같은 문자열이 들어간다. 코드를 읽기 좋고, DB에서 직접 쿼리할 때도 의미가 명확하다.

하지만 레거시 DB가 Integer로 상태를 저장하고 있었다. `status = 1`은 생성, `status = 2`는 확인, `status = 3`은 배송. 레거시 PHP 시스템이 아직 같은 DB를 읽고 있었다. Enum String으로 바꾸면 레거시 시스템이 깨진다.

DB 마이그레이션(ALTER TABLE + UPDATE)으로 해결할 수 있지만, 전환 비용이 이점보다 컸다. 대신 `getStatusEnum()` 메서드로 코드 레벨에서는 Enum을 사용하되, DB에는 Integer를 저장하는 타협을 했다.

```java
// 코드에서는 Enum으로 다룬다
if (order.getStatusEnum() == OrderStatus.CONFIRMED) { ... }

// DB에는 Integer로 저장된다
// status = 2  (OrderStatus.CONFIRMED.getCode())
```

### 타협 2: ID 참조

DDD 교과서에서는 Aggregate 간 참조를 ID로 하라고 한다. `@ManyToOne`을 쓰지 않고 `Long memberId`를 쓴다.

```java
// 우리의 방식: ID 참조
@Column(name = "member_id")
private Long memberId;

// 교과서적 JPA: 객체 참조
// @ManyToOne(fetch = FetchType.LAZY)
// @JoinColumn(name = "member_id")
// private Member member;
```

이것은 DDD 원칙을 따른 것이기도 하지만, 현실적인 이유도 있었다.

1. **순환 참조 방지** — Member가 Order를 참조하고, Order가 Member를 참조하면 직렬화 시 무한 루프
2. **지연 로딩 이슈 방지** — `@ManyToOne(LAZY)`로 해도 프록시 초기화 시점 문제가 있다
3. **Aggregate 경계 명확화** — Order와 Member는 다른 Aggregate. ID로 참조하면 경계가 명확하다

대신 여러 Aggregate의 데이터가 필요하면 Facade에서 조합한다. [Application-level Join](/posts/application-level-join-facade-reusability/)에서 다룬 패턴이다.

### 타협 3: 카운터 메서드의 null 방어

```java
public class Article {

    public void increaseViewCount() {
        this.viewCount = (this.viewCount == null ? 0 : this.viewCount) + 1;
    }

    public void increaseLikeCount() {
        this.likeCount = (this.likeCount == null ? 0 : this.likeCount) + 1;
    }

    public void decreaseLikeCount() {
        this.likeCount = (this.likeCount == null || this.likeCount <= 0) ? 0 : this.likeCount - 1;
    }
}
```

`this.viewCount == null ? 0 : this.viewCount` — 레거시 데이터에 null이 있다. 새로 생성되는 엔티티는 팩토리에서 0으로 초기화하지만, 이미 DB에 있는 기존 데이터는 null일 수 있다.

DB 마이그레이션(`UPDATE article SET view_count = 0 WHERE view_count IS NULL`)으로 해결할 수 있다. 하지만 데이터가 많고, 마이그레이션 중 서비스 영향이 있을 수 있다. 코드에서 null 방어를 하는 게 현실적이었다.

`decreaseLikeCount()`에서 0 이하로 내려가지 않도록 방어하는 것도 같은 맥락이다. 레거시 데이터에 음수 좋아요 수가 있을 수 없지만, 동시성 이슈로 음수가 될 수 있다.

### 타협 4: Hibernate 종속 어노테이션

```java
// Hibernate 종속
@CreationTimestamp
private LocalDateTime createdAt;

@UpdateTimestamp
private LocalDateTime updatedAt;
```

`@CreationTimestamp`와 `@UpdateTimestamp`는 JPA 표준이 아니라 Hibernate 전용이다. JPA 표준으로 하려면 `@PrePersist`와 `@PreUpdate` 콜백을 써야 한다.

```java
// JPA 표준 방식
@PrePersist
protected void onCreate() {
    this.createdAt = LocalDateTime.now();
    this.updatedAt = LocalDateTime.now();
}

@PreUpdate
protected void onUpdate() {
    this.updatedAt = LocalDateTime.now();
}
```

표준 방식이 더 깔끔하고 명시적이다. 하지만 Hibernate 어노테이션이 더 간결하고, 실질적으로 Hibernate를 벗어날 일이 없다. JPA 구현체를 EclipseLink로 바꿀 가능성은 제로에 가깝다. 이론적인 이식성보다 실용적인 간결함을 택했다.

---

## 6. 필드 순서 규칙

엔티티가 30개를 넘으면서 필드 순서 규칙이 필요해졌다. 엔티티마다 필드 배치가 다르면 코드를 읽을 때 "이 필드가 어디 있지?"를 매번 찾아야 한다.

```
PK → FK → Business Fields → Status → Audit
→ Factory Methods → Enum Conversion → Domain Logic → Validation
```

각 섹션을 주석으로 구분한다.

```java
// ==================== Primary Key ====================
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

// ==================== Foreign Key ====================
@Column(name = "member_id")
private Long memberId;

// ==================== Business Fields ====================
private String title;
private LocalDateTime deadline;

// ==================== Status ====================
private Integer status;

// ==================== Audit ====================
@CreationTimestamp
private LocalDateTime createdAt;
@UpdateTimestamp
private LocalDateTime updatedAt;

// ==================== Factory Methods ====================
public static Order create(...) { ... }

// ==================== Enum Conversion ====================
public OrderStatus getStatusEnum() { ... }

// ==================== State Transition ====================
public void confirm(Long memberId) { ... }

// ==================== Domain Logic ====================
public boolean canBeConfirmed() { ... }
public boolean isExpired() { ... }
```

"왜 순서를 정했나?" — 엔티티를 열었을 때 예측 가능한 구조여야 한다. PK가 항상 맨 위에 있고, Audit가 항상 필드의 마지막에 있으면, 처음 보는 엔티티도 빠르게 파악할 수 있다. 엔티티가 5개일 때는 불필요하지만, 30개를 넘으면 일관된 구조가 생산성에 직접적으로 영향을 준다.

---

## 7. 돌아보며

Rich Domain Model은 스펙트럼이다. Anemic에서 Full DDD Aggregate까지 다양한 수준이 있고, 프로젝트 상황에 맞는 위치를 찾아야 한다. 교과서를 그대로 따르는 게 아니라, 레거시 호환성, 팀의 역량, 마이그레이션 비용을 고려해서 실용적인 수준을 정했다.

> **1. Rich Domain Model은 all-or-nothing이 아니다.** 정적 팩토리와 상태 전이 메서드를 도입하는 것만으로도 Anemic Model보다 훨씬 안전하다. Full DDD까지 가지 않아도 `setStatus(99)` 같은 실수를 막을 수 있다. 프로젝트 상황에 맞는 수준을 찾는 것이 중요하다.

> **2. 레거시와의 호환성은 현실적인 제약이다.** Integer 상태 코드, null 방어, Hibernate 어노테이션 — 이론적으로는 더 나은 방법이 있지만, 전환 비용이 이점을 초과하는 경우가 있다. "나중에 바꾸겠다"는 계획보다 "지금 동작하는 타협"이 현실적이다.

> **3. 금지 목록은 타협 없이 지켜야 한다.** `@Setter`, `@Builder`, `@AllArgsConstructor` — 이 세 가지는 불변식을 우회하는 통로다. Integer 상태 코드나 null 방어는 타협해도 되지만, 외부에서 엔티티 상태를 마음대로 바꿀 수 있는 통로는 절대 열어두면 안 된다. 이 통로를 한번 열면, 비즈니스 규칙을 엔티티에 넣는 의미가 사라진다.

---

**시리즈 연결:**
- [DomainService vs Facade — 경계를 정한 과정](/posts/domain-service-vs-facade-boundary/) — 로직의 **경계**
- [쿼리 복잡도에 따른 Repository 전략](/posts/repository-query-tier-selection/) — Repository **쿼리**
- [에러 코드 아키텍처](/posts/domain-error-code-architecture/) — **에러** 흐름
- 이 글 — Rich Domain Model의 **타협**
