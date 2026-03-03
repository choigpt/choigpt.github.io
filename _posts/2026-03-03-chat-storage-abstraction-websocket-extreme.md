---
title: 채팅 도메인 — Storage 추상화, WHERE IN 풀스캔, WebSocket 1,000VU 극한 테스트
date: 2026-03-03
tags: [Java, Spring, WebSocket, STOMP, MySQL, MongoDB, Redis, 부하테스트, k6, 성능최적화]
permalink: /chat-storage-websocket-extreme-test/
excerpt: "채팅 Storage를 MySQL↔MongoDB 전환 가능하게 추상화했다. 부하 테스트에서 채팅방 목록 API가 30초 타임아웃 — WHERE IN 서브쿼리가 3.1M 행 풀스캔. derived JOIN으로 30.9배 개선. WebSocket 극한 테스트에서 Redis Pub/Sub 풀 포화를 발견하고, 1,000VU 스파이크까지 안정화한 기록."
---

## 개요

[이전 채팅 부하 테스트](/chat-subquery-redis-listener/)에서 상관 서브쿼리, Redis 리스너 미등록, N+1을 잡고 기본 부하 테스트를 통과했다. 이번에는 Storage 추상화와 WebSocket 극한 테스트로 넘어간다.

채팅 도메인은 실시간 전송(Redis Pub/Sub)과 영속 저장(MySQL)이 분리된 구조다.

| 항목 | 현재 구현 |
|------|----------|
| Storage | MySQL (message, chat_room, user_chat_room) |
| 실시간 전송 | Redis Pub/Sub (`chat.room.*` 채널) |
| WebSocket | STOMP (`/pub/chat/{roomId}/messages` → `/sub/chat/{roomId}/messages`) |
| 비동기 저장 | `@Async` + `@Retryable` (3회, 실패시 Redis list 백업) |
| 페이지네이션 | 커서 기반 (sentAt + messageId 튜플) |

알림 도메인에서 MySQL → MongoDB 전환으로 47배 성능 차이를 확인한 경험이 있어, 채팅도 Storage를 추상화해 동일한 전환을 가능하게 만들었다.

---

## Storage 추상화

### 왜 Pub/Sub은 건드리지 않았나

Redis Pub/Sub이 채팅에 최적이다. fire-and-forget, 1ms 이하 latency, 채널 패턴 매칭. Kafka로 바꾸면 10~50ms latency가 생기고, 메시지 영구 저장은 이미 DB에서 하고 있으니 중복이다. RabbitMQ도 마찬가지. 진짜 병목은 Storage 쪽이었다.

```
WebSocket → Redis Pub/Sub (1ms, 병목 아님)
         → @Async → MySQL save (여기가 병목 후보)
         → MySQL 조회 (커서 페이지네이션 + JOIN, 여기가 병목 후보)
```

### 구현

`ChatMessageStoragePort` 인터페이스를 만들고 MySQL/MongoDB 어댑터를 구현했다. `app.chat.storage: mysql | mongodb`로 전환 가능하다.

| 파일 | 목적 |
|------|------|
| `ChatMessageStoragePort.java` | 저장소 추상화 인터페이스 (6개 메서드) |
| `ChatMessageItemDto.java` | Storage 무관 중간 DTO (record) |
| `MysqlChatMessageStorageAdapter.java` | MySQL 어댑터 (`matchIfMissing=true` 기본값) |
| `MongoChatMessageStorageAdapter.java` | MongoDB 어댑터 (Segment 할당, `$or` 커서, `$group` aggregation) |
| `MessageDocument.java` | MongoDB document 엔티티 (sender 비정규화, 복합인덱스) |
| `MongoChatConfig.java` | `@EnableMongoAuditing` |

서비스 계층(`MessageCommandService`, `MessageQueryService`, `ChatRoomQueryService`)은 `MessageRepository` 직접 의존에서 `ChatMessageStoragePort`로 전환했다. 기존 테스트 3개도 Mock 대상을 변경해 전체 50개 테스트 통과를 확인했다.

