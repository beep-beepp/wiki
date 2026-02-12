---
title: "About Slack"
parent: "ETC"
permalink: /etc/about-slack
nav_order: 1
---

# Slack Bot 개발 가이드

Slack Bot을 만들면서 배운 것들을 정리한 문서.
DCS-AI 프로젝트에서 워크플로우 엔진의 첫 번째 채널 어댑터로 Slack Bot을 구현하면서 작성했다.

---

## 1. WebSocket vs HTTP API

### HTTP API (일반적인 방식)

```
클라이언트                    서버
    │                          │
    │  "주문이요" (요청)        │
    │ ────────────────────→    │
    │                          │
    │  "여기요" (응답)          │
    │ ←────────────────────    │
    │                          │
    │  (연결 끊김)              │
```

**식당 카운터**와 같다. 매번 가서 주문하고, 받고, 돌아온다. 서버가 먼저 말을 걸 수 없다.

### WebSocket (양방향 상시 연결)

```
클라이언트                    서버
    │                          │
    │  "연결할게요" (핸드셰이크) │
    │ ────────────────────→    │
    │                          │
    │  "OK, 연결됨"             │
    │ ←────────────────────    │
    │                          │
    │ ════════════════════════ │  ← 파이프가 열린 상태 유지
    │                          │
    │  서버가 먼저 말함          │
    │ ←────────────────────    │
    │                          │
    │  클라이언트도 아무때나     │
    │ ────────────────────→    │
    │                          │
    │ ════════════════════════ │  ← 계속 열려있음
```

**전화 통화**와 같다. 한번 연결하면 양쪽 다 아무때나 말할 수 있다.

### 핵심 차이

| | HTTP API | WebSocket |
|---|---|---|
| **연결** | 요청마다 열고 닫음 | 한번 열면 계속 유지 |
| **방향** | 클라이언트 → 서버만 | 양방향 자유 |
| **서버가 먼저?** | 불가능 | 가능 |
| **비유** | 카운터 주문 | 전화 통화 |

---

## 2. Slack Bot의 두 가지 연결 방식

핵심 차이: **누가 먼저 연결하느냐**.

```
HTTP Mode:    Slack ──→ 우리 서버  (Slack이 찾아옴)
Socket Mode:  우리 서버 ──→ Slack  (우리가 찾아감)
```

### HTTP Mode

```
누가 @bot 멘션함
       │
       ▼
Slack 서버: "봇한테 알려줘야지"
       │
       ▼
Slack ──HTTP POST──→ https://our-server.com/slack/events
                     (우리가 미리 등록한 공개 URL)
       │
       ▼
우리 서버: 요청 받음 → 서명 검증 → 처리 → 응답 반환
```

Slack이 **우리 서버를 찾아온다**. 그래서 Slack이 찾아올 수 있는 **공개 URL이 필수**다.

```
인터넷
  │
  ▼
방화벽 ──→ ❌ 막힘! Slack이 못 들어옴
  │
우리 서버 (사내망)
```

사내망에 있으면 Slack이 접근할 수 없어서, ngrok 같은 터널링이 필요하다.

### Socket Mode (우리가 쓰는 방식)

```
서버 시작됨
       │
       ▼
우리 서버 ──WebSocket 연결──→ Slack 서버
       (우리가 먼저 나감)
       │
       ▼
누가 @bot 멘션함
       │
       ▼
Slack 서버: "이미 연결되어 있으니 그냥 보내면 됨"
       │
       ▼
Slack ──(기존 WebSocket)──→ 우리 서버
```

우리 서버가 **먼저 Slack에 나간다**. 그래서 공개 URL이 필요 없다.

```
인터넷
  │
  ▲
방화벽 ──→ ✅ 나가는 건 OK!
  │
우리 서버 (사내망) ──→ Slack에 연결
```

**나가는 건 막지 않으니까** 방화벽 뒤에서도 동작한다.

### 비교

| | HTTP Mode | Socket Mode |
|---|---|---|
| **연결 주체** | Slack이 우리를 찾아옴 | 우리가 Slack에 연결 |
| **공개 URL** | 필수 | 불필요 |
| **방화벽** | 인바운드 포트 열어야 함 | 아웃바운드만 되면 OK |
| **보안** | 매 요청 서명 검증 필요 | 이미 인증된 연결 |
| **설정** | URL 등록, SSL 인증서 등 | 토큰 2개만 있으면 됨 |
| **확장성** | 서버 무한 확장 가능 | 앱당 10개 연결 |
| **안정성** | 높음 (Slack 공식 권장) | WebSocket 끊길 수 있음 |
| **적합한 곳** | 공개 앱, 대규모 | 사내 앱, 개발 |

