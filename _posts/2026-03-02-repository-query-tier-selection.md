---
title: "Method Naming → @Query → QueryDSL — 쿼리 복잡도에 따른 Repository 전략"
date: 2026-03-02 22:00:00 +0900
categories: [Dev, Architecture]
tags: [spring-boot, jpa, querydsl, repository, query-optimization, retrospective]
description: "Repository 쿼리를 어떤 방식으로 구현할 것인가. Method Naming, @Query, QueryDSL 세 단계의 선택 기준과, 실제로 전환한 사례를 기록한다."
---

> Repository 쿼리를 어떤 방식으로 구현할 것인가. Method Naming, @Query, QueryDSL 세 단계의 선택 기준과, 실제로 전환한 사례를 기록한다.

---

## 1. 상황

> 이 글은 [DomainService vs Facade 경계](/posts/domain-service-vs-facade-boundary/)와 [Application-level Join의 발견](/posts/application-level-join-facade-reusability/)의 연장이다. DomainService가 Repository를 호출하고, Facade가 DomainService를 조합한다는 계층 구조를 다뤘다면, 여기서는 Repository 내부에서 **쿼리를 어떤 방식으로 구현할 것인가**를 기록한다.

처음에는 전부 Spring Data JPA의 Method Naming으로 시작했다. `findByEmail()`, `existsByNickname()` — 메서드명만으로 쿼리가 생성되니 편했다.

문제는 쿼리가 복잡해지면서 생겼다. 조건이 3~4개 붙으면 메서드명이 `findByCategoryIdAndIsActiveOrderBySortOrderAscItemIdAsc` 수준으로 길어졌다. 읽기 어렵고, 오타가 나기 쉬웠다.

그래서 QueryDSL을 도입했다. 그런데 이번에는 반대 문제가 생겼다. `findByEmail()` 같은 단순한 쿼리까지 QueryDSL로 작성하고 있었다. `queryFactory.selectFrom(member).where(member.email.eq(email)).fetchFirst()` — 한 줄이면 될 것을 5줄로 쓰고 있었다.

기준이 필요했다.

---

## 2. 3단계 전략

쿼리의 복잡도에 따라 세 가지 접근법을 구분했다.

| 기준 | Tier 1: Method Naming | Tier 2: @Query | Tier 3: QueryDSL |
|------|----------------------|----------------|-------------------|
| 복잡도 | 단일 엔티티, 단순 조건 | DISTINCT, IN, 1-2 JOIN | 동적 필터, 3+ JOIN, 서브쿼리 |
| 장점 | 메서드명 = 쿼리 의도, 컴파일 안전 | JPQL로 가독성 유지, 적당한 유연성 | 타입 안전, 동적 쿼리, 커스텀 DTO |
| 한계 | 조건 3개 이상이면 메서드명 폭발 | 동적 쿼리 불가, 문자열 기반 | 단순 쿼리에는 오버엔지니어링 |
| 파일 위치 | JpaRepository 인터페이스 | 같은 인터페이스, @Query 추가 | 별도 @Repository 클래스 |

프로젝트의 현재 분포를 보면 이렇다.

| 접근법 | Repository 수 | 비율 |
|--------|-------------|------|
| Method Naming Only | 34 | 59% |
| @Query 포함 | 7 (11 메서드) | 12% |
| QueryDSL | 17 | 29% |

절반 이상이 Method Naming만으로 충분하다. 대부분의 쿼리는 단순하다.

선택 기준을 플로차트로 정리했다.

```
쿼리가 필요한가?
  ↓
단일 엔티티 + 단순 조건?
  ├─ YES → Method Naming
  └─ NO  ↓
DISTINCT / IN / 단순 JOIN / 기본 집계?
  ├─ YES → @Query (JPQL)
  └─ NO  ↓
동적 필터 / 3+ JOIN / 커스텀 DTO / 서브쿼리?
  ├─ YES → QueryDSL
  └─ NO  → @Query 재검토
```

---

## 3. Tier 1: Method Naming