---

## MySQL 부하 테스트 — 채팅방 목록 30초 타임아웃

### 결과

| Phase | p95 | 성공률 | 판정 |
|-------|-----|--------|------|
| 메시지 전송 (150VU) | 15.80ms | 100% | PASS |
| 최신 목록 (150VU) | 23.93ms | 100% | PASS |
| 커서 페이지네이션 (150VU) | 16.16ms | 99.97% | PASS |
| **채팅방 목록 (100VU)** | **6,870ms** | 100% | **FAIL** |
| WebSocket (100VU) | 13ms | 99.90% | PASS |
| 스파이크 (300VU) | 274ms | 100% | PASS |

전송/조회/커서/WebSocket/스파이크는 전부 양호. **채팅방 목록 API만 p95 6.87초**로 threshold 초과.

### 원인: WHERE IN 서브쿼리 풀스캔

`findLastMessagesByChatRoomIds()`가 320만 건 message 테이블에서 서브쿼리(GROUP BY + MAX)로 각 채팅방의 마지막 메시지를 조회하고 있었다.

```sql
EXPLAIN SELECT m.* FROM message m JOIN user u ON m.user_id = u.user_id
WHERE m.message_id IN (
    SELECT MAX(m2.message_id) FROM message m2
    WHERE m2.chat_room_id IN (?) AND m2.deleted = false
    GROUP BY m2.chat_room_id
);
```

```
id  select_type  table  type     rows
1   PRIMARY      m      ALL      3,157,362   ← 풀 테이블 스캔
2   SUBQUERY     m2     range    333,570     ← 인덱스 탐
```

서브쿼리는 인덱스를 타지만, **외부 쿼리에서 `message_id IN (서브쿼리결과)`를 할 때 PK 인덱스 대신 풀 테이블 스캔(3.1M rows)**을 하고 있었다. MySQL이 서브쿼리 결과를 materialized temp table로 만든 뒤 PK lookup을 못 하는 알려진 이슈다.

### 수정: derived table JOIN

```sql
-- 수정 전: WHERE IN (서브쿼리) — type=ALL 풀스캔
WHERE m.message_id IN (SELECT MAX(m2.message_id) ...)

-- 수정 후: derived table JOIN — type=eq_ref PK lookup
JOIN (
    SELECT MAX(m2.message_id) AS max_id FROM message m2
    WHERE m2.chat_room_id IN (?) AND m2.deleted = false
    GROUP BY m2.chat_room_id
) sub ON m.message_id = sub.max_id
```

```
id  select_type  table       type     rows
1   PRIMARY      <derived2>  ALL      333,570  ← 예상 행수 (실제 결과 10건)
1   PRIMARY      m           eq_ref   1        ← PK lookup
```

IN 서브쿼리: ~1초. JOIN derived: 즉시 완료. 이 차이가 동시 부하에서 30초까지 벌어진 것이다.

### 수정 결과

| 지표 | 수정 전 | 수정 후 | 개선 |
|------|---------|---------|------|
| p95 | 30.94s | 1.00s | **30.9배** |
| avg | 29.43s | 411ms | 71.4배 |
| 성공률 | 21.62% | 99.97% | — |
| 처리량 | 296건 | 11,909건 | 40배 |

---

## MongoDB 비교 테스트

동일 시나리오를 MongoDB 어댑터로 전환해 테스트했다.

| Phase | MySQL p95 | MongoDB p95 |
|-------|-----------|-------------|
| 메시지 전송 | 15.80ms | 14.47ms |
| 최신 목록 | 23.93ms | 17.79ms |
| 커서 페이지네이션 | 16.16ms | 15.83ms |
| 채팅방 목록 | 1.00s (수정 후) | 10.5s |
| 스파이크 | 274ms | 266ms |