### 우리가 Socket Mode를 쓰는 이유

1. **사내 앱**이라 마켓플레이스 등록 불필요
2. **사내망**에서 실행하므로 공개 URL 노출 불필요
3. **설정이 간단** — 토큰만 넣으면 끝
4. 규모가 크지 않아 10개 연결 제한에 걸릴 일 없음

### 나중에 HTTP Mode로 전환이 필요하면?

코드 변경은 거의 없다. 연결 방식만 바꾸면 된다.

```typescript
// Socket Mode (지금)
new App({ token, appToken, socketMode: true });

// HTTP Mode (전환 시)
new App({ token, signingSecret: '...' });
// + 서버에 /slack/events 엔드포인트 노출
```

핸들러 코드(`app.event`, `app.command`, `app.action`)는 **전혀 안 바뀐다**.

---

## 3. Socket Mode 연결 과정

```
우리 서버                          Slack
   │                                │
   │  1. POST apps.connections.open │
   │  (App Token: xapp-...)         │
   │ ─────────────────────────────→ │
   │                                │
   │  2. WebSocket URL 반환          │
   │  "wss://wss.slack.com/..."     │
   │ ←───────────────────────────── │
   │                                │
   │  3. WebSocket 연결              │
   │ ═══════════════════════════════│  (양방향 상시 연결)
   │                                │
   │  4. "hello" 메시지              │
   │ ←───────────────────────────── │
   │                                │
   │  이후: 이벤트가 이 연결로 push │
```

### 토큰 2개가 필요한 이유

| 토큰 | 역할 | 비유 |
|------|------|------|
| **App Token** (`xapp-`) | WebSocket 연결을 여는 열쇠 | 건물 출입카드 |
| **Bot Token** (`xoxb-`) | 메시지 보내기, 유저 조회 등 API 호출 | 업무용 사원증 |

App Token으로 연결을 열고, Bot Token으로 실제 일을 한다.

### URL이 하드코딩되지 않는 이유

Socket Mode는 `@slack/bolt`가 내부적으로 Slack의 WebSocket 엔드포인트에 연결한다.
URL을 지정할 필요가 없기 때문에 `.env`에 토큰만 있으면 **로컬이든 EC2든 ECS든 어디서 실행해도 동일하게 동작**한다.

```typescript
// 우리 코드에서 URL은 없다. 토큰만 있으면 됨.
this.app = new App({
  token: botToken,      // .env의 SLACK_BOT_TOKEN
  appToken: appToken,   // .env의 SLACK_APP_TOKEN
  socketMode: true,
});
```

### 주의: 같은 토큰으로 동시 실행 시

같은 토큰으로 두 곳(로컬 + 배포 서버)에서 동시 실행하면, Slack이 이벤트를 **둘 중 하나에만 랜덤으로** 보낸다.
해결 방법: **개발용 Slack App / 운영용 Slack App을 별도로 만들거나**, 환경변수로 on/off 제어.

---

## 4. 코드 구조: API와 비교

### 연결 준비

```typescript
// API라면:
const app = express();
app.listen(3000);  // 포트 열고 기다려

// Slack Bot은:
this.app = new App({ token, appToken, socketMode: true });
await this.app.start();  // Slack에 WebSocket 연결
```

API는 **문을 열고 손님을 기다리는 것**, WebSocket은 **우리가 Slack에 전화를 거는 것**.

### 핸들러 등록

```typescript
// API라면:
app.post('/mention', handler);      // URL에 매핑
app.post('/approve', handler);      // URL에 매핑
app.post('/button-click', handler); // URL에 매핑

// Slack Bot은:
this.app.event('app_mention', handler);      // 이벤트 타입에 매핑
this.app.command('/approve', handler);       // 커맨드 이름에 매핑
this.app.action('approve_request', handler); // 버튼 action_id에 매핑
```

---

## 5. 핸들러에서 쓰는 객체들

### 들어온 데이터 (req.body에 해당)

**`event`** — 멘션, 메시지 등 이벤트

```typescript
// "@bot @김이사 이거 승인해줘" 라고 치면
this.app.event('app_mention', async ({ event }) => {
  event.user     // "U034ZC28ZDX" — 누가 멘션했는지
  event.text     // "<@BOT> <@U999> 이거 승인해줘" — 전체 메시지
  event.channel  // "C012ABC" — 어느 채널에서
  event.ts       // "1707654321.001" — 메시지 타임스탬프
});
```

