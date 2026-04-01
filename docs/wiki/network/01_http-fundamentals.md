---
title: "01. HTTP 기초 복습"
parent: Network
permalink: /network/http-fundamentals
nav_order: 1
---

# 01. HTTP 기초 복습
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## TCP — HTTP의 기반

**TCP (Transmission Control Protocol)** 는 인터넷에서 데이터를 **안전하게, 순서대로, 빠짐없이** 전달하는 규약이다. HTTP는 TCP 위에서 동작한다.

### 비유

택배라고 생각하면 된다:
- 큰 짐(데이터)을 여러 상자(패킷)로 나눠서 보냄
- 각 상자에 번호를 매겨서 순서 보장
- 상자 도착하면 수신자가 "잘 받았어" 확인(ACK) 보냄
- 안 오면 다시 보냄

### TCP 연결 수립 (3-way handshake)

```
클라이언트                서버
    │                       │
    │── SYN ──────────────▶│  "연결하고 싶어"
    │                       │
    │◀── SYN + ACK ────────│  "좋아, 나도 준비됐어"
    │                       │
    │── ACK ──────────────▶│  "확인, 시작하자"
    │                       │
    │══ 데이터 전송 시작 ═══│
```

### HTTP와의 관계

```
[네트워크 계층 구조]

HTTP   ← "어떤 데이터를 주고받을지" (요청/응답 포맷)
TCP    ← "데이터를 안전하게 전달" (순서 보장, 재전송)
IP     ← "어디로 보낼지" (주소 라우팅)
```

HTTP 요청을 보내려면 먼저 TCP 연결을 맺고, 그 파이프 위로 HTTP 메시지를 주고받는 구조다. 이 문서에서 나오는 "TCP 연결", "TCP FIN", "TCP 핸드셰이크"가 전부 이 개념이다.

### TCP 연결은 매번 새로 맺나?

HTTP 버전에 따라 다르다:

```
[HTTP/1.0 — 매 요청마다 연결]

요청1: TCP 연결 → 요청 → 응답 → TCP 종료
요청2: TCP 연결 → 요청 → 응답 → TCP 종료  ← 또 연결!
요청3: TCP 연결 → 요청 → 응답 → TCP 종료  ← 또 연결!

→ 매번 3-way handshake 비용 발생. 느림.
```

```
[HTTP/1.1 — Keep-Alive (기본값)]

TCP 연결 한 번
    │── 요청1 → 응답1
    │── 요청2 → 응답2  ← 같은 연결 재사용!
    │── 요청3 → 응답3  ← 같은 연결 재사용!
    │
    (일정 시간 안 쓰면 자동 종료)

→ Connection: keep-alive가 기본값. 한 번 연결하면 재사용.
→ 다만 한 연결에서 한 번에 하나씩 순차 처리.
```

```
[HTTP/2 — 하나의 연결로 전부]

TCP 연결 딱 하나
    │── 스트림1: 요청A/응답A ─────▶
    │── 스트림2: 요청B/응답B ──▶     ← 동시에!
    │── 스트림3: 요청C/응답C ───▶    ← 동시에!
    │
    (연결 하나로 모든 요청 병렬 처리)

→ 멀티플렉싱으로 하나의 연결에서 동시 처리.
```

이 연결 재사용 방식의 차이가 곧 HTTP/1.1 vs HTTP/2 성능 차이의 핵심이다.

---

## HTTP/1.1 vs HTTP/2

### HTTP/1.1의 한계

HTTP/1.1은 **하나의 TCP 연결에서 한 번에 하나의 요청-응답**만 처리할 수 있다. 브라우저는 이 제약을 우회하기 위해 도메인당 6~8개의 TCP 연결을 동시에 열지만, 각 연결마다 TCP 핸드셰이크 + TLS 핸드셰이크 비용이 발생한다.

**Head-of-Line Blocking** 문제도 있다. 앞선 요청이 느리면 뒤의 요청도 대기해야 한다.

```
[HTTP/1.1 — 직렬 처리]

연결1: ──요청A──▶◀──응답A── ──요청B──▶◀──응답B──
연결2: ──요청C──▶◀──응답C── ──요청D──▶◀──응답D──
연결3: ──요청E──▶◀──응답E── (대기...)

→ 각 연결에서 하나씩 순차 처리
→ 도메인당 6개 연결 제한
```

### HTTP/2의 개선

HTTP/2는 **하나의 TCP 연결 위에서 여러 스트림을 동시에** 처리한다(멀티플렉싱). 바이너리 프레임 레이어를 도입해서, 하나의 연결을 논리적 스트림으로 분할한다.

```
[HTTP/2 — 멀티플렉싱]

단일 TCP 연결:
  스트림1: ─프레임─프레임─프레임─▶ (요청A/응답A)
  스트림2: ──프레임──프레임──▶      (요청B/응답B)
  스트림3: ─프레임──프레임─프레임─▶ (요청C/응답C)

→ 하나의 연결에서 병렬 처리
→ 연결 수 제한 없이 동시 전송
```

| 특성 | HTTP/1.1 | HTTP/2 |
|------|----------|--------|
| 프로토콜 형식 | 텍스트 기반 | 바이너리 프레임 |
| 동시 요청 | 연결당 1개 | 연결당 다수 (스트림) |
| 헤더 처리 | 매 요청마다 전체 전송 | HPACK 압축 (중복 제거) |
| 서버 푸시 | 불가 | 가능 (서버가 먼저 리소스 전송) |
| HOL Blocking | TCP 연결 레벨 | TCP 레벨에선 여전히 존재 |

