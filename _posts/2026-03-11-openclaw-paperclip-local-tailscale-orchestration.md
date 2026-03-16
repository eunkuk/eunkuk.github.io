---
title: "OpenClaw 이후 방향이 바뀌었다 — Paperclip을 보며 고정 에이전트와 오케스트레이션을 다시 본 기록"
date: 2026-03-11 14:30:00 +0900
categories: [AI]
tags: [paperclip, openclaw, tailscale, self-hosted, ai-agent, orchestration, retrospective]
description: "OpenClaw 이후 Paperclip을 로컬에 붙여보며, 내 관심사가 더 똑똑한 단일 에이전트가 아니라 고정 에이전트를 운영하는 오케스트레이션 레이어 쪽으로 옮겨갔다는 걸 정리한 기록."
---

> [OpenClaw 셋팅 회고](/posts/building-my-ai-trading-assistant-retrospective/) 이후, 다음으로 본 건 더 센 단일 에이전트가 아니었다. [Paperclip](https://github.com/paperclipai/paperclip)을 로컬에 올리고 [Tailscale](https://tailscale.com/)로 외부 접근을 열어보면서, 내가 원했던 건 결국 고정 에이전트를 두고 그 위를 운영하는 오케스트레이션 레이어 쪽이라는 걸 더 분명하게 보게 됐다.

## OpenClaw 다음에 막힌 지점

처음 [OpenClaw](https://github.com/openclaw/openclaw)를 붙였을 때 좋았던 건 명확했다.

- 텔레그램 같은 기존 채널에서 바로 붙는다
- 로컬 환경을 원격으로 다루는 감각이 빠르게 온다
- 에이전트 하나를 꽤 "자율적인 작업자"처럼 다룰 수 있다

실제로 OpenClaw 공식 문서도 이 도구를 개인 비서 세팅 관점에서 설명한다.  
워크스페이스에 `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `HEARTBEAT.md`를 두고, 필요하면 `MEMORY.md`를 더해 세션과 행동을 이어가는 구조다.

문제는 그 다음이었다.

내가 만들고 싶은 게 단순히 "채팅창에서 부르면 일하는 에이전트 1명"이 아니라,
**역할이 다른 에이전트 몇 개를 묶어서 돌아가게 하는 개인화 앱** 쪽으로 바뀌기 시작했다.

그 시점부터 필요한 건 이런 것들이었다.

- 지금 누가 무슨 일을 맡고 있는지
- 어떤 목표 아래 작업이 연결되는지
- 승인 포인트를 어디에 둘지
- 비용이 어디서 얼마나 타고 있는지
- 밖에서도 상태를 보고 중간 개입할 수 있는지

여기서부터는 "에이전트가 똑똑한가"보다 **오케스트레이션 레이어가 있는가**가 더 중요해졌다.

## 그래서 Paperclip을 붙였다

[Paperclip 공식 README](https://github.com/paperclipai/paperclip)는 이 도구를 꽤 정확하게 설명한다.  
OpenClaw가 개별 작업자에 가깝다면, Paperclip은 그 작업자들을 조직으로 운영하는 쪽에 가깝다.

README 기준으로 Paperclip은 이런 성격이다.

- Node.js 서버 + React UI 기반
- org chart, goal alignment, budget, governance, ticket system 제공
- 여러 에이전트를 하나의 회사처럼 운영하는 레이어
- "챗봇"이 아니라 에이전트 운영 관리 도구에 더 가까움

내 경우에는 온보드 스크립트보다, 소스를 직접 보고 붙이는 쪽이 더 맞았다.

```bash
git clone https://github.com/paperclipai/paperclip.git
cd paperclip
pnpm install
pnpm dev
```

공식 README에 따르면 로컬 개발 시 API 서버는 `http://localhost:3100`에서 뜨고, 임베디드 PostgreSQL도 자동 생성된다.  
이 지점이 꽤 좋았다. 초기 실험 단계에서는 "일단 로컬에 바로 올려서 구조를 본다"가 훨씬 중요했기 때문이다.

## 외부 접근은 Tailscale로 열었다

여기서 마음에 들었던 포인트 하나는, Paperclip README FAQ도 **solo entrepreneur라면 Tailscale로 이동 중 접근하는 방식**을 바로 예시로 든다는 점이었다.

내가 원하는 것도 딱 그 정도였다.

- 아직 퍼블릭 서비스로 바로 노출할 단계는 아니다
- 하지만 밖에서 대시보드와 상태는 보고 싶다
- 같은 로컬 인스턴스를 안전하게 계속 쓰고 싶다

그래서 공개 포트부터 여는 대신,
**로컬에 올린 Paperclip 인스턴스를 Tailscale VPN으로 묶어서 외부에서 접근하는 형태**로 갔다.

이 방식의 장점은 단순했다.

- 초기 배포 하드닝 부담이 작다
- 개인 도구 성격에 맞다
- 로컬 퍼스트 흐름을 유지한 채 밖에서 붙을 수 있다

지금 단계에서는 이게 가장 실용적이었다.

## OpenClaw와 뭐가 다른가

이 둘을 써보면서 느낀 건, 자꾸 경쟁 제품처럼 비교하면 오히려 감이 흐려진다는 점이다.

Flowtivity의 [OpenClaw vs Paperclip 비교 글](https://flowtivity.ai/blog/openclaw-vs-paperclip-ai-agent-framework-comparison/)도 비슷한 프레임을 쓰는데, 실제로 만져보니 그 방향이 맞았다.  
둘은 대체재라기보다 **층이 다르다.**

| 관점 | OpenClaw | Paperclip |
| --- | --- | --- |
| 중심 개념 | 자율적으로 움직이는 개별 에이전트 | 여러 에이전트를 운영하는 조직 레이어 |
| 기본 인터페이스 | Telegram/WhatsApp/Discord 같은 채널 친화적 | 대시보드 중심 |
| 기억 방식 | 워크스페이스 + `SOUL.md` + `MEMORY.md` + heartbeat | 목표, 티켓, 예산, 조직 문맥 중심 |
| 실행 성격 | 에이전트 스스로 판단하고 행동 | 할당된 일과 조직 규칙 안에서 조율 |
| 강한 지점 | 개인 비서, 자동화, 원격 작업자 감각 | 역할 분리, 추적, 거버넌스, 비용 통제 |
| 잘 맞는 상황 | 1인용 강한 에이전트 1~2개 | 여러 에이전트를 묶어 운영해야 할 때 |

내 체감으로 정리하면 이렇다.

- OpenClaw는 "일 잘하는 한 명"에 가깝다
- Paperclip은 "그 한 명, 혹은 여러 명을 굴리는 운영판"에 가깝다

그래서 지금 만드는 개인화 앱에는 Paperclip 쪽이 더 잘 맞는다.
문제가 "잘 대답하는가"가 아니라, **"여러 역할을 어떻게 이어붙이고 관리할 것인가"** 쪽으로 넘어갔기 때문이다.

## 그렇다고 OpenClaw를 버린 건 아니다

오히려 반대다.

OpenClaw를 먼저 써봤기 때문에 Paperclip이 왜 필요한지 더 빨리 이해됐다.

OpenClaw는 여전히 이런 역할로 강하다.

- 직접 실행하고 반응하는 개인 비서형 에이전트
- 채널 기반 인터랙션
- 로컬 환경과 가까운 자동화

Paperclip은 그 위에서 이런 걸 맡기 좋다.

- 역할별 에이전트 분리
- 작업 추적과 상태 관리
- 승인 흐름
- 예산/비용 통제
- 여러 에이전트의 책임 분배

그래서 지금은 둘 중 하나를 고르는 문제라기보다,
**OpenClaw와 이런 오케스트레이션 레이어를 적절히 같이 쓰는 쪽이 더 맞겠다**는 생각이 든다.

예를 들면 이런 그림이다.

- OpenClaw 쪽에는 기억과 역할이 이어지는 고정 에이전트를 둔다
- 필요할 때만 잠깐 돌릴 작업은 임시 작업자나 서브에이전트로 처리한다
- 그 위에서 목표, 승인, 흐름 관리는 상위 운영 레이어가 맡는다

이렇게 보면 "내가 OpenClaw가 된다"기보다,
**OpenClaw 같은 고정 직원들을 두고 그 위를 운영하는 쪽**에 더 가깝다.

내 느낌에는 순서도 이게 자연스럽다.

1. 먼저 OpenClaw 같은 걸로 "에이전트 하나가 실제로 얼마나 일을 할 수 있는가"를 본다.
2. 그 다음 "이제 여러 개를 어떻게 통제할 건데?"에서 Paperclip 같은 레이어가 필요해진다.

## 그리고 이런 생각도 들었다

Paperclip을 보면서 계속 든 생각이 하나 있었다.

어떻게 보면 이건 예전에 쓴
[`Tauri + Claude CLI로 만든 Multi-Agent 프로젝트 관리 도구`](/posts/tauri-claude-multi-agent-system/)
글에서 내가 만들려던 것의 **최종본에 더 가까운 형태 아닌가?** 하는 생각이었다.

그때도 본질적으로 풀고 싶었던 문제는 비슷했다.

- 역할이 다른 에이전트를 나누고
- 각 에이전트의 상태와 출력을 보고
- 세션과 작업 흐름을 이어서 관리하고
- 하나의 프로젝트 목표 아래서 오케스트레이션하는 것

차이가 있다면 나는 그걸
**Tauri + Claude CLI 기반의 로컬 데스크톱 도구** 관점에서 풀고 있었고,
Paperclip은 처음부터
**조직, 목표, 티켓, 거버넌스, 예산** 같은 운영 레이어까지 포함한 제품 관점으로 더 밀어붙였다는 점이다.

그래서 약간 묘한 기분이 들었다.

- 방향 자체는 틀리지 않았다는 확인
- 하지만 내가 만들던 건 "도구"였고, Paperclip은 더 노골적으로 "운영 체계" 쪽이었다는 차이

지금 다시 보면, 당시 내가 직접 만들던 멀티에이전트 대시보드의 문제의식이
이번 Paperclip 경험에서 다른 형태로 다시 이어진 셈이다.

## 결국 바뀐 방향

지금은 이 로컬 + Tailscale 구성 위에서,
개인화 앱을 **AI 오케스트레이션 중심**으로 개발하고 있다.

예전에는 좋은 에이전트 하나를 더 만드는 쪽에 가까웠다면,
지금은 방향이 조금 다르다.

- 기억과 역할이 이어지는 고정 에이전트를 두고
- 필요할 때만 임시 작업자나 서브에이전트를 붙이고
- 사용자 맥락에 맞게 역할을 나누고
- 각 역할이 하는 일을 연결하고
- 사람이 중간에 개입할 지점을 남기고
- 그 전체 흐름을 개인 도구처럼 다루는 것

아직은 초기 단계지만, 적어도 하나는 분명해졌다.

OpenClaw 이후의 다음 문제는 더 똑똑한 단일 에이전트가 아니라,
**에이전트를 운영하는 구조**였다.

그리고 지금 내 관심사는 정확히 그쪽으로 옮겨갔다.

결국 바뀐 건 툴이 아니라 방향이었다.

- "좋은 에이전트 하나를 더 붙일까?"에서
- "고정 에이전트들을 어떻게 운영할까?"로

Paperclip은 그 변화를 분명하게 보여준 참고점이었고,
OpenClaw는 그 안에서 계속 살아남을 고정 작업자에 더 가까웠다.

## 참고 링크

- [Paperclip GitHub 저장소](https://github.com/paperclipai/paperclip)
- [OpenClaw GitHub 저장소](https://github.com/openclaw/openclaw)
- [OpenClaw Personal Assistant Setup 문서](https://docs.openclaw.ai/start/openclaw)
- [Flowtivity - OpenClaw vs Paperclip 비교 글](https://flowtivity.ai/blog/openclaw-vs-paperclip-ai-agent-framework-comparison/)
