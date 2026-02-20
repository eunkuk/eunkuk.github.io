---
title: "115개 PHP 모델을 44개 엔티티로 재설계하다 — 마이그레이션 회고 (2) 도메인 재설계"
date: 2026-02-17 22:00:00 +0900
categories: [Dev, Migration]
tags: [spring-boot, jpa, ddd, entity, domain-service, migration, retrospective, series]
description: "PHP CodeIgniter의 115개 모델을 Spring Boot 44개 엔티티로 재설계한 과정. 테이블 1:1 모델에서 DDD 기반 도메인 서비스 계층까지, 마이그레이션은 번역이 아니라 재설계였다."
---

> PHP CodeIgniter의 115개 모델을 Spring Boot 44개 엔티티로 재설계한 과정. 테이블 1:1 모델에서 DDD 기반 도메인 서비스 계층까지, 마이그레이션은 번역이 아니라 재설계였다.

---

## 1. PHP 모델 구조

PHP CodeIgniter에서 모델은 테이블 래퍼다. 테이블 하나에 모델 하나. 서비스 계층이 없다.

```php
// Member_model.php — 테이블 래퍼
class Member_model extends CB_Model
{
    public $tablename = 'member';
    public $primary_key = 'mem_id';

    public function get_by_token($token) {
        return $this->db->where('mem_token', $token)
                        ->get($this->tablename)
                        ->row_array();
    }

    public function update_login_time($mem_id) {
        $this->db->where('mem_id', $mem_id)
                 ->update($this->tablename, [
                     'mem_lastlogin_datetime' => date('Y-m-d H:i:s')
                 ]);
    }
}
```

비즈니스 로직이 어디에 있는가? 컨트롤러다.

```php
// Api_member.php — 비즈니스 로직이 컨트롤러에
public function login()
{
    $userid = $this->input->post('userid');
    $password = $this->input->post('password');

    $member = $this->Member_model->get_by_userid($userid);
    if (!$member) {
        echo json_encode(['status' => 'error', 'message' => '회원 없음']);
        exit;
    }
    if ($member['mem_denied'] == 1) {
        echo json_encode(['status' => 'error', 'message' => '차단된 회원']);
        exit;
    }
    // ... 비밀번호 검증, 토큰 발급, 로그인 로그 저장이 전부 여기에
}
```

이 구조의 문제는 명확했다.

| 문제 | 증상 |
|------|------|
| 서비스 계층 부재 | 비즈니스 로직이 컨트롤러에 흩어짐 |
| 모델 = 테이블 래퍼 | 도메인 규칙 없이 CRUD만 수행 |
| 로직 중복 | 같은 검증을 API와 Admin에서 각각 구현 |
| 테스트 불가 | 컨트롤러에 로직이 있으니 단위 테스트가 안 됨 |

마이그레이션을 하러 입사했지만, 115개 모델을 그대로 115개 엔티티로 옮기는 건 의미가 없었다. 구조를 바꿔야 했다.

---

## 2. 도메인별 모델 → 엔티티 매핑

115개 PHP 모델을 6개 도메인 그룹으로 분류하고, 각 그룹별로 엔티티를 재설계했다. 결과를 요약하면 이렇다.

| 도메인 | PHP 모델 | Spring Boot 엔티티 | 변화율 |
|--------|---------|-------------------|--------|
| Member | 14 | 6 | -57% |
| Voucher/Gene | 7 | 13 | +86% |
| Post/Board | 16+ | 11 | -31% |
| Banner/SMS | 9 | 6 | -33% |
| Config/Stat/기타 | 69+ | 8 | -88% |
| **합계** | **115** | **44** | **-62%** |

수치만 보면 줄어든 도메인이 대부분이지만, Voucher/Gene처럼 오히려 늘어난 곳이 있다. 줄인 게 아니라 **재설계한** 것이다. 재설계에서 반복된 패턴이 세 가지 있었다.

### Wide Table 분해

