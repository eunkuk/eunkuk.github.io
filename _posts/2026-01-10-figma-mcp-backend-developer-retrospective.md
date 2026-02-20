---
title: "Figma MCP로 홈페이지를 만들어본 회고 — 디자인 디테일을 코드로 살리는 삽질의 기록"
date: 2026-01-10 22:00:00 +0900
categories: [Dev, Frontend]
tags: [figma, mcp, claude-code, react, responsive, css-in-js]
description: "Figma 디자인을 픽셀 단위로 코드에 옮기는 건 기능 구현과는 다른 영역이었다. Figma MCP + Claude Code 조합을 실무에 써본 3주간의 기록."
---

## 상황

회사 홈페이지를 만들어야 했다. React로 기능을 구현하는 건 할 수 있는데, Figma 디자인을 픽셀 단위로 재현하는 건 다른 영역이었다. 기능이 동작하게 만드는 것과 디자인 디테일을 살리는 것은 요구하는 감각이 다르다.

디자이너가 Figma로 디자인을 만들어줬다. Desktop, Tablet, Mobile 3개 뷰포트에 대해 각 페이지별로 프레임이 나왔다. 문제는 이걸 코드로 옮길 사람이 나밖에 없다는 것이었다.

기존 방식은 이랬다. Figma에서 요소를 하나씩 클릭하고, 오른쪽 인스펙트 패널에서 값을 확인하고, 에디터로 돌아와서 타이핑한다. 한 요소의 padding, fontSize, color, lineHeight를 옮기는 데 클릭 4번, 창 전환 4번이다. 카드 하나에 속성이 10개면 40번을 반복한다.

```
Figma 요소 클릭 → 인스펙트 패널에서 padding 확인 → 에디터 전환 → 값 입력
                → 인스펙트 패널에서 fontSize 확인 → 에디터 전환 → 값 입력
                → 인스펙트 패널에서 color 확인   → 에디터 전환 → 값 입력
                → ... (속성 수만큼 반복)
```

값 자체는 정확하다. Figma가 알려주는 숫자를 그대로 가져오니까. 문제는 이 과정이 모든 요소, 모든 뷰포트(Desktop/Tablet/Mobile)에 대해 반복된다는 것이었다. 한 페이지에 요소가 30개이고 뷰포트가 3개면, 단순 계산으로도 수백 번의 클릭-확인-전환-입력을 해야 한다.

## Figma MCP를 써보기로 한 계기

Claude Code에 Figma MCP를 연결하면 디자인 파일에서 레이아웃 정보를 자동으로 추출할 수 있다는 걸 알게 됐다. Figma에서 일일이 클릭해서 확인하던 수치를 API 한 번 호출로 구조화된 데이터로 가져오는 방식이다.

첫 반응은 기대였다. "Figma URL만 주면 알아서 코드로 변환해주는 거 아냐?" 현실은 달랐다.

### 초기 설정부터 삽질

Claude Code에서 Figma MCP를 처음 연결하려니 `No MCP servers configured` 에러가 떴다. 토큰 인증, MCP 서버 설정, 권한 허용까지 — 설정 파일을 만지작거리며 시간을 썼다.

```json
// .claude/settings.local.json에 허용 목록 추가
{
  "permissions": {
    "allow": [
      "mcp__figma-desktop__get_design_context",
      "mcp__figma-desktop__get_screenshot",
      "mcp__figma-desktop__get_metadata"
    ]
  }
}
```

연결이 되고 나서 `get_design_context`를 처음 호출했을 때, JSON으로 쏟아지는 디자인 정보를 보고 "이건 되겠다" 싶었다. px 단위의 정확한 크기, hex 색상 코드, 폰트 패밀리와 사이즈, 자식 노드의 레이아웃 구조까지 다 나왔다.

## 실제로 써보니 — 좋았던 점

### 한 번 호출로 전체 구조 추출

Figma에서 수동으로 작업할 때는 요소를 하나씩 클릭해서 값을 확인해야 한다. 카드 컴포넌트의 padding을 보려면 카드를 클릭하고, 타이틀의 fontSize를 보려면 타이틀을 클릭하고, 날짜의 color를 보려면 날짜를 클릭한다. 각각 별도의 동작이다.

`get_design_context`는 프레임 하나를 호출하면 그 안의 모든 자식 요소 정보를 트리 구조로 한 번에 반환한다.

