---
title: "Tauri + Claude CLI로 만든 Multi-Agent 프로젝트 관리 도구"
date: 2026-01-20 22:00:00 +0900
categories: [AI]
tags: [tauri, rust, claude, multi-agent, streaming, session-management, desktop-app, retrospective]
description: "9개 전문 AI 에이전트를 오케스트레이션하는 데스크톱 도구를 Tauri(Rust) + React로 개발한 경험을 기록한다. CLI 추상화, 실시간 스트리밍, 세션 관리까지."
---

> 9개 전문 AI 에이전트를 오케스트레이션하는 데스크톱 도구를 Tauri(Rust) + React로 개발한 경험을 기록한다. CLI 추상화, 실시간 스트리밍, 세션 관리까지.

---

## 빠른 요약

- 문제: Claude CLI를 요구사항, 설계, 구현, 리뷰마다 따로 돌리면서 컨텍스트 단절과 세션 관리 문제가 생겼다.
- 맡은 범위: 9개 에이전트 역할 정의, Tauri + Rust + React 데스크톱 구조 설계, CLI 스트리밍과 세션 관리 구현.
- 결과: 역할 분리, stdout 스트리밍, 로컬 세션 추적이 되는 오케스트레이션 도구를 만들었다.

---

## 1. 상황

Claude CLI를 프로젝트마다 반복 실행하고 있었다.

새 기능을 만들 때의 흐름이 대략 이랬다. 터미널을 열고, Claude CLI로 요구사항을 정리해달라고 하고, 결과를 복사해서 다시 아키텍처 설계를 요청하고, 그걸 또 복사해서 구현을 요청한다. 리뷰도 따로, 문서화도 따로. 매번 프롬프트를 처음부터 작성하고, 컨텍스트를 수동으로 넘겨줘야 했다.

```
[터미널 1] claude "요구사항 분석해줘" → 결과 복사
[터미널 2] claude "아키텍처 설계해줘" → 결과 복사
[터미널 3] claude "구현해줘" → 결과 복사
[터미널 4] claude "리뷰해줘" → ...반복
```

문제는 세 가지였다.

| 문제 | 영향 |
|------|------|
| 컨텍스트 단절 | 에이전트 간 결과를 수동으로 전달, 정보 손실 |
| 역할 구분 없음 | 하나의 CLI에 "만능" 프롬프트, 결과 품질 들쭉날쭉 |
| 세션 미관리 | 이전 대화 내용 추적 불가, 동일 작업 반복 |

"만능 AI"에게 모든 걸 시키는 것보다, 역할을 좁힌 전문 에이전트가 더 나은 결과를 내는 경험이 쌓이고 있었다. 요구사항 분석만 하는 에이전트, 코드 리뷰만 하는 에이전트 — 역할이 명확할수록 출력 품질이 올라갔다.

그래서 만들기로 했다. 9개의 전문 에이전트를 오케스트레이션하는 데스크톱 도구를.

### 왜 Tauri인가

선택지는 세 가지였다.

| 선택지 | 장점 | 단점 |
|--------|------|------|
| 웹앱 (Next.js) | 배포 간편, 크로스 플랫폼 | 로컬 CLI 실행 불가, 서버 필요 |
| Electron | 생태계 성숙, Node.js | 번들 100MB+, 메모리 500MB+ |
| **Tauri** | **번들 4-13MB, Rust 백엔드** | 생태계 상대적으로 작음 |

Claude CLI는 로컬에서 실행되는 프로세스다. 웹앱에서는 로컬 CLI를 직접 실행할 수 없다. 데스크톱 앱이어야 했다.

Tauri를 고른 결정적 이유는 Rust 백엔드다. `std::process::Command`로 CLI 프로세스를 세밀하게 제어할 수 있고, Tokio 비동기 런타임으로 여러 에이전트를 동시에 돌릴 수 있다. 프론트엔드는 React + Tailwind CSS로 빠르게 만들 수 있다. Rust 백엔드의 성능과 React 프론트엔드의 유연성, 거기에 데스크톱 네이티브 실행까지. 조합이 맞았다.

> CLI 프로세스를 직접 관리해야 하는 도구에서 Tauri(Rust)는 자연스러운 선택이었다. `Command` + Tokio 조합으로 프로세스 생성, stdout 스트리밍, 비동기 실행을 모두 안정적으로 처리할 수 있다.

## 2. 9개 에이전트 설계

소프트웨어 개발 라이프사이클을 단계별로 쪼개서 9개 에이전트를 정의했다.