전송/조회/커서는 MongoDB가 소폭 빠르지만, 채팅방 목록은 MongoDB의 `$group` aggregation + 각 방의 lastMessage 조회가 MySQL derived JOIN보다 느렸다. Storage 추상화는 전환 가능성을 확보한 것이고, 현재 데이터 규모에서는 MySQL이 적합하다.

---

## WebSocket 극한 테스트 — 1,000VU

### 테스트 설계

기존 테스트는 100VU, 3초 세션, 5개 메시지로 가벼웠다. 극한 테스트를 설계했다.

| Phase | 내용 | VU | 특징 |
|-------|------|-----|------|
| 1 | Warmup | 10 | 기본 연결 확인 |
| 2 | Mass Connect | 600 | 점진적 동시 접속 |
| 3 | Message Storm | 400 | 세션당 50개 메시지 폭풍 |
| 4 | Sustained | 300 | 20초 유지 + 2초마다 전송 |
| 5 | Broadcast | 300 | 7개 방 전체 구독 |
| 6 | Spike | 1,000 | 동시 접속 폭발 |
| 7 | Mixed | 700 | HTTP + WS 혼합 |
| 8 | Soak | 80 | 3분 장시간 |

### 1차 결과 — Redis Pub/Sub 풀 포화

```
pool size = 32, active threads = 32, queued tasks = 1000
→ RejectedExecutionException
```

Redis Pub/Sub 메시지 디스패치용 ThreadPool이 포화됐다. `corePoolSize=8, maxPoolSize=32, queueCapacity=1000`이 전부 차서 메시지를 드랍했다.

STOMP inbound/outbound는 64~128 스레드로 여유가 있었지만, Redis → STOMP 브릿지 구간의 풀이 병목이었다.

### 풀 튜닝 — 2라운드

**1차 튜닝**

```
RedisChatPubSubConfig:
Before: corePoolSize=8,  maxPoolSize=32,  queueCapacity=1000
After:  corePoolSize=64, maxPoolSize=200, queueCapacity=5000
```

TaskRejectedException 해소, 수신 처리량 6,322/s → 17,777/s (2.8배 증가).

**2차 튜닝 — CallerRunsPolicy 적용**

```
RedisChatPubSubConfig:
After:  corePoolSize=128, maxPoolSize=400, queueCapacity=10000
거부 정책: AbortPolicy → CallerRunsPolicy

customAsyncExecutor (DB 저장 등):
After:  corePoolSize=64, maxPoolSize=200
거부 정책: AbortPolicy → CallerRunsPolicy
```

`CallerRunsPolicy`가 핵심이다. 큐가 꽉 차면 예외를 던지는 대신 호출 스레드가 직접 태스크를 실행한다. 메시지 드랍 0건, 자연스러운 백프레셔로 전송 속도가 조절된다.

> [이전 채팅 부하 테스트](/chat-subquery-redis-listener/)에서는 DB 저장 풀의 `CallerRunsPolicy`를 제거했다. 당시 풀 크기가 8/32로 작아서 WebSocket 스레드가 DB I/O를 직접 실행하며 블로킹됐기 때문이다. 이번에는 풀을 64/200으로 충분히 키운 뒤 재적용했다. 풀이 크면 CallerRunsPolicy가 발동하는 빈도 자체가 낮고, 발동하더라도 순간적이라 극한 부하에서 메시지 드랍을 방지하는 이점이 더 크다.

### 최종 결과

| Phase | VU | 성공률 | p95 | 판정 |
|-------|-----|--------|-----|------|
| Mass Connect | 600 | 100% | 72ms | PASS |
| Message Storm (50msg/세션) | 400 | 99.5% | — | PASS |
| Sustained (20초 세션) | 300 | 100% | — | PASS |
| Broadcast (7방 전체) | 300 | 99.5% | — | PASS |
| Spike | 1,000 | 84.9% | 161ms | PASS |
| Mixed WS+HTTP | 700 | 14~36% | 1.16s | FAIL |