가장 단순한 방법이다. 메서드명이 곧 쿼리 의도다.

```java
public interface MemberRepository extends JpaRepository<Member, Long> {

    Optional<Member> findByEmail(String email);

    boolean existsByEmail(String email);
    boolean existsByNickname(String nickname);
    boolean existsByPhone(String phone);

    boolean existsByNicknameAndStateNot(String nickname, Integer state);
    boolean existsByEmailAndStateNot(String email, Integer state);

    Optional<Member> findByPhone(String phone);
    Optional<Member> findByPhoneAndStateNot(String phone, Integer state);

    long countByState(Integer state);
}
```

메서드명만 보면 "무엇을 조회하는가"가 바로 보인다. `existsByEmail()` — 이메일로 존재 여부를 확인한다. 설명이 필요 없다.

컴파일 타임에 안전하다. 필드명을 오타내면 애플리케이션 시작 시점에 에러가 난다. `findByEmal()` (오타) 같은 실수를 런타임이 아니라 시작 시점에 잡아준다.

```java
public interface CategoryRepository extends JpaRepository<Category, Long> {

    Optional<Category> findByCode(String code);
    boolean existsByCode(String code);
    List<Category> findAllByActive(Integer active);
    List<Category> findAllByActiveAndType(Integer active, Integer type);
}
```

4개의 메서드. 전부 단일 엔티티, 단순 조건이다. 이 수준이면 @Query도 QueryDSL도 필요 없다.

### 한계

조건이 많아지면 메서드명이 폭발한다.

```java
// 이 수준이면 가독성이 떨어진다
List<ProductOption> findByCategoryIdAndIsActiveOrderBySortOrderAscOptionIdAsc(
    Long categoryId, Integer isActive);
```

메서드명만으로 의도를 파악하기 어려워진다. 이때가 @Query로 올릴 시점이다.

---

## 4. Tier 2: @Query

DISTINCT, IN절, 단순 JOIN, 기본 집계가 필요하면 @Query를 쓴다.

```java
public interface ProductOptionRepository extends JpaRepository<ProductOption, Long> {

    // ── Method Naming: 단순 조건 ──
    List<ProductOption> findByCategoryId(Long categoryId);
    boolean existsByCategoryId(Long categoryId);
    List<ProductOption> findByCategoryIdAndType(Long categoryId, String type);

    // ── @Query: DISTINCT, IN 등 중간 복잡도 ──
    @Query("SELECT DISTINCT p.type FROM ProductOption p WHERE p.categoryId = :categoryId ORDER BY p.type")
    List<String> findDistinctTypesByCategoryId(@Param("categoryId") Long categoryId);

    @Query("SELECT DISTINCT p.groupName FROM ProductOption p WHERE p.categoryId = :categoryId ORDER BY p.groupName")
    List<String> findDistinctGroupsByCategoryId(@Param("categoryId") Long categoryId);

    @Query("SELECT p FROM ProductOption p WHERE p.categoryId IN :categoryIds")
    List<ProductOption> findByCategoryIdIn(@Param("categoryIds") List<Long> categoryIds);

    @Query("SELECT p FROM ProductOption p WHERE p.categoryId = :categoryId AND p.optionCode IN :optionCodes")
    List<ProductOption> findByCategoryIdAndOptionCodeIn(
        @Param("categoryId") Long categoryId,
        @Param("optionCodes") List<String> optionCodes);
}
```

하나의 Repository에 Method Naming과 @Query가 공존한다. 단순한 건 Method Naming으로, DISTINCT나 IN이 필요한 건 @Query로 — 각 메서드의 복잡도에 맞는 방법을 선택했다.

### @Param 필수 규칙

`@Query`를 사용할 때 모든 파라미터에 `@Param`을 붙인다.

```java
// Good
@Query("SELECT o FROM OrderItem o WHERE o.orderId = :orderId AND o.status != :excludeStatus")
List<OrderItem> findActiveByOrderId(@Param("orderId") Long orderId,
                                     @Param("excludeStatus") Integer excludeStatus);

// Bad
@Query("SELECT o FROM OrderItem o WHERE o.orderId = ?1 AND o.status != ?2")
List<OrderItem> findActiveByOrderId(Long orderId, Integer excludeStatus);
```

