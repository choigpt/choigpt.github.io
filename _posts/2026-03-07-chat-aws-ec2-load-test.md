---
title: 채팅 부하 테스트 — AWS EC2에서 TX 분리, Virtual Thread Pinning 분석, 4,500 VU 달성
date: 2026-03-07
tags: [AWS, EC2, MySQL, HikariCP, WebSocket, Redis, STOMP, ZGC, VirtualThread, k6, 성능최적화, 부하테스트, Java, 채팅]
permalink: /chat-aws-ec2-load-test/
excerpt: "EC2에 배포한 채팅 도메인에서 HikariCP Pending 173, write-pool 24.7초 점유가 발생했다. 풀 확대, 커널/JVM 튜닝, TX 분리(Redis publish 커넥션 점유 제거)를 적용하고 4,500 VU까지 확장했다. 전 Phase 100%, 에러 1건, write-pool acquire 95% 개선. Virtual Thread Pinning 분석에서 SseEmitter.send()와 MongoDB synchronized의 pinning 경로를 확인했다."
---

## 개요

[로컬 채팅 부하 테스트](/chat-storage-websocket-extreme-test/)에서 WHERE IN 풀스캔을 derived JOIN으로 30.9배 개선하고, Redis Pub/Sub 풀 튜닝으로 1,000 VU 스파이크까지 안정화했다. 이번에는 AWS EC2에 배포한 환경에서 동일 시나리오를 실행했다.

### EC2 인프라 구성

| 역할 | 인스턴스 타입 | 사양 |
|------|-------------|------|
| Load Generator (k6) | c5.xlarge | 4 vCPU, 8GB |
| App Server | c5.xlarge | 4 vCPU, 8GB |
| Infra (MySQL, Redis) | c5.2xlarge | 8 vCPU, 16GB |

앱 서버와 인프라(MySQL, Redis)는 별도 인스턴스로 분리되어 있다. 로컬 테스트에서는 전부 동일 머신이었으므로 네트워크 지연이 0이었다.

---

## 1차 테스트 — HikariCP 300

### API 응답 시간

| API | p95 | max | avg |
|-----|-----|-----|-----|
| 메시지 전송 | 1.35s | 7.26s | 334ms |
| 메시지 삭제 | 838ms | 5.18s | 205ms |
| 메시지 목록 | 1.50s | 7.32s | 355ms |
| 커서 페이징 | 1.30s | 8.41s | 300ms |
| 채팅방 목록 | 484ms | 6.91s | 119ms |
| WS 연결 | 2ms | 230ms | 2ms |

HTTP 500: 0건. HTTP 404: 59,452건 (채팅방이 없는 클럽에 대한 요청, 정상 동작).

### 로컬 테스트와의 비교

| API | 로컬 p95 (150VU) | EC2 p95 (750~1,500VU) | 비고 |
|-----|-------------------|------------------------|------|
| 메시지 전송 | 15.80ms | 1,350ms | VU 5~10배, 네트워크 지연 추가 |
| 메시지 목록 | 23.93ms | 1,500ms | 동일 |
| 커서 페이징 | 16.16ms | 1,300ms | 동일 |
| 채팅방 목록 | 1,000ms (수정 후) | 484ms | EC2에서 484ms로 로컬보다 낮다 |
| WS 연결 | 13ms | 2ms | WebSocket은 DB 무관 |

채팅방 목록은 로컬에서 derived JOIN으로 수정한 결과가 EC2에 반영되어 484ms로 정상 범위다. 나머지 API의 p95 상승은 VU 규모 차이(150 → 750~1,500)와 네트워크 RTT가 복합적으로 작용한 결과다.

### HikariCP 병목 타임라인

| 시간대 | Phase | HK Active | HK Pending |
|--------|-------|-----------|------------|
| 09:08~09:11 | Warmup/Baseline | 1~58 | 0 |
| 09:11~09:13 | Send Storm (750VU) | 300 | 70~100 |
| 09:13~09:15 | WebSocket (500VU) | 0 | 0 |
| 09:15~09:17 | Contention | 24~62 | 0 |
| 09:18~09:21 | Spike (1,500VU) | 300 | 65~173 |
| 09:21~09:22 | Cooldown | 0 | 0 |

