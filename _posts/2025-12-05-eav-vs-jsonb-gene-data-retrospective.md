---
title: "유전자검사 결과 저장: JSON → EAV 마이그레이션 회고"
date: 2025-12-05 22:00:00 +0900
categories: [Dev, Database]
tags: [eav, jsonb, mariadb, postgresql, database-design, spring-boot, jpa, migration]
description: "유전자검사 서비스에서 W-GRS 점수를 JSON 컬럼에서 EAV 패턴으로 전환한 경험을 기록한다. PostgreSQL JSONB도 검토했지만, MariaDB + EAV를 선택한 이유와 그 결과를 정리한다."
---

> 유전자검사 서비스에서 W-GRS 점수를 JSON 컬럼에서 EAV 패턴으로 전환한 경험을 기록한다. PostgreSQL JSONB도 검토했지만, MariaDB + EAV를 선택한 이유와 그 결과를 정리한다.

---

## 1. 상황

유전자검사 서비스를 만들고 있다. 검체에서 추출한 유전자 데이터(callData)를 가중치 정보(WeightedInfo)와 조합해 W-GRS 점수를 계산하고, 그 결과를 저장해야 한다.

처음에는 검사 결과를 `gene` 테이블의 JSON 컬럼 하나에 통째로 넣었다.

```
gene 테이블
┌──────────┬──────────────┬──────────────────────────────────────────┐
│ gene_id  │ call_data    │ results (JSON)                           │
├──────────┼──────────────┼──────────────────────────────────────────┤
│ 1        │ {...}        │ {"circle_type1":"2.34","vitamin_d":"1.8"}│
│ 2        │ {...}        │ {"circle_type1":"1.56","man_type1":"3.2"}│
└──────────┴──────────────┴──────────────────────────────────────────┘
```

검사 항목이 5~6개였을 때는 이걸로 충분했다.

---

## 2. JSON 컬럼의 한계

항목이 20개를 넘어가면서 문제가 보이기 시작했다.

### 부분 조회가 안 된다

특정 항목만 꺼내려면 `JSON_EXTRACT`를 써야 하는데, MariaDB에서는 JSON 내부 필드에 인덱스를 걸 수 없다.

```sql
-- vitamin_d 결과만 조회하고 싶다
SELECT JSON_EXTRACT(results, '$.vitamin_d') FROM gene WHERE gene_id = 1;
-- 인덱스 활용 불가, 풀스캔
```

### 부분 업데이트가 안 된다

한 항목만 바꾸려고 해도 전체 JSON을 읽고, 수정하고, 다시 저장해야 한다.

```java
// JSON 컬럼 시절의 업데이트
String json = gene.getResults();                    // 1. 전체 읽기
Map<String, String> map = objectMapper.readValue(json, ...); // 2. 파싱
map.put("vitamin_d", "2.1");                        // 3. 한 필드 수정
gene.setResults(objectMapper.writeValueAsString(map));       // 4. 직렬화
geneRepository.save(gene);                          // 5. 전체 저장
```

5단계. 한 필드 바꾸는 데 5단계.

### 스키마가 불투명하다

JPA 엔티티에서 results 필드는 그냥 `String`이다. 어떤 키가 들어있는지, 값이 숫자인지 문자인지 컴파일러가 모른다. 필드명에 오타를 내도 런타임까지 발견할 수 없다.

### 플랜별 동적 항목

가장 큰 문제는 플랜마다 검사 항목이 다르다는 점이었다. PRO 플랜은 26개 항목, BASIC은 12개. JSON 스키마를 고정할 수가 없다. 어느 플랜에서 어떤 키를 쓰는지를 JSON 자체만으로는 알 수 없다.

---

## 3. 선택지: Wide Table vs JSONB vs EAV

세 가지 방식을 검토했다.

### Wide Table (컬럼형)

```sql
CREATE TABLE gene_result (
    gene_id      BIGINT,
    circle_type1 VARCHAR(50),
    vitamin_d    VARCHAR(50),
    man_type1    VARCHAR(50),
    ...26개 컬럼
);
```

**장점**: 직관적이다. 각 컬럼에 타입과 인덱스를 걸 수 있다. SQL 조회가 자연스럽다.

