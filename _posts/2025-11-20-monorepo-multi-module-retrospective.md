---
title: "1인 개발자가 모노레포 + 6모듈 구조를 선택하기까지 — Spring Boot 멀티모듈 회고"
date: 2025-11-20 22:00:00 +0900
categories: [Dev, Architecture]
tags: [monorepo, multi-module, gradle, spring-boot, hexagonal-architecture]
description: "두 개의 프로젝트를 따로 운영하다가 모노레포 + 멀티모듈로 전환한 1인 개발자의 기록. 왜 합쳤고, 헥사고날 아키텍처를 어떻게 모듈 구조에 녹였는지."
---

## 상황

유전자검사 서비스를 만들고 있다. 개발자는 나 하나다.

처음에 만든 건 Service API였다. 사용자가 이용권을 등록하고, 검사 키트를 수령하고, 유전자검사 결과를 조회하는 REST API. 모바일 앱과 웹 클라이언트가 이 API를 호출하는 구조다. 하나의 Spring Boot 프로젝트로 시작했고, 서비스 자체는 잘 돌아갔다.

문제는 그다음이었다. 관리자용 Admin API가 필요해졌다. 이용권 발급, 키트 배송 관리, 유전자 데이터 업로드, 검사 결과 확정, 엑셀 다운로드 — Service와는 성격이 완전히 다른 애플리케이션이었다. Thymeleaf로 서버 사이드 렌더링을 해야 했고, Apache POI로 엑셀을 뽑아야 했고, 레거시 DB와 연동하기 위한 MyBatis 매퍼도 필요했다.

별도의 프로젝트로 Admin을 만들었다. 당연한 선택이었다. Service와 Admin은 다른 애플리케이션이니까.

## 두 프로젝트 동시 운영

처음에는 괜찮았다. Service는 Service대로, Admin은 Admin대로 개발하면 됐다.

수정사항이 나오기 시작하면서 문제가 보였다.

### 한 번의 변경, 두 번의 작업

유전자검사 도메인은 하나의 검사 흐름이 양쪽 애플리케이션을 관통한다. Admin에서 이용권을 발급하고, Service에서 사용자가 등록하고, Admin에서 검사 결과를 업로드하고, Service에서 결과를 조회한다.

`Gene` 엔티티에 필드가 추가되면 양쪽 프로젝트에서 동시에 수정해야 한다. `Voucher`의 상태 전이 로직이 바뀌면 양쪽에서 동시에 수정해야 한다. Repository 쿼리가 바뀌면 양쪽에서 동시에 수정해야 한다.

```java
// service 프로젝트의 Gene.java
@Entity
public class Gene {
    private Long id;
    private String vouKitBarcode;
    private Integer testState;    // 0=대기, 1=진행, 2=완료
    private Integer qcState;      // 0=대기, 1=승인
    private Integer confirmed;    // 0=대기, 1=확정
    private String callData;      // SNP 원시 데이터 (JSON)
    // ...
}

// admin 프로젝트의 Gene.java — 똑같은 코드
@Entity
public class Gene {
    private Long id;
    private String vouKitBarcode;
    private Integer testState;
    private Integer qcState;
    private Integer confirmed;
    private String callData;
    // ...
}
```

엔티티가 5개였다면 참을 수 있었을 것이다. 그런데 엔티티가 늘어나면서 양쪽에 복사해놓은 코드가 점점 많아졌다. 수정할 때마다 두 프로젝트를 열고, 같은 파일을 찾아서, 같은 수정을 하고, 각각 커밋하고, 각각 배포했다.

### 1인 개발의 오버헤드

팀이었다면 "Admin은 A가, Service는 B가 담당"하는 식으로 분담할 수 있다. 하지만 나 혼자 양쪽 다 하고 있으니 모든 오버헤드가 두 배다.

| 작업 | 멀티레포에서 실제로 해야 했던 일 |
|------|------|
| 엔티티 필드 추가 | 양쪽 프로젝트에서 같은 수정, 각각 커밋 |
| 도메인 로직 버그 수정 | 양쪽에서 같은 패치, 배포 순서도 신경 |
| 공통 의존성 업그레이드 | 양쪽에서 각각 버전 올리고 호환성 확인 |
| 새 엔티티 추가 | Entity, Repository, Service 전부 양쪽에 복사 |

