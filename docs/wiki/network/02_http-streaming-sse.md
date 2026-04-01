---
title: "02. HTTP Streaming & SSE"
parent: Network
permalink: /network/http-streaming-sse
nav_order: 2
---

# 02. HTTP Streaming & SSE
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## HTTP Streaming 동작 원리

HTTP Streaming은 서버가 응답을 한 번에 보내지 않고, **생성되는 대로 조각(chunk)씩 밀어내는** 방식이다. TCP 연결은 하나 유지하면서, 응답 본문을 점진적으로 전송한다.

### 서버에서 청크를 밀어내는 과정

```
[HTTP Streaming 내부 동작]

1. 클라이언트 → TCP 연결 수립 → HTTP 요청 전송
2. 서버 → 응답 헤더 전송 (Transfer-Encoding: chunked)
3. 서버 → 데이터가 준비될 때마다 청크 전송
   ┌──────────────────────────────────────┐
   │  서버 내부 (NestJS)                    │
   │                                        │
   │  Anthropic SDK                         │
   │    │                                   │
   │    ├── 토큰 생성 → res.write(chunk1)  │──▶ TCP 전송
   │    ├── 토큰 생성 → res.write(chunk2)  │──▶ TCP 전송
   │    ├── 토큰 생성 → res.write(chunk3)  │──▶ TCP 전송
   │    └── 완료      → res.end()          │──▶ TCP FIN
   │                                        │
   └──────────────────────────────────────┘
4. 클라이언트 → 청크 수신 즉시 처리 (화면 표시)
5. 서버 → 마지막 청크(크기 0) 전송 → 연결 종료
```

### 핵심 메커니즘

**TCP 수준에서 일어나는 일:**
- 서버가 `res.write(data)`를 호출하면, Node.js는 데이터를 TCP 송신 버퍼에 쓴다
- TCP 스택이 데이터를 세그먼트로 쪼개서 전송
- `res.end()`를 호출하면 TCP FIN 패킷으로 연결 종료를 알림

**HTTP 수준에서 일어나는 일:**
- `Transfer-Encoding: chunked` 헤더가 있으면, HTTP 파서는 `Content-Length` 없이도 데이터를 수신
- 각 청크의 크기 필드를 읽고 → 해당 크기만큼 데이터를 읽고 → 다음 청크 대기
- 크기가 0인 청크 = 응답 완료

### 버퍼링 주의사항

스트리밍이 실제로 작동하려면 **중간 계층의 버퍼링을 비활성화**해야 한다:

```
[버퍼링이 스트리밍을 망치는 경우]

서버 → [Nginx 버퍼] → [CDN 버퍼] → 클라이언트
         ↑ 여기서 청크를 모아두다가
           한꺼번에 보내면 스트리밍 효과 사라짐
```

Nginx 설정 예시:
```nginx
location /chat {
    proxy_buffering off;          # 프록시 버퍼링 비활성화
    proxy_cache off;              # 캐싱 비활성화
}
```

또는 NestJS 서버에서 응답 헤더로 직접 제어할 수도 있다:
```typescript
res.setHeader('X-Accel-Buffering', 'no');  // Nginx에게 이 응답은 버퍼링하지 말라고 알림
```

---

## SSE (Server-Sent Events)

SSE는 HTTP Streaming 위에 **표준화된 이벤트 포맷**을 얹은 프로토콜이다.

### HTTP Streaming vs SSE

```
[일반 HTTP Streaming]
응답 본문: "안녕하세요, 무엇을 도와드릴까요?"
→ 원시 텍스트. 어디서 끊어 읽을지 클라이언트가 판단해야 함

[SSE]
응답 본문:
data: {"token":"안녕"}\n\n
data: {"token":"하세요"}\n\n
data: {"token":", 무엇을"}\n\n
event: done\ndata: {}\n\n
→ 각 이벤트가 \n\n로 구분. 브라우저가 자동 파싱
```

### SSE 이벤트 포맷