**단점**: 항목 하나 추가할 때마다 `ALTER TABLE ADD COLUMN`. 플랜별로 사용하는 항목이 다르니 대부분의 컬럼이 NULL로 채워진다. 검사 항목이 계속 늘어나는 서비스에서 이건 지속 불가능하다.

### PostgreSQL JSONB

```sql
CREATE TABLE gene_result (
    gene_id BIGINT,
    results JSONB
);

-- GIN 인덱스로 내부 필드 검색 가능
CREATE INDEX idx_results ON gene_result USING gin(results);

-- 부분 조회
SELECT results->>'vitamin_d' FROM gene_result WHERE gene_id = 1;

-- 조건 검색
SELECT * FROM gene_result WHERE results @> '{"vitamin_d": "2.1"}';
```

**장점**: 유연한 스키마 + 인덱싱 + 부분 조회. JSON의 편리함과 관계형의 성능을 둘 다 가져간다. `jsonb_each()`로 피벗도 가능하다.

**단점**: 우리 DB는 MariaDB다. PostgreSQL로 전환해야 한다.

### EAV (Entity-Attribute-Value)

```sql
CREATE TABLE gene_result (
    result_id       BIGINT AUTO_INCREMENT,
    gene_id         BIGINT NOT NULL,
    attribute_key   VARCHAR(100) NOT NULL,
    attribute_value VARCHAR(255),
    created_at      DATETIME
);
```

**장점**: MariaDB 그대로 쓸 수 있다. 항목 추가에 DDL 변경이 없다. NULL인 항목은 행 자체가 없으니 공간 낭비가 없다.

**단점**: 행 수가 늘어난다. SQL만으로 피벗 쿼리가 어렵다. 앱에서 `List → Map` 변환을 해야 한다.

---

## 4. 왜 EAV를 선택했나

### MariaDB를 유지해야 했다

이미 운영 중인 MariaDB 인프라가 있다. PostgreSQL로 바꾸려면:

- DB 마이그레이션 스크립트 작성
- 기존 쿼리 전수 검증 (MariaDB 방언 → PostgreSQL 방언)
- 운영 환경 재구성 (백업, 모니터링, 커넥션 풀)
- MyBatis 레거시 쿼리까지 전부 재검증

1인 개발에서 이건 "결과 저장 방식 개선"이 아니라 **"인프라 이관 프로젝트"**다. 목적에 비해 비용이 너무 크다.

### 조회 패턴이 단순하다

우리 서비스의 검사 결과 조회 패턴은 하나다: **gene_id로 해당 검사의 전체 결과를 한 번에 가져온다.**

```java
List<GeneResult> results = geneResultRepository.findByGeneId(geneId);
```

"vitamin_d만 조회"하거나 "특정 조건으로 필터링"하는 일이 없다. JSONB의 GIN 인덱스나 `@>` 연산자가 빛나는 건 부분 조회가 빈번할 때인데, 우리는 항상 전체를 가져간다. JSONB의 핵심 장점이 우리 유스케이스에서는 의미가 없었다.

### NULL 미저장 = 공간 효율

PRO 플랜은 26개 항목이지만, 실제로 결과가 산출되는 항목은 검체에 따라 다르다. Wide Table이었다면 결과 없는 항목도 NULL 컬럼으로 자리를 차지하지만, EAV에서는 결과가 없는 항목의 행 자체가 없다.

코드에서도 이걸 명시적으로 처리한다:

```java
// GeneResultDomainService.saveResults()
resultsMap.forEach((key, value) -> {
    if (value == null || value.isBlank()) {
        return; // NULL이면 행을 만들지 않는다
    }
    // ...
});
```

이 방식으로 **평균 62%의 저장 공간을 절약**했다.

### PlanGeneConfig로 EAV의 약점을 보완했다

EAV의 대표적인 약점은 "속성 정보가 없다"는 것이다. `attribute_key`가 `"circle_type1"`이라는 문자열일 뿐, 그게 뭔지, 어떤 순서로 보여줘야 하는지, 활성화된 항목인지 알 수 없다.

이걸 `plan_gene_config` 테이블로 해결했다:

```
plan_gene_config 테이블
┌───────────┬────────────────┬────────────────┬────────────┬──────────────┐
│ plan_id   │ attribute_key  │ attribute_name │ sort_order │ is_activated │
├───────────┼────────────────┼────────────────┼────────────┼──────────────┤
│ 1 (PRO)   │ circle_type1   │ 혈당 조절      │ 1          │ true         │
│ 1 (PRO)   │ vitamin_d      │ 비타민 D 농도  │ 2          │ true         │
│ 2 (BASIC) │ circle_type1   │ 혈당 조절      │ 1          │ true         │
└───────────┴────────────────┴────────────────┴────────────┴──────────────┘
```

속성의 메타데이터(이름, 정렬, 활성화, 계산 방식)를 별도 테이블에서 관리하니, EAV의 행 자체에는 순수한 key-value만 남긴다. 메타데이터 분리는 오히려 JSON 컬럼 시절보다 깔끔하다.

---

## 5. 전환 후 뭐가 달라졌나

### GeneResult 엔티티

```java
@Entity
@Table(name = "gene_result")
public class GeneResult {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "result_id")
    private Long id;

    @Column(name = "gene_id", nullable = false)
    private Long geneId;

    @Column(name = "attribute_key", nullable = false, length = 100)
    private String attributeKey;

    @Column(name = "attribute_value", length = 255)
    private String attributeValue;

    @CreationTimestamp
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    public static GeneResult create(Long geneId, String attributeKey, String attributeValue) {
        GeneResult result = new GeneResult();
        result.geneId = geneId;
        result.attributeKey = attributeKey;
        result.attributeValue = attributeValue;
        return result;
    }
}
```

JSON 컬럼이 사라지고, 한 행이 하나의 검사 항목 결과를 담는다. 정적 팩토리 메서드로 생성하고, `@DynamicInsert`/`@DynamicUpdate`로 변경된 필드만 쿼리에 포함한다.

### 조회: 한 번의 쿼리, Java Stream 변환

```java
// GeneResultDomainService
public Map<String, String> getResultsAsMap(Long geneId) {
    List<GeneResult> results = geneResultRepository.findByGeneId(geneId);
    return results.stream()
            .collect(Collectors.toMap(
                    GeneResult::getAttributeKey,
                    GeneResult::getAttributeValue,
                    (existing, replacement) -> replacement
            ));
}
```

`findByGeneId()` 한 번이면 된다. Spring Data JPA의 메서드 네이밍 쿼리. `SELECT * FROM gene_result WHERE gene_id = ?` — 이보다 단순한 SQL이 있을까. 결과를 `Map<String, String>`으로 변환하는 건 Java Stream이 해준다.

정렬이 필요하면 `PlanGeneConfig`의 `sortOrder`를 사용한다:

```java
// GeneResultDomainService
public Map<String, String> getResultsAsMapWithSort(Long geneId, Long planId) {
    List<GeneResult> results = geneResultRepository.findByGeneId(geneId);
    List<PlanGeneConfig> configs = planGeneConfigDomainService.getConfigsByPlanId(planId);

    Map<String, Integer> sortOrderMap = configs.stream()
            .collect(Collectors.toMap(
                    PlanGeneConfig::getAttributeKey,
                    PlanGeneConfig::getSortOrder,
                    (existing, replacement) -> existing
            ));

    return results.stream()
            .sorted((a, b) -> Integer.compare(
                    sortOrderMap.getOrDefault(a.getAttributeKey(), 999),
                    sortOrderMap.getOrDefault(b.getAttributeKey(), 999)
            ))
            .collect(LinkedHashMap::new,
                    (map, r) -> map.put(r.getAttributeKey(), r.getAttributeValue()),
                    LinkedHashMap::putAll);
}
```

`LinkedHashMap`을 써서 정렬 순서를 보존한다. SQL에서 `ORDER BY`를 쓰는 대신 Java에서 정렬하는 이유는, 정렬 기준이 다른 테이블(`plan_gene_config`)에 있기 때문이다.

### 저장: Upsert 로직으로 멱등성 보장