| 에이전트 | 역할 | 아이콘 |
|---------|------|--------|
| Orchestrator | 작업 분석 및 분배 | 🎯 |
| RequirementAnalyst | 요구사항 분석 | 📋 |
| UxDesigner | UX 설계 | 🎨 |
| TechArchitect | 기술 아키텍처 설계 | 🏗️ |
| Planner | 구현 계획 수립 | 📝 |
| TestDesigner | 테스트 케이스 설계 (TDD) | 🧪 |
| Developer | 코드 구현 | 💻 |
| Reviewer | 코드 리뷰 | 🔍 |
| Documenter | 문서화 | 📚 |

Rust 쪽에서 `AgentRole` enum으로 정의한다.

```rust
#[derive(Clone, Debug, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum AgentRole {
    Orchestrator,
    RequirementAnalyst,
    UxDesigner,
    TechArchitect,
    Planner,
    TestDesigner,
    Developer,
    Reviewer,
    Documenter,
}
```

### 시스템 프롬프트 관리

각 에이전트의 시스템 프롬프트는 `.claude/agents/{id}.md` 파일로 관리한다. 코드에 하드코딩하지 않고 파일로 분리한 이유는, 프롬프트를 수정할 때 앱을 다시 빌드하지 않기 위해서다.

```
.claude/agents/
├── orchestrator.md          # "당신은 작업을 분석하고 적절한 에이전트에게 분배하는 오케스트레이터입니다..."
├── requirement-analyst.md   # "당신은 요구사항을 분석하는 전문가입니다..."
├── ux-designer.md           # "당신은 UX 설계 전문가입니다..."
├── tech-architect.md        # "당신은 기술 아키텍처를 설계하는 전문가입니다..."
├── planner.md               # "당신은 구현 계획을 수립하는 전문가입니다..."
├── test-designer.md         # "당신은 TDD 기반 테스트 케이스를 설계하는 전문가입니다..."
├── developer.md             # "당신은 코드를 구현하는 개발자입니다..."
├── reviewer.md              # "당신은 코드를 리뷰하는 전문가입니다..."
└── documenter.md            # "당신은 기술 문서를 작성하는 전문가입니다..."
```

핵심은 각 에이전트의 역할 경계를 명확히 하는 것이다. RequirementAnalyst에게 코드를 작성하라고 하지 않고, Developer에게 UX 설계를 요청하지 않는다. 역할이 좁을수록 출력 품질이 높아진다.

### AgentStatus 상태 머신

각 에이전트는 상태 머신으로 관리된다.

```
              start()
    Idle ──────────────> Running
     ↑                    │   │
     │        resume()    │   │
     │    ┌──────────────┘   │
     │    ↓                   │
     │  Paused                │
     │                        │
     │   complete()/error()   │
     └────────── ←────────────┘
              Completed / Error
```

```rust
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub enum AgentStatus {
    Idle,
    Running,
    Paused,
    Completed,
    Error(String),
}
```

`AgentManager`가 `HashMap<AgentRole, AgentStatus>`로 전체 에이전트의 상태를 추적한다. 프론트엔드에서 대시보드를 렌더링할 때 이 맵을 조회해서, 어떤 에이전트가 실행 중이고 어떤 에이전트가 완료됐는지를 실시간으로 보여준다.

```rust
pub struct AgentManager {
    agents: HashMap<AgentRole, AgentStatus>,
    outputs: HashMap<AgentRole, Vec<String>>,
}

impl AgentManager {
    pub fn new() -> Self {
        let mut agents = HashMap::new();
        // 모든 에이전트를 Idle로 초기화
        for role in AgentRole::all() {
            agents.insert(role, AgentStatus::Idle);
        }
        Self {
            agents,
            outputs: HashMap::new(),
        }
    }

    pub fn set_status(&mut self, role: &AgentRole, status: AgentStatus) {
        self.agents.insert(role.clone(), status);
    }

    pub fn append_output(&mut self, role: &AgentRole, line: String) {
        self.outputs
            .entry(role.clone())
            .or_insert_with(Vec::new)
            .push(line);
    }
}
```

## 3. Rust 백엔드: CliProvider 트레이트

여기가 핵심 설계 결정이다.

처음에는 Claude CLI만 지원하면 됐다. 그런데 개발 도중에 Gemini CLI도 지원해야 하는 상황이 생겼다. 프로젝트에 따라 다른 LLM을 쓰고 싶은 경우가 있었다.

CLI 도구를 추상화하는 트레이트를 만들기로 했다.

```rust
pub trait CliProvider: Send + Sync {
    fn cli_name(&self) -> &str;              // "claude" or "gemini"
    fn skip_permission_flag(&self) -> &str;  // "--dangerously-skip-permissions" or "--yolo"
    fn display_name(&self) -> &str;
    fn working_dir(&self) -> Option<&str>;

    async fn run_streaming(
        &self,
        app: AppHandle,
        agent_role: &str,
        prompt: &str,
    ) -> Result<String, String>;

    async fn run(&self, prompt: &str) -> Result<String, String>;
}
```

