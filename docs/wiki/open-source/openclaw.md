---
title: "OpenClaw"
parent: "Open Source"
permalink: /open-source/openclaw
nav_order: 1
---

# OpenClaw - Personal AI Assistant

- **GitHub**: [openclaw/openclaw](https://github.com/openclaw/openclaw)
- **라이선스**: MIT
- **언어**: TypeScript (ESM), Swift (macOS/iOS), Kotlin (Android)
- **런타임**: Node.js 22+
- **패키지 매니저**: pnpm (Bun도 부분 지원)
- **버전**: 2026.3.14 (CalVer 방식 - `YYYY.M.D`)

---

## 스터디 발표 가이드

### 도입 — "이게 뭔데?" (3분)

내 디바이스에서 돌아가는 개인용 AI 게이트웨이. ChatGPT처럼 "AI 전용 앱"을 새로 만든 게 아니라, **이미 쓰고 있는 메신저(WhatsApp, Telegram, Slack, Discord 등 20개+)에 AI를 꽂는 인프라**다.

```
[사용자] → [메시징 채널] → [OpenClaw Gateway] → [AI Provider] → [응답]
             WhatsApp                                 OpenAI
             Telegram          라우팅/세션/보안         Anthropic
             Discord                                  Google
             Slack                                    Ollama (로컬)
```

- "앱"이 아니라 "플랫폼". 채널도 플러그인, AI 프로바이더도 플러그인, 도구도 플러그인.
- src 2,800+ 파일, extensions 60개+, 15명+ 메인테이너의 대형 오픈소스.
- 코어는 얇게, 확장은 전부 플러그인으로 — 이 설계 철학이 코드 전체에 녹아있다.

### 본론 — 코드로 보는 설계 패턴 3가지 (10분)

#### 1. Optional Adapter 패턴 — `src/channels/plugins/types.plugin.ts` (92줄)

채널 플러그인 인터페이스가 하나의 거대한 interface가 아니라, **선택적 어댑터 조합**으로 되어있다.

```typescript
interface ChannelPlugin {
  capabilities: { ... };       // 이 채널이 뭘 지원하는지 선언
  config?: ConfigAdapter;      // 설정 위자드 (optional)
  auth?: AuthAdapter;          // 인증 (optional)
  messaging?: MessagingAdapter; // 메시지 전송 (optional)
  streaming?: StreamAdapter;   // 스트리밍 (optional)
  security?: SecurityAdapter;  // 보안 정책 (optional)
  actions?: ActionsAdapter;    // 리액션, 편집 등 (optional)
  // ...20개+ 어댑터, 전부 optional
}
```

**핵심 포인트:**
- 새 채널(예: LINE) 추가 시 코어 코드 변경 0. 플러그인 계약만 구현하면 됨.
- 처음엔 messaging만 구현하고, 나중에 streaming, actions를 점진적으로 추가 가능.
- "전부 구현하거나 말거나"가 아니라 **필요한 것만 골라서 구현**하는 구조.

#### 2. 커스텀 아키텍처 Lint — `scripts/check-channel-agnostic-boundaries.mjs` (344줄)

모듈 경계 규칙을 **문서가 아니라 CI에서 자동으로 잡는다.**

```
이 스크립트가 하는 일:
1. TypeScript 컴파일러 API로 소스 파일의 AST를 파싱
2. "채널 무관(agnostic)" 코드에서 채널 특화 참조를 탐지
   - import 경로에 특정 채널명이 있는지
   - config 경로에 채널별 키가 있는지
   - 문자열 비교에서 채널 ID를 쓰는지
3. 위반이 발견되면 CI 실패
```

**핵심 포인트:**
- OpenClaw은 이런 스크립트가 20개+. 모듈 경계를 전부 자동 검증한다.
- `check-extension-plugin-sdk-boundary.mjs`: 확장이 코어를 직접 import 못하게
- `check-no-extension-src-imports.ts`: 코어가 확장을 직접 import 못하게
- 규칙은 문서에 쓰면 안 지켜진다. **CI에서 빌드를 깨뜨려야 지켜진다.**

#### 3. 헬스 모니터 — `src/gateway/channel-health-monitor.ts` (203줄)

외부 연결(채널)의 상태를 감시하고 자동 복구하는 패턴.

```
핵심 흐름:
  check → evaluate → throttle → restart → record

특별한 점:
- "소켓은 살아있는데 이벤트가 안 오는" stale 상태를 별도 감지
  (연결 자체는 OK지만 실제로는 죽어있는 경우)
- 수동으로 멈춘 채널은 자동 재시작 대상에서 제외
- 시간당 재시작 횟수 제한 + 쿨다운 사이클로 재시작 폭주 방지
- 정책(policy)을 분리해서 채널마다 다른 임계값 적용 가능
```

**핵심 포인트:**
- 단순 ping/pong이 아니라 **"반쯤 죽은" 상태를 구분**하는 게 실전적.
- 재시작 횟수 제한과 쿨다운은 외부 API 연동이 많은 서비스에서 바로 써먹을 수 있는 패턴.

### 마무리 (2분)

- TypeScript 프로젝트지만 설계 패턴은 언어 무관. 어디서든 적용 가능.
- 공통 키워드: **"코어를 보호하고, 확장은 계약으로 열어둔다"**
  - 플러그인 계약 → 코어 변경 없이 기능 추가
  - 아키텍처 Lint → 계약 위반을 CI가 잡음
  - 헬스 모니터 → 외부 연결 장애를 자동 복구
- 4,000+ 파일 규모의 오픈소스가 15명+ 메인테이너로 굴러가려면, 이런 **자동화된 경계 보호**가 필수적이다.

---

## 다른 AI 에이전트와의 핵심 차이점

대부분의 AI 에이전트(ChatGPT, Claude Code, Cursor 등)는 **"AI와 대화하는 앱"**을 만든 것이다. OpenClaw은 **"내가 이미 쓰는 모든 메신저에 AI를 꽂는 인프라"**다.

| | 일반 AI 에이전트 | OpenClaw |
|---|---|---|
| **인터페이스** | 자체 UI 1개 (웹/CLI) | 20개+ 기존 메시징 채널을 그대로 사용 |
| **채널 연동** | 하드코딩 | 플러그인 계약 기반, 런타임에 동적 구성 |
| **AI 프로바이더** | 빌드 타임에 고정 | 플러그인으로 동적 교체, failover 지원 |
| **실행 위치** | 클라우드 | 내 로컬 머신 (privacy-first) |
| **설계 사고** | "앱" | "플랫폼/게이트웨이" |

핵심 아키텍처 특징:
- **멀티채널 라우팅**: 다단계 바인딩 계층 (peer → guild role → guild → account → channel → default)
- **채널 플러그인 계약**: 20개+ 선택적 어댑터(lifecycle, outbound, security, config, actions)로 구성. 코어 변경 없이 새 채널 추가 가능.
- **게이트웨이 = 오케스트레이션 플레인**: 채널 생명주기, 헬스 모니터링, 설정 hot-reload, WS+HTTP API 동시 제공

---

## 한 줄 요약

> 내 디바이스에서 돌아가는 개인용 AI 어시스턴트. 이미 쓰고 있는 메시징 채널(WhatsApp, Telegram, Slack, Discord 등 20개+)에서 AI와 대화할 수 있게 해주는 **멀티채널 AI 게이트웨이**.

---

## 왜 흥미로운 프로젝트인가?

1. **실용적 아키텍처**: 단순한 챗봇이 아니라, 메시징 채널 추상화 + AI 프로바이더 추상화 + 플러그인 시스템을 갖춘 본격 gateway
2. **규모**: src 2,800+ 파일, extensions 1,200+ 파일의 대형 TypeScript 프로젝트
3. **멀티플랫폼**: CLI + macOS 앱(Swift) + iOS 앱(Swift) + Android 앱(Kotlin) + Web UI
4. **플러그인 생태계**: 60개+ extensions, plugin-sdk를 통한 확장
5. **활발한 오픈소스**: 15명+ 메인테이너, 주단위 릴리즈

---

## 핵심 컨셉

### Gateway 패턴

```
[사용자] → [메시징 채널] → [OpenClaw Gateway] → [AI Provider] → [응답]
             WhatsApp                                 OpenAI
             Telegram          라우팅/세션/보안         Anthropic
             Discord                                  Google
             Slack                                    Ollama (로컬)
             Signal                                   ...
             ...
```

- Gateway는 **컨트롤 플레인** 역할. 실제 "제품"은 어시스턴트 자체
- 로컬 머신에서 데몬으로 실행 (launchd/systemd)
- 싱글 유저 설계: 개인 비서 컨셉

### 채널 추상화

- 모든 메시징 플랫폼을 통합된 인터페이스로 추상화
- 인바운드 메시지 수신 -> 세션 관리 -> AI 처리 -> 아웃바운드 응답
- 채널별 특성(쓰레드, 리액션, 미디어 등) 처리

### 프로바이더 추상화

- OpenAI, Anthropic, Google, Mistral, Ollama 등 다양한 AI 프로바이더 지원
- OAuth/API Key 인증, 모델 failover, 스트리밍 응답
- Tool use, 웹 검색, 이미지 생성 등 기능별 추상화

---

## 프로젝트 구조

```
openclaw/
├── src/                    # 코어 소스코드
│   ├── entry.ts            # CLI 엔트리포인트
│   ├── index.ts            # 라이브러리 엔트리 (public API)
│   ├── cli/                # CLI 프레임워크 (commander 기반)
│   ├── commands/           # CLI 커맨드 구현 (agent, gateway, onboard, doctor 등)
│   ├── gateway/            # 게이트웨이 서버 핵심 (WebSocket + HTTP)
│   ├── channels/           # 채널 추상화 레이어
│   ├── providers/          # AI 프로바이더 공통 코드
│   ├── plugins/            # 플러그인 로딩/관리
│   ├── plugin-sdk/         # 플러그인 개발 SDK
│   ├── agents/             # 에이전트 실행 엔진
│   ├── routing/            # 메시지 라우팅
│   ├── security/           # 보안 정책, 샌드박스
│   ├── config/             # 설정 관리
│   ├── media/              # 미디어 파이프라인 (이미지, 음성 등)
│   ├── sessions/           # 세션/대화 관리
│   ├── hooks/              # 이벤트 훅 시스템
│   ├── infra/              # 인프라 유틸리티
│   ├── terminal/           # 터미널 UI (테이블, 컬러 팔레트)
│   ├── tui/                # TUI (Terminal UI)
│   ├── web-search/         # 웹 검색 기능
│   ├── tts/                # Text-to-Speech
│   └── ...
├── extensions/             # 플러그인/확장 (60개+)
│   ├── openai/             # OpenAI 프로바이더
│   ├── anthropic/          # Anthropic 프로바이더
│   ├── google/             # Google 프로바이더
│   ├── ollama/             # Ollama (로컬 LLM)
│   ├── telegram/           # Telegram 채널
│   ├── discord/            # Discord 채널
│   ├── slack/              # Slack 채널
│   ├── whatsapp/           # WhatsApp 채널
│   ├── voice-call/         # 음성 통화
│   ├── memory-lancedb/     # 벡터 메모리 (LanceDB)
│   └── ...
├── apps/                   # 네이티브 앱
│   ├── macos/              # macOS 앱 (SwiftUI)
│   ├── ios/                # iOS 앱 (SwiftUI)
│   ├── android/            # Android 앱 (Kotlin)
│   └── shared/             # 공유 Swift 코드 (OpenClawKit)
├── ui/                     # Web UI (Lit 기반)
├── docs/                   # 문서 (Mintlify)
├── scripts/                # 빌드/릴리즈/테스트 스크립트
├── skills/                 # 번들 스킬
├── test/                   # E2E 테스트
└── packages/               # 내부 패키지
```

---

## 아키텍처 심화

### 1. Gateway Server (`src/gateway/`)

게이트웨이는 프로젝트의 핵심이며 가장 복잡한 모듈이다.

- **WebSocket 기반 실시간 통신**: 클라이언트(모바일 앱, CLI, Web UI)와의 양방향 통신
- **HTTP API**: OpenAI-compatible API 제공 (`openai-http.ts`, `openresponses-http.ts`)
- **인증/인가**: 토큰 기반, 디바이스 페어링, 브라우저 하드닝, 역할 기반 접근 제어
- **플러그인 런타임**: 플러그인 로딩, HTTP 핸들러 등록, 이벤트 브로드캐스트
- **세션 관리**: 채널별/사용자별 대화 세션 관리
- **Config hot-reload**: 설정 변경 시 재시작 없이 적용
- **채널 헬스 모니터링**: 각 채널 연결 상태 감시

### 2. Plugin SDK (`src/plugin-sdk/`)

확장 가능한 구조의 핵심. package.json의 exports를 보면 80개+ 서브 엔트리를 가진다.

```typescript
// 플러그인 구조 예시
import { definePlugin } from "openclaw/plugin-sdk";

export default definePlugin({
  name: "my-plugin",
  channels: { ... },      // 채널 통합
  providers: { ... },      // AI 프로바이더
  tools: { ... },          // 에이전트 도구
  hooks: { ... },          // 이벤트 훅
});
```

주요 런타임 인터페이스:
- `channel-runtime`: 채널 생명주기 관리
- `agent-runtime`: 에이전트 실행 컨텍스트
- `provider-stream`: AI 프로바이더 스트리밍
- `security-runtime`: 보안 정책 적용
- `hook-runtime`: 이벤트 훅 등록/실행
- `media-runtime`: 미디어 처리 파이프라인

### 3. 채널 시스템 (`src/channels/` + `extensions/`)

```
인바운드 메시지
  → allowlist 검사 (allow-from)
  → 커맨드 게이팅 (command-gating)
  → 멘션 게이팅 (mention-gating)
  → 세션 매핑 (session-envelope)
  → 디바운스 (inbound-debounce-policy)
  → 상태 머신 (run-state-machine)
  → AI 프로바이더로 전달
  → 응답 스트리밍
  → 채널별 포맷 변환
  → 아웃바운드 전송
```

### 4. 에이전트 시스템 (`src/commands/agent.ts`)

- `--local` 모드: 게이트웨이 없이 직접 AI 호출
- 게이트웨이 모드: WebSocket을 통한 원격 실행
- 도구 사용(tool use), 승인 관리(exec-approval), 샌드박스 실행
- ACP (Agent Client Protocol) 지원

---

## 기술 스택 상세

| 영역 | 기술 |
|------|------|
| 언어 | TypeScript (ESM), Swift, Kotlin |
| 런타임 | Node.js 22+, Bun (선택) |
| 빌드 | tsdown (Rolldown 기반) |
| 타입체크 | tsc + tsgo (native TS preview) |
| 린트/포맷 | Oxlint + Oxfmt (Rust 기반, 빠름) |
| 테스트 | Vitest + V8 coverage |
| HTTP | Hono (게이트웨이), Express (레거시) |
| WebSocket | ws |
| 스키마 | TypeBox (@sinclair/typebox), Zod |
| 메시징 | grammy(Telegram), @buape/carbon(Discord), @slack/bolt, Baileys(WhatsApp) |
| AI | 각 프로바이더 SDK 직접 통합 |
| 벡터DB | LanceDB, sqlite-vec |
| 미디어 | sharp (이미지), pdfjs-dist (PDF), playwright-core (브라우저) |
| 네이티브 | node-pty (터미널), node-edge-tts (TTS) |
| 문서 | Mintlify |
| CI/CD | GitHub Actions |

---

## 주요 CLI 커맨드

```bash
openclaw onboard          # 초기 설정 위자드
openclaw gateway run      # 게이트웨이 서버 실행
openclaw agent            # 에이전트 실행 (대화/작업)
openclaw message send     # 메시지 전송
openclaw config set       # 설정 변경
openclaw channels status  # 채널 상태 확인
openclaw doctor           # 진단/마이그레이션
openclaw status            # 전체 상태 요약
```

---

## 설계 원칙 & 인사이트

### Privacy-first 설계
- 모든 데이터가 로컬 머신에 저장 (`~/.openclaw/`)
- 클라우드 서비스 의존 없음 (AI API 호출 제외)
- 세션 데이터, 인증 정보 모두 로컬

### 보안 레이어링
- 샌드박스 실행 (Docker 기반)
- 실행 승인 시스템 (exec-approval)
- 역할 기반 접근 제어
- 채널별 allowlist
- 입력 검증/새니타이징

### 확장성 전략
- **Core를 얇게 유지**: 핵심 기능만 코어에, 나머지는 플러그인
- **Plugin SDK**: 80개+ 서브 엔트리를 가진 풍부한 SDK
- **Extension 패턴**: 채널, 프로바이더, 도구 모두 동일한 플러그인 인터페이스
- **MCP 지원**: mcporter를 통한 브릿지 (코어에 직접 빌트인하지 않음)

### 코드 품질 관리
- 커스텀 lint 규칙 20개+ (`scripts/check-*.mjs`)
- 모듈 경계 보호 (plugin-sdk boundary, extension import boundary)
- 파일당 500 LOC 가이드라인
- 70% 커버리지 threshold

---

## 배울 점

1. **대규모 TypeScript 모노레포 운영**: pnpm workspace + 60개 extension 패키지 관리
2. **플러그인 아키텍처 설계**: plugin-sdk의 세분화된 엔트리포인트로 tree-shaking 최적화
3. **멀티채널 추상화**: 20개+ 메시징 플랫폼을 통합 인터페이스로 다루는 패턴
4. **보안과 편의성 균형**: "Strong defaults without killing capability"
5. **CalVer 릴리즈**: `YYYY.M.D` 형식의 날짜 기반 버전 관리
6. **Rust 기반 도구 적극 활용**: Oxlint, Oxfmt, tsdown(Rolldown) 등으로 빌드 성능 확보
7. **커스텀 아키텍처 lint**: `scripts/check-*.mjs`로 모듈 경계/import 규칙을 CI에서 강제

---

## 코드 리딩 포인트

아래 파일들은 dcs-ai-project 관점에서 특히 참고할만한 패턴을 담고 있다.

### 1. 플러그인 계약 인터페이스 — `src/channels/plugins/types.plugin.ts` (92 LOC)

```
왜 볼까: Optional adapter 패턴의 교과서적 구현
```

채널 플러그인이 지원하는 기능을 `capabilities`로 선언하고, config/auth/messaging/streaming 등 각 어댑터가 독립적으로 optional이다. "전부 구현하거나 아예 안 하거나"가 아니라 **점진적으로 기능을 확장**할 수 있는 구조.

→ **dcs-ai 적용점**: `tools.service.ts`에서 KG/MCP/custom 도구를 if문으로 합치는 대신, 도구 소스마다 동일한 계약을 구현하게 하면 확장이 쉬워짐.

### 2. 플러그인 레지스트리 — `src/plugins/registry.ts` (1,008 LOC)

```
왜 볼까: 대규모 플러그인 시스템의 로딩/검증/진단 패턴
```

멀티 스테이지 로딩, 네임스페이스 충돌 감지, 타입별 훅 보안 정책, graceful degradation (스냅샷용 setup-only 모드). tools/providers/hooks/channels/CLI/services/HTTP routes를 **하나의 레지스트리에서 일관되게** 관리한다.

→ **dcs-ai 적용점**: MCP 서버 + KG 도구 + custom 도구 + 스킬을 통합 레지스트리로 관리하면 도구 카탈로그 일관성 확보.

### 3. Config Hot-Reload — `src/gateway/config-reload.ts` (247 LOC)

```
왜 볼까: 파일 감시 → 디바운스 → diff → 핫리로드/재시작 분기의 깔끔한 상태 머신
```

chokidar로 설정 파일 변경 감지 → 스냅샷 검증 → 변경된 경로를 재귀적으로 diff → hot-reload 가능한 것과 재시작 필요한 것을 분기. 설정 파일이 깨졌을 때 retry 로직도 포함.

→ **dcs-ai 적용점**: 프롬프트 템플릿, 도구 활성/비활성, MCP 서버 연결 등을 서버 무중단으로 반영하는 패턴.

### 4. 헬스 모니터 — `src/gateway/channel-health-monitor.ts` (203 LOC)

```
왜 볼까: "반쯤 죽은" 연결을 감지하는 패턴
```

소켓은 살아있지만 이벤트가 안 오는 "stale" 상태를 별도 판정. 수동 중지 존중, 시간당 재시작 횟수 제한, 쿨다운 사이클. check → evaluate → throttle → restart → record 흐름.

→ **dcs-ai 적용점**: MCP 커넥션, KG API, 외부 API 연결에 동일한 헬스 모니터 + 자동 복구 적용.

### 5. 라우팅 바인딩 해소 — `src/routing/resolve-route.ts` (804 LOC)

```
왜 볼까: 다단계 바인딩 해소 + WeakMap 캐싱 전략
```

peer → parent peer(쓰레드 상속) → guild+roles → guild → team → account → channel → default 순으로 매칭. 인덱스 기반 빠른 조회, 크로스 채널 사용자 통합(identity linking), config 객체를 WeakMap 키로 사용해 GC 자동 정리.

→ **dcs-ai 적용점**: 멀티채널 확장(Slack, Teams 등) 시 세션 라우팅 설계 참고. WeakMap 캐싱은 config 갱신 시 캐시 무효화를 자연스럽게 해결하는 좋은 트릭.

### 6. 세션 키 파생 — `src/routing/session-key.ts` (253 LOC)

```
왜 볼까: agent + channel + account + peer 조합으로 세션을 격리하는 설계
```

DM 스코핑 모드 4가지(`main`, `per-peer`, `per-channel-peer`, `per-account-channel-peer`), ID 정규화(영숫자+대시, 64자 제한), 파싱 유틸리티까지 포함.

→ **dcs-ai 적용점**: 현재 `chat_room` 테이블 기반 세션 관리를 멀티채널로 확장할 때 키 설계 참고.

### 7. 스트리밍 상태 머신 — `src/channels/run-state-machine.ts` (99 LOC)

```
왜 볼까: 99줄로 스트리밍 실행 상태를 완벽히 관리하는 미니멀 구현
```

start에서 카운터 증가, end에서 감소, 주기적 heartbeat. busy 플래그, 활성 run 수, 마지막 활동 시각을 상태 패치로 발행.

→ **dcs-ai 적용점**: `chat-streaming.controller.ts`의 스트리밍 상태 추적을 더 체계적으로 만들 때 참고.

### 8. 스트리밍 컨트롤 합성 — `src/channels/draft-stream-controls.ts` (142 LOC)

```
왜 볼까: 상속 없이 함수 파라미터로 스트리밍 생명주기를 합성하는 패턴
```

throttled loop → stop(최종 마킹, in-flight 대기) → clear(취소 시 메시지 삭제)를 **각각 독립 함수로 분리**하고 조합. 상속 대신 합성(composition over inheritance)의 실용적 예시.

→ **dcs-ai 적용점**: StreamingContext 패턴을 더 세분화할 때 참고.

### 9. 커스텀 아키텍처 Lint — `scripts/check-channel-agnostic-boundaries.mjs` (344 LOC)

```
왜 볼까: TypeScript 컴파일러 API로 AST 레벨 모듈 경계 검증
```

채널 무관(agnostic) 코드에서 채널 특화 참조(import, config 경로, 문자열 비교)를 감지. CI에서 자동 실행되어 아키텍처 규칙 위반을 빌드 타임에 잡아냄.

→ **dcs-ai 적용점**: FSD 계층 규칙(entities가 features를 import 못하게)을 스크립트로 강제하는 데 바로 응용 가능. 가장 빠르게 도입할 수 있는 패턴.

### 10. 훅 시스템 — `src/hooks/types.ts` (67 LOC) + `src/hooks/internal-hooks.ts`

```
왜 볼까: 이벤트 드리븐 확장의 선언적(declarative) 설계
```

훅 메타데이터를 프론트매터로 선언(이벤트 목록, 플랫폼 요구사항, 설치 스펙). 타입된 훅 이벤트(agent bootstrap, gateway startup, message received/sent)와 각 이벤트가 받는 컨텍스트 객체(config, workspace, session, channel, message) 정의.

→ **dcs-ai 적용점**: dcs-ai-plugin의 훅 시스템을 더 타입 안전하게 만들 때 참고.

---

## 참고 자료

- [공식 문서](https://docs.openclaw.ai)
- [DeepWiki 분석](https://deepwiki.com/openclaw/openclaw)
- [VISION.md](https://github.com/openclaw/openclaw/blob/main/VISION.md)
- [CONTRIBUTING.md](https://github.com/openclaw/openclaw/blob/main/CONTRIBUTING.md)
- [SECURITY.md](https://github.com/openclaw/openclaw/blob/main/SECURITY.md)