```java
// GeneResultDomainService
@Transactional
public void saveResults(Long geneId, Map<String, String> resultsMap) {
    List<GeneResult> existingResults = geneResultRepository.findByGeneId(geneId);
    Map<String, GeneResult> existingMap = existingResults.stream()
            .collect(Collectors.toMap(GeneResult::getAttributeKey, r -> r, (a, b) -> a));

    List<GeneResult> toSave = new ArrayList<>();

    resultsMap.forEach((key, value) -> {
        if (value == null || value.isBlank()) {
            return; // NULL 미저장
        }
        GeneResult result = existingMap.get(key);
        if (result != null) {
            result.updateValue(value.trim()); // 기존 행 업데이트
        } else {
            result = GeneResult.create(geneId, key, value.trim()); // 새 행 생성
        }
        toSave.add(result);
    });

    if (!toSave.isEmpty()) {
        geneResultRepository.saveAll(toSave); // 배치 저장
    }
}
```

기존 결과가 있으면 업데이트, 없으면 생성. `saveAll()`로 배치 처리해서 N+1을 방지한다. 재계산을 여러 번 돌려도 결과가 같다 — 멱등성.

### 동적 항목 관리: PlanGeneConfig

새 검사 항목을 추가하려면:

1. `plan_gene_config`에 행 추가 (attributeKey, 표시명, 정렬 순서)
2. `weighted_info`에 가중치 데이터 업로드
3. 끝.

`ALTER TABLE` 없음. 배포 없음. 관리자 페이지에서 설정만 바꾸면 된다.

`PlanGeneConfig`에는 단순 매핑 외에도 계산 방식을 정의하는 필드가 있다:

```java
// PlanGeneConfig — 계산 방식 필드
@Column(name = "cb_weighted_category", length = 100)
private String cbWeightedCategory;     // weighted_info.category와 매핑

@Column(name = "aggregation_type", length = 20)
private String aggregationType;        // NULL(직접), "SUM"(합계), "COUNT"(카운트)

@Column(name = "aggregate_source_keys", length = 500)
private String aggregateSourceKeys;    // SUM 대상 키 (예: "circle_type1t,circle_type1h")
```

이로써 "어떤 항목이 어떤 방식으로 계산되는가"까지 데이터로 관리한다. 계산 로직 변경에 코드 수정이 필요 없다.

### 등급 판정: PlanGeneGradeConfig

W-GRS 점수를 등급(매우 안심 ~ 매우 주의)으로 변환하는 임계값도 테이블로 관리한다:

```java
// PlanGeneGradeConfig — 5단계 등급 임계값
@Column(name = "grade_1_max") private BigDecimal grade1Max;
@Column(name = "grade_2_max") private BigDecimal grade2Max;
@Column(name = "grade_3_max") private BigDecimal grade3Max;
@Column(name = "grade_4_max") private BigDecimal grade4Max;
@Column(name = "is_reverse")  private Boolean isReverse;
```

- **기본 (낮을수록 좋음)**: `score ≤ grade1Max → 매우 안심`, `> grade4Max → 매우 주의`
- **역방향 (높을수록 좋음)**: 방향을 뒤집는다. `is_reverse = true`
- **3단계 항목**: `grade1Max`, `grade4Max`가 NULL이면 안심/보통/주의 3단계로 동작

플랜마다 같은 `attribute_key`에 다른 등급 기준을 적용할 수 있다. PRO와 BASIC에서 "혈당 조절"의 등급 커트라인이 다를 수 있고, 그걸 코드가 아니라 데이터로 처리한다.

### 계산 흐름 전체 그림

```
Gene.callData (원본 JSON)
    │
    ▼
GeneCalculationFacade.parseCallData()
    │  probesetId + type → callData 매핑
    ▼
buildGeneValueMap()
    │  WeightedInfo 조인
    │  위험 대립유전자 카운트
    │  가중치 곱셈 → weightedNum
    ▼
GeneCalculationRiskService.calculateScores()
    │  Pass 1: 직접 계산 (weighted sum / count)
    │  Pass 2: 집계 계산 (SUM of other keys)
    │  Pass 3: BigDecimal → String 직렬화
    ▼
Map<String, String> calculatedScores
    │
    ▼
GeneResultDomainService.saveResults()
    │  NULL 필터링, Upsert, 배치 저장
    ▼
gene_result 테이블 (EAV)
```

callData는 여전히 원본 JSON으로 `gene` 테이블에 보관한다. 원본을 건드리지 않기 때문에, 가중치가 업데이트되면 동일한 callData로 재계산할 수 있다.