Claude와 Gemini 구현체는 이렇게 된다.

```rust
pub struct ClaudeCli {
    working_dir: Option<String>,
}

impl CliProvider for ClaudeCli {
    fn cli_name(&self) -> &str { "claude" }
    fn skip_permission_flag(&self) -> &str { "--dangerously-skip-permissions" }
    fn display_name(&self) -> &str { "Claude CLI" }
    fn working_dir(&self) -> Option<&str> { self.working_dir.as_deref() }

    async fn run_streaming(
        &self,
        app: AppHandle,
        agent_role: &str,
        prompt: &str,
    ) -> Result<String, String> {
        // Claude CLI 실행 + stdout 스트리밍
        execute_cli_streaming(self, app, agent_role, prompt).await
    }

    async fn run(&self, prompt: &str) -> Result<String, String> {
        execute_cli(self, prompt).await
    }
}
```

```rust
pub struct GeminiCli {
    working_dir: Option<String>,
}

impl CliProvider for GeminiCli {
    fn cli_name(&self) -> &str { "gemini" }
    fn skip_permission_flag(&self) -> &str { "--yolo" }
    fn display_name(&self) -> &str { "Gemini CLI" }
    fn working_dir(&self) -> Option<&str> { self.working_dir.as_deref() }

    async fn run_streaming(
        &self,
        app: AppHandle,
        agent_role: &str,
        prompt: &str,
    ) -> Result<String, String> {
        execute_cli_streaming(self, app, agent_role, prompt).await
    }

    async fn run(&self, prompt: &str) -> Result<String, String> {
        execute_cli(self, prompt).await
    }
}
```

`CliProvider` 트레이트 덕분에, 새 LLM CLI가 나와도 구현체 하나만 추가하면 된다. 나머지 코드 — 스트리밍, 세션 관리, 에이전트 오케스트레이션 — 는 `CliProvider`를 통해 추상화되어 있으므로 수정할 필요가 없다.

> 다형성(polymorphism)의 클래식한 활용이다. "새 CLI가 추가될 때 기존 코드를 수정하지 않아도 된다"는 OCP(Open-Closed Principle)를 Rust 트레이트로 구현한 것. 이 결정이 프로젝트 전체에서 가장 큰 ROI를 냈다.

### Windows에서 콘솔 창 숨기기

데스크톱 앱에서 CLI를 subprocess로 실행하면, Windows에서는 기본적으로 콘솔 창이 뜬다. 사용자에게 검은 창이 번쩍거리는 건 좋지 않다.

```rust
#[cfg(target_os = "windows")]
const CREATE_NO_WINDOW: u32 = 0x08000000;

fn build_command(provider: &dyn CliProvider, prompt: &str) -> Command {
    let mut cmd = Command::new(provider.cli_name());

    cmd.arg("--print")
       .arg(provider.skip_permission_flag())
       .arg("--output-format").arg("text")
       .arg("--verbose")
       .arg("--prompt").arg(prompt);

    if let Some(dir) = provider.working_dir() {
        cmd.current_dir(dir);
    }

    // Windows에서 콘솔 창 미표시
    #[cfg(target_os = "windows")]
    {
        use std::os::windows::process::CommandExt;
        cmd.creation_flags(CREATE_NO_WINDOW);
    }

    cmd.stdout(std::process::Stdio::piped());
    cmd.stderr(std::process::Stdio::piped());

    cmd
}
```

`#[cfg(target_os = "windows")]` 조건부 컴파일로 Windows에서만 `CREATE_NO_WINDOW` 플래그를 설정한다. 크로스 플랫폼 빌드에서 이런 분기는 Rust의 `cfg` 매크로가 깔끔하게 처리해준다.

### Tokio 비동기 + stdout 스트리밍

CLI 프로세스의 stdout을 줄 단위로 읽어서 프론트엔드로 보내는 부분이다.

```rust
use tokio::io::{AsyncBufReadExt, BufReader};
use tokio::process::Command;

async fn execute_cli_streaming(
    provider: &dyn CliProvider,
    app: AppHandle,
    agent_role: &str,
    prompt: &str,
) -> Result<String, String> {
    let mut child = build_async_command(provider, prompt)
        .spawn()
        .map_err(|e| format!("{} 실행 실패: {}", provider.display_name(), e))?;

    let stdout = child.stdout.take()
        .ok_or("stdout 캡처 실패")?;

    let mut reader = BufReader::new(stdout).lines();
    let mut full_output = String::new();

    while let Ok(Some(line)) = reader.next_line().await {
        full_output.push_str(&line);
        full_output.push('\n');

        // Tauri 이벤트로 프론트엔드에 실시간 전송
        let _ = app.emit("mas:agent-output", serde_json::json!({
            "role": agent_role,
            "text": line,
        }));
    }

    let status = child.wait().await
        .map_err(|e| format!("프로세스 대기 실패: {}", e))?;

    if status.success() {
        Ok(full_output)
    } else {
        Err(format!("{} 실행 실패 (exit code: {:?})", provider.display_name(), status.code()))
    }
}
```