유전자검사 결과는 틀리면 안 되는 데이터다. `Gene` 엔티티의 상태 전이 — 검사 시작 → 검사 완료 → QC 승인 → 최종 확정 — 이 로직에 버그가 있어서 한쪽만 수정하고 다른 쪽을 까먹으면, Admin에서는 QC 승인이 되는데 Service에서는 이전 로직으로 동작하는 상황이 올 수 있다.

> 멀티레포의 장점은 팀이 클 때 극대화된다. 팀 간 독립성, 책임 분리, 독립 배포 주기가 의미를 가지려면 각 레포를 담당하는 사람이 따로 있어야 한다. 1인 개발자에게 멀티레포는 "똑같은 일을 두 번 하는 구조"였다.

### 공유 라이브러리라는 선택지

엔티티와 공통 로직을 별도의 라이브러리로 만들어서 private Maven 또는 Gradle 저장소에 올리고, 양쪽 프로젝트에서 의존성으로 가져오는 방법도 있다. 이론적으로는 맞지만, 1인 개발자가 이걸 유지하려면:

| 해야 하는 일 | 빈도 |
|------------|------|
| 공유 라이브러리 버전 올리기 | 엔티티 변경할 때마다 |
| 양쪽 프로젝트에서 버전 업데이트 | 라이브러리 올릴 때마다 |
| private 저장소 유지보수 | 항상 |
| 라이브러리 호환성 테스트 | 매 릴리즈마다 |

배보다 배꼽이다. 두 프로젝트를 동시에 수정하는 것보다 나을 게 없었다.

## 모노레포를 구상하다

뭔가 더 나은 방법이 필요했다. 정리해보면 문제의 핵심은 세 가지였다.

1. **코드 중복** — 같은 엔티티, Repository, 도메인 로직이 두 곳에 존재한다
2. **동기화 부담** — 하나를 고치면 다른 하나도 반드시 맞춰야 한다
3. **경계 부재** — 그렇다고 하나의 프로젝트에 합치면 Admin과 Service의 의존성이 섞인다

하나의 Git 레포지토리 안에서 코드를 공유하되, Admin과 Service의 경계는 유지할 수 있는 구조. **모노레포 + Gradle 멀티모듈**이 그 답이었다.

## 헥사고날 아키텍처를 모듈에 녹이다

모노레포로 합치기로 했으면 모듈 구조를 어떻게 나눌 것인지가 다음 문제다. 아무렇게나 나누면 나중에 또 뒤섞인다.

헥사고날 아키텍처(포트 앤 어댑터)의 핵심 원칙을 차용했다. **도메인이 중심에 있고, 인프라와 애플리케이션은 도메인에 의존하되 도메인은 바깥을 모른다.** 이 방향성을 Gradle 모듈로 물리적으로 강제한 것이다.

```groovy
rootProject.name = 'project-api'

include 'project-common'
include 'project-infra'
include 'project-domain'
include 'project-admin-be'
include 'project-service-be'
include 'project-app-common'
```

6개 모듈. 의존 관계는 안쪽에서 바깥쪽으로 흐른다.

```
project-common          ← 유틸리티, 예외 기본 클래스 (Spring 의존 없음)
       ↓
project-domain          ← 엔티티, Repository, DomainService (핵심 도메인)
       ↓
project-infra           ← 보안, Swagger, 캐시, 메일/PDF, API 응답 (인프라 어댑터)
       ↓
project-app-common      ← Facade, 공통 DTO (애플리케이션 서비스, 유스케이스 조율)
       ↓
project-admin-be        ← 관리자 API (인바운드 어댑터: Thymeleaf, POI)
project-service-be      ← 사용자 API (인바운드 어댑터: 순수 REST)
```

헥사고날의 용어로 대응시키면 이렇다.

| 헥사고날 계층 | 모듈 | 역할 |
|------------|------|------|
| 도메인 코어 | `common` + `domain` | 엔티티, 비즈니스 규칙, DomainService |
| 포트 (유스케이스) | `app-common` | Facade가 유스케이스를 정의, Admin·Service 공유 |
| 인프라 어댑터 | `infra` | JWT, 캐시, 메일, PDF, FCM 등 외부 연동 |
| 인바운드 어댑터 | `admin-be`, `service-be` | HTTP 컨트롤러, 뷰 렌더링 |

핵심은 `project-domain`이 `project-infra`나 `project-admin-be`를 모른다는 점이다. 도메인 로직에 인프라 코드가 스며드는 걸 Gradle 의존성이 컴파일 타임에 막아준다. 패키지 컨벤션이 아니라 빌드 시스템이 아키텍처를 강제한다.

