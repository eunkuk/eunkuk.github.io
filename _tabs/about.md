---
# the default layout is 'page'
icon: fas fa-user
order: 4
---

백엔드 중심으로 일하고, 필요하면 프론트엔드, 배포, 내부 도구까지 직접 붙여서 끝내는 개발자입니다.

이 블로그에는 실제로 만들고 운영하면서 부딪힌 구조 선택, 마이그레이션, 설계 타협, 자동화 경험을 기록합니다.

> 화려한 이론보다, 혼자서도 유지 가능한 구조와 실제로 돌아가는 선택을 더 중요하게 봅니다.

## 어떤 글을 쓰는가

- Spring Boot 기반 백엔드와 도메인/계층 설계
- 운영 중인 서비스의 마이그레이션과 레거시 정리
- 유전자검사 서비스 도메인에서의 데이터 모델링과 결과지 설계
- Tauri, AI 툴링, 자동화 도구 같은 사이드 프로젝트 기록

## 지금 하는 방식

- 서비스와 내부 도구를 직접 만들고 운영합니다.
- 백엔드가 주력이지만 배포, 운영, 간단한 프론트엔드까지 같이 다룹니다.
- 일단 동작하게 만든 뒤, 반복되는 비용을 줄이는 방향으로 구조를 다듬는 편입니다.
- "깔끔한 정답"보다 현재 제약 안에서 오래 버티는 선택을 더 중요하게 봅니다.

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

> 소개 페이지는 계속 다듬는 중입니다. 블로그 전체를 처음 보는 사람이 어떤 글을 기대하면 되는지 빠르게 보이도록 정리하고 있습니다.