```
Frame "PR_News_Desktop"
  ├── padding: top=80, bottom=120, left=0, right=0
  ├── gap: 24
  ├── Grid
  │   ├── columns: 6
  │   ├── columnGap: 24
  │   └── rowGap: 32
  └── Card
      ├── width: 384, height: 280
      ├── borderRadius: 4
      ├── title: fontSize=18, fontWeight=700, color=#0E1825
      └── date: fontSize=14, fontWeight=400, color=#666666
```

Figma 인스펙트 패널에서 10번 클릭해서 확인할 정보가 API 한 번 호출로 나온다. 값의 정확도는 같다 — 어차피 같은 Figma 파일에서 나오는 숫자다. 차이는 속도와 누락이다. 수동으로 하면 한두 개 속성을 빼먹기 쉬운데, MCP는 구조 전체를 빠짐없이 반환한다.

### 3개 뷰포트 체계적 처리

Desktop, Tablet, Mobile 각각의 Figma 프레임에서 디자인 정보를 추출한 뒤, 차이점을 비교할 수 있었다.

| 속성 | Desktop | Tablet | Mobile |
|------|---------|--------|--------|
| 그리드 열 | 6열 | 2열 | 1열 |
| 카드 간격 | 24px | 20px | 16px |
| 컨테이너 패딩 | 0 (센터 정렬) | 40px | 16px |
| 타이틀 폰트 | 18px | 16px | 14px |

이 비교표를 기반으로 `constants.ts`에 반응형 상수를 정의했다.

```typescript
export const POST_LIST_LAYOUT = {
  CONTAINER_PADDING: {
    DESKTOP: { top: 80, bottom: 120, horizontal: 0 },
    TABLET: { top: 60, bottom: 100, horizontal: 40 },
    MOBILE: { top: 40, bottom: 80, horizontal: 16 },
  },
  GRID_GAP: {
    DESKTOP: { column: 24, row: 32 },
    TABLET: { column: 20, row: 24 },
    MOBILE: { column: 0, row: 16 },
  },
} as const;
```

수동으로 Figma에서 하나하나 클릭해서 옮겼으면 이 비교표를 만드는 데만 반나절이 걸렸을 것이다.

### 워크플로우 문서화

약 3주간 11개 이상의 페이지를 구현하면서 반복되는 패턴을 문서로 정리했다.

| 문서 | 역할 |
|------|------|
| `figma-workflow-guide.md` | Figma → 코드 작업 프로세스 |
| `figma-responsive-implementation.md` | 반응형 구현 스킬 (Phase 0~5) |
| `design-system-guide.md` | 디자인 토큰 가이드 |
| `constants.ts` | 실제 디자인 상수 모음 |

특히 `figma-responsive-implementation.md`는 Claude Code의 스킬 파일로 등록해서, 새 페이지를 구현할 때마다 동일한 프로세스를 따르도록 했다. Phase 0(사전 분석)에서 3개 뷰포트의 디자인 정보를 전부 추출하고, Phase 1에서 상수를 정의하고, Phase 2~5에서 컴포넌트를 구현하는 흐름이다.

## 삽질 기록 — 에로사항들

좋은 점만 쓰면 광고가 된다. 실제로 겪은 문제들을 기록한다.

### 텍스트 불일치 — LLM 문맥 보정 이슈

PostDetailPage를 구현할 때, 목록 버튼 텍스트가 디자인과 다르게 들어간 적이 있었다.

```tsx
<button>{t("목록으로")}</button>
```

디자이너 피드백은 "버튼 텍스트가 '목록보기'인데요?"였다.

Figma를 다시 확인하니 원문은 `목록보기`가 맞았다. 이건 내가 임의로 오타를 낸 문제가 아니라, LLM이 문맥상 더 자연스럽다고 판단해 표현을 바꿔 넣은 케이스였다.

이후에는 텍스트 처리 원칙을 바꿨다. UI 카피는 자연어 보정 대상이 아니라 **디자인 원문 고정값**으로 취급하고, MCP로 추출한 문구를 그대로 사용했다.

### 디자인 디테일의 무한 수정

수치가 정확해도 "보이는 것"은 또 다른 문제다.

**Border Radius**: Figma에서 버튼의 `borderRadius`가 `4px`인 것과 `20px`인 것이 있었다. 4px는 살짝 둥근 사각형이고, 20px은 캡슐형이다. 처음에 전부 `borderRadius: 4`로 통일했다가, "이 버튼은 둥근 형태인데요"라는 피드백을 받았다.