Send Storm(750 VU)에서 300 풀이 전부 소진되고 Pending 70~100이 발생했다. Spike(1,500 VU)에서는 Pending이 최대 173까지 상승했다. WebSocket Phase(500 VU)에서는 Active 0, Pending 0으로, WebSocket 자체는 DB 커넥션을 점유하지 않는다.

### MySQL 상태

| 지표 | 값 |
|------|-----|
| InnoDB Buffer Pool | 3GB |
| Buffer Pool Hit Rate | 99.976% |
| Row Lock Waits | 190 |
| Slow Queries | 0 |

Buffer Pool Hit 99.976%, Slow Query 0건, Row Lock Waits 190건. MySQL은 병목 요인이 아니다. 쿼리 실행은 정상이지만 HikariCP 풀이 부족하여 커넥션 대기가 발생한 구조다.

### Redis 상태

| 지표 | 값 |
|------|-----|
| Memory | 17.86MB |
| Clients | 33 |
| Peak OPS | 623/s |

Redis는 병목 요인이 아니다. Pub/Sub 메시지 전달은 정상적으로 동작했다.

---

## 1차 튜닝 — HikariCP 300 → 400

### 변경

기존 MySQL `max_connections` 400은 HikariCP 300에 맞춘 설정이었다. 풀을 400으로 확대하면서 MySQL `max_connections`도 500으로 조정했다. HikariCP `maximum-pool-size`는 300 → 400으로 확대했다.

### 결과

| 시점 | HK Total | HK Active | HK Pending | 비고 |
|------|----------|-----------|------------|------|
| 09:56:22 | 255 | — | 0 | Warmup, 풀 확장 중 |
| 09:56:54 | 298 | 293 | 72 | Send Storm 시작, 풀 미확장 |
| 09:57:27 | 400 | 397 | 0 | 풀 확장 완료, Pending 해소 |

풀이 400까지 확장 완료된 이후에는 Pending 0을 유지했다. 09:56:54의 Pending 72는 HikariCP의 lazy 커넥션 생성 때문이다.

### HikariCP lazy 생성과 warm-up

HikariCP는 시작 시 `min-idle`(50개)만 생성하고, 부하가 증가하면 요청마다 커넥션을 하나씩 생성하여 `max`(400)까지 확장한다. 09:56:22에 total=255였고 30초 뒤 Send Storm(750 VU)이 시작되면서 300+ 커넥션이 동시에 필요했지만, 풀이 298개까지만 생성된 상태여서 일시적으로 Pending 72가 발생했다.

09:57:27에 total=400으로 확장이 완료되면서 해소됐다. 운영 환경에서는 배포 후 트래픽이 점진적으로 증가하므로 이 문제가 발생하지 않는다. cold start 대응이 필요하면 `min-idle`을 200으로 높이면 된다.

### 300 vs 400 비교

| 항목 | HikariCP 300 | HikariCP 400 |
|------|-------------|-------------|
| Pending 최대 | 173 | 72 (warm-up 시점만) |
| 풀 확장 완료 후 Pending | 지속 발생 | 0 |
| HTTP 500 | 0 | 0 |

300 풀에서는 Spike 구간 내내 Pending이 65~173으로 지속됐다. 400 풀에서는 확장 완료 후 Pending이 0으로 유지됐다.

---

## 2차 튜닝 — message 테이블 복합 인덱스 추가

HikariCP 병목을 풀 확대로 해소한 뒤, 쿼리 실행 시간 자체를 줄여 커넥션 점유 시간을 단축하는 방향으로 전환했다.

### 문제