```
[SSE 메시지 구조]

event: message_start          ← 이벤트 타입 (선택)
id: 42                        ← 이벤트 ID (선택, 재연결 시 Last-Event-ID로 전송)
retry: 3000                   ← 재연결 대기 시간 ms (선택)
data: {"type":"start"}        ← 데이터 (필수, 여러 줄 가능)
                              ← 빈 줄(\n\n)로 이벤트 종료
```

### EventSource API

브라우저 내장 API로, SSE를 쉽게 수신할 수 있다:

```javascript
const source = new EventSource('/api/stream');

// 기본 메시지 수신
source.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log(data.token); // 토큰 하나씩 수신
};

// 커스텀 이벤트 수신
source.addEventListener('done', (event) => {
  source.close(); // 스트림 종료
});

// 에러 처리 (자동 재연결 시도)
source.onerror = (event) => {
  if (source.readyState === EventSource.CLOSED) {
    console.log('서버가 연결을 종료함');
  }
  // CONNECTING 상태면 브라우저가 자동으로 재연결 시도
};
```

### SSE의 장점과 한계

| 장점 | 한계 |
|------|------|
| 자동 재연결 (retry 필드) | 단방향만 가능 (서버→클라이언트) |
| Last-Event-ID로 이어받기 | HTTP/1.1에서 도메인당 6개 연결 제한 |
| 텍스트 기반으로 디버깅 쉬움 | 바이너리 데이터 전송 불편 |
| 프록시/방화벽 친화적 | GET 요청만 지원 (POST 불가) |

{: .important }
> **EventSource의 한계**: GET만 지원하고 커스텀 헤더를 보낼 수 없다. 그래서 실무에서는 `fetch()` + `ReadableStream`으로 직접 구현하거나, `@ai-sdk/react` 같은 라이브러리를 사용한다.

---

## ReadableStream — 클라이언트에서 스트림 수신

`fetch()` API는 응답 본문을 `ReadableStream`으로 제공한다. 이걸 이용하면 **SSE의 제약 없이** POST 요청 + 커스텀 헤더 + 스트리밍 수신이 가능하다.

### 기본 사용법

```javascript
async function streamChat(message) {
  const response = await fetch('/api/chat', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer token',
    },
    body: JSON.stringify({ message }),
  });

  // response.body가 ReadableStream
  const reader = response.body.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    // value는 Uint8Array → 문자열로 디코딩
    const chunk = decoder.decode(value, { stream: true });
    console.log(chunk); // 청크 단위로 수신
  }
}
```

### 스트림 파싱 흐름

```
[ReadableStream 처리 파이프라인]

fetch() 응답
    │
    ▼
response.body (ReadableStream<Uint8Array>)
    │
    ▼
reader.read() ──▶ { done: false, value: Uint8Array }
    │                          │
    │                          ▼
    │               TextDecoder.decode()
    │                          │
    │                          ▼
    │                "data: {\"token\":\"안녕\"}\n\n"
    │                          │
    │                          ▼
    │               SSE 라인 파싱 (data: 접두사 제거)
    │                          │
    │                          ▼
    │               JSON.parse() → { token: "안녕" }
    │                          │
    │                          ▼
    │               UI 업데이트 (setState)
    │
    ▼
{ done: true } → 스트림 종료
```

### @ai-sdk/react가 내부적으로 하는 일

`useChat()` 훅을 호출하면 내부적으로 이런 과정이 일어난다:

```
[useChat() 내부 흐름]

1. 사용자 메시지 입력
2. fetch('/api/chat', { method: 'POST', body: messages })
3. response.body → ReadableStream 획득
4. AI SDK의 스트림 프로토콜에 따라 청크 파싱
   - 텍스트 델타: "0:토큰텍스트\n"
   - 도구 호출: "9:{...}\n"
   - 완료 신호: "d:{...}\n"
5. 파싱된 토큰을 React 상태에 누적
6. 상태 변경 → 컴포넌트 리렌더링 → 화면에 글자 추가
```