---

## 6. 대신 포기한 것들

공짜 점심은 없다. EAV를 선택하면서 포기한 것들이 있다.

### SQL 레벨 분석

"vitamin_d 점수가 40 이상인 검사 건수"를 구하고 싶다면:

```sql
-- EAV: 가능하지만 불편하다
SELECT COUNT(DISTINCT gene_id)
FROM gene_result
WHERE attribute_key = 'vitamin_d'
  AND CAST(attribute_value AS DECIMAL) > 40;
```

`CAST`가 들어가는 순간 인덱스 활용이 어렵다. JSONB였다면:

```sql
-- JSONB: 더 자연스럽고, GIN 인덱스를 탈 수 있다
SELECT COUNT(*)
FROM gene_result
WHERE (results->>'vitamin_d')::numeric > 40;
```

Wide Table이었다면:

```sql
-- Wide Table: 가장 자연스럽다
SELECT COUNT(*) FROM gene_result WHERE vitamin_d > 40;
```

현재 이런 분석 쿼리가 필요하지 않기 때문에 수용했지만, 통계 대시보드가 필요해지면 별도 분석용 뷰나 배치를 만들어야 한다.

### 행 수 증가

검사 1건에 항목 26개면 EAV 행이 26개 생긴다.

```
검사 10,000건 기준:
- Wide Table: 10,000행
- EAV:        ~260,000행 (26항목 × 10,000건)
- JSONB:      10,000행
```

인덱스 크기와 조회 I/O가 증가한다. 다만 현재 서비스 규모에서 26만 행은 부담이 아니다. `gene_id`에 인덱스가 있으니 `findByGeneId()` 성능은 충분하다.

### 타입 안전성

`attribute_value`가 `VARCHAR(255)`. W-GRS 점수는 숫자지만 문자열로 저장된다. 타입 검증은 앱 레벨에서 해야 한다.

```java
// 문자열을 숫자로 쓰려면 매번 변환
BigDecimal score = new BigDecimal(result.getAttributeValue());
```

JSONB라면 숫자, 배열, 객체를 네이티브로 지원한다. Wide Table이라면 컬럼 타입 자체가 `DECIMAL`이다. EAV는 모든 값을 문자열로 평탄화하는 대가를 치른다.

---

## 7. EAV는 안티패턴인가

EAV는 데이터베이스 설계에서 대표적인 안티패턴으로 불린다. Bill Karwin의 *SQL Antipatterns*에서도 첫 번째 장에서 다루는 패턴이다. 이유는 명확하다.

- **타입 안전성 없음**: 모든 값이 `VARCHAR`에 저장된다. 숫자, 날짜, 불리언 구분이 없다. `CAST`를 쓰는 순간 인덱스가 무력화된다.
- **쿼리 복잡도**: 한 엔티티의 속성을 컬럼처럼 보려면 피벗 쿼리가 필요하다. `JOIN`을 속성 수만큼 반복하거나, 앱에서 `List → Map` 변환을 해야 한다.
- **FK 제약 불가**: `attribute_value`가 문자열이므로, 다른 테이블을 참조하는 외래키를 걸 수 없다. 참조 무결성을 DB가 보장하지 못한다.
- **스키마 검증 불가**: 어떤 엔티티에 어떤 속성이 있어야 하는지를 DB 스키마만으로는 알 수 없다. `NOT NULL` 제약도 걸 수 없다.

이런 이유로 "EAV는 쓰지 마라"는 조언이 나온다. 그리고 대부분의 경우 그 조언은 맞다.

### 하지만 안티패턴 ≠ 절대 금지

"안티패턴"의 원래 의미는 **"자주 반복되는 나쁜 패턴"**이지 "절대 쓰면 안 되는 패턴"이 아니다. 디자인 패턴이 모든 상황에서 좋은 패턴이 아니듯, 안티패턴도 모든 상황에서 나쁜 패턴이 아니다. 맥락이 중요하다.

EAV가 합리적인 선택이 되는 조건이 있다:

| 조건 | 우리 상황 |
|------|----------|
| DB 전환이 불가능하다 | MariaDB 유지 필수 ✓ |
| 속성이 동적으로 추가된다 | 검사 항목이 계속 늘어남 ✓ |
| 조회 패턴이 단순하다 | gene_id로 전체 조회만 함 ✓ |
| SQL 분석 쿼리가 거의 없다 | 현재 없음 ✓ |

이 네 가지가 동시에 성립할 때, EAV의 약점은 크게 줄어들고 유연성은 그대로 남는다. 핵심은 **약점을 인지하고 보완책을 갖추는 것**이다. 우리는 `PlanGeneConfig`라는 메타데이터 테이블로 속성 정보, 정렬, 계산 방식을 관리했고, 타입 검증은 애플리케이션 레벨에서 처리했다. 약점을 모르고 쓰는 것과, 약점을 알고 보완하며 쓰는 것은 다르다.

### 실제로 EAV가 쓰이는 곳

EAV가 안티패턴이라면서도 실전에서 광범위하게 쓰인다는 점은 아이러니다.

- **Magento (Adobe Commerce)**: 상품 속성을 EAV로 관리한다. 의류 상품의 "사이즈", "색상"과 전자제품의 "해상도", "배터리 용량"이 같은 테이블 구조에 공존한다. 수만 개의 상품이 각각 다른 속성 집합을 가져야 하는 e-commerce에서 EAV는 현실적인 선택이다.
- **WordPress**: `wp_postmeta` 테이블이 전형적인 EAV다. `meta_key`와 `meta_value`로 포스트에 임의의 메타데이터를 붙인다. 플러그인 생태계가 이 구조 위에서 작동한다.
- **의료 시스템**: HL7 FHIR 같은 의료 데이터 표준에서도 EAV 유사 패턴을 사용한다. 환자마다 기록해야 하는 항목이 다르고, 항목 자체가 계속 추가되기 때문이다.

이들의 공통점은 **속성 집합이 사전에 고정되지 않는다**는 것이다. 우리의 유전자검사 서비스도 같은 상황이었다.

---

## 8. 돌아보며

EAV는 "최고의 방법"이 아니다. "지금 이 상황에서 가장 현실적인 방법"이다.

기술적으로 우아한 해답은 PostgreSQL JSONB였다. 유연한 스키마, GIN 인덱싱, 부분 조회, 네이티브 타입 지원 — 모든 면에서 EAV보다 낫다. 하지만 그걸 쓰려면 DB를 통째로 바꿔야 한다. 운영 중인 MariaDB를 PostgreSQL로 전환하는 건 "결과 저장 방식 개선"의 범위를 한참 넘는다.

결국 의사결정의 핵심은 이거였다:

| 조건 | EAV 유리 | JSONB 유리 |
|------|---------|-----------|
| 현재 DB | MariaDB ✓ | PostgreSQL 전환 필요 |
| 조회 패턴 | 전체 조회 ✓ | 부분 조회/필터링 빈번 |
| 개발 인력 | 1인 ✓ | DB 전환을 감당할 팀 |
| 항목 변동 | 자주 추가됨 ✓ | 자주 추가됨 ✓ |
| SQL 분석 | 불필요 ✓ | SQL 분석 빈번 |

5개 조건 중 4개가 EAV를 가리켰다. JSONB가 유일하게 이기는 건 "항목이 자주 추가될 때"인데, 그건 EAV도 가능하다.

EAV의 약점 — 메타데이터 부재, 정렬 정보 없음, 타입 안전성 없음 — 은 `PlanGeneConfig`와 `PlanGeneGradeConfig`라는 별도 테이블로 보완했다. 속성 정의, 정렬 순서, 등급 임계값, 계산 방식까지 데이터로 관리한다. 코드 수정 없이 검사 항목을 추가하고, 등급 기준을 바꿀 수 있다.

> 기술 선택은 제약 조건 안에서의 최적화다. 제약이 없다면 항상 더 좋은 방법이 있다. 하지만 제약이 없는 프로젝트는 없다.

MariaDB를 유지하면서 동적 스키마를 확보하고, NULL 미저장으로 62%의 공간을 절약하고, 메타데이터 테이블로 EAV의 약점을 메꿨다. 완벽하지 않지만 돌아간다. 그리고 돌아가는 코드가 우아한 설계도보다 낫다.