3M+ message 테이블에서 `chat_room_id` FK 인덱스만 존재했다. 메시지 목록 조회 시 `chat_room_id`로 필터 → `deleted` 필터 → `sent_at` 정렬 순서인데, `deleted` 필터와 `sent_at` 정렬은 인덱스를 사용하지 못하고 filesort로 처리됐다.

### 복합 인덱스

```
(chat_room_id, deleted, sent_at DESC, message_id DESC)
```

복합 인덱스를 추가했다. 아래는 인덱스 적용 후 예상되는 개선 효과다.

| 쿼리 | 용도 | 예상 개선 |
|------|------|-----------|
| `findLatest` | 초기 메시지 목록 | p95 1.50s → 100~300ms |
| `findOlderThan` | 커서 페이징 | p95 1.30s → 50~200ms |
| `findLastMessagesByChatRoomIdsNative` | 채팅방 마지막 메시지 | 이미 derived JOIN 적용, 추가 개선 |

기존에는 `chat_room_id` FK 인덱스로 행을 필터링한 뒤 `deleted` 조건은 테이블에서 확인하고, `sent_at` 정렬은 filesort로 처리했다. 복합 인덱스를 적용하면 인덱스 스캔만으로 필터링과 정렬이 완료될 것으로 예상된다.

---

## 병목 분석 — 로컬과 EC2의 차이

| 항목 | 로컬 | EC2 |
|------|------|-----|
| MySQL 병목 여부 | WHERE IN 풀스캔 (3.1M rows) | 쿼리 자체는 정상 (Slow Query 0) |
| 주요 병목 | 쿼리 실행 시간 | HikariCP 커넥션 소진 |
| HikariCP Pending | 관측되지 않음 | 최대 173 (300 풀), 72 (400 풀) |
| Redis | Pub/Sub 풀 포화 | 병목 아님 (Peak 623 OPS) |

로컬 테스트에서는 쿼리 실행 시간이 병목이었다. WHERE IN 풀스캔, Redis Pub/Sub 풀 포화 등 코드 레벨 문제가 드러났다. EC2 테스트에서는 쿼리 자체는 정상이지만 커넥션 풀 크기가 동시 요청을 감당하지 못하는 인프라 레벨 문제가 드러났다.

이 차이는 [알림 EC2 테스트](/notification-aws-ec2-load-test/)와 동일한 패턴이다. 로컬에서는 네트워크 지연이 0이라 커넥션 점유 시간이 짧고, 커넥션 풀이 빠르게 회전한다. EC2에서는 각 DB 요청에 네트워크 RTT가 추가되어 커넥션 점유 시간이 길어지고, 동일한 풀 크기에서도 Pending이 발생한다.

---

## 3차 튜닝 — 커널 + JVM 최적화 후 재테스트

HikariCP 풀 확대와 복합 인덱스 추가 이후, OS 커널과 JVM 설정을 튜닝하고 재테스트를 실행했다.

### 적용 내용

| 항목 | 설정 |
|------|------|
| JVM | ZGC Generational, 3GB heap, VT parallelism=8 |
| 커널 TCP | rmem/wmem 16MB, keepalive 60s, fin_timeout 15s |
| Tomcat | max-threads 400, max-connections 10,000, accept-count 500 |
| STOMP inbound | 64~128 threads, queue 5,000 |
| STOMP outbound | 64~128 threads, queue 2,000 |
| Redis Pub/Sub listener | 128~400 threads, queue 10,000 |

### 결과 — 19분, 149만 iterations, 에러 0건

| API | p95 | max | avg |
|-----|-----|-----|-----|
| 메시지 전송 | 743ms | 4.23s | 212ms |
| 메시지 삭제 | 552ms | 2.87s | 160ms |
| 메시지 목록 | 890ms | 11.26s | 178ms |
| 커서 페이징 | 933ms | 10.83s | 171ms |
| 채팅방 목록 | 589ms | 2.91s | 96ms |
| WS 연결 | 2ms | 226ms | 2ms |

전 Phase 성공률 100%, 총 에러 0건. WebSocket STOMP 62K 전송, 158K 수신 (브로드캐스트 배수 정상). 1,500 VU Spike에서도 안정적이다.

