---
# the default layout is 'page'
icon: fas fa-user
order: 4
---

백엔드 중심으로 일하지만, 레거시 분석부터 API, 배포, 운영, 내부 도구까지 필요하면 범위를 넓혀 문제를 끝까지 닫는 개발자입니다.

이 블로그에는 실제 서비스와 내부 도구를 만들고 운영하면서 내린 구조 선택, 마이그레이션 판단, 테스트 전략, 자동화 경험을 기록합니다.

> 화려한 이론보다, 팀이 설명할 수 있고 운영이 버틸 수 있는 구조를 더 중요하게 봅니다.

## 이 블로그를 보면 알 수 있는 것

- PHP 레거시를 Spring Boot 기반 구조로 다시 세우며 어떤 판단을 했는지
- 도메인 설계, 테스트 전략, 멀티테넌시 같은 문제를 과설계 없이 어떻게 풀었는지
- 반복 업무를 줄이기 위해 내부 도구와 자동화를 어떻게 만들었는지
- 기술 결정을 코드, 문서, 테스트, 협업 언어로 어떻게 연결했는지

## 일하는 방식

- 백엔드가 주력이지만, 문제를 닫기 위해 프론트엔드, 배포, 운영, 문서화도 직접 맡습니다.
- 구조는 추상적으로 예쁘게보다 변경 비용과 운영 비용이 줄어드는 방향으로 고릅니다.
- 설계 의도를 코드와 문서와 테스트로 남겨 팀이 같이 다룰 수 있게 만드는 편입니다.
- 기술 용어를 그대로 던지기보다, 비즈니스 영향과 운영 리스크로 번역해 설명합니다.

## 처음 보면 좋은 글

- [1인 개발로 마이그레이션을 끝내며 — 마이그레이션 회고 (5) 교훈과 수치](/posts/migration-lessons-learned-retrospective/) - 레거시 전환에서 맡은 범위와 결과
- [레거시 혼합 환경에서 테스트 전략 다시 세우기 — ArchUnit, E2E, H2 전환기](/posts/testing-strategy-legacy-archunit-e2e/) - 계층 규칙과 검증 전략을 팀 규칙으로 만든 과정
- [유전자검사 서비스에 벤더사가 붙기 시작했다 — 멀티테넌시를 고민할 시점](/posts/gene-test-vendor-multi-tenancy-architecture/) - 과설계를 피하면서 확장 경로를 남긴 설계 판단
- [Python 스크립트 하나로 돌리던 유전자 분석을 데스크톱 앱으로 만들기까지 — GeneTitan Agent 회고](/posts/genetitan-agent-retrospective/) - 운영 자동화와 내부 도구 개발 경험
- [스트래티지 패턴을 못 말했던 이유 — 설계 감각과 설계 언어는 다르다](/posts/pattern-language-llm-strategy-pattern/) - 실무 감각을 팀 언어로 연결하는 방식

### 마이그레이션 회고

- [PHP CodeIgniter 프로젝트를 Spring Boot로 옮기기까지 — 마이그레이션 회고 (1) 왜 다시 만들었나](/posts/php-codeigniter-spring-boot-migration-why/)
- [115개 PHP 모델을 44개 엔티티로 재설계하다 — 마이그레이션 회고 (2) 도메인 재설계](/posts/php-to-spring-boot-domain-redesign/)
- [96개 PHP 어드민 컨트롤러를 42개로 재구성하다 — 마이그레이션 회고 (3) 컨트롤러 전환과 신규 API](/posts/admin-controller-migration-legacy-compatibility/)
- [6모듈 멀티모듈에서 인프라를 다시 세우다 — 마이그레이션 회고 (4) 인증, 보안, 인프라](/posts/spring-boot-infrastructure-auth-security/)
- [1인 개발로 마이그레이션을 끝내며 — 마이그레이션 회고 (5) 교훈과 수치](/posts/migration-lessons-learned-retrospective/)

### 아키텍처 / 설계

- [1:N 강의 동시 번역 시스템 PoC 회고](/posts/realtime-translation-flow-control-final-retrospective/)
- [Spring Boot DDD — Facade는 왜 Domain이 아닌 Application 레이어에 있는가](/posts/spring-boot-facade-transaction-architecture/)
- [DomainService vs Facade — 이 로직은 어디에 둬야 하는가](/posts/domain-service-vs-facade-boundary/)
- [Method Naming → @Query → QueryDSL — 쿼리 복잡도에 따른 Repository 전략](/posts/repository-query-tier-selection/)
- [유전자검사 서비스에 벤더사가 붙기 시작했다 — 멀티테넌시를 고민할 시점](/posts/gene-test-vendor-multi-tenancy-architecture/)

### 사이드 프로젝트 / 도구

- [Python 스크립트 하나로 돌리던 유전자 분석을 데스크톱 앱으로 만들기까지 — GeneTitan Agent 회고](/posts/genetitan-agent-retrospective/)
- [Tauri + Claude CLI로 만든 Multi-Agent 프로젝트 관리 도구](/posts/tauri-claude-multi-agent-system/)
- [스트래티지 패턴을 못 말했던 이유 — 설계 감각과 설계 언어는 다르다](/posts/pattern-language-llm-strategy-pattern/)

> 빠르게 훑고 싶다면 위의 "처음 보면 좋은 글"부터 보는 편이 좋습니다. 이 블로그의 결이 가장 선명하게 보이는 글들입니다.