PHP에서는 하나의 모델에 모든 정보를 넣는 Wide Table 구조가 많았다. `Voucher_model` 하나에 이용권 정보, 키트 배송 정보, 동의서 정보가 전부 들어있었다. 컬럼이 40개가 넘는다. Spring Boot에서는 Aggregate를 분해해서 `Voucher`, `VoucherKit`, `VoucherUsage`, `VoucherAgreement`로 나눴다. 이것이 Voucher/Gene 도메인의 엔티티가 오히려 늘어난 이유다. 검사 결과 저장도 Gene 테이블의 JSON 컬럼에서 `GeneResult` EAV 테이블로 전환했다. 이 결정의 배경은 [EAV vs JSONB 마이그레이션](/posts/eav-vs-jsonb-gene-data-retrospective/)에서 자세히 다룬다.

### 별도 테이블 → 상태값 전환

Member 도메인이 대표적이다. PHP에서는 휴면 회원을 별도 테이블(`Member_dormant_model`)에 복사해서 관리했다. 같은 회원 데이터가 두 테이블에 존재하는 구조. Spring Boot에서는 `Member.state` 필드 하나로 활성/휴면/탈퇴를 관리한다.

```java
// Member.java — 상태 기반 관리
@Column(name = "mem_state", nullable = false)
private Integer state;

public boolean isActive() {
    return MemberState.fromCode(this.state) == MemberState.ACTIVE;
}

public boolean isDormant() {
    return MemberState.fromCode(this.state) == MemberState.DORMANT;
}

public void activate() {
    if (!isDormant()) {
        throw new DomainException(DomainErrorCode.INVALID_STATE_TRANSITION);
    }
    this.state = MemberState.ACTIVE.getCode();
}
```

이런 식으로 중복 테이블, 미사용 기능, EAV 패턴을 정리하면서 14개 모델이 6개 엔티티로 줄었다.

### 불필요한 것 제거

Config(8개 모델)은 `application.yml`로 대체했고, Stat(4개)은 외부 분석 도구로 대체했다. 마이그레이션은 모든 것을 옮기는 게 아니라, 옮길 것과 버릴 것을 판단하는 과정이기도 했다.

---

## 3. DomainService 계층

PHP에 없던 것이 Spring Boot에 생겼다. 서비스 계층.

PHP는 Controller → Model 2단 구조다. 서비스 계층이 없으니 비즈니스 로직이 컨트롤러에 흩어지고, 같은 규칙이 API와 Admin에서 각각 중복 구현된다. "이용권 상태가 1이면 사용 가능"이라는 규칙이 `Api_voucher.php`에도 있고, Admin 컨트롤러에도 있었다. 같은 규칙이 두 곳 이상에 복사되어 있으면 반드시 불일치가 생긴다.

Spring Boot에서는 DomainService로 비즈니스 규칙을 한 곳에 집중시켰다. DomainService는 하나의 Aggregate에 대한 비즈니스 규칙을 캡슐화하는 계층이다.

```java
// VoucherDomainService.java
public Voucher useVoucher(Long voucherId, Long memberId) {
    Voucher voucher = getById(voucherId);

    if (!voucher.canBeUsed()) {
        throw new DomainException(DomainErrorCode.VOUCHER_NOT_AVAILABLE);
    }
    if (voucher.isExpired()) {
        throw new DomainException(DomainErrorCode.VOUCHER_EXPIRED);
    }

    voucher.markAsUsed(memberId);
    return voucherRepository.save(voucher);
}
```

`canBeUsed()`, `isExpired()`, `markAsUsed()` — 비즈니스 규칙이 엔티티와 DomainService 안에 있다. API 컨트롤러에서든 Admin 컨트롤러에서든 `voucherDomainService.useVoucher()`를 호출하면 같은 규칙이 적용된다.