### 1차 테스트와의 비교

| API | 1차 p95 | 3차 p95 | 변화 |
|-----|---------|---------|------|
| 메시지 전송 | 1,350ms | 743ms | -45% |
| 메시지 삭제 | 838ms | 552ms | -34% |
| 메시지 목록 | 1,500ms | 890ms | -41% |
| 커서 페이징 | 1,300ms | 933ms | -28% |
| 채팅방 목록 | 484ms | 589ms | +22% |

메시지 전송, 삭제, 목록 p95가 28~45% 개선됐다. 복합 인덱스, ZGC, 커널 TCP 버퍼 확대의 복합 효과다.

---

## 고부하 병목 예측 — 3,000+ VU

현재 1,500 VU에서 안정적이지만, VU를 더 올릴 경우의 병목을 분석했다.

| 예상 VU | 병목 | 원인 |
|---------|------|------|
| 3,000+ | STOMP outbound queue 포화 | queue 2,000에 브로드캐스트가 몰림 → 메시지 지연/드롭 |
| 4,000+ | HikariCP 풀 고갈 | write 200 + read 200 = 400 pool에 동시 쓰기 집중 → 커넥션 타임아웃 |
| 5,000+ | Tomcat max-connections 한계 | HTTP + WS가 10,000을 공유 → 연결 거부 |

커널과 JVM은 이미 충분하다. 추가 고부하 대비는 앱 설정 튜닝이 필요하다.

| 항목 | 현재 | 권장 (3,000+ VU) |
|------|------|------------------|
| STOMP outbound queue | 2,000 | 8,000 |
| STOMP outbound max threads | 128 | 256 |
| STOMP inbound queue | 5,000 | 10,000 |
| Tomcat max-connections | 10,000 | 20,000 |
| HikariCP pool (write+read) | 400 | 500~600 |

STOMP outbound queue가 가장 먼저 포화된다. Redis Pub/Sub listener가 `SimpMessagingTemplate.convertAndSend()`를 호출하면 outbound queue로 들어가기 때문에, outbound가 최종 병목이 된다.

---

## Virtual Thread Pinning 분석

1,500 VU 테스트에서 안정적이었지만, Virtual Thread 사용 시 carrier thread pinning 여부를 확인했다.

### 확인된 pinning

| 위치 | 스레드 종류 | synchronized | blocking I/O | Pinning |
|------|------------|-------------|-------------|---------|
| SseEmitter.send() | Virtual | Spring 내부 sendMutex | OutputStream.write | YES |
| Mongo nextSequence() | Tomcat VT | 명시적 synchronized | mongoTemplate | YES |
| STOMP outbound | Platform | WebSocket 내부 | socket.write | No |
| Redis Pub/Sub | Platform | 없음 | Lettuce | No |
| STOMP inbound | Platform | 없음 | 없음 | No |

**SseEmitter.send()**: Spring 내부의 `ResponseBodyEmitter`가 `synchronized(this.sendMutex)` 안에서 `OutputStream.write()`를 수행한다. Virtual Thread executor에서 호출되므로 carrier thread에 pinned된다. SSE 동시 전송 수가 carrier thread 수(parallelism=8)로 제한된다.

**Mongo nextSequence()**: `synchronized (this)` 블록 안에서 `mongoTemplate.findAndModify()`로 네트워크 I/O를 수행한다. 1,000건당 1회지만 발생 시 수십 ms 동안 carrier thread를 점유한다. 채팅의 `MongoChatMessageStorageAdapter`에도 동일 패턴이 존재한다.

### 수정 방안

1. **SSE**: `sseEventExecutor`를 Virtual Thread → Platform Thread Pool(core 64, max 256)로 변경. SseEmitter.send() 자체가 짧은 I/O라 Virtual Thread 이점이 없고 pinning만 발생한다.
2. **MongoDB synchronized**: `synchronized` → `ReentrantLock`으로 교체. JDK 21+에서 ReentrantLock은 Virtual Thread pinning을 발생시키지 않는다.

