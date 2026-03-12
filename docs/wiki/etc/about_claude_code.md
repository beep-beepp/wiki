---
title: "About Claude Code"
parent: "ETC"
permalink: /etc/about-claude-code
nav_order: 2
---

# Claude Code 고급 기능 가이드

기본 사용법(채팅, 파일 편집, 터미널 실행)은 다 알고 있다는 전제로, 프로젝트에서 실제로 써본 **고급 기능** 위주로 정리했다.

---

## 1. CLAUDE.md — 프로젝트 컨텍스트 주입

Claude Code가 세션 시작 시 **자동으로 읽는** 마크다운 파일. 프로젝트의 구조, 컨벤션, 명령어 등을 매번 설명하지 않아도 된다.

### 로딩 계층

```
1. /Library/Application Support/ClaudeCode/CLAUDE.md   ← 시스템 (Managed Policy)
2. ~/.claude/CLAUDE.md                                  ← 유저 전역
3. <project>/CLAUDE.md                                  ← 프로젝트 루트
4. <project>/<subdir>/CLAUDE.md                         ← 하위 디렉토리
```

**상위 → 하위** 순서로 모두 로딩된다. 하위가 상위를 덮어쓰는 게 아니라 **누적**된다.

### 실전 예시

```markdown
# CLAUDE.md (프로젝트 루트)

## Repository Structure
| Project     | Stack              |
|-------------|--------------------|
| dcs-ai/     | Next.js 15 + NestJS |
| dcs-ai-cli/ | Rust (clap + tokio) |

## Commands
pnpm dev        # 전체 실행
pnpm dev:client # 클라이언트만

## Architecture
Frontend: Feature-Sliced Design (FSD)
상위 레이어가 하위만 import. 역방향 금지.

## Coding Conventions
- API prefix: /server (not /api)
- Auth: NextAuth → Azure AD → JWT httpOnly cookie
```

Claude는 이 파일을 읽고 "이 프로젝트는 NestJS를 쓰고, FSD 구조이며, API prefix가 /server" 같은 맥락을 파악한 상태로 대화를 시작한다.

### Managed Policy (시스템 레벨)

조직 전체에 적용할 정책을 시스템 경로에 배포할 수 있다:

```bash
# macOS
/Library/Application Support/ClaudeCode/CLAUDE.md

# Linux
/etc/claude-code/CLAUDE.md
```

사용자가 수정할 수 없는 위치이므로, **회사 표준 정책**을 강제할 때 유용하다. 예를 들어 응답 포맷 통일, 보안 가이드라인 등.

---

## 2. Hooks — 라이프사이클 이벤트 자동화

Claude Code가 특정 **이벤트** 발생 시 자동으로 실행하는 셸 명령어. 설정만 해두면 사용자 개입 없이 동작한다.

### Hook 이벤트 종류

| 이벤트 | 발생 시점 | 용도 예시 |
|--------|----------|----------|
| `UserPromptSubmit` | 사용자가 프롬프트 전송 시 | 프롬프트 로깅, 입력 검증 |
| `PreToolUse` | 도구 실행 **직전** | 실행 시간 측정 시작, 위험 명령 차단 |
| `PostToolUse` | 도구 실행 **직후** | 결과 로깅, 에러 캡처 |
| `PostToolUseFailure` | 도구 실행 **실패** 시 | 에러 리포트 전송 |
| `Stop` | Claude 응답 완료 시 | 버퍼된 이벤트 전송, 상태 정리 |
| `SubagentStop` | 서브에이전트 종료 시 | 서브에이전트 에러 캡처 |
| `SessionEnd` | 세션 종료 시 | 자동 업데이트, 정리 작업 |

### Hook 설정 (hooks.json)

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "node send-event.js pre-tool"
          }
        ],
        "description": "Record tool start time"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'input=$(cat); echo \"$input\" | grep -q error && echo \"$input\" | node capture-error.js || true'"
          }
        ],
        "description": "Capture errors from tool results"
      }
    ]
  }
}
```

### matcher 패턴

- `""` (빈 문자열) — **모든** 도구에 반응
- `"*"` — 모든 도구 (와일드카드)
- `"Bash"` — Bash 도구만
- `"mcp__dcsai__*"` — 특정 MCP 도구만

### Hook 데이터 흐름

Hook 스크립트는 **stdin**으로 이벤트 데이터(JSON)를 받는다:

```
Claude Code ──(이벤트 발생)──→ stdin으로 JSON 전달 ──→ Hook 스크립트 실행
```

```javascript
// send-event.js 예시
const input = JSON.parse(require('fs').readFileSync('/dev/stdin', 'utf8'));