**그라데이션**: Figma의 그라데이션 정보가 MCP를 통해 나오긴 하는데, CSS `linear-gradient`로 옮기면 미묘하게 다르게 보이는 경우가 있었다. 각도와 색상 중단점(color stop)의 표현 방식이 Figma와 CSS 사이에서 1:1로 대응되지 않는 부분이 있다.

**"타이틀이 너무작네.."** — `get_design_context`에서 가져온 `fontSize: 36`을 그대로 적용했는데, 실제 화면에서 Figma 미리보기보다 작게 느껴졌다. `lineHeight`나 `letterSpacing` 차이 때문이었다.

| 피드백 | 원인 |
|--------|------|
| "타이틀이 너무작네.." | lineHeight 누락 |
| "about us 위치 이상" | 컨테이너 padding 불일치 |
| "중간화면에 크기가 다름" | Tablet 뷰포트 미처리 |
| "버튼이 각져 보여요" | borderRadius 4px vs 20px 혼동 |

### 반복되는 MCP 호출

하나의 페이지를 구현하려면 MCP를 최소 3번 호출해야 한다. Desktop 프레임 한 번, Tablet 프레임 한 번, Mobile 프레임 한 번.

Hover 상태가 있는 버튼이나 카드가 있으면 추가 호출이 필요하다. Figma에서 컴포넌트의 `state=hover` variant를 별도로 조회해야 하기 때문이다.

```
PostListPage 구현 시 MCP 호출 횟수:
├── Desktop 프레임 ........... 1회
├── Tablet 프레임 ............ 1회
├── Mobile 프레임 ............ 1회
├── 카드 Hover 상태 .......... 1회
├── 더보기 버튼 Hover 상태 .... 1회
├── 누락된 정보 재조회 ........ 1~2회
└── 합계 .................... 6~7회
```

한 번에 필요한 모든 정보를 얻지 못하는 경우도 있다. 노드 트리가 깊으면 하위 컴포넌트의 세부 정보가 생략되기도 해서, 해당 NodeId를 다시 조회해야 한다.

## 대신 얻은 것들

### 워크플로우 표준화

`figma-workflow-guide.md`를 작성했다. Figma URL을 받으면 어떤 순서로 작업하는지, MCP의 어떤 도구를 쓰는지, 반응형은 어떤 원칙으로 처리하는지를 정리했다.

```
1. Figma URL 수신
2. get_design_context로 Desktop/Tablet/Mobile 정보 추출
3. 차이점 비교표 작성
4. constants.ts에 반응형 상수 정의
5. 컴포넌트 구현 (3단계 분기)
6. 인터랙션 효과 추가
7. 빌드 검증
```

이 순서를 따르면 실수가 줄었다. 특히 Phase 0(사전 분석)을 건너뛰고 바로 코딩부터 시작하면, 나중에 Tablet 레이아웃이 완전히 다르다는 걸 뒤늦게 발견하고 전부 뜯어고치는 일이 생긴다.

### 실수 패턴 문서화

스킬 파일에 "흔한 실수" 섹션을 만들었다. 한 번 실수한 건 다시 하지 않기 위해서다.

| 실수 | 원인 | 해결 |
|------|------|------|
| Tablet과 Mobile을 묶어서 처리 | "어차피 작은 화면이니까" | 3단계 분기 원칙 |
| 텍스트 내용 확인 소홀 | "비슷한 텍스트니까" | MCP 텍스트 복사-붙여넣기 |
| 인터랙션 효과 누락 | "hover는 나중에" | Phase 0에서 미리 확인 |
| borderRadius 혼동 | 4px와 20px 구분 안 함 | 상수로 분리 관리 |

### 디자인 시스템 상수화

`constants.ts`를 단일 소스로 유지한 이유는 명확하다. 디자인 값이 한곳에 모여 있어 수정 시 영향 범위를 빠르게 파악하고 반영할 수 있기 때문이다.

```typescript
// 페이지별 레이아웃 상수가 전부 여기에
export const POST_LIST_LAYOUT = { ... }
export const POST_DETAIL_LAYOUT = { ... }
export const CONTACT_PAGE_LAYOUT = { ... }
export const ABOUT_PAGE_LAYOUT = { ... }
export const CEO_MESSAGE_PAGE_LAYOUT = { ... }
export const HISTORY_PAGE_LAYOUT = { ... }
export const RND_PAGE_COMMON_LAYOUT = { ... }
```

