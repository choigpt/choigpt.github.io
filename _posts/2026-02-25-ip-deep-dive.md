---
layout: post
title: "네트워크 — OSI 7계층 & TCP/IP 심층 정리 (v2)"
date: 2026-02-25 00:00:00
tags: [네트워크, CS, TCP, IP, 소켓, Spring]
category: CS
---

> 참조: 영상(OSI 7계층 강의), IEEE 802.3, RFC 791(IP), RFC 793(TCP), RFC 768(UDP), Shannon-Hartley Theorem, Linux Kernel Source

---

## 출발점 — 왜 전선에 0101을 그냥 보내면 안 되는가?

모든 파일과 프로그램은 0과 1의 나열이다. 두 컴퓨터가 통신한다는 것 = 0과 1을 주고받는 것.

단순하게 생각하면: `1 = +5V`, `0 = -5V` 전압을 전선으로 흘려보내면 되지 않을까?

**안 된다.** 핵심 이유는 **전자기파의 주파수 특성**에 있다.

### 전자기파와 주파수

모든 전기 신호는 전자기파다. 시간(X축) vs 전압(Y축)의 변화 그래프로 표현된다.

- **주파수(Hz)**: 1초당 진동 횟수. 1초에 4번 진동 → 4Hz
- 전선은 모든 주파수를 통과시키지 못한다 → 특정 범위(대역폭)만 통과

직류 신호(계속 +5V)는 0Hz다. 일반 전선의 하한 주파수는 0Hz보다 높아서, 직류 신호는 통과 자체가 안 된다. 0과 1을 단순히 전압 변화로 보내면 전선 통과 불가.

**해결책**: 디지털 신호를 **반송파(carrier wave)에 실어 보내는 것 = 변조(Modulation)**. 이것이 모뎀(Modem)의 역할.

---

## 1계층 — 물리 계층 (Physical Layer)

디지털 신호를 아날로그로, 아날로그를 다시 디지털로 변환하는 계층.

- 신호: 전기(구리선), 빛(광섬유), 전파(Wi-Fi)
- 장치: 모뎀, 리피터, 허브
- 주소 없음, 오류 감지 없음 → 그냥 0과 1을 신호로 전달

### 여러 컴퓨터 연결 문제

하나의 전선에 N대가 연결되면 신호가 모두에게 간다. 이를 해결하는 것이 **스위치(Layer 2)**와 **라우터(Layer 3)**.

---

## 2계층 — 데이터 링크 계층 (Data Link Layer)

### 프레이밍 (Framing)

연속된 비트열을 의미 있는 단위로 끊어야 한다.

```
[ Preamble | SFD | Dst MAC | Src MAC | EtherType | Payload | FCS ]
    7B        1B     6B        6B         2B        46~1500B   4B
```

- **Preamble**: 동기화 신호 (10101010 × 7)
- **SFD**: 프레임 시작 구분자 (10101011)
- **FCS**: 오류 검출 (CRC-32)
- → **랜카드(NIC)**가 담당

### MAC 주소

하드웨어에 구운 48비트 주소. `AA:BB:CC:DD:EE:FF`
- 앞 24비트: 제조사(OUI)
- 뒤 24비트: 시리얼

같은 스위치 내에서만 유효. 라우터를 넘어가면 MAC 주소가 교체된다.

---

## 3계층 — 네트워크 계층 (Network Layer)

### IP 주소와 패킷

서로 다른 네트워크 간 통신 담당. **IP 주소**로 목적지 식별.

**IPv4 패킷 헤더 (주요 필드)**:

```
Version(4b) | IHL(4b) | DSCP+ECN(8b) | Total Length(16b)
Identification(16b) | Flags(3b) | Fragment Offset(13b)
TTL(8b) | Protocol(8b) | Header Checksum(16b)
Source IP(32b)
Destination IP(32b)
```

- **TTL**: 라우터를 거칠 때마다 1씩 감소. 0이 되면 패킷 폐기 (루프 방지)
- **Protocol**: 상위 계층 프로토콜 번호 (6=TCP, 17=UDP, 1=ICMP)

