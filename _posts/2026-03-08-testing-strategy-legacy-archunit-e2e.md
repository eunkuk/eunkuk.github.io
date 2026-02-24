---
title: "테스트 코드 때문에 개같았던 경험 — 레거시 환경, ArchUnit, E2E 전환기"
date: 2026-03-08 22:00:00 +0900
categories: [Dev, Testing]
tags: [spring-boot, testing, tdd, archunit, e2e-test, mybatis, h2, legacy, retrospective]
description: "MyBatis + JPA 혼합 환경에서 H2 테스트가 깨지고, Controller가 몰래 계층을 위반하고, MockMvc 테스트는 아무것도 못 잡았다. 삽질 끝에 정착한 테스트 전략을 기록한다."
---

> MyBatis + JPA 혼합 환경에서 H2 테스트가 깨지고, Controller가 몰래 계층을 위반하고, MockMvc 테스트는 아무것도 못 잡았다. 삽질 끝에 정착한 테스트 전략을 기록한다.

---

## 1. 상황

> 이 글은 [DomainService vs Facade 경계](/posts/domain-service-vs-facade-boundary/)와 [Rich Domain Model 타협기](/posts/rich-domain-model-pragmatic-compromises/)의 연장이다. 계층을 나누고, 도메인 로직을 분리했다면 — 그걸 **테스트로 어떻게 지키느냐**가 다음 문제였다.

프로젝트 구조는 이렇다.

```
Controller → Application Service → Domain Service → Repository
                                        ↑
                                   비즈니스 로직
```

레거시 시스템(PHP)에서 마이그레이션 중이라 JPA 엔티티와 MyBatis로 조회하는 레거시 테이블이 공존했다. 테스트 환경은 H2 인메모리 DB.

처음에는 "테스트 작성하면 되지 뭐가 어렵겠어"라고 생각했다. 실제로는 세 가지 문제가 동시에 터졌다.

1. **H2에서 MyBatis 쿼리가 전부 실패** — 레거시 테이블이 없으니까
2. **계층 위반을 아무도 모르게 저지름** — Controller에서 DomainService 직접 호출
3. **`@WebMvcTest` 테스트가 아무것도 못 잡음** — Mock으로 다 가려버리니까

---

## 2. 문제 1 — MyBatis + JPA 혼합 환경에서 H2 테스트 깨짐

### 증상

```java
@SpringBootTest
@ActiveProfiles("test")
class DashboardE2ETest {

    @Test
    void 대시보드_조회() throws Exception {
        mockMvc.perform(get("/api/v0/dashboard"))
            .andExpect(status().isOk());  // 500 터짐
    }
}
```

500 에러. 로그를 보면:

```
org.apache.ibatis.binding.BindingException:
  Invalid bound statement (not found):
  com.example.dashboard.mapper.DashboardMyBatisMapper.countPosts
```

### 원인

JPA의 `ddl-auto: create`는 `@Entity`가 붙은 테이블만 생성한다. MyBatis가 쿼리하는 레거시 테이블(`cb_post`, `cb_comment` 등)은 JPA가 모르는 테이블이다. H2에 존재하지 않으니 당연히 실패한다.

거기에 하나 더. `application-test.yml`에 MyBatis 매퍼 경로 설정이 빠져 있었다.

```yaml
# application-local.yml에는 있는데
mybatis:
  mapper-locations: classpath:mybatis/**/*.xml

# application-test.yml에는 없었다
# → MyBatis가 XML 매퍼를 아예 로딩하지 않음
# → BindingException: Invalid bound statement
```

이건 진짜 짜증나는 유형의 버그다. 로컬에서는 잘 되고, 테스트에서만 터진다. 원인은 설정 파일 누락.

### 해결

**첫 번째**: `application-test.yml`에 MyBatis 설정 추가.

```yaml
# application-test.yml
mybatis:
  mapper-locations: classpath:mybatis/**/*.xml
```

**두 번째**: 레거시 테이블 최소 스키마 SQL 작성.

```sql
-- src/test/resources/sql/legacy-schema.sql
CREATE TABLE IF NOT EXISTS cb_post (
    post_id       BIGINT AUTO_INCREMENT PRIMARY KEY,
    board_id      BIGINT,
    post_title    VARCHAR(255),
    post_nickname VARCHAR(100),
    post_datetime TIMESTAMP,
    post_del      INT DEFAULT 0
);

CREATE TABLE IF NOT EXISTS cb_comment (
    cmt_id       BIGINT AUTO_INCREMENT PRIMARY KEY,
    post_id      BIGINT,
    cmt_content  TEXT,
    cmt_nickname VARCHAR(100),
    cmt_datetime TIMESTAMP,
    cmt_del      INT DEFAULT 0
);
```