> Gradle 모듈 분리는 헥사고날 아키텍처의 의존성 방향을 "규칙"이 아닌 "강제"로 만드는 장치다. 도메인 모듈에서 인프라를 import하면 컴파일 에러가 난다. 의지가 아니라 빌드가 지켜준다.

## 전환 후 뭐가 달라졌나

### 엔티티와 Repository: 한 곳에서 관리

두 프로젝트에 흩어져 있던 엔티티와 Repository를 전부 `project-domain`으로 모았다. 이제 Admin에서도 Service에서도 이 모듈을 의존성으로 가져다 쓸 뿐이다.

```groovy
// admin-be/build.gradle
dependencies {
    implementation project(':project-domain')
    implementation project(':project-infra')
    implementation project(':project-app-common')
    // Admin 전용
    implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
    implementation 'org.apache.poi:poi-ooxml:5.2.3'
}
```

```groovy
// service-be/build.gradle
dependencies {
    implementation project(':project-domain')
    implementation project(':project-infra')
    implementation project(':project-app-common')
    // Thymeleaf? POI? 없다. 필요 없으니까.
}
```

`Gene` 엔티티의 상태 전이 로직을 수정하면 `project-domain`에서 한 번만 고치면 된다. 이전처럼 두 프로젝트를 열고 같은 파일을 찾아서 같은 수정을 하는 일이 사라졌다.

### Facade: 두 프로젝트에 복사하던 로직의 한 곳

유전자검사 서비스의 핵심 흐름 — 이용권 발급, 키트 배송, 검사 결과 조회 — 은 Admin과 Service에서 모두 필요한 로직이다. 이전에는 양쪽 프로젝트에 비슷한 서비스 클래스가 각각 있었다. 이제는 `project-app-common`의 Facade 하나로 모았다.

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class GenePlanFacade {

    private final GeneDomainService geneDomainService;
    private final GeneResultDomainService geneResultDomainService;
    private final VoucherKitDomainService voucherKitDomainService;
    private final VoucherPlanDomainService voucherPlanDomainService;

    public GeneResultResponse getGeneByVoucherId(Long voucherId) {
        // Voucher → VoucherKit(barcode) → Gene → GeneResult(W-GRS 점수)
        VoucherKit kit = voucherKitDomainService.getLatestKit(voucherId);
        Gene gene = geneDomainService.getByBarcode(kit.getBarcode());
        Map<String, String> wgrsScores = geneResultDomainService.getResultMap(gene.getId());
        // ... 공통 결과 조회 로직
    }
}
```

Admin에서 검사 결과를 확인할 때도, Service에서 사용자가 앱으로 결과를 조회할 때도, `GenePlanFacade`를 호출한다. 이용권 → 키트 → 유전자 데이터 → 검사 결과로 이어지는 도메인 탐색 로직이 한 곳에만 존재한다.

| Facade | 역할 |
|--------|------|
| `VoucherFacade` | 이용권 발급·검증·상태 관리 |
| `VoucherKitFacade` | 검사 키트 배송·수령 처리 |
| `GenePlanFacade` | 유전자 검사 결과·W-GRS 점수 조회 |
| `PlanFacade` | 검사 플랜(상품) 관리 |
| `AuthFacade` | 인증·토큰 갱신 |
| `MemberFacade` | 회원 등록·수정 |
| `MemberQueryFacade` | 회원 조회·검색 |
| `MemberProfileFacade` | 프로필 관리 |

### Atomic Change

이전에는 수정사항이 나오면 두 프로젝트에서 각각 커밋, 각각 배포를 해야 했다. 이제는 하나의 커밋으로 전체 모듈에 반영된다.

```bash
# 엔티티 변경 + Facade 수정 + Admin/Service 컨트롤러 반영을 하나의 커밋으로
git add project-domain/src/.../Gene.java
git add project-app-common/src/.../GenePlanFacade.java
git add project-admin-be/src/.../GeneTestAdminController.java
git add project-service-be/src/.../GeneResultController.java
git commit -m "feat(gene): 재검사 상태 필드 추가 및 API 반영"
```

두 프로젝트를 동시에 수정하고 배포 순서를 맞추던 시절과 비교하면, 이게 가장 체감이 크다.

### Infra: 보안 설정의 일관성

보안 설정, 에러 핸들링, API 응답 포맷, 캐시 정책 같은 인프라성 코드는 `project-infra`에 들어 있다. JWT 토큰 검증 로직을 수정하면 Admin과 Service가 동시에 반영된다.

유전자검사 결과는 민감한 개인정보다. 보안 설정이 양쪽에서 일관되지 않으면 한쪽에서 뚫리는 건 시간문제다. 이전에는 두 프로젝트의 Security 설정을 각각 관리해야 했는데, 이제는 한 곳에서 관리한다.

## 주의할 점: Facade가 갓클래스가 되는 순간

Facade 계층을 두면 공통 로직을 한 곳에 모을 수 있다. 문제는 **"한 곳"이 너무 편하다**는 것이다.

처음에는 `VoucherFacade` 하나로 시작한다. 이용권 발급, 검증, 상태 조회. 깔끔하다. 그런데 키트 배송 로직이 추가되고, 동의서 처리가 추가되고, 이용권에 연결된 플랜 조회가 추가되면서 하나의 Facade가 점점 비대해진다. Admin에서 필요한 메서드, Service에서 필요한 메서드가 전부 한 클래스에 쌓이기 시작한다.

```java
// 이렇게 되면 안 된다
@Service
public class VoucherFacade {