if (eventType === 'pre-tool') {
  // input.tool_name, input.tool_input 등 접근 가능
  recordStartTime(input.session_id, Date.now());
}

if (eventType === 'post-tool') {
  // MCP 도구 사용 감지 → 세션 마킹
  if (input.tool_name?.startsWith('mcp__')) {
    markSessionAsMcp(input.session_id);
  }
}
```

### 핵심 원칙

1. **에러를 삼켜야 한다** — Hook이 실패하면 Claude Code 전체가 블로킹된다. 항상 `|| true`로 감싼다.
2. **빨라야 한다** — Hook은 동기 실행이므로 무거운 작업은 비동기로 넘기거나 버퍼링한다.
3. **상태는 파일로** — `~/.claude/`에 JSON 파일로 상태를 저장하고 읽는다.

---

## 3. MCP (Model Context Protocol) — 외부 도구 연결

Claude Code에 **외부 API/도구를 연결**하는 프로토콜. Claude가 직접 외부 시스템을 호출할 수 있게 된다.

### 개념

```
사용자: "매출 데이터 조회해줘"
         │
         ▼
Claude Code: "MCP 도구가 있네 → 호출하자"
         │
         ▼
MCP Client ──HTTP/SSE──→ MCP Server (우리 백엔드)
         │                    │
         │                    ├→ DB 조회
         │                    ├→ API 호출
         │                    └→ 결과 반환
         │
         ▼
Claude Code: "결과를 바탕으로 답변 생성"
```

기존에는 "데이터 복사해서 붙여넣기 → Claude에게 분석 요청"이었다면, MCP는 **Claude가 직접 데이터를 가져온다**.

### MCP 서버 설정

프로젝트의 `.mcp.json` 또는 플러그인의 `.mcp.json`에 설정:

```json
{
  "mcpServers": {
    "dcsai": {
      "type": "http",
      "url": "https://dcsai.fnf.co.kr/server/mcp"
    }
  }
}
```

### MCP 도구 호출 흐름

```
1. Claude가 사용 가능한 MCP 도구 목록을 받음
2. 사용자 요청에 맞는 도구를 선택
3. MCP 프로토콜로 서버에 요청
4. 서버가 실제 작업 수행 후 결과 반환
5. Claude가 결과를 해석하여 사용자에게 전달
```

도구 이름은 `mcp__<plugin>_<namespace>__<tool_name>` 패턴을 따른다:

```
mcp__plugin_dcs-ai-common_dcsai__execute_kg_api_to_context
mcp__plugin_dcs-ai-common_dcsai__get_artifacts
mcp__plugin_dcs-ai-common_dcsai__get_intents
```

### MCP 인증

MCP OAuth를 통해 인증하면 토큰이 `~/.claude/.credentials.json`에 저장된다. 이후 MCP 호출과 Hook 스크립트 모두 이 토큰을 사용한다.

```
사용자 ──OAuth 로그인──→ 토큰 발급
                           │
                ┌──────────┼──────────┐
                ▼                      ▼
         MCP 호출 시 자동 사용    Hook에서 API 호출 시 사용
```

### Built-in MCP 연결 (claude.ai 통합)

Claude Code는 claude.ai의 MCP 커넥터도 지원한다:

```
claude.ai Slack      → Slack 채널 읽기/메시지 보내기
claude.ai Notion     → Notion 페이지 검색/생성
claude.ai Snowflake  → SQL 쿼리 실행
claude.ai Microsoft 365 → 이메일/캘린더 검색
```

설정은 claude.ai 웹에서 한 번 연결하면, Claude Code에서도 사용 가능하다.

---

## 4. Plugin System — 확장 패키지

Commands, Agents, Skills, Rules, Hooks를 **하나의 패키지로 묶어** 배포하는 시스템.

### 플러그인 구조

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json        ← 메타데이터 (이름, 버전, 설명)
├── .mcp.json              ← MCP 서버 설정
├── commands/              ← 슬래시 커맨드 (/plugin:Command)
│   ├── Init.md
│   ├── Deploy.md
│   └── Issue.md
├── agents/                ← 특수 목적 에이전트
│   ├── build-error-resolver.md
│   └── figma-migrator.md
├── skills/                ← 지식 모듈 (SKILL.md)
│   ├── data-processing/
│   └── chart-visualization/
├── rules/                 ← 코딩 규칙 (.claude/rules/에 배포)
│   ├── common/
│   ├── python/
│   └── typescript/
├── hooks/                 ← 라이프사이클 훅
│   ├── hooks.json
│   └── send-event.js
└── scripts/               ← 설치/업데이트 스크립트
    └── install.sh
```

### 플러그인 설치/사용