| 지표 | 튜닝 전 | 튜닝 후 | 변화 |
|------|---------|---------|------|
| 수신 처리량 | 6,322/s | 15,849/s | **2.5배** |
| TaskRejectedException | 다수 | 0건 | 해소 |
| 1,000VU 스파이크 성공률 | — | 84.9% | — |

Phase 7(Mixed 700VU) FAIL은 Phase 1~6 누적 부하(10분+) 후 WS+HTTP 700VU 혼합이 로컬 환경 자원 한계에 도달한 것이다. 1,000VU 스파이크 단독은 84.9% 성공이므로 WebSocket 자체는 건강하다.

---

## SimpleBroker vs 외부 브로커

테스트 결과를 기반으로 판단하면, 아직 외부 브로커(RabbitMQ)는 불필요하다.

| 구간 | 결과 | 의미 |
|------|------|------|
| 600VU 동시접속 | 100% | 단일 서버로 충분 |
| 400VU 메시지 폭풍 | 99.5% | 브로커 병목 아님 |
| 300VU 장시간 세션 | 100% | 안정적 |
| 1,000VU 스파이크 | 84.9% | 단독이면 처리 가능 |

외부 브로커가 필요한 시점은 세 가지다. 서버를 2대 이상 띄울 때(SimpleBroker는 JVM 메모리 내 구독 관리라 서버 간 공유 불가), 동시 접속 2,000+ 이상이 필요할 때, 메시지 유실이 절대 안 되는 요구사항이 있을 때.

현재 단계에서는 **SimpleBroker + Redis Pub/Sub 조합이 최적**이다. 외부 브로커는 인프라 복잡도 대비 현재 규모에서는 오버엔지니어링이다.

---

## 정리하며

채팅 도메인의 실질적 병목은 두 곳이었다.

첫째, **WHERE IN 서브쿼리의 MySQL 풀스캔.** `message_id IN (SELECT MAX(...))`에서 MySQL이 PK 인덱스를 타지 않고 3.1M 행을 풀스캔했다. derived table JOIN으로 바꾸니 30.9배 빨라졌다. 단독 실행에서는 ~1초 차이지만, 동시 부하에서 30초 타임아웃까지 벌어진다.

둘째, **Redis Pub/Sub 디스패치 풀 포화.** 메시지 수신 후 STOMP 구독자에게 전달하는 ThreadPool이 32스레드로 고정되어 있었다. 극한 부하에서 큐가 꽉 차면 `RejectedExecutionException`으로 메시지를 드랍했다. `CallerRunsPolicy`로 전환하면 드랍 대신 호출 스레드가 직접 처리해 자연스러운 백프레셔가 걸린다.

Storage 추상화는 MySQL↔MongoDB 전환 가능성을 확보한 것이지, 현재 데이터 규모에서 MongoDB가 무조건 빠른 것은 아니다. 채팅방 목록처럼 aggregation이 필요한 쿼리는 MySQL의 derived JOIN이 더 효율적이었다.

---

## 수정 내역 요약

| # | 변경 내용 | 효과 |
|---|----------|------|
| 1 | `ChatMessageStoragePort` 인터페이스 + MySQL/MongoDB 어댑터 | Storage 전환 가능 |
| 2 | `findLastMessagesByChatRoomIds` WHERE IN → derived JOIN | 채팅방 목록 p95 30.9배 개선 |
| 3 | Redis Pub/Sub 풀 8/32/1000 → 128/400/10000 | 수신 처리량 2.5배 |
| 4 | 거부 정책 AbortPolicy → CallerRunsPolicy | 메시지 드랍 0건 |
| 5 | Async 풀 32/100 → 64/200 + CallerRunsPolicy | 비동기 저장 안정화 |

---

## 시리즈 탐색

**◀ 이전 글**
[검색 도메인 부하 테스트 — NOT IN 한 줄이 만든 62배 차이](/search-load-test-not-exists-optimization/)

**▶ 다음 글**
[멀티 도메인 리팩토링 — ErrorCode 분리부터 29파일 코드 품질 정리까지](/multi-domain-refactoring-errorcode-code-quality/)