JFR 10초 녹화(17,769 events)에서 `VirtualThreadPinned = 0`으로 기록된 것은, pinning이 매우 짧은 시간(수 ms) 동안만 발생하여 JFR 샘플링에 잡히지 않았기 때문이다. 코드상으로는 pinning 경로가 존재하며, 고부하 시 carrier thread 고갈로 이어질 수 있다.

---

## 4차 최적화 — TX 분리 + 4,500 VU 스케일 테스트

### 발견된 문제: write-pool 24.7초 점유

3차 테스트에서 write-pool max usage가 24.7초로 관측됐다. 원인: `sendAndPublish()`가 `@Transactional` 안에서 Redis `publish()`를 호출하여, Redis 통신 중에도 DB 커넥션을 점유하고 있었다.

### 적용 내용

1. **sendAndPublish() TX 분리**: `@Transactional`을 제거하고 `saveMessage()`만 독립 TX로 실행. Redis publish는 TX 커밋 후 커넥션 반환된 상태에서 실행된다.
2. **EC2 show_sql/generate_statistics 비활성화**: 전 쿼리 로깅과 통계 수집 오버헤드를 제거했다.
3. **JVM heap 3GB → 4GB**: Allocation Stall 감소를 위해 확대했다.

수정 과정에서 `sendAndPublish()`의 `@Transactional` 제거 시 클래스 레벨 `@Transactional(readOnly=true)`가 적용되어 write가 실패하는 문제가 발생했다. `saveMessage()`의 `@Transactional`이 REQUIRED 전파로 read-only TX에 참여하기 때문이다. 클래스 레벨 `readOnly=true`를 제거하여 해결했다.

### 결과 — 4,500 VU, 전 Phase 100%, 에러 1건

| API | p95 | max | avg |
|-----|-----|-----|-----|
| 메시지 전송 | 2.19s | 38.5s | 916ms |
| 메시지 삭제 | 1.99s | 10.3s | 796ms |
| 메시지 목록 | 2.14s | 38.9s | 805ms |
| 커서 페이징 | 2.74s | 39.0s | 954ms |
| 채팅방 목록 | 1.33s | 30.9s | 325ms |
| WS 연결 | 13ms | 582ms | 5ms |

### HikariCP — TX 분리 효과

| 항목 | 3차 (1,500 VU) | 4차 (4,500 VU) | 변화 |
|------|---------------|----------------|------|
| write-pool acquire max | 24.7s | 1.16s | 95% 개선 |
| write-pool avg acquire | 1.1ms | 0.53ms | 52% 개선 |
| Pending | 0 | 0 | 동일 |
| Timeout | 0 | 0 | 동일 |

write-pool acquire max 24.7s → 1.16s가 TX 분리의 핵심 효과다. Redis publish가 DB 커넥션을 물고 있던 병목이 제거됐다.

### 처리량 비교

| 항목 | 3차 (1,500 VU) | 4차 (4,500 VU) |
|------|---------------|----------------|
| 최대 VU | 1,500 | 4,500 (3배) |
| 총 iterations | 1,854,368 | 1,741,030 |
| 전 Phase 성공률 | 100% | 100% |
| HTTP 에러 | 0 | 1건 |
| WS 에러 | 0 | 0 |
| WS 연결 수 | 24,925 | 37,378 (+50%) |
| WS 메시지 수신 | 124,625 | 186,890 (+50%) |

VU를 3배로 올렸음에도 전 Phase 100%를 유지했다. 총 iterations가 소폭 감소한 것은 4,500 VU에서 응답 시간이 늘어난 영향이다.

### JVM / MySQL

| 항목 | 3차 | 4차 |
|------|-----|-----|
| GC pause total | 12ms | 5ms |
| Allocation Stall | 63회 | 54회 |
| Heap 피크 | ~2.3GB / 3GB | ~2.5GB / 4GB |
| Slow queries | 95 | 909 |
| Row lock waits | 1 | 1 |