**HPACK 헤더 압축**: HTTP/1.1에서는 매 요청마다 `Cookie`, `User-Agent` 같은 수백 바이트의 헤더를 반복 전송한다. HTTP/2는 정적/동적 테이블을 이용해 이전에 보낸 헤더를 인덱스로 참조하고, 새로운 헤더만 허프만 코딩으로 압축해서 보낸다.

---

## Transfer-Encoding: chunked

일반 HTTP 응답은 `Content-Length` 헤더로 전체 크기를 미리 알려준다. 하지만 **AI 응답처럼 생성 중인 데이터**는 전체 크기를 알 수 없다. 이때 `Transfer-Encoding: chunked`를 사용한다.

### 동작 원리

```
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Content-Type: text/plain

7\r\n         ← 청크 크기 (16진수로 7바이트)
Hello, \r\n   ← 청크 데이터
6\r\n         ← 다음 청크 크기
World!\r\n    ← 청크 데이터
0\r\n         ← 크기 0 = 전송 완료
\r\n          ← 트레일러 종료
```

**핵심**: 각 청크는 `크기\r\n데이터\r\n` 형식이고, 크기 0인 청크가 오면 전송이 끝났다는 신호다.

---

## Content-Type: text/event-stream (SSE의 기반)

`text/event-stream`은 SSE(Server-Sent Events)에서 사용하는 MIME 타입이다. 서버가 이 Content-Type으로 응답하면, 브라우저는 이 연결을 **이벤트 스트림**으로 인식하고 자동으로 파싱한다.

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"token": "안녕"}\n\n
data: {"token": "하세요"}\n\n
event: done\ndata: {}\n\n
```

### Chunked Transfer vs SSE 상세 비교

| | **Chunked Transfer** | **SSE (Server-Sent Events)** |
|---|---|---|
| **본질** | HTTP 전송 메커니즘 (Transport) | 이벤트 프로토콜 (Application) |
| **포맷** | 원시 바이트 스트림, 구조 없음 | `data:`, `event:`, `id:` 규격화된 포맷 |
| **파싱** | 클라이언트가 직접 구현 | 브라우저 `EventSource` API가 자동 처리 |
| **HTTP 메서드** | POST, GET 등 자유 | EventSource는 **GET만** 가능 |
| **커스텀 헤더** | fetch()로 자유롭게 설정 | EventSource는 **불가** |
| **자동 재연결** | 직접 구현해야 함 | 브라우저가 자동 재연결 + `Last-Event-ID`로 이어받기 |
| **이벤트 구분** | 직접 구분자 설계 필요 | `event:` 필드로 타입 분리 가능 |
| **바이너리** | 가능 | 텍스트만 (바이너리는 Base64 인코딩 필요) |

**chunked가 유리한 경우**: POST로 body를 보내야 하거나, 인증 헤더가 필요하거나, 자체 프로토콜이 있을 때. dcs-ai가 이 케이스 — `fetch()` + `ReadableStream`으로 chunked를 직접 읽는다.

**SSE가 유리한 경우**: 단순 구독형 스트림 (알림 피드, 주식 시세). GET 하나 열어두면 브라우저가 재연결/이어받기까지 알아서 해준다.

**실무 하이브리드**: SSE 포맷(`data:\n\n`)을 쓰되 `EventSource` 대신 `fetch()`로 수신하는 방식이 많다. POST + 커스텀 헤더의 자유도를 가져가면서, SSE 포맷의 파싱 편의성도 챙기는 방식이다.

---

## CORS (Cross-Origin Resource Sharing)

### 왜 필요한가

브라우저는 **Same-Origin Policy**를 적용한다. `https://app.example.com`에서 실행되는 JavaScript가 `https://api.example.com`으로 요청을 보내면, 브라우저가 차단한다. 출처(Origin = 프로토콜 + 도메인 + 포트)가 다르기 때문이다.

```
[Same-Origin Policy]

https://app.example.com (프론트엔드)
    │
    ├──▶ https://app.example.com/api  ✅ 같은 출처 → 허용
    │
    └──▶ https://api.example.com/chat ❌ 다른 출처 → 차단!
```

### CORS 해결 흐름

서버가 응답 헤더에 `Access-Control-Allow-Origin`을 포함시키면 브라우저가 허용한다.

```
[Preflight 요청 흐름 (Content-Type: application/json 등 비단순 요청)]

클라이언트                           서버
    │                                  │
    │── OPTIONS /chat ───────────────▶│  ← 사전 확인(Preflight)
    │   Origin: https://app.example.com│
    │                                  │
    │◀── 200 OK ──────────────────────│
    │   Access-Control-Allow-Origin:   │
    │     https://app.example.com      │
    │   Access-Control-Allow-Methods:  │
    │     POST                         │
    │                                  │
    │── POST /chat ──────────────────▶│  ← 실제 요청
    │                                  │
    │◀── 200 OK (스트리밍 응답) ──────│
```

### CORS_ORIGINS 환경변수

dcs-ai 서버에서 `CORS_ORIGINS` 환경변수를 설정하는 이유:

```typescript
// NestJS main.ts 예시
app.enableCors({
  origin: process.env.CORS_ORIGINS?.split(','), // ['https://app.example.com']
  credentials: true,
});
```

- 허용할 출처를 명시적으로 지정해서 **인가된 프론트엔드만** API에 접근 가능
- 와일드카드(`*`)를 쓰면 **아무 사이트에서나** API를 호출할 수 있어 위험
- `credentials: true`와 와일드카드는 **동시에 사용 불가** (브라우저가 거부)