여러 DomainService를 조합해야 하는 유스케이스에는 Facade를 둔다. 예를 들어 이용권 발급은 Voucher + Kit + Agreement를 조합하고, 검사 결과 조회는 Gene + GeneResult + Plan을 조합한다. DomainService가 단일 Aggregate의 규칙이라면, Facade는 여러 Aggregate를 조율하는 유스케이스 계층이다. Facade가 `app-common` 모듈에 위치하는 이유와 트랜잭션 경계에 대해서는 [Facade는 왜 Domain이 아닌 Application 레이어에 있는가](/posts/spring-boot-facade-transaction-architecture/)에서, Facade를 Command와 Query로 분리한 경량 CQRS 패턴은 [경량 CQRS](/posts/lightweight-cqrs-facade-separation/)에서 이어진다.

---

## 4. PHP → Spring Boot: 구조 비교

전환 전후를 한눈에 비교하면 이렇다.

| 항목 | PHP CodeIgniter | Spring Boot |
|------|----------------|-------------|
| 아키텍처 | Controller → Model (2계층) | Controller → Service → Facade → DomainService → Repository (다계층) |
| 모델/엔티티 | 115개 (테이블 1:1) | 44개 (Aggregate 기반) |
| 서비스 계층 | 없음 | DomainService 40개 + Facade 14개 |
| 비즈니스 로직 위치 | 컨트롤러 | 엔티티 + DomainService |
| 상태 관리 | 매직넘버 (`1`, `2`) | Enum (`VoucherState.ISSUED`) |
| 검증 | 문자열 규칙 | 타입 시스템 + 도메인 메서드 |
| 데이터 접근 | `$this->db->where()->get()` | JPA Repository + QueryDSL |
| 변경 감사 | 수동 히스토리 테이블 | Hibernate Envers `@Audited` |

---

## 5. 돌아보며

마이그레이션은 번역이 아니다. 재설계다.

115개 모델을 44개 엔티티로 줄인 게 아니다. PHP의 테이블 래퍼 구조를 DDD 기반의 도메인 모델로 다시 만든 것이다. 어떤 도메인(Voucher/Gene)은 오히려 엔티티가 늘었다 — 하나의 Wide Table에 섞여 있던 책임을 분리하면 당연히 그렇게 된다.

가장 큰 변화는 수치에 드러나지 않는다. PHP에 없던 DomainService와 Facade 계층이 생기면서 "비즈니스 규칙이 어디에 있는가"라는 질문에 답할 수 있게 되었다. 컨트롤러 안 어딘가가 아니라, `VoucherDomainService.useVoucher()`에 있다고 말할 수 있게 되었다.

정리하면 세 가지였다.

> 1. **1:1 전환은 마이그레이션이 아니다.** PHP 모델 115개를 Java 엔티티 115개로 옮기면 레거시를 Java로 번역한 것뿐이다. 구조를 바꿔야 마이그레이션의 의미가 있다.
>
> 2. **엔티티 수가 줄었다고 좋은 게 아니다.** Voucher/Gene 도메인은 오히려 늘었다. 줄어든 건 중복과 불필요한 테이블이고, 늘어난 건 비즈니스를 표현하는 데 필요한 구조다.
>
> 3. **서비스 계층의 부재가 가장 큰 기술 부채였다.** PHP의 문제는 언어가 아니라 구조다. Controller → Model 2단 구조에서는 비즈니스 로직이 흩어질 수밖에 없다. DomainService와 Facade를 도입한 것이 이 마이그레이션의 핵심이다.

이 도메인 재설계가 [모노레포 + 6모듈 구조](/posts/monorepo-multi-module-retrospective/)의 기반이 되었다. `domain` 모듈에 엔티티와 DomainService가 들어가고, `app-common` 모듈에 Facade가 들어간다. 모듈 구조가 도메인 설계를 물리적으로 강제한다.

---

**시리즈:**
- [Part 1](/posts/php-codeigniter-spring-boot-migration-why/) — 왜 다시 만들었나
- **Part 2 (이 글)** — 도메인 재설계
- [Part 3](/posts/admin-controller-migration-legacy-compatibility/) — 레거시 호환과 신규 API
- [Part 4](/posts/spring-boot-infrastructure-auth-security/) — 인증, 보안, 인프라
- [Part 5](/posts/migration-lessons-learned-retrospective/) — 교훈과 수치