### 라우팅

라우터는 목적지 IP를 보고 다음 홉(hop)을 결정한다.

```
목적지 패킷 도착
    ↓
라우팅 테이블 참조 (목적지 IP → 다음 라우터 IP)
    ↓
다음 라우터로 전달 (이때 MAC 주소는 교체, IP는 유지)
```

---

## 4계층 — 전송 계층 (Transport Layer)

### 포트 번호

IP 주소 = 컴퓨터 식별 / 포트 번호 = 프로세스 식별

- Well-known: 0~1023 (HTTP=80, HTTPS=443, SSH=22)
- Registered: 1024~49151
- Dynamic: 49152~65535 (클라이언트 임시 포트)

### TCP

신뢰성 있는 데이터 전송.

**세그먼트 헤더**:
```
Source Port(16b) | Destination Port(16b)
Sequence Number(32b)
Acknowledgment Number(32b)
Data Offset(4b) | Reserved(3b) | Flags(9b) | Window Size(16b)
Checksum(16b) | Urgent Pointer(16b)
```

**3-Way Handshake**:
```
Client → Server: SYN (seq=x)
Server → Client: SYN-ACK (seq=y, ack=x+1)
Client → Server: ACK (ack=y+1)
연결 수립 ✅
```

**4-Way Termination**:
```
Client → Server: FIN
Server → Client: ACK
Server → Client: FIN
Client → Server: ACK → TIME_WAIT(2MSL) → CLOSED
```

**흐름 제어**: Window Size로 수신 버퍼 크기 알림
**혼잡 제어**: Slow Start, AIMD, Fast Retransmit

### UDP

헤더 8바이트. Source Port | Dest Port | Length | Checksum. 신뢰성 없지만 빠름.

---

## 5-6계층 — 세션 / 프레젠테이션 계층

TCP/IP 4계층 모델에서는 Application에 통합.

- **세션**: 연결 수립/유지/종료 관리. TCP가 상당 부분 담당.
- **프레젠테이션**: 데이터 형식 변환, 암호화(TLS), 압축.

---

## 7계층 — 응용 계층 (Application Layer)

소켓 API를 통해 애플리케이션이 네트워크를 사용한다.

```java
// Java 소켓 프로그래밍 예시
ServerSocket serverSocket = new ServerSocket(8080);
Socket client = serverSocket.accept();
InputStream in = client.getInputStream();
OutputStream out = client.getOutputStream();
```

OS가 제공하는 `socket()`, `bind()`, `listen()`, `accept()`, `connect()`, `send()`, `recv()`, `close()` 시스템 콜 위에 HTTP, WebSocket, gRPC 등이 구현된다.

---

## Spring에서 네트워크 계층 활용

### WebSocket / STOMP

```
HTTP Upgrade → WebSocket 연결 (TCP 지속 연결)
STOMP는 WebSocket 위에서 메시지 라우팅 담당
```

### SSE (Server-Sent Events)

```
HTTP/1.1 기반, TCP 연결 유지
Content-Type: text/event-stream
단방향 (서버 → 클라이언트)
```

### Spring Boot 포트 관리

`application.yml`의 `server.port`가 Transport Layer의 포트를 지정한다. 8080 기본값.

---

## 요약

| 계층 | 주요 개념 | 장치/구현 |
|------|-----------|-----------|
| 7 Application | HTTP, WebSocket, DNS | 앱 소프트웨어 |
| 6 Presentation | TLS, 인코딩 | 라이브러리 |
| 5 Session | 연결 관리 | OS/라이브러리 |
| 4 Transport | TCP/UDP, 포트 | OS 커널 |
| 3 Network | IP, 라우팅 | OS 커널, 라우터 |
| 2 Data Link | MAC, 프레임, Ethernet | NIC, 스위치 |
| 1 Physical | 신호, 변조 | 모뎀, 케이블 |