`BufReader::lines()`로 줄 단위 비동기 읽기를 하면서, 각 줄마다 `app.emit()`으로 Tauri 이벤트를 발생시킨다. 프론트엔드는 이 이벤트를 리스닝해서 실시간으로 화면을 업데이트한다.

## 4. 실시간 스트리밍 아키텍처

전체 파이프라인을 그림으로 보면 이렇다.

```
Rust subprocess (claude/gemini CLI)
    │ stdout line-by-line
    ▼
BufReader::lines() (Tokio async)
    │ 각 줄마다
    ▼
app.emit("mas:agent-output", { role, text })
    │ Tauri IPC (WebView ↔ Rust)
    ▼
React event listener (useEffect + listen)
    │ state 업데이트
    ▼
UI 업데이트 (자동 스크롤, 출력 버퍼링)
```

Tauri의 이벤트 시스템은 Rust에서 React로의 단방향 스트리밍 채널이다. `app.emit()`은 WebView의 JavaScript 런타임으로 이벤트를 보내고, React 쪽에서 `listen()`으로 받는다. HTTP도 WebSocket도 아닌, Tauri 자체의 IPC 브릿지를 사용하기 때문에 네트워크 오버헤드가 없다.

React 쪽의 이벤트 리스너는 이렇게 구성된다.

```typescript
import { listen } from '@tauri-apps/api/event';

interface AgentOutputEvent {
  role: string;
  text: string;
}

function useAgentOutput() {
  const [outputs, setOutputs] = useState<Record<string, string[]>>({});

  useEffect(() => {
    const unlisten = listen<AgentOutputEvent>('mas:agent-output', (event) => {
      const { role, text } = event.payload;
      setOutputs(prev => ({
        ...prev,
        [role]: [...(prev[role] || []), text],
      }));
    });

    return () => {
      unlisten.then(fn => fn());
    };
  }, []);

  return outputs;
}
```

줄 단위로 emit하기 때문에 사용자는 에이전트가 "생각하는 과정"을 실시간으로 볼 수 있다. ChatGPT나 Claude 웹에서 응답이 한 글자씩 나오는 것과 비슷한 경험을 CLI 기반 도구에서 재현한 것이다.

> 스트리밍의 핵심은 "줄 단위"다. `BufReader::lines()`가 줄바꿈을 만날 때마다 즉시 emit하므로, CLI가 한 줄을 출력하는 순간 프론트엔드에 반영된다. 전체 출력을 기다렸다가 한 번에 보여주는 것과 체감이 완전히 다르다.

## 5. 세션 & 체크포인트

### Session 관리

에이전트를 실행할 때마다 세션이 생성된다. UUID 기반 세션 ID로 식별하고, JSON 파일로 영속화한다.

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct Session {
    pub id: String,                            // UUID v4
    pub agent_role: String,                    // "Developer", "Reviewer" 등
    pub started_at: DateTime<Utc>,
    pub completed_at: Option<DateTime<Utc>>,
    pub input: String,                         // 사용자 프롬프트
    pub output: Option<String>,                // 에이전트 출력
    pub usage: TokenUsage,
    pub status: SessionStatus,
    pub working_dir: Option<String>,
}

#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct TokenUsage {
    pub input_tokens: u32,
    pub output_tokens: u32,
}

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub enum SessionStatus {
    Running,
    Paused,
    Completed,
    Failed(String),
}
```

세션의 라이프사이클은 이렇다.

```
create() → Running → complete() → Completed
                   → fail()     → Failed(reason)
                   → pause()    → Paused → resume() → Running
```

`SessionManager`가 세션의 생성, 조회, 상태 변경, 영속화를 담당한다.

```rust
pub struct SessionManager {
    sessions_dir: PathBuf,   // sessions/ 디렉토리
}

impl SessionManager {
    pub fn create_session(
        &self,
        agent_role: &str,
        input: &str,
        working_dir: Option<&str>,
    ) -> Result<Session, String> {
        let session = Session {
            id: uuid::Uuid::new_v4().to_string(),
            agent_role: agent_role.to_string(),
            started_at: Utc::now(),
            completed_at: None,
            input: input.to_string(),
            output: None,
            usage: TokenUsage { input_tokens: 0, output_tokens: 0 },
            status: SessionStatus::Running,
            working_dir: working_dir.map(|s| s.to_string()),
        };
        self.save_session(&session)?;
        Ok(session)
    }