    // 이용권 관련
    public VoucherDetailResponse getVoucherDetail(Long voucherId) { ... }
    public void validateVoucherCode(String code) { ... }

    // 키트 배송 관련 — 여기 있는 게 맞나?
    public VoucherKitInfo getLatestKitInfo(Long userId) { ... }
    public void updateDeliveryAddress(Long voucherId, String address) { ... }

    // 동의서 관련 — 이것도?
    public String getPdfKey(Long userId, String barcode) { ... }

    // 검사 결과 관련 — 이건 확실히 아닌데
    public GeneResultResponse getGeneResult(Long voucherId) { ... }
    public List<Plan> getActivePlans() { ... }
}
```

"공통 로직이니까 Facade에 넣자"를 반복하면 결국 갓클래스가 된다. 두 프로젝트에 흩어져 있던 중복 코드를 한 곳에 모은 건 좋았는데, 모은 곳 자체가 정리되지 않으면 또 다른 문제가 된다.

해결은 **Facade를 책임 단위로 쪼개는 것**이다. 실제로 이 프로젝트에서는 이용권 관련 로직만 해도 세 개의 Facade로 분리했다.

| Facade | 책임 |
|--------|------|
| `VoucherFacade` | 이용권 자체의 발급·검증·상태 관리 |
| `VoucherKitFacade` | 키트 배송·수령·주소 변경 |
| `GenePlanFacade` | 검사 결과 조회·W-GRS 점수·플랜 연동 |

회원 도메인도 마찬가지다. `MemberFacade` 하나가 아니라 `MemberFacade`(등록·수정), `MemberQueryFacade`(조회·검색), `MemberProfileFacade`(프로필)로 나눴다.

> Facade는 "공통 로직을 모으는 곳"이 아니라 "하나의 유스케이스를 조율하는 곳"이다. 하나의 Facade에 DomainService 주입이 5개가 넘어가면 책임이 섞이고 있다는 신호다. 쪼개야 할 때다.

Facade를 `app-common`에 둔 이유, 트랜잭션 경계를 어디서 긋는지, Entity 생성 규칙까지 — 이 구조 위에서 만든 구체적인 규칙은 [Facade는 왜 Domain이 아닌 Application 레이어에 있는가](/posts/spring-boot-facade-transaction-architecture/)에서 이어진다.

## 대신 포기한 것들

공짜 점심은 없다.

### 빌드 시간

`project-domain`을 수정하면 이에 의존하는 `project-infra`, `project-app-common`, `project-admin-be`, `project-service-be`가 전부 다시 빌드된다. Gradle의 증분 빌드가 어느 정도 완화해주지만, 엔티티를 수정하면 사실상 전체 리빌드에 가깝다.

```bash
# 엔티티 하나 수정 후 빌드
./gradlew clean build