**`command`** — 슬래시 커맨드

```typescript
// "/approve 마케팅 예산 500만원" 이라고 치면
this.app.command('/approve', async ({ command }) => {
  command.user_id    // "U034ZC28ZDX" — 누가 쳤는지
  command.user_name  // "topseon23" — 유저 이름
  command.text       // "마케팅 예산 500만원" — /approve 뒤의 내용
  command.channel_id // "C012ABC" — 어느 채널에서
});
```

**`body`** — 버튼 클릭

```typescript
// [✅ 승인] 버튼을 누르면
this.app.action('approve_request', async ({ body }) => {
  body.user.id              // "U999" — 누가 버튼을 눌렀는지
  body.actions[0].action_id // "approve_request" — 어떤 버튼인지
  body.actions[0].value     // '{"requester":"U034ZC28ZDX","content":"승인해줘"}'
                            // ↑ 버튼 만들 때 넣어둔 JSON
});
```

### 응답 (res.json()에 해당)

**`say()`** — 같은 채널에 새 메시지

```typescript
this.app.event('app_mention', async ({ event, say }) => {
  await say({ text: `<@${event.user}> 승인자에게 DM 보냈습니다.` });
});
```

**`respond()`** — 원본 메시지에 응답/교체

```typescript
this.app.action('approve_request', async ({ respond }) => {
  await respond({
    replace_original: true,  // true면 원본 교체, false면 새 메시지
    text: '승인 완료',
  });
});
```

| | `say()` | `respond()` |
|---|---|---|
| 대상 | 이벤트가 발생한 채널 | 원본 메시지 |
| 용도 | 새 메시지 보내기 | 기존 메시지 교체/이어 답변 |

### 별도 API 호출 (axios.post()에 해당)

**`client`** — Slack Web API 호출

```typescript
this.app.event('app_mention', async ({ client }) => {
  // 유저 ID를 channel에 넣으면 DM이 된다
  await client.chat.postMessage({
    channel: 'U999',
    text: '승인 요청입니다',
    blocks: [{ ... }],
  });
});
```

`say()`는 "지금 대화 중인 곳에 응답", `client`는 "아무 데나 메시지 보내기".

### ack() — WebSocket 전용 (API에는 없음)

```typescript
this.app.command('/approve', async ({ ack, respond }) => {
  await ack();  // 반드시 3초 내에 호출!
  // ack() 이후에 무거운 작업 수행
  await respond({ ... });
});
```

Slack이 이벤트를 보내면 "잘 받았어?"를 확인한다. 3초 안에 `ack()`를 안 하면 Slack이 **같은 이벤트를 다시 보낸다**.

> `event`에는 `ack()`가 없다. `@slack/bolt`가 자동으로 ack 해준다. `command`와 `action`만 수동으로 해야 한다.

### 종합 비교

| API | Slack Bot | 역할 |
|-----|-----------|------|
| `req.body` | `event`, `command`, `body` | 들어온 데이터 |
| `res.json()` | `say()`, `respond()` | 바로 응답 |
| `axios.post()` | `client.chat.postMessage()` | 별도 API 호출 |
| — | `ack()` | "이벤트 받았어" 확인 (3초 내) |

---

## 6. Block Kit — 메시지 UI 만들기

Slack에서는 **Block Kit**이라는 UI 프레임워크로 메시지를 구성한다. JSON 블록을 레고처럼 조합하는 방식.

### 기본 구조

```typescript
await say({
  text: '폴백 텍스트',  // 알림에 표시되는 텍스트
  blocks: [             // 실제 보여지는 UI 블록들
    { ... },            // 블록 1
    { ... },            // 블록 2
  ],
});
```

### 블록 타입

**header** — 제목

```typescript
{
  type: 'header',
  text: { type: 'plain_text', text: '📋 승인 요청' },
}
```

**section** — 본문 (텍스트 또는 2열 레이아웃)

```typescript
// 한 줄 텍스트
{
  type: 'section',
  text: { type: 'mrkdwn', text: '*굵게* _기울임_ `코드`' },
}

// 2열 레이아웃 (fields)
{
  type: 'section',
  fields: [
    { type: 'mrkdwn', text: '*요청자:*\n홍길동' },
    { type: 'mrkdwn', text: '*금액:*\n500만원' },
  ],
}
```

**actions** — 버튼