    pub fn complete_session(
        &self,
        session_id: &str,
        output: &str,
    ) -> Result<Session, String> {
        let mut session = self.load_session(session_id)?;
        session.status = SessionStatus::Completed;
        session.completed_at = Some(Utc::now());
        session.output = Some(output.to_string());
        session.usage = self.estimate_usage(&session.input, output);
        self.save_session(&session)?;
        Ok(session)
    }

    fn estimate_usage(&self, input: &str, output: &str) -> TokenUsage {
        // 보수적 휴리스틱: 문자 수 / 3 ≈ 토큰 수
        TokenUsage {
            input_tokens: (input.chars().count() / 3) as u32,
            output_tokens: (output.chars().count() / 3) as u32,
        }
    }

    fn save_session(&self, session: &Session) -> Result<(), String> {
        let path = self.sessions_dir.join(format!("{}.json", session.id));
        let json = serde_json::to_string_pretty(session)
            .map_err(|e| format!("직렬화 실패: {}", e))?;
        std::fs::write(&path, json)
            .map_err(|e| format!("저장 실패: {}", e))
    }

    fn load_session(&self, session_id: &str) -> Result<Session, String> {
        let path = self.sessions_dir.join(format!("{}.json", session_id));
        let json = std::fs::read_to_string(&path)
            .map_err(|e| format!("읽기 실패: {}", e))?;
        serde_json::from_str(&json)
            .map_err(|e| format!("역직렬화 실패: {}", e))
    }
}
```

토큰 사용량 추정은 `(chars.count() / 3) as u32`로 한다. 정확한 토큰 수를 알려면 토크나이저를 돌려야 하지만, CLI 출력에서 토큰 수를 직접 가져올 수 없는 상황에서 보수적 휴리스틱을 적용했다. 한국어는 토큰 효율이 영어보다 낮으므로 실제보다 적게 나올 수 있지만, 대략적인 사용량 추적에는 충분하다.

### Checkpoint 시스템

세션의 중간 상태를 스냅샷으로 저장하고 복원하는 기능이다. 긴 작업 중에 "여기까지의 진행 상태를 저장해두고 싶다"는 요구에서 나왔다.

```rust
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct Checkpoint {
    pub id: String,               // UUID v4
    pub session_id: String,       // 연관 세션 ID
    pub created_at: DateTime<Utc>,
    pub description: String,      // "UX 설계 완료 시점" 등
    pub session_state: Session,   // 전체 세션 상태 복제
}
```

```rust
pub struct CheckpointManager {
    checkpoints_dir: PathBuf,
}

impl CheckpointManager {
    /// 현재 세션 상태를 스냅샷으로 저장
    pub fn save_checkpoint(
        &self,
        session: &Session,
        description: &str,
    ) -> Result<Checkpoint, String> {
        let checkpoint = Checkpoint {
            id: uuid::Uuid::new_v4().to_string(),
            session_id: session.id.clone(),
            created_at: Utc::now(),
            description: description.to_string(),
            session_state: session.clone(),
        };
        let path = self.checkpoints_dir.join(format!("{}.json", checkpoint.id));
        let json = serde_json::to_string_pretty(&checkpoint)
            .map_err(|e| format!("직렬화 실패: {}", e))?;
        std::fs::write(&path, json)
            .map_err(|e| format!("저장 실패: {}", e))?;
        Ok(checkpoint)
    }

    /// 체크포인트로부터 세션 상태 복원
    pub fn restore_checkpoint(&self, checkpoint_id: &str) -> Result<Session, String> {
        let path = self.checkpoints_dir.join(format!("{}.json", checkpoint_id));
        let json = std::fs::read_to_string(&path)
            .map_err(|e| format!("읽기 실패: {}", e))?;
        let checkpoint: Checkpoint = serde_json::from_str(&json)
            .map_err(|e| format!("역직렬화 실패: {}", e))?;
        Ok(checkpoint.session_state)
    }

    /// 특정 세션의 모든 체크포인트 조회
    pub fn list_checkpoints(&self, session_id: &str) -> Result<Vec<Checkpoint>, String> {
        // checkpoints_dir 내의 JSON 파일을 순회하며 session_id가 일치하는 것만 반환
        // ...
    }

    /// 체크포인트 삭제
    pub fn delete_checkpoint(&self, checkpoint_id: &str) -> Result<(), String> {
        let path = self.checkpoints_dir.join(format!("{}.json", checkpoint_id));
        std::fs::remove_file(&path)
            .map_err(|e| format!("삭제 실패: {}", e))
    }
}
```

체크포인트는 세션 상태의 전체 복제(`session.clone()`)를 저장한다. 부분 복제보다 저장 공간을 더 쓰지만, 복원 로직이 단순하고 데이터 정합성 문제가 없다. JSON 파일 하나에 세션의 모든 정보가 담겨 있으므로, 파일 하나만 읽으면 완전한 상태 복원이 가능하다.

### Dexie(IndexedDB) 프론트엔드 저장소

Rust 백엔드의 파일 기반 저장소 외에, 프론트엔드에서도 로컬 데이터를 관리한다. 프로젝트 메타데이터, 목표, 태스크, 일일 평가 같은 UI 레벨 데이터는 IndexedDB에 저장한다.

```typescript
import Dexie, { type Table } from 'dexie';