```bash
# 마켓플레이스 등록 (1회)
/plugin marketplace add https://github.com/org/my-plugin.git

# 플러그인 설치
/plugin install my-plugin@marketplace-name

# 커맨드 실행
/my-plugin:Init
/my-plugin:Deploy
```

### Commands vs Agents vs Skills vs Rules

| 구성요소 | 형태 | 트리거 | 용도 |
|---------|------|--------|------|
| **Commands** | Markdown 프롬프트 | `/plugin:Command` 입력 시 | 워크플로우 자동화 (배포, PR 생성 등) |
| **Agents** | Markdown 프롬프트 | Claude가 필요 시 자동 호출 | 특수 작업 위임 (빌드 에러 수정, Figma 변환) |
| **Skills** | Markdown 문서 | Claude가 관련 작업 시 참조 | 도메인 지식 (차트 그리는 법, 데이터 처리 법) |
| **Rules** | Markdown 문서 | 항상 컨텍스트에 포함 | 코딩 컨벤션 강제 (네이밍, 패턴, 테스트) |

### 실전: 에이전트 정의 예시

```markdown
# build-error-resolver.md

당신은 빌드 에러를 수정하는 전문 에이전트입니다.

## 행동 규칙
1. 에러 메시지를 분석한다
2. 관련 파일만 최소한으로 수정한다
3. 수정 후 빌드를 다시 실행하여 검증한다
4. 새로운 에러가 발생하면 반복한다

## 모델
Opus (정확도 우선)
```

Claude는 빌드 에러를 만나면 이 에이전트를 **서브프로세스로 실행**하여 에러 수정을 위임한다. 메인 컨텍스트를 오염시키지 않는다.

---

## 5. Subagents — 병렬 작업 위임

Claude Code는 복잡한 작업을 **서브에이전트에게 위임**할 수 있다. 각 서브에이전트는 독립된 컨텍스트에서 실행된다.

### 내장 서브에이전트 타입

| 타입 | 모델 | 용도 |
|------|------|------|
| `general-purpose` | 기본 | 범용 (검색, 코드 실행, 멀티스텝) |
| `Explore` | 기본 | 코드베이스 탐색 전문 |
| `Plan` | 기본 | 구현 계획 설계 |

### 왜 서브에이전트를 쓰는가

```
메인 에이전트 (컨텍스트 윈도우 = 한정적)
   │
   ├─→ 서브에이전트 A: "src/ 디렉토리에서 인증 관련 코드 찾아"
   │     └─→ 50개 파일 탐색 → 결과 요약만 반환
   │
   ├─→ 서브에이전트 B: "테스트 실행하고 결과 알려줘"
   │     └─→ 전체 테스트 실행 → 실패 건만 반환
   │
   └─→ 메인: 요약된 결과만 받아서 판단
```

서브에이전트가 없으면 50개 파일 내용이 전부 메인 컨텍스트에 쌓인다. 서브에이전트를 쓰면 **요약된 결과만** 메인으로 돌아온다.

### 병렬 실행

독립적인 작업은 **동시에** 여러 서브에이전트를 실행할 수 있다:

```
메인 ──┬──→ 에이전트 A (파일 탐색)     ──┐
       ├──→ 에이전트 B (테스트 실행)    ──┤──→ 결과 수집 → 종합 판단
       └──→ 에이전트 C (문서 검색)      ──┘
```

### Worktree 격리

`isolation: "worktree"` 옵션으로 서브에이전트를 **별도 git worktree**에서 실행할 수 있다. 메인 작업 디렉토리를 건드리지 않고 독립적으로 코드를 수정/테스트한다.

---

## 6. Memory — 대화 간 기억 유지

Claude Code 세션은 기본적으로 **독립적**이다. 이전 대화 내용을 기억하지 못한다. Memory 시스템으로 이를 보완한다.

### Memory 저장 위치

```
~/.claude/projects/<project-path>/memory/
├── MEMORY.md              ← 인덱스 (항상 로딩됨)
├── user_role.md           ← 유저 정보
├── feedback_testing.md    ← 피드백
└── project_auth.md        ← 프로젝트 맥락
```

### Memory 타입

| 타입 | 용도 | 예시 |
|------|------|------|
| `user` | 사용자 역할/선호 | "시니어 개발자, Go 전문, React 초보" |
| `feedback` | 교정/지시 | "테스트에서 DB 모킹 금지. 실제 DB 사용" |
| `project` | 프로젝트 맥락 | "3/5부터 머지 프리즈, 모바일 릴리스" |
| `reference` | 외부 시스템 참조 | "버그 트래커는 Linear INGEST 프로젝트" |

### 기억시키기

