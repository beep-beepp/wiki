---
title: "03. WebSocket"
parent: Network
permalink: /network/websocket
nav_order: 3
---

# 03. WebSocket
{: .no_toc }

## Table of Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## WebSocket 핸드셰이크

WebSocket은 HTTP에서 시작해서 프로토콜을 **업그레이드**하는 방식으로 연결을 수립한다.

### 핸드셰이크 과정

```
[WebSocket 핸드셰이크]

클라이언트                                    서버
    │                                           │
    │── GET /ws HTTP/1.1 ─────────────────────▶│
    │   Host: api.example.com                   │
    │   Upgrade: websocket         ← 핵심 헤더  │
    │   Connection: Upgrade                     │
    │   Sec-WebSocket-Key: dGhlIHNhbXBsZS...   │
    │   Sec-WebSocket-Version: 13              │
    │                                           │
    │◀── HTTP/1.1 101 Switching Protocols ─────│ ← 101 = 프로토콜 전환
    │    Upgrade: websocket                     │
    │    Connection: Upgrade                    │
    │    Sec-WebSocket-Accept: s3pPLMBiT...    │
    │                                           │
    │◀══════════ WebSocket 연결 수립 ══════════▶│
    │     (이후 HTTP가 아닌 WS 프레임으로 통신)   │
    │                                           │
    │══ WS 프레임: "안녕하세요" ══════════════▶│
    │◀══ WS 프레임: "네, 반갑습니다" ══════════│
    │                                           │
    │══ Close 프레임 ════════════════════════▶│ ← 정상 종료
    │◀══ Close 프레임 ════════════════════════│
```

### 핵심 헤더 설명

| 헤더 | 역할 |
|------|------|
| `Upgrade: websocket` | HTTP → WebSocket으로 프로토콜 전환 요청 |
| `Connection: Upgrade` | 현재 연결을 업그레이드하겠다는 의미 |
| `Sec-WebSocket-Key` | 클라이언트가 생성한 랜덤 Base64 키 |
| `Sec-WebSocket-Accept` | 서버가 Key를 매직 스트링과 합쳐 SHA-1 해시 후 Base64 인코딩한 값 |
| `101 Switching Protocols` | HTTP → WebSocket 전환 성공 응답 코드 |

**Sec-WebSocket-Accept 검증**: 서버는 `Sec-WebSocket-Key` + `"258EAFA5-E914-47DA-95CA-C5AB0DC85B11"` (고정 매직 스트링)을 SHA-1 해시하고 Base64 인코딩해서 반환한다. 클라이언트는 이 값을 검증해서 서버가 진짜 WebSocket을 지원하는지 확인한다.

---

## HTTP Streaming vs WebSocket 비교

### 통신 패턴 비교

```
[HTTP Streaming — 단방향, 요청-응답 모델]

클라이언트                        서버
    │── POST /chat ──────────▶│
    │                          │
    │◀── chunk: 토큰1 ────────│
    │◀── chunk: 토큰2 ────────│
    │◀── chunk: 토큰3 ────────│
    │◀── [종료] ──────────────│
    │                          │
    │── POST /chat ──────────▶│  ← 새 질문 = 새 요청
    │◀── chunk: 토큰1 ────────│
    ...

→ 매 질문마다 새 HTTP 요청
→ 서버→클라이언트 단방향 스트림
```

```
[WebSocket — 양방향, 지속 연결]

클라이언트                        서버
    │══ 핸드셰이크 ═══════════▶│
    │◀══════════════════════════│
    │                          │
    │══ "안녕?" ══════════════▶│
    │◀══ "안녕하세요!" ════════│
    │◀══ "무엇을 도와..." ════│  ← 서버가 먼저 보내기도 가능
    │                          │
    │══ "고마워" ═════════════▶│  ← 같은 연결 재사용
    │◀══ "별말씀을요!" ═══════│
    │                          │
    │══ Close ═══════════════▶│
    │◀══ Close ═══════════════│

→ 한번 연결하면 계속 유지
→ 양쪽 모두 자유롭게 전송
```

### 상세 비교표

| 항목 | HTTP Streaming / SSE | WebSocket |
|------|---------------------|-----------|
| **방향** | 단방향 (서버→클라) | 양방향 |
| **프로토콜** | HTTP (기존 인프라 호환) | ws:// / wss:// (별도 프로토콜) |
| **연결 수명** | 요청당 1개 (짧은 수명) | 장기 연결 (수분~수시간) |
| **오버헤드** | HTTP 헤더 (요청마다) | 프레임 헤더 2~14바이트 (매우 낮음) |
| **프록시 통과** | 자연스러움 | 일부 프록시에서 문제 |
| **로드밸런서** | 표준 HTTP LB 사용 | Sticky Session 또는 WS 지원 LB 필요 |
| **재연결** | SSE는 자동 / fetch는 수동 | 직접 구현해야 함 |
| **HTTP/2 호환** | 완벽 (멀티플렉싱 활용) | HTTP/2 위에서는 별도 표준 필요 |
| **서버 스케일링** | Stateless (쉬움) | Stateful 연결 관리 (복잡) |

### 언제 뭘 써야 하는가

**HTTP Streaming / SSE가 적합한 경우:**
- AI 챗봇 응답 (질문→답변 단방향)
- 실시간 알림 피드
- 주식 시세 업데이트
- 빌드/배포 로그 스트리밍

**WebSocket이 적합한 경우:**
- 실시간 채팅 (다자간 양방향)
- 온라인 게임 (저지연 양방향)
- 협업 도구 (Google Docs, Figma)
- IoT 디바이스 양방향 제어

---

## 양방향 통신이 필요한 케이스

### 실시간 채팅

```
[다자간 채팅 — WebSocket 필요]

User A ══▶ 서버 ══▶ User B
                ══▶ User C
                ══▶ User D

User B ══▶ 서버 ══▶ User A
                ══▶ User C
                ══▶ User D

→ 누구든 메시지를 보내면 즉시 다른 참가자에게 전파
→ HTTP로는 "User B가 메시지를 보냈는지" 폴링해야 함
→ WebSocket이면 서버가 즉시 push 가능
```

### 협업 도구 (Google Docs 스타일)

```
[동시 편집]

User A: "Hello" 입력 ══▶ 서버 ══▶ User B 화면에 실시간 반영
User B: 커서 이동     ══▶ 서버 ══▶ User A 화면에 커서 위치 표시

→ 양방향: 둘 다 동시에 편집하고 상대방 변경을 즉시 수신
→ 저지연 필요: 타이핑할 때마다 즉시 전파
→ 높은 빈도: 초당 수십 개 이벤트
```

### 온라인 게임

```
[게임 상태 동기화]

플레이어 입력 (60fps) ══▶ 서버 ══▶ 게임 상태 브로드캐스트
                                      ║
                          ◀════════════╝
                          모든 플레이어에게 상태 전파

→ 초저지연 필수 (16ms 프레임마다)
→ HTTP 오버헤드가 병목
→ WebSocket 프레임 헤더 2바이트면 충분
```