interface Project {
  id?: number;
  name: string;
  description: string;
  workingDir: string;
  createdAt: Date;
  updatedAt: Date;
}

interface Goal {
  id?: number;
  projectId: number;
  title: string;
  status: 'active' | 'completed' | 'archived';
  createdAt: Date;
}

interface Task {
  id?: number;
  goalId: number;
  projectId: number;
  title: string;
  agentRole: string;
  status: 'pending' | 'in-progress' | 'done';
  sessionId?: string;  // Rust 세션과 연결
}

interface DailyEvaluation {
  id?: number;
  projectId: number;
  date: string;       // "2026-02-21"
  summary: string;
  score: number;       // 1-10
}

class MasDatabase extends Dexie {
  projects!: Table<Project>;
  goals!: Table<Goal>;
  tasks!: Table<Task>;
  dailyEvaluations!: Table<DailyEvaluation>;

  constructor() {
    super('mas-db');
    this.version(1).stores({
      projects: '++id, name, createdAt',
      goals: '++id, projectId, status',
      tasks: '++id, goalId, projectId, agentRole, status',
      dailyEvaluations: '++id, projectId, date',
    });
  }
}

export const db = new MasDatabase();
```

데이터 정리 유틸리티도 함께 만들었다.

```typescript
// 프로젝트 삭제 시 연관 데이터 전체 삭제 (Cascade Delete)
async function cascadeDeleteProject(projectId: number): Promise<void> {
  await db.transaction('rw', [db.projects, db.goals, db.tasks, db.dailyEvaluations], async () => {
    await db.dailyEvaluations.where('projectId').equals(projectId).delete();
    await db.tasks.where('projectId').equals(projectId).delete();
    await db.goals.where('projectId').equals(projectId).delete();
    await db.projects.delete(projectId);
  });
}

// 고아 레코드 정리 (프로젝트가 삭제된 후 남은 Goal, Task 등)
async function cleanOrphanedData(): Promise<number> {
  const projectIds = (await db.projects.toArray()).map(p => p.id!);
  let cleaned = 0;

  const orphanedGoals = await db.goals
    .filter(g => !projectIds.includes(g.projectId))
    .toArray();
  cleaned += orphanedGoals.length;
  await db.goals.bulkDelete(orphanedGoals.map(g => g.id!));

  const orphanedTasks = await db.tasks
    .filter(t => !projectIds.includes(t.projectId))
    .toArray();
  cleaned += orphanedTasks.length;
  await db.tasks.bulkDelete(orphanedTasks.map(t => t.id!));

  return cleaned;
}

// IndexedDB 사용량 모니터링
async function getStorageQuota(): Promise<{ usage: number; quota: number }> {
  if (navigator.storage && navigator.storage.estimate) {
    const { usage = 0, quota = 0 } = await navigator.storage.estimate();
    return { usage, quota };
  }
  return { usage: 0, quota: 0 };
}
```

`cascadeDeleteProject()`는 트랜잭션 안에서 연관 테이블을 순서대로 삭제한다. IndexedDB에는 FK 제약이 없으므로 애플리케이션 레벨에서 직접 관리해야 한다. `cleanOrphanedData()`는 데이터 정합성이 깨졌을 때 정리하는 안전장치다.

## 6. 프론트엔드: AgentDashboard

대시보드의 핵심은 9개 에이전트의 상태를 한눈에 보여주는 것이다.

```tsx
import { useState, useEffect } from 'react';
import { listen } from '@tauri-apps/api/event';
import { invoke } from '@tauri-apps/api/core';

interface AgentCardProps {
  role: string;
  icon: string;
  status: 'idle' | 'running' | 'completed' | 'error';
  output: string[];
}