# 실제로는 6개 모듈 전체가 빌드됨
> Task :project-common:compileJava
> Task :project-domain:compileJava
> Task :project-infra:compileJava
> Task :project-app-common:compileJava
> Task :project-admin-be:compileJava
> Task :project-service-be:compileJava
```

두 프로젝트로 분리되어 있을 때는 필요한 프로젝트만 빌드하면 됐다. 모노레포에서는 변경이 전파되는 구조이기 때문에 하위 모듈 전체에 영향을 준다.

### 초기 설계 비용

6개 모듈의 경계를 어디서 나눌 것인가, 각 모듈의 책임은 무엇인가, 의존성 방향은 어떻게 설정할 것인가. 기존 두 프로젝트의 코드를 모듈 구조에 맞게 재배치하는 데 상당한 시간이 들었다.

특히 유전자검사 도메인은 Aggregate 간 경계가 애매한 부분이 많다. 이용권(Voucher)과 유전자(Gene)는 서로 다른 Aggregate인데, 키트 바코드를 통해 연결된다. 이 관계를 domain 모듈 안에서 어떻게 표현할 것인지, Facade에서 어떻게 조율할 것인지를 결정하는 데 꽤 고민이 필요했다.

### 순환 의존

모듈이 많아지면 "A에서 B의 뭔가가 필요한데, B도 A의 뭔가가 필요한" 상황이 생긴다. Gradle은 이걸 빌드 에러로 잡아주지만, 그걸 해결하기 위해 공통 모듈로 빼거나 구조를 재설계해야 하는 경우가 있다. 두 프로젝트로 분리되어 있을 때는 이런 고민 자체가 없었다.

> 모듈 간 순환 의존이 발생하면 보통 책임 분리가 잘못된 신호다. 당장은 귀찮지만, 순환 의존을 해결하는 과정에서 구조가 더 깔끔해지는 경우가 많다.

## 현재 규모

구조가 실제로 어떤 규모인지 감을 잡기 위해 수치를 정리했다.

| 모듈 | 타입 | 주요 구성 |
|------|------|----------|
| common | Library | 유틸리티, 예외 기본 클래스 (Spring 의존 없음) |
| domain | Library | 엔티티 44개, Repository 58개 (QueryDSL 17개), DomainService 40개 |
| infra | Library | 보안, JWT, Swagger, 캐시, 메일/PDF, FCM 푸시, 에러 핸들링 |
| app-common | Library | Facade 13개, 공통 DTO |
| admin-be | Application | 컨트롤러 44개 (Thymeleaf, 엑셀, 유전자 데이터 관리) |
| service-be | Application | 컨트롤러 15개 (순수 REST, 이용권·결과 조회) |

두 프로젝트로 운영하던 시절에는 이 엔티티 44개와 Repository 58개가 양쪽에 각각 존재했다. 그걸 한 곳으로 모은 것만으로도 전환할 가치가 있었다.

## 돌아보며

1인 개발자에게 모노레포 + 멀티모듈 구조는 **"혼자서도 두 서비스를 안전하게 운영하기 위한 최소한의 장치"**였다.

두 프로젝트를 따로 운영해본 경험이 있었기 때문에, 모노레포로 전환했을 때 뭐가 좋아졌는지를 구체적으로 체감할 수 있었다. 수정사항이 나올 때마다 두 프로젝트를 동시에 열고, 같은 수정을 두 번 하고, 배포 순서를 맞추던 그 오버헤드가 사라진 것. 그리고 헥사고날 아키텍처의 의존성 방향을 Gradle 모듈이 컴파일 타임에 강제해주는 것.

정리하면 세 가지였다.

> 1. **코드 중복 제거** — 엔티티, Repository, 공통 로직을 한 곳에서 관리
> 2. **컴파일 타임 경계 강제** — 헥사고날 아키텍처의 의존성 방향을 빌드가 강제
> 3. **Atomic Change** — 하나의 커밋으로 전체 모듈에 변경을 반영

팀이 있었다면 멀티레포도 괜찮았을 것이다. 각 레포를 담당하는 사람이 있고, 코드 리뷰로 경계 위반을 잡아줄 사람이 있었다면. 하지만 혼자서는 그 모든 역할을 동시에 할 수 없었고, 구조가 대신 해줘야 했다.

완벽한 구조는 아니다. 빌드 시간도 아쉽고, 초기 전환에 들인 시간도 적지 않았다. 하지만 `Gene` 엔티티의 상태 전이 로직을 고칠 때 한 곳만 수정하면 되는 안도감, 실수로 Service에서 Admin 전용 `GeneDataUploadService`를 import하면 빌드가 깨져서 바로 알 수 있는 안전감 — 그게 혼자서 유전자검사 서비스를 운영하면서 잠을 잘 수 있게 해주는 것들이다.