위치 기반 파라미터(`?1`, `?2`)는 순서가 바뀌면 버그가 생긴다. `@Param`은 이름으로 바인딩하니 순서와 무관하다.

### JOIN이 포함된 @Query

1-2개의 JOIN까지는 @Query로 충분하다.

```java
@Query("SELECT oi FROM OrderItem oi JOIN Order o ON oi.orderId = o.id " +
       "WHERE oi.barcode = :barcode AND o.status != :cancelledStatus " +
       "ORDER BY oi.sequence DESC")
List<OrderItem> findActiveItemsByBarcode(@Param("barcode") String barcode,
                                          @Param("cancelledStatus") Integer cancelledStatus);
```

JOIN이 하나이고, 조건이 고정되어 있다. 동적 필터링이 필요 없으면 @Query가 QueryDSL보다 간결하다.

### 왜 QueryDSL로 올리지 않았나

ProductOptionRepository는 @Query 메서드가 4개 있지만 QueryDSL로 분리하지 않았다. 이유는 단순하다.

1. 동적 쿼리가 아니다 — 모든 파라미터가 고정이다
2. JOIN이 없다 — 단일 엔티티 조회다
3. 커스텀 DTO projection이 없다 — 엔티티 그대로 반환하거나 단순 필드 반환이다

QueryDSL은 강력하지만, 이 수준의 쿼리에는 오버엔지니어링이다.

---

## 5. Tier 3: QueryDSL

동적 필터, 3개 이상의 JOIN, 커스텀 DTO projection, 서브쿼리 — 이 중 하나라도 필요하면 QueryDSL을 쓴다.

### 별도 @Repository 클래스로 분리

QueryDSL 쿼리는 JpaRepository 인터페이스에 넣지 않고, 별도의 클래스로 분리한다.

```java
@Repository
@RequiredArgsConstructor
public class ArticleQuerydslRepository {

    private final JPAQueryFactory queryFactory;

    // 쿼리 메서드들
}
```

인터페이스(JpaRepository)와 클래스(QueryDSL)가 분리되어 있으면, Method Naming 쿼리와 복잡한 쿼리가 뒤섞이지 않는다. DomainService에서는 용도에 따라 적절한 Repository를 주입받는다.

### Query Object Pattern

복잡한 쿼리를 읽기 쉽게 만드는 패턴이다. 기본 쿼리(SELECT/FROM/JOIN)와 동적 조건을 분리한다.

```java
@Repository
@RequiredArgsConstructor
public class BannerQuerydslRepository {

    private final JPAQueryFactory queryFactory;

    /**
     * 앱 배너 목록 조회 — 기본 쿼리 + 동적 조건 조합
     */
    public List<BannerAppRead> findAppBanners(String position, String type, int limit) {
        StringExpression imageUrl = Expressions.stringTemplate(
            "CONCAT({0}, {1})", banner.imagePath, banner.imageFile);

        BooleanExpression where = applyDisplayConditions(position);
        OrderSpecifier<?>[] orderSpecifiers = applyOrderCondition(type);

        return baseSelectBanner(imageUrl)
            .where(where)
            .orderBy(orderSpecifiers)
            .limit(limit)
            .fetch()
            .stream()
            .map(tuple -> new BannerAppRead(/* ... */))
            .toList();
    }

    /**
     * 기본 SELECT + FROM
     */
    private JPAQuery<Tuple> baseSelectBanner(StringExpression imageUrl) {
        return queryFactory
            .select(banner.id, banner.title, imageUrl, banner.linkUrl, banner.position)
            .from(banner)
            .where(banner.isDeleted.eq(0));
    }

    /**
     * 동적 조건: 위치, 날짜, 노출 상태
     */
    private BooleanExpression applyDisplayConditions(String position) {
        LocalDateTime now = LocalDateTime.now();
        return banner.position.eq(position)
            .and(banner.displayStartDate.loe(now))
            .and(banner.displayEndDate.goe(now))
            .and(banner.isDisplay.eq(1));
    }

    /**
     * 동적 정렬: 타입에 따라 다른 정렬 기준
     */
    private OrderSpecifier<?>[] applyOrderCondition(String type) {
        if ("main".equals(type)) {
            return new OrderSpecifier[]{banner.sortOrder.asc(), banner.id.desc()};
        }
        return new OrderSpecifier[]{banner.id.desc()};
    }
}
```