```typescript
{
  type: 'actions',
  elements: [
    {
      type: 'button',
      text: { type: 'plain_text', text: '✅ 승인' },
      style: 'primary',              // 초록색 (없으면 회색)
      action_id: 'approve_request',  // ← 핸들러와 연결되는 키
      value: JSON.stringify({        // ← 클릭 시 전달할 데이터
        requester: 'U034ZC28ZDX',
        content: '마케팅 예산',
      }),
    },
    {
      type: 'button',
      text: { type: 'plain_text', text: '❌ 거절' },
      style: 'danger',              // 빨간색
      action_id: 'reject_request',
      value: JSON.stringify({ ... }),
    },
  ],
}
```

**divider** — 구분선

```typescript
{ type: 'divider' }
```

**context** — 작은 부가 정보 (회색 텍스트)

```typescript
{
  type: 'context',
  elements: [
    { type: 'mrkdwn', text: '요청 시간: 2026-02-11 17:30' },
  ],
}
```

### 버튼 → 핸들러 연결 원리

```
버튼 정의:   action_id: 'approve_request'
                         ↓ 클릭하면
핸들러:      this.app.action('approve_request', handler)
                         ↓ handler에서
데이터 복원: body.actions[0].value → JSON.parse → { requester, content }
```

버튼의 `value`에 JSON을 넣어서 상태를 전달한다. DB 없이도 "누가 요청했고 내용이 뭔지"를 버튼 클릭 시점에 복원할 수 있다.

### 직접 만들어보기

Slack에서 **Block Kit Builder**라는 시각적 도구를 제공한다.
블록을 드래그해서 배치하면 JSON이 생성되고, 그걸 코드에 넣으면 된다.

https://app.slack.com/block-kit-builder

---

## 7. 동시 사용자와 연결 수

### "10개 연결 제한"은 동시 사용자가 아니다

```
사용자 A ──→ Slack ──┐
사용자 B ──→ Slack ──┤
사용자 C ──→ Slack ──┼── WebSocket 1개 ──→ 우리 서버
사용자 D ──→ Slack ──┤
  ...100명  Slack ──┘
```

사용자는 Slack에 연결하는 것이고, **우리 서버와 Slack 사이**가 WebSocket이다. 사용자가 몇 명이든 연결 1개로 충분하다.

### 10개 연결은 고가용성(HA) 용도

```
서버 인스턴스 1 ──WebSocket──→ Slack  (연결 1)
서버 인스턴스 2 ──WebSocket──→ Slack  (연결 2)
서버 인스턴스 3 ──WebSocket──→ Slack  (연결 3)
```

서버를 여러 대 띄워서 하나가 죽어도 나머지가 이벤트를 받을 수 있게 하는 용도다.

### 실제 병목은 Slack API Rate Limit

WebSocket 연결 수가 아니라 **Slack API Rate Limit**이 병목이다.
하지만 사내 앱에서 1초에 수십 명이 동시에 봇을 호출하는 경우는 현실적으로 없으니, 일반적인 규모에서는 문제가 되지 않는다.

---

## 8. DCS-AI에서의 구현

### 데이터 흐름

```
채널: @bot @승인자 승인해줘
  │
  ├→ 채널: "승인자님에게 DM 보냈습니다"
  │
  └→ 승인자 DM: [승인] [거절] 버튼
                    │
                    ├→ 승인자 DM: 버튼 → "승인 완료" 텍스트로 교체
                    │
                    └→ 요청자 DM: "OO님이 승인했습니다"
```

### 채널에 없는 사람에게도 DM 가능

`client.chat.postMessage({ channel: 유저ID })`에서 channel에 유저 ID를 넣으면 DM이 된다.
채널 참여 여부와 상관없이, **같은 Slack 워크스페이스 멤버이기만 하면** DM을 보낼 수 있다.

### 파일 구조

```
server/src/common/slack-bot/
├── slack-bot.service.ts  ← 핵심 서비스 (연결, 핸들러 등록)
└── slack-bot.module.ts   ← NestJS 모듈
```

### 환경 변수

```bash
SLACK_BOT_TOKEN=xoxb-...   # Bot Token (API 호출용)
SLACK_APP_TOKEN=xapp-...   # App Token (WebSocket 연결용)
```

토큰이 없으면 봇이 자동으로 비활성화된다.

---

## 참고 자료

- [Comparing HTTP & Socket Mode](https://docs.slack.dev/apis/events-api/comparing-http-socket-mode/)
- [Using Socket Mode](https://docs.slack.dev/apis/events-api/using-socket-mode)
- [Socket Mode Overview](https://api.slack.com/apis/socket-mode)
- [Block Kit Builder](https://app.slack.com/block-kit-builder)