function AgentCard({ role, icon, status, output }: AgentCardProps) {
  const statusColor = {
    idle: 'bg-gray-100 dark:bg-gray-800',
    running: 'bg-blue-50 dark:bg-blue-900/30 border-blue-400',
    completed: 'bg-green-50 dark:bg-green-900/30 border-green-400',
    error: 'bg-red-50 dark:bg-red-900/30 border-red-400',
  }[status];

  return (
    <div className={`rounded-lg border-2 p-4 ${statusColor} transition-all`}>
      <div className="flex items-center justify-between mb-2">
        <span className="text-lg">{icon}</span>
        <span className="text-sm font-medium">{role}</span>
        {status === 'running' && (
          <span className="animate-pulse text-blue-500">●</span>
        )}
      </div>
      <div className="h-32 overflow-y-auto font-mono text-xs">
        {output.map((line, i) => (
          <div key={i} className="text-gray-700 dark:text-gray-300">{line}</div>
        ))}
      </div>
    </div>
  );
}
```

3열 그리드 레이아웃으로 9개 에이전트를 배치한다.

```tsx
function AgentDashboard() {
  const [agents, setAgents] = useState<Record<string, AgentCardProps>>({});
  const [workingDir, setWorkingDir] = useState<string>('');

  useEffect(() => {
    const unlisten = listen<{ role: string; text: string }>(
      'mas:agent-output',
      (event) => {
        const { role, text } = event.payload;
        setAgents(prev => ({
          ...prev,
          [role]: {
            ...prev[role],
            status: 'running',
            output: [...(prev[role]?.output || []), text],
          },
        }));
      }
    );
    return () => { unlisten.then(fn => fn()); };
  }, []);

  const handleRunAgent = async (role: string, prompt: string) => {
    await invoke('run_agent', {
      role,
      prompt,
      workingDir: workingDir || undefined,
    });
  };

  const agentDefs = [
    { role: 'Orchestrator', icon: '🎯' },
    { role: 'RequirementAnalyst', icon: '📋' },
    { role: 'UxDesigner', icon: '🎨' },
    { role: 'TechArchitect', icon: '🏗️' },
    { role: 'Planner', icon: '📝' },
    { role: 'TestDesigner', icon: '🧪' },
    { role: 'Developer', icon: '💻' },
    { role: 'Reviewer', icon: '🔍' },
    { role: 'Documenter', icon: '📚' },
  ];

  return (
    <div className="p-6">
      {/* 작업 디렉토리 설정 */}
      <div className="mb-6 flex gap-2">
        <input
          type="text"
          value={workingDir}
          onChange={e => setWorkingDir(e.target.value)}
          placeholder="작업 디렉토리 경로"
          className="flex-1 rounded border px-3 py-2 dark:bg-gray-800"
        />
      </div>

      {/* 에이전트 그리드 (3열) */}
      <div className="grid grid-cols-3 gap-4">
        {agentDefs.map(def => (
          <AgentCard
            key={def.role}
            role={def.role}
            icon={def.icon}
            status={agents[def.role]?.status || 'idle'}
            output={agents[def.role]?.output || []}
          />
        ))}
      </div>
    </div>
  );
}
```

부가 기능으로 에이전트 프롬프트 조회 모달, 실행 로그 뷰어, Dark mode(Tailwind CSS `dark:` 프리픽스)를 지원한다. 자동 스크롤은 출력이 추가될 때 `scrollIntoView()`로 마지막 줄을 따라가는 방식이다.

```tsx
function OutputViewer({ output }: { output: string[] }) {
  const bottomRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [output]);

  return (
    <div className="h-64 overflow-y-auto bg-gray-900 rounded p-3 font-mono text-sm">
      {output.map((line, i) => (
        <div key={i} className="text-green-400">{line}</div>
      ))}
      <div ref={bottomRef} />
    </div>
  );
}
```

## 7. 돌아보며

### Local-first AI 도구의 장점

이 도구의 데이터는 전부 로컬에 남는다. 세션 기록, 체크포인트, 프로젝트 메타데이터 — 외부 서버로 나가지 않는다. API 키도 앱이 관리하지 않는다. Claude CLI와 Gemini CLI가 각자의 인증을 처리하므로, 이 도구는 CLI를 호출만 할 뿐 자격 증명을 모른다.

| 특성 | 설명 |
|------|------|
| 데이터 로컬 보관 | 세션, 체크포인트, 프로젝트 데이터가 로컬 디스크에만 저장 |
| API 키 불필요 | CLI가 자체적으로 인증 관리, 앱은 자격 증명을 다루지 않음 |
| 오프라인 세션 리뷰 | 인터넷 없이도 이전 세션 기록 조회 가능 |
| 프라이버시 | 코드와 프롬프트가 앱 개발자 서버에 전송되지 않음 |

### 한계

현재 한계도 뚜렷하다.

**CLI 프로세스 관리의 복잡성.** subprocess를 직접 관리하는 건 예상보다 까다롭다. 프로세스가 hang 걸리거나, 예상치 못한 stderr 출력이 나오거나, exit code가 비정상인 경우를 모두 처리해야 한다. 특히 Windows에서 프로세스 종료가 깔끔하지 않을 때가 있다.

**토큰 추정의 부정확성.** `chars.count() / 3`은 어디까지나 휴리스틱이다. 한국어, 코드, 영어가 섞인 프롬프트에서 실제 토큰 수와 차이가 날 수 있다. CLI가 토큰 사용량을 직접 리포트하는 API가 있다면 정확한 수치를 쓸 수 있겠지만, 현재는 근사값으로 만족하고 있다.

**에이전트 간 컨텍스트 전달.** 현재는 에이전트 A의 출력을 에이전트 B의 입력으로 수동 연결하는 구조다. Orchestrator가 자동으로 파이프라이닝하는 기능은 아직 구현 중이다.

### 그리고 멈췄다

개발 도중, 멀티에이전트 프레임워크들이 쏟아져 나왔다.

- **Claude Code**가 자체적으로 `Task`/subagent 기능을 도입했다. 메인 에이전트가 하위 에이전트를 생성하고, 결과를 자동으로 수집한다. CLI를 subprocess로 감싸서 수동으로 하던 것을 프레임워크가 네이티브로 지원하기 시작한 것이다.
- **OpenAI Agents SDK**가 나왔다. 에이전트 정의, 핸드오프, 가드레일을 프레임워크 레벨에서 제공한다.
- **Google ADK (Agent Development Kit)**도 등장했다. 멀티에이전트 오케스트레이션을 공식 지원한다.

내가 `CliProvider` 트레이트로 추상화하고, `AgentManager`로 상태 머신을 만들고, Tauri 이벤트로 스트리밍 파이프라인을 구축한 것 — 이 모든 것을 프레임워크가 몇 줄의 설정으로 해결해주기 시작했다.

전형적인 **Build vs Buy** 문제다. 직접 만든 솔루션이 제품화되기 전에 에코시스템이 따라잡으면, 커스텀 솔루션의 존재 이유가 줄어든다. 유지보수 비용, 버그 수정, 새 LLM 지원 — 프레임워크를 쓰면 이 모든 것이 커뮤니티의 몫이 된다. 1인 개발자가 감당하기엔 프레임워크의 진화 속도가 더 빨랐다.

그래서 멈췄다.

하지만 만들면서 배운 것은 남는다. Rust 트레이트로 CLI 도구를 추상화하는 패턴, Tokio 비동기 스트리밍 아키텍처, 상태 머신 기반 세션 관리 — 이것들은 어떤 프레임워크를 쓰든 밑바탕이 되는 설계 감각이다. 프레임워크가 "무엇을" 해주는지 이해하려면, 한 번은 직접 만들어봐야 한다.

### 핵심 설계 결정 회고

프로젝트 전체를 돌아보면, **`CliProvider` 트레이트 패턴**이 가장 큰 영향을 미친 결정이었다. 처음에는 "Claude만 쓸 건데 추상화가 필요한가?"라고 생각했지만, Gemini CLI 지원을 추가하는 데 걸린 시간이 30분이 안 됐을 때 확신이 생겼다. 새 구현체 하나만 추가하면 됐으니까.

두 번째로 중요한 결정은 **9개 에이전트로의 역할 분리**다. 이건 "만능 AI"보다 역할을 좁힌 전문 에이전트가 더 나은 결과를 내는 경험에서 비롯됐다. 시스템 프롬프트에 "당신은 코드 리뷰 전문가입니다. 구현은 하지 마세요."라고 명시하는 것만으로도, 리뷰 품질이 체감할 수 있을 만큼 올라갔다.

세 번째는 **줄 단위 실시간 스트리밍**이다. 전체 출력을 기다렸다가 한 번에 보여주는 것과, 줄이 나올 때마다 바로 보여주는 것은 체감이 완전히 다르다. 에이전트가 "생각하고 있다"는 피드백을 주는 것만으로 사용자 경험이 크게 개선된다.

> 1. **CliProvider 트레이트** — LLM 교체 비용을 최소화. 새 CLI 도구를 30분 안에 통합 가능
> 2. **역할 분리** — "만능"보다 "전문"이 낫다. 시스템 프롬프트의 역할 경계가 출력 품질을 결정한다
> 3. **줄 단위 스트리밍** — CLI 도구에서도 ChatGPT 수준의 실시간 피드백 경험을 구현할 수 있다
> 4. **Local-first** — 데이터가 로컬에 남고, 앱이 자격 증명을 모르는 구조가 보안과 프라이버시 면에서 유리하다

CLI를 반복 실행하던 것에서, 9개 전문 에이전트를 대시보드에서 오케스트레이션하는 것으로. 도구의 형태가 바뀌면 작업 방식이 바뀐다. 비록 이 도구가 제품으로 완성되지는 못했지만, 직접 만들어보면서 얻은 설계 감각 — 트레이트 추상화, 스트리밍 아키텍처, 세션 관리 — 은 프레임워크를 쓸 때도 "왜 이렇게 동작하는지"를 이해하는 기반이 된다. 때로는 완성하지 못한 프로젝트가 가장 많이 가르쳐준다.