**세 번째**: E2E 테스트에 `@Sql`로 스키마 주입.

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Transactional
@Sql(scripts = "/sql/legacy-schema.sql",
     executionPhase = Sql.ExecutionPhase.BEFORE_TEST_CLASS)
class DashboardE2ETest {
    // 이제 MyBatis 쿼리가 동작한다
}
```

### 교훈

> **프로파일별 설정 파일은 "빠진 것"이 문제다.** 있는 것은 눈에 보이지만, 없는 것은 에러 메시지를 역추적해야 찾을 수 있다.

레거시 마이그레이션 중인 프로젝트라면 테스트 프로파일 설정을 운영 프로파일과 diff 해보는 습관이 필요하다.

```bash
diff application-local.yml application-test.yml
```

---

## 3. 문제 2 — Controller가 DomainService를 직접 호출

### 증상

코드 리뷰 없이 들어간 커밋. 대시보드 Controller가 이렇게 되어 있었다.

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/v0/dashboard")
public class DashboardController {

    private final OrderDomainService orderDomainService;  // 계층 위반
    private final DashboardMyBatisMapper mybatisMapper;

    @GetMapping
    public ApiResponse<DashboardResponse> index() {
        long orderCount = orderDomainService.countAll();  // Controller → DomainService 직접
        long postCount = mybatisMapper.countPosts();
        // ...
    }
}
```

문제는 이게 **잘 동작한다**는 것이다. 컴파일도 되고, 런타임 에러도 없다. 코드 리뷰에서도 놓칠 수 있다. "돌아가니까 괜찮겠지"로 넘어간다.

하지만 이러면 Controller가 비즈니스 로직을 알게 된다. 나중에 같은 로직을 다른 Controller에서도 쓰고 싶으면? 복붙한다. 그러면 중복이 생기고, 수정할 때 한 곳만 고치고 나머지를 놓친다.

### ArchUnit으로 강제하기