"타이틀 폰트 크기 2px 키워주세요" 같은 요청이 오면, `constants.ts`에서 해당 값만 바꾸면 된다. 컴포넌트 코드를 뒤질 필요가 없다.

## 돌아보며

Figma MCP는 "마법"이 아니라 "도구"다.

도구가 해주는 건 Figma에서 일일이 클릭하며 확인하던 수치를 API 한 번으로 구조화된 데이터로 뽑아주는 것이다. `paddingTop: 80`, `fontSize: 18`, `color: #0E1825` — 같은 숫자를 수동으로 옮기는 대신 자동으로 추출한다. 값의 정확도가 올라가는 게 아니라, 같은 정확도를 훨씬 빠르게 달성한다.

하지만 도구가 해주지 않는 것들이 있다.

**"이게 맞는 건지" 확인하는 눈.** MCP가 `borderRadius: 4`를 반환했을 때, 이게 캡슐형 버튼인지 살짝 둥근 카드인지는 컨텍스트를 보고 판단해야 한다. Figma 전체 화면을 보면서 "이 컴포넌트가 디자인 안에서 어떤 역할을 하는지" 이해하는 건 사람의 몫이다.

**Tablet과 Mobile이 왜 다른지 이해하는 감각.** MCP가 Desktop은 6열, Tablet은 2열, Mobile은 1열이라고 알려줘도, "왜 Tablet에서 2열인지"를 이해하지 못하면 다음에 비슷한 페이지를 만들 때 또 실수한다.

**디자인 의도를 읽는 능력.** 디자이너가 특정 요소에 `24px` 간격을 준 건 그냥 숫자가 아니다. 시각적 그룹핑, 정보의 위계, 사용자 시선의 흐름 — 이런 의도가 담겨 있다. MCP는 `24px`이라는 숫자를 주지만, 그 숫자의 의미까지 알려주지는 않는다.

3주간 11개 이상의 페이지를 구현하면서, 가장 어려웠던 건 MCP 자체가 아니라 **디자인 디테일을 살리는 감각**이었다. 특히 백엔드 개발자였던 나에게는, 수치 매핑보다 미세 인터랙션(hover 전환감, 상태 변화 강도, 타이포 밸런스) 같은 영역이 훨씬 어려웠다. 기능은 동작하게 만들 수 있지만, 정확한 수치를 적용해도 "이 화면이 자연스러운지" 판단하는 건 또 다른 능력이다. 결국 디자이너의 피드백에 의존할 수밖에 없었다.

그래도 수동으로 하던 시절과 비교하면 효율이 다르다. Figma에서 요소 하나하나 클릭하며 값을 옮기던 시간이 줄어드니, 그 시간을 구조를 잡고 인터랙션 품질을 다듬는 데 쓸 수 있게 됐다. 피드백의 성격도 달라졌다. 단순히 숫자를 맞추는 확인보다, "이 부분 인터랙션이 빠졌다", "전환감이 어색하다", "Tablet에서 리듬이 다르다" 같은 품질 중심 피드백으로 이동했다.

그리고 이 과정을 `figma-responsive-implementation.md` 같은 스킬 파일로 정형화한 것도 컸다. 작업 순서와 검수 항목을 고정해두니, 페이지가 바뀌어도 품질 편차가 줄고 재작업이 확실히 줄었다.

1. **추출은 자동화해도, 판단은 사람이 한다** — MCP가 수동 작업을 줄여줘도, 그 값이 화면에서 어떻게 보이는지 확인하는 건 사람의 눈이다
2. **미세 인터랙션 품질은 별도 역량이다** — MCP로 수치를 가져오는 것과, 상태 전환의 자연스러움까지 맞추는 것은 다른 일이다
3. **삽질을 문서화하고 스킬로 정형화하면 자산이 된다** — 워크플로우 가이드와 스킬 파일은 3주간의 시행착오에서 나왔다. 같은 실수를 반복하지 않게 해주는 게 이 문서들의 가치다
4. **디자이너의 피드백은 패치 노트다** — 도구를 아무리 좋은 것을 써도, "이게 자연스러운 화면인지" 판단하는 감각은 경험에서 온다. 디자이너의 피드백을 귀찮아하지 말고, 그 피드백이 디자인 감각을 기르는 과정이라고 생각해야 한다