`baseSelectBanner()` — SELECT와 FROM을 정의한다. `applyDisplayConditions()` — 동적 WHERE 조건을 만든다. `applyOrderCondition()` — 동적 ORDER BY를 만든다. 각 조각이 독립적이어서 조합하기 쉽다.

### 커스텀 DTO Projection

엔티티 전체가 아니라 필요한 필드만 뽑아서 DTO로 만들 때 QueryDSL의 projection이 유용하다.

```java
// 서브쿼리로 파일 첨부 개수를 함께 조회
JPAExpressions.select(attachment.count())
    .from(attachment)
    .where(attachment.refId.eq(article.id)
        .and(attachment.refType.eq(RefType.ARTICLE))
        .and(attachment.isDeleted.eq(0)))
```

서브쿼리로 관련 파일 개수를 메인 쿼리에 포함시킨다. @Query JPQL로는 표현하기 어렵고, 엔티티로 가져온 뒤 애플리케이션에서 세면 N+1 문제가 생긴다.

### 동적 필터링

검색 화면에서 사용자가 입력한 조건에 따라 쿼리가 달라져야 할 때 BooleanBuilder를 쓴다.

```java
public Page<ArticleSummary> searchArticles(String keyword, String tag, Pageable pageable) {
    BooleanBuilder where = new BooleanBuilder();

    if (keyword != null && !keyword.isBlank()) {
        where.and(article.title.containsIgnoreCase(keyword));
    }

    if (tag != null && !tag.isBlank()) {
        where.and(JPAExpressions.selectOne()
            .from(articleTag)
            .where(articleTag.articleId.eq(article.id).and(articleTag.tag.eq(tag)))
            .exists());
    }

    return baseSelectArticleSummary()
        .where(where)
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize())
        .fetchPage();
}
```

keyword가 없으면 제목 조건이 빠지고, tag가 없으면 태그 조건이 빠진다. Method Naming이나 @Query로는 이런 동적 조합이 불가능하다.

### CaseBuilder — 조건부 값 변환

```java
// Boolean을 Integer로 안전하게 변환
new CaseBuilder()
    .when(agreement.agreeTerms.isTrue()
        .and(agreement.agreePrivacy.isTrue()))
    .then(1)
    .otherwise(0)
```

NULL 비교 이슈를 방지하면서 Boolean을 Integer로 변환한다. DB마다 Boolean 처리가 다를 수 있어서, 명시적으로 Integer로 변환하는 게 안전하다.

### 집계와 그룹핑

```java
// 월별 주문 통계
var yearExpr = Expressions.numberTemplate(Integer.class, "YEAR({0})", order.orderedAt);
var monthExpr = Expressions.numberTemplate(Integer.class, "MONTH({0})", order.orderedAt);

List<Tuple> rows = queryFactory
    .select(yearExpr, monthExpr, order.count())
    .from(order)
    .groupBy(yearExpr, monthExpr)
    .fetch();
```

DB 함수(YEAR, MONTH)를 QueryDSL에서 사용한다. 통계 쿼리는 @Query JPQL로도 가능하지만, 동적 기간 필터링이 추가되면 QueryDSL이 더 유연하다.

---

## 6. 전환 시점

언제 한 단계 올리는가? 실제로 전환을 결정한 기준들이다.

### Method Naming → @Query

- 메서드명이 30자를 넘으면
- DISTINCT, IN절이 필요하면
- 간단한 JOIN이 필요하면 (1-2개)

### @Query → QueryDSL