[ArchUnit](https://www.archunit.org/)은 아키텍처 규칙을 **테스트 코드로 검증**하는 라이브러리다.

```groovy
// build.gradle
testImplementation 'com.tngtech.archunit:archunit-junit5:1.3.0'
```

```java
@AnalyzeClasses(packages = "com.example",
                importOptions = ImportOption.DoNotIncludeTests.class)
class ArchitectureTest {

    @ArchTest
    static final ArchRule controller_should_not_depend_on_domain_service =
        noClasses()
            .that().resideInAPackage("..controller..")
            .should().dependOnClassesThat()
            .resideInAPackage("..domain..service..");

    @ArchTest
    static final ArchRule controller_should_only_use_application_service =
        classes()
            .that().resideInAPackage("..controller..")
            .should().onlyDependOnClassesThat()
            .resideInAnyPackage(
                "..controller..",
                "..service..",        // Application Service
                "..dto..",
                "..response..",
                "java..",
                "jakarta..",
                "org.springframework..",
                "io.swagger..",
                "lombok.."
            );
}
```

이 테스트를 추가하고 `./gradlew test`를 돌리면:

```
Architecture Violation [Priority: MEDIUM]
Rule 'no classes that reside in a package '..controller..'
should depend on classes that reside in a package '..domain..service..''
was violated (1 times):

DashboardController depends on OrderDomainService
in (DashboardController.java:0)
```

**컴파일은 되지만 테스트는 실패한다.** 이게 핵심이다.

### 수정

Application Service를 만들어서 중간 계층을 넣는다.

```java
// Before: Controller → DomainService (위반)
// After:  Controller → DashboardService → DomainService (정상)

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class DashboardService {

    private final OrderDomainService orderDomainService;
    private final DashboardMyBatisMapper mybatisMapper;
    private final DashboardMapper mapper;

    public DashboardResponse getDashboardData() {
        long orderCount = orderDomainService.countAll();
        long postCount = mybatisMapper.countPosts();
        // ... 조합 로직
        return new DashboardResponse(orderCount, postCount, ...);
    }
}
```

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/v0/dashboard")
public class DashboardController {

    private final DashboardService dashboardService;  // Application Service만 의존

    @GetMapping
    public ApiResponse<DashboardResponse> index() {
        return ApiResponse.success(dashboardService.getDashboardData());
    }
}
```

Controller는 얇아지고, 비즈니스 로직은 Service에 모인다. ArchUnit 테스트도 통과한다.

### ArchUnit을 쓰면 좋은 규칙들

```java
// Repository는 domain 패키지 안에서만 접근
@ArchTest
static final ArchRule repository_access =
    noClasses()
        .that().resideOutsideOfPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAPackage("..domain..repository..");

// 순환 의존 금지
@ArchTest
static final ArchRule no_cycles =
    slices().matching("com.example.(*)..").should().beFreeOfCycles();

// Entity는 Controller 패키지에서 직접 사용 금지
@ArchTest
static final ArchRule no_entity_in_controller =
    noClasses()
        .that().resideInAPackage("..controller..")
        .should().dependOnClassesThat()
        .resideInAPackage("..entity..");
```

이걸 한번 세팅해두면 **CI에서 자동으로 계층 위반을 잡는다.** 코드 리뷰에서 눈으로 찾을 필요가 없다.

---

## 4. 문제 3 — `@WebMvcTest`가 아무것도 못 잡는다

### `@WebMvcTest`의 한계

처음에는 Controller 테스트를 이렇게 작성했다.

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private OrderService orderService;

    @Test
    void 주문_조회() throws Exception {
        // Service를 Mock으로 대체
        when(orderService.getOrder(1L))
            .thenReturn(new OrderResponse(1L, "상품A", 10000));

        mockMvc.perform(get("/api/v0/orders/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.orderId").value(1));
    }
}
```

이 테스트는 **항상 성공한다.** Service를 Mock으로 대체했으니까. 실제 Service가 어떤 에러를 던지든, DB 쿼리가 실패하든, Mock이 다 가려버린다.

이러면 뭘 테스트하는 건가? Controller의 `@GetMapping` 경로가 맞는지? `jsonPath`가 맞는지? 그 정도다. **비즈니스 로직은 전혀 검증하지 못한다.**

더 큰 문제는 이것이다:

```java
// Security 설정이 실제와 다르다
@WebMvcTest  // Security 자동 설정이 부분적으로만 로딩됨

// Filter chain이 실제와 다르다
// JWT 인증 필터, 권한 체크 — 다 빠져있다

// 실제 에러 핸들링이 동작하지 않는다
// GlobalExceptionHandler가 로딩되지 않을 수 있다
```

결국 `@WebMvcTest`로 테스트한 건 **가짜 환경에서의 가짜 성공**이었다.

### E2E 테스트로 전환

`@SpringBootTest`로 전체 컨텍스트를 띄우고, 실제 DB(H2)를 사용하고, 실제 인증 토큰을 발급받아서 테스트한다.

```java
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Transactional
class OrderControllerE2ETest {

    @Autowired private MockMvc mockMvc;
    @Autowired private ObjectMapper objectMapper;
    @Autowired private MemberRepository memberRepository;
    @Autowired private OrderRepository orderRepository;

    private String accessToken;

    @BeforeEach
    void setUp() throws Exception {
        // 실제 회원 생성
        Member member = Member.create("testuser", "test@test.com", ...);
        memberRepository.save(member);

        // 실제 로그인해서 토큰 발급
        String loginResponse = mockMvc.perform(post("/api/v0/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"userId": "testuser", "password": "password123"}
                """))
            .andReturn().getResponse().getContentAsString();

        accessToken = objectMapper.readTree(loginResponse)
            .get("data").get("accessToken").asText();
    }

    @Test
    void 주문_조회_성공() throws Exception {
        // 실제 데이터 생성
        Order order = Order.create(1L, "상품A", 10000);
        orderRepository.save(order);

        // 실제 인증 토큰으로 요청
        mockMvc.perform(get("/api/v0/orders/" + order.getId())
                .header("Authorization", "Bearer " + accessToken))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value("0000"))
            .andExpect(jsonPath("$.data.orderId").value(order.getId()));
    }

    @Test
    void 인증_없이_요청하면_401() throws Exception {
        mockMvc.perform(get("/api/v0/orders/1"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    void 존재하지_않는_주문_조회시_404() throws Exception {
        mockMvc.perform(get("/api/v0/orders/99999")
                .header("Authorization", "Bearer " + accessToken))
            .andExpect(status().isNotFound());
    }
}
```

### 뭐가 다른가

| 항목 | `@WebMvcTest` | `@SpringBootTest` E2E |
|------|---------------|----------------------|
| Spring Context | Controller + 일부만 | 전체 로딩 |
| DB | 없음 (Mock) | H2 인메모리 |
| Security | 부분 로딩 | 실제 필터 체인 |
| Service 로직 | Mock | 실제 실행 |
| 검증 범위 | 라우팅 정도 | 전체 흐름 |
| 버그 발견 | 거의 못함 | 실제 버그 발견 |

E2E 테스트가 느리다는 단점은 있다. `@SpringBootTest`는 전체 컨텍스트를 띄우니까. 하지만 **가짜 성공보다는 느린 진짜 검증이 낫다.**

---

## 5. 최종 테스트 전략

삽질 끝에 정착한 구조다.

```
테스트 피라미드 (우리 버전)

         ┌─────────┐
         │  E2E    │  ← API 전체 흐름 검증
         │ 테스트   │    실제 DB, 실제 인증
         ├─────────┤
         │ App     │  ← Service 조합 로직 검증
         │ Service │    Mock 사용 (DomainService Mock)
         │ 테스트   │
         ├─────────┤
         │ Domain  │  ← 비즈니스 로직 단위 테스트
         │ Service │    순수 로직, Mock 최소화
         │ 테스트   │
         ├─────────┤
         │ArchUnit │  ← 계층 규칙 자동 검증
         └─────────┘
```

### 각 계층별 역할

**Domain Service 테스트** — 비즈니스 로직 검증. 이게 핵심이다.

```java
class OrderDomainServiceTest {

    @Mock private OrderRepository orderRepository;
    @InjectMocks private OrderDomainService orderDomainService;

    @Test
    void 만료된_주문은_취소할_수_없다() {
        Order expiredOrder = createOrder(OrderStatus.EXPIRED);
        when(orderRepository.findById(1L)).thenReturn(Optional.of(expiredOrder));

        assertThatThrownBy(() -> orderDomainService.cancel(1L))
            .isInstanceOf(DomainException.class)
            .hasFieldOrPropertyWithValue("errorCode", DomainErrorCode.ORDER_ALREADY_EXPIRED);
    }

    @Test
    void 정상_주문_취소_성공() {
        Order activeOrder = createOrder(OrderStatus.ACTIVE);
        when(orderRepository.findById(1L)).thenReturn(Optional.of(activeOrder));

        orderDomainService.cancel(1L);

        assertThat(activeOrder.getStatus()).isEqualTo(OrderStatus.CANCELLED);
    }
}
```

**Application Service 테스트** — 조합 로직과 DTO 변환 검증.

```java
class OrderAppServiceTest {

    @Mock private OrderDomainService orderDomainService;
    @Mock private NotificationService notificationService;
    @InjectMocks private OrderAppService orderAppService;

    @Test
    void 주문_취소시_알림_발송() {
        Order order = createOrder(OrderStatus.ACTIVE);
        when(orderDomainService.getById(1L)).thenReturn(order);

        orderAppService.cancelOrder(1L);

        verify(orderDomainService).cancel(1L);
        verify(notificationService).sendCancelNotification(1L);
    }
}
```

**E2E 테스트** — 전체 흐름 통합 검증. 위에서 본 것.

**ArchUnit** — 계층 규칙 자동 검증. CI에서 돌린다.

### 금지한 것

```
금지: @WebMvcTest + @MockBean 조합
이유: 가짜 환경에서 가짜 성공을 만들어냄
대안: E2E 테스트로 전체 흐름 검증
```

---

## 6. 돌아보며

처음에는 테스트를 "CI 통과용"으로 생각했다. 빨간불 안 뜨면 되는 거 아닌가. 그래서 `@WebMvcTest`로 Mock 떡칠해서 전부 녹색으로 만들었다. 테스트 커버리지도 올라갔다. 뿌듯했다.

그런데 실제 배포하면 터졌다. 테스트에서는 잡지 못한 버그들. Security 필터가 빠져서 인증 안 타는 것, MyBatis 매퍼가 로딩 안 되는 것, 계층 위반으로 중복 로직이 생기는 것.

테스트가 "진짜 동작을 검증"하려면 가짜를 최소화해야 했다. Mock은 DomainService 단위 테스트에서만 쓰고, 나머지는 실제 환경에 가깝게 돌려야 했다. 느리더라도.

정리하면:

| 삽질 | 해결 |
|------|------|
| H2에서 MyBatis 테이블 없음 | `@Sql`로 레거시 스키마 주입 |
| 테스트 프로파일 설정 누락 | 운영 프로파일과 diff |
| Controller 계층 위반 | ArchUnit으로 자동 검증 |
| `@WebMvcTest` 가짜 성공 | E2E 테스트로 전환 |

**테스트는 "통과하기 위해" 쓰는 게 아니라 "실패하기 위해" 쓰는 것이다.** 진짜 문제가 있을 때 실패해주는 테스트가 좋은 테스트다.

---

## 참고

- [ArchUnit 공식 문서](https://www.archunit.org/userguide/html/000_Index.html)
- [Spring Boot Testing — 공식 가이드](https://docs.spring.io/spring-boot/reference/testing/index.html)
- [DomainService vs Facade 경계](/posts/domain-service-vs-facade-boundary/)
- [Rich Domain Model 타협기](/posts/rich-domain-model-pragmatic-compromises/)