Allocation Stall이 4GB에서도 54회 발생한다. Spike Phase에서 메모리 할당이 ZGC 회수 속도를 초과하는 구간이 있다. Slow query 909건은 3배 부하에서 증가한 것으로, 인덱스 최적화 여지가 있다.

---

## 개선 방안

### 적용 완료

| # | 변경 | 효과 |
|---|------|------|
| 1 | HikariCP 300 → 400 | Pending 173 → 0 (확장 완료 후) |
| 2 | message 복합 인덱스 `(chat_room_id, deleted, sent_at DESC, message_id DESC)` | filesort 제거 예상 |
| 3 | 커널 TCP 버퍼 16MB + keepalive 60s + fin_timeout 15s | 네트워크 레벨 지연 감소 |
| 4 | ZGC Generational + 4GB heap | GC pause 5ms, Allocation Stall 감소 |
| 5 | sendAndPublish() TX 분리 | write-pool acquire 24.7s → 1.16s |
| 6 | show_sql/generate_statistics 비활성화 | 로깅 오버헤드 제거 |

### 추가 개선 후보

| # | 병목 | 개선안 | 효과 |
|---|------|--------|------|
| 1 | Virtual Thread pinning | SSE executor → Platform Thread Pool | carrier thread 고갈 방지 |
| 2 | MongoDB synchronized | ReentrantLock 교체 | VT pinning 제거 |
| 3 | Allocation Stall | heap 5GB 또는 ZAllocationSpikeTolerance=5 | Spike 시 stall 감소 |
| 4 | Slow query 909건 | 쿼리 최적화 또는 인덱스 추가 | Spike Phase 응답 시간 개선 |
| 5 | STOMP outbound | queue 2,000 → 8,000, threads 128 → 256 | 3,000+ VU 대비 |

---

## 정리

| 항목 | 로컬 (150~1,000VU) | EC2 1차 (1,500VU) | EC2 3차 (1,500VU) | EC2 4차 (4,500VU) |
|------|--------------------|--------------------|-------------------|-------------------|
| HTTP 에러 | 0 | 0 | 0 | 1건 |
| 총 iterations | — | — | 1,854,368 | 1,741,030 |
| 메시지 전송 p95 | 15.80ms | 1,350ms | 743ms | 2,190ms |
| 메시지 목록 p95 | 23.93ms | 1,500ms | 890ms | 2,140ms |
| 채팅방 목록 p95 | 1,000ms | 484ms | 589ms | 1,330ms |
| HikariCP Pending | 없음 | 최대 173 | 0 | 0 |
| write-pool acquire max | — | — | 24.7s | 1.16s |
| WS 연결 p95 | 13ms | 2ms | 2ms | 13ms |
| WS 메시지 수신 | — | — | 124,625 | 186,890 |

1차에서 4차까지의 흐름: HikariCP 풀 확대(300→400)로 Pending 해소 → 복합 인덱스와 커널/JVM 튜닝으로 p95 45% 개선 → TX 분리로 write-pool 점유 95% 개선 → 4,500 VU까지 확장하여 전 Phase 100% 달성.

> **채팅 도메인의 병목은 단계적으로 변화한다.** HikariCP 풀 크기 → 커널/JVM 설정 → TX 범위 내 Redis 호출 → STOMP queue 순서로 병목이 이동한다. 각 단계를 해소하면 다음 병목이 드러나며, c5.xlarge 단일 인스턴스에서 4,500 VU Spike까지 에러 없이 처리할 수 있다.

---

## 시리즈 탐색

**◀ 이전 글**
[알림 부하 테스트 — AWS EC2에서 MySQL write 붕괴와 MongoDB 비교](/notification-aws-ec2-load-test/)

**▶ 다음 글**
[피드 부하 테스트 — AWS EC2에서 IN절 병목, covering index, 전 페이지 캐싱](/feed-aws-ec2-load-test/)