- 동적 필터링이 필요하면 — 사용자 입력에 따라 조건이 달라질 때
- JOIN이 3개 이상이면
- 커스텀 DTO projection이 필요하면
- 서브쿼리가 필요하면

### 한 Repository에 3단계가 공존할 수 있다

ProductOptionRepository는 Method Naming과 @Query가 하나의 인터페이스에 공존한다. 이게 정상이다. 메서드마다 복잡도가 다르니, 각 메서드에 맞는 접근법을 쓰면 된다.

다만 QueryDSL은 별도 클래스다. 같은 도메인에 `OrderRepository` (JpaRepository)와 `OrderQuerydslRepository` (QueryDSL)가 함께 존재한다. DomainService에서 용도에 따라 적절한 Repository를 주입받는다.

```java
@Service
@RequiredArgsConstructor
public class OrderDomainService {

    private final OrderRepository orderRepository;               // Method Naming + @Query
    private final OrderQuerydslRepository orderQuerydslRepository; // QueryDSL

    public Order getById(Long id) {
        return orderRepository.findById(id)  // Method Naming
            .orElseThrow(() -> new DomainException(DomainErrorCode.NOT_FOUND_ORDER));
    }

    public Page<OrderSearchRead> search(OrderSearchCondition condition, Pageable pageable) {
        return orderQuerydslRepository.search(condition, pageable);  // QueryDSL
    }
}
```

### 성능 개선 사례

QueryDSL로 전환하면서 성능이 개선된 사례가 있었다.

```java
/**
 * 변경 전: orderId 조회(1) + itemStatus 조회(2) + paymentStatus 조회(3) = 3회 DB 왕복
 * 변경 후: 1회 LEFT JOIN 쿼리로 모든 데이터 조회
 * 쿼리 수 66% 감소
 */
```

개별 쿼리 3번이 LEFT JOIN 하나로 합쳐졌다. Application-level Join이 적합한 경우도 있지만, 목록 조회에서 여러 테이블의 상태를 한꺼번에 가져와야 할 때는 DB JOIN이 효율적이다.

---

## 7. 돌아보며

3단계 전략이 처음부터 있었던 게 아니다. Method Naming으로 시작했고, 한계가 보이면서 @Query를 쓰기 시작했고, 동적 쿼리가 필요해지면서 QueryDSL을 도입했다. 경험이 쌓이면서 자연스럽게 기준이 생긴 것이다.

> **1. 가장 단순한 방법으로 시작하고, 복잡해지면 올린다.** `findByEmail()`을 QueryDSL로 쓸 이유가 없다. Method Naming으로 시작하고, DISTINCT나 IN이 필요하면 @Query로, 동적 쿼리가 필요하면 QueryDSL로 올린다. 반대로 QueryDSL에서 시작하면 단순한 쿼리까지 불필요하게 복잡해진다.

> **2. QueryDSL은 강력하지만, 간단한 쿼리에는 오버엔지니어링이다.** `queryFactory.selectFrom(member).where(member.email.eq(email)).fetchFirst()` — 이건 `findByEmail()`로 충분하다. 도구의 강력함과 적합함은 다른 문제다.

> **3. 기준을 문서화하면 매번 고민하지 않아도 된다.** "이 쿼리에 @Query를 쓸까 QueryDSL을 쓸까?" — 매번 고민하면 시간이 낭비된다. 판단 기준을 문서화하고 팀에서 공유하면, 개인의 취향이 아니라 합의된 기준으로 결정할 수 있다.

---

**시리즈 연결:**
- [Facade는 왜 Domain이 아닌 Application 레이어에 있는가](/posts/spring-boot-facade-transaction-architecture/) — Facade **위치**
- [Facade로 공용 로직을 뽑았더니 JOIN이 사라졌다](/posts/application-level-join-facade-reusability/) — Application-level **Join**
- [DomainService vs Facade — 경계를 정한 과정](/posts/domain-service-vs-facade-boundary/) — 로직의 **경계**
- 이 글 — 쿼리 복잡도에 따른 Repository **전략**