```
사용자: "앞으로 커밋 메시지는 한국어로 작성해줘"
Claude: [feedback 메모리에 저장]

# 다음 세션에서...
사용자: "이 변경사항 커밋해줘"
Claude: "feat: 로그인 페이지 반응형 레이아웃 적용"  ← 한국어로 작성
```

---

## 7. Permissions — 도구 실행 제어

Claude Code가 실행할 수 있는 도구/명령을 제어한다.

### 설정 파일

```
~/.claude/settings.json                         ← 유저 전역
<project>/.claude/settings.json                 ← 프로젝트 (git 공유)
<project>/.claude/settings.local.json           ← 프로젝트 로컬 (gitignore)
```

### 허용 패턴

```json
{
  "permissions": {
    "allow": [
      "Bash(git:*)",           // git 명령 전체 허용
      "Bash(pnpm dev:*)",     // pnpm dev 관련 허용
      "Bash(cargo build:*)",  // cargo build 허용
      "Read(//Users/me/project/**)",  // 프로젝트 파일 읽기 허용
      "WebFetch(domain:api.example.com)"  // 특정 도메인 fetch 허용
    ]
  }
}
```

패턴 문법:
- `Tool(pattern)` — 특정 도구의 특정 패턴
- `*` — 와일드카드
- `**` — 재귀 와일드카드 (디렉토리 전체)

허용하지 않은 도구는 **실행 전 사용자에게 확인**을 요청한다.

---

## 8. 실전 조합 — 이 모든 게 어떻게 엮이는가

DCS AI 프로젝트에서 실제로 이 기능들이 어떻게 조합되는지:

```
사용자가 Claude Code 실행
    │
    ├─ CLAUDE.md 로딩
    │   ├─ 시스템: Managed Policy (회사 공통 규칙)
    │   └─ 프로젝트: 구조, 커맨드, 아키텍처 정보
    │
    ├─ 플러그인 로딩 (dcs-ai-dashboard-maker)
    │   ├─ MCP 서버 연결 → DCS AI 백엔드
    │   ├─ Rules → 코딩 컨벤션 주입
    │   ├─ Skills → 도메인 지식 참조 가능
    │   ├─ Agents → 특수 작업 위임 가능
    │   └─ Hooks → 이벤트 자동 수집 시작
    │
    ├─ Memory 로딩 → 이전 대화 맥락 복원
    │
    └─ Permissions → 허용된 도구만 자동 실행
```

### 실제 시나리오: "매출 대시보드 만들어줘"

```
1. 사용자: "/dcs-ai-dashboard-maker:Init" → 프로젝트 초기 설정
2. 사용자: "매출 대시보드 만들어줘"

3. Claude: MCP로 데이터 스키마 조회 (execute_kg_api_to_context)
4. Claude: Skills 참조 (chart-visualization, data-processing)
5. Claude: Rules 준수하며 코드 작성 (coding-style, tech-stack)
6. Claude: 서브에이전트로 빌드 에러 수정 위임 (build-error-resolver)
7. Claude: "/dcs-ai-dashboard-maker:Dashboard-Deploy" → EC2 배포

   ↕ 이 모든 과정에서 Hooks가 이벤트를 수집하여 DCS AI 서버로 전송
```

---

## 9. 유용한 팁

### 모델 선택

```
/model                    # 현재 모델 확인/변경
/model opus               # Opus (정확도 우선, 느림)
/model sonnet             # Sonnet (균형)
/model haiku              # Haiku (속도 우선, 저렴)
```

서브에이전트별로 모델을 다르게 지정할 수 있다:

```markdown
# 에이전트 정의 시
모델: Haiku (단순 규칙 동기화에는 Haiku면 충분)
```

### Plan Mode

복잡한 작업 전에 계획을 먼저 세우고 확인받는 모드:

```
/plan            # Plan 모드 진입
사용자: "인증 시스템 리팩토링해줘"
Claude: 계획 제시 (코드 수정 없음)
사용자: "좋아, 진행해"
/exitplan        # 실행 모드로 전환
```

### 이미지 입력

스크린샷을 직접 붙여넣거나 드래그앤드롭 할 수 있다:

```
사용자: [스크린샷 붙여넣기]
사용자: "이 디자인대로 구현해줘"
```

### Prompt Queue

프롬프트를 **여러 개 미리 입력**해두면 순차적으로 실행된다. 긴 작업을 걸어두고 다른 일 할 때 유용하다.

---

## 참고 자료

- [Claude Code 공식 문서](https://docs.anthropic.com/en/docs/claude-code)
- [MCP 프로토콜 스펙](https://modelcontextprotocol.io/)
- [Claude Code Plugin 가이드](https://docs.anthropic.com/en/docs/claude-code/plugins)
- [Claude Code Hooks 가이드](https://docs.anthropic.com/en/docs/claude-code/hooks)
