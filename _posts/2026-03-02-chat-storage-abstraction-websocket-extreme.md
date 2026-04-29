---
layout: post
title: 채팅 도메인 — Storage 추상화, WHERE IN 풀스캔, WebSocket 1,000VU 극한 테스트
date: 2026-03-02
tags: [Java, Spring, WebSocket, STOMP, MySQL, MongoDB, Redis, 부하테스트, k6, 성능최적화]
permalink: /chat-storage-websocket-extreme-test/
excerpt: "채팅 Storage를 MySQL↔MongoDB 전환 가능하게 추상화했다. 부하 테스트에서 채팅방 목록 API가 30초 타임아웃 — WHERE IN 서브쿼리가 3.1M 행 풀스캔. derived JOIN으로 30.9배 개선. WebSocket 극한 테스트에서 Redis Pub/Sub 풀 포화를 발견하고, 1,000VU 스파이크까지 안정화한 기록."
---

## 개요

[이전 채팅 부하 테스트](/chat-subquery-redis-listener/)에서 상관 서브쿼리와 N+1을 잡고 기본 부하 테스트를 통과했다. 이번에는 Storage 추상화와 WebSocket 극한 테스트로 넘어간다. Port/Adapter 패턴의 상세 설명과 MySQL↔MongoDB 비교 방법론은 [알림 도메인 Storage 추상화](/notification-storage-abstraction-mysql-mongodb/)에서 다뤘다. 이 글에서는 채팅 도메인 고유의 차이점과 WebSocket 극한 테스트에 집중한다.

채팅 도메인은 실시간 전송(Redis Pub/Sub)과 영속 저장(MySQL)이 분리된 구조다.

현재 구현에서 Storage는 MySQL(message, chat_room, user_chat_room)을 사용하고, 실시간 전송은 Redis Pub/Sub(`chat.room.*` 채널)으로 처리한다. WebSocket은 STOMP(`/pub/chat/{roomId}/messages`에서 `/sub/chat/{roomId}/messages`)를 사용하며, 비동기 저장은 `@Async` + `@Retryable`(3회, 실패 시 Redis list 백업)로 구현돼 있다. 페이지네이션은 커서 기반(sentAt + messageId 튜플)이다.

알림 도메인에서 MySQL → MongoDB 전환으로 47배 성능 차이를 확인한 경험이 있어, 채팅도 Storage를 추상화해 동일한 전환을 가능하게 만들었다.

---

## Storage 추상화

### 왜 Pub/Sub은 건드리지 않았나

Redis Pub/Sub은 채팅의 요구사항에 적합하다. fire-and-forget, 1ms 이하 latency, 채널 패턴 매칭을 지원한다. Kafka로 바꾸면 10~50ms latency가 추가되고, 메시지 영구 저장은 이미 DB에서 처리하고 있어 중복이다. RabbitMQ도 동일하다. 실제 병목은 Storage 쪽이었다.

```
WebSocket → Redis Pub/Sub (1ms, 병목 아님)
         → @Async → MySQL save (여기가 병목 후보)
         → MySQL 조회 (커서 페이지네이션 + JOIN, 여기가 병목 후보)
```

### 구현

`ChatMessageStoragePort` 인터페이스를 만들고 MySQL/MongoDB 어댑터를 구현했다. `app.chat.storage: mysql | mongodb`로 전환 가능하다.

`ChatMessageStoragePort.java`는 6개 메서드를 가진 저장소 추상화 인터페이스이고, `ChatMessageItemDto.java`는 Storage에 무관한 중간 DTO(record)다. `MysqlChatMessageStorageAdapter.java`는 MySQL 어댑터(`matchIfMissing=true` 기본값)이며, `MongoChatMessageStorageAdapter.java`는 MongoDB 어댑터(Segment 할당, `$or` 커서, `$group` aggregation)다. `MessageDocument.java`는 MongoDB document 엔티티(sender 비정규화, 복합인덱스)이고, `MongoChatConfig.java`는 `@EnableMongoAuditing` 설정이다.

서비스 계층(`MessageCommandService`, `MessageQueryService`, `ChatRoomQueryService`)은 `MessageRepository` 직접 의존에서 `ChatMessageStoragePort`로 전환했다. 기존 테스트 3개도 Mock 대상을 변경해 전체 50개 테스트 통과를 확인했다.

---

## MySQL 부하 테스트 — 채팅방 목록 30초 타임아웃

### 결과

메시지 전송(150VU)은 p95 15.80ms, 성공률 100%(PASS). 최신 목록(150VU)은 p95 23.93ms(PASS). 커서 페이지네이션(150VU)은 p95 16.16ms, 성공률 99.97%(PASS). **채팅방 목록(100VU)은 p95 6,870ms로 threshold 초과(FAIL)**. WebSocket(100VU)은 p95 13ms(PASS). 스파이크(300VU)는 p95 274ms(PASS).

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

서브쿼리는 인덱스를 타지만, **외부 쿼리에서 `message_id IN (서브쿼리결과)`를 할 때 PK 인덱스 대신 풀 테이블 스캔(3.1M rows)**을 하고 있었다. MySQL 옵티마이저가 IN 서브쿼리를 materialized temp table로 변환하고, 통계가 왜곡됐을 때 PK lookup이 아니라 풀스캔을 선택해버리는 케이스다 (8.0의 semi-join transformation이 항상 PK 경로로 풀어지지는 않는다).

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

수정 후 p95는 30.94초에서 1.00초로 **30.9배** 개선, avg는 29.43초에서 411ms로 71.4배 개선됐다. 성공률은 21.62%에서 99.97%로 올랐고, 처리량은 296건에서 11,909건으로 40배 증가했다.

---

## MongoDB 비교 테스트

동일 시나리오를 MongoDB 어댑터로 전환해 테스트했다.

메시지 전송은 MySQL p95 15.80ms, MongoDB p95 14.47ms로 거의 동일하다. 최신 목록은 MySQL 23.93ms, MongoDB 17.79ms. 커서 페이지네이션은 MySQL 16.16ms, MongoDB 15.83ms. 채팅방 목록은 MySQL 1.00초(수정 후), MongoDB 10.5초로 MySQL이 압도적으로 빨랐다. 스파이크는 MySQL 274ms, MongoDB 266ms로 비슷했다.

전송/조회/커서는 MongoDB가 소폭 빠르지만, 채팅방 목록은 MongoDB의 `$group` aggregation + 각 방의 lastMessage 조회가 MySQL derived JOIN보다 느렸다. Storage 추상화는 전환 가능성을 확보한 것이고, 현재 데이터 규모에서는 MySQL이 적합하다.

---

## WebSocket 극한 테스트 — 1,000VU

### 테스트 설계

기존 테스트는 100VU, 3초 세션, 5개 메시지로 가벼웠다. 극한 테스트를 설계했다.

**Phase 1 (Warmup):** 10 VU, 기본 연결 확인. **Phase 2 (Mass Connect):** 600 VU, 점진적 동시 접속. **Phase 3 (Message Storm):** 400 VU, 세션당 50개 메시지 폭풍. **Phase 4 (Sustained):** 300 VU, 20초 유지 + 2초마다 전송. **Phase 5 (Broadcast):** 300 VU, 7개 방 전체 구독. **Phase 6 (Spike):** 1,000 VU, 동시 접속 폭발. **Phase 7 (Mixed):** 700 VU, HTTP + WS 혼합. **Phase 8 (Soak):** 80 VU, 3분 장시간.

### 1차 결과 — Redis Pub/Sub 풀 포화

```
pool size = 32, active threads = 32, queued tasks = 1000
→ RejectedExecutionException
```

Redis Pub/Sub 메시지 디스패치용 ThreadPool이 포화됐다. `corePoolSize=8, maxPoolSize=32, queueCapacity=1000`이 전부 차서 메시지를 드랍했다.

STOMP inbound/outbound는 64~128 스레드로 여유가 있었지만, Redis → STOMP 브릿지 구간의 풀이 병목이었다.

### 스레드풀 설정 조정

**1차 조정**

```
RedisChatPubSubConfig:
Before: corePoolSize=8,  maxPoolSize=32,  queueCapacity=1000
After:  corePoolSize=64, maxPoolSize=200, queueCapacity=5000
```

TaskRejectedException 해소, 수신 처리량 6,322/s → 17,777/s (2.8배 증가).

**2차 조정 — CallerRunsPolicy 적용**

```
RedisChatPubSubConfig:
After:  corePoolSize=128, maxPoolSize=400, queueCapacity=10000
거부 정책: AbortPolicy → CallerRunsPolicy

customAsyncExecutor (DB 저장 등):
After:  corePoolSize=64, maxPoolSize=200
거부 정책: AbortPolicy → CallerRunsPolicy
```

`CallerRunsPolicy`가 주요 변경 사항이다. 큐가 꽉 차면 예외를 던지는 대신 호출 스레드가 직접 태스크를 실행한다. 메시지 드랍 0건, 백프레셔로 전송 속도가 조절된다.

> [이전 채팅 부하 테스트](/chat-subquery-redis-listener/)에서는 DB 저장 풀의 `CallerRunsPolicy`를 제거했다. 당시 풀 크기가 8/32로 작아서 WebSocket 스레드가 DB I/O를 직접 실행하며 블로킹됐기 때문이다. 이번에는 풀을 64/200으로 충분히 키운 뒤 재적용했다. 풀이 크면 CallerRunsPolicy가 발동하는 빈도 자체가 낮고, 발동하더라도 순간적이라 극한 부하에서 메시지 드랍을 방지하는 이점이 더 크다.

### 최종 결과

Mass Connect(600VU)는 성공률 100%, p95 72ms(PASS). Message Storm(400VU, 50msg/세션)은 성공률 99.5%(PASS). Sustained(300VU, 20초 세션)은 성공률 100%(PASS). Broadcast(300VU, 7방 전체)는 성공률 99.5%(PASS). Spike(1,000VU)는 성공률 84.9%, p95 161ms(PASS). Mixed WS+HTTP(700VU)는 성공률 14~36%, p95 1.16초(FAIL).

튜닝 전후를 비교하면, 수신 처리량은 6,322/s에서 15,849/s로 **2.5배** 증가, TaskRejectedException은 다수에서 0건으로 해소, 1,000VU 스파이크 성공률 84.9%를 달성했다.

Phase 7(Mixed 700VU) FAIL은 Phase 1~6 누적 부하(10분+) 후 WS+HTTP 700VU 혼합이 로컬 환경 자원 한계에 도달한 것이다. 1,000VU 스파이크 단독은 84.9% 성공이므로 WebSocket 자체는 건강하다.

---

## SimpleBroker vs 외부 브로커

테스트 결과를 기반으로 판단하면, 아직 외부 브로커(RabbitMQ)는 불필요하다.

600VU 동시접속에서 성공률 100%로 단일 서버로 충분하며, 400VU 메시지 폭풍에서 99.5%로 브로커 병목이 아님을 확인했다. 300VU 장시간 세션은 100%로 안정적이고, 1,000VU 스파이크는 84.9%로 단독이면 처리 가능한 수준이다.

외부 브로커가 필요한 시점은 세 가지다. 서버를 2대 이상 띄울 때(SimpleBroker는 JVM 메모리 내 구독 관리라 서버 간 공유 불가), 동시 접속 2,000+ 이상이 필요할 때, 메시지 유실이 절대 안 되는 요구사항이 있을 때.

현재 단계에서는 **SimpleBroker + Redis Pub/Sub 조합이 최적**이다. 외부 브로커는 인프라 복잡도 대비 현재 규모에서는 오버엔지니어링이다.

---

## 정리하며

채팅 도메인의 실질적 병목은 두 곳이었다.

첫째, **WHERE IN 서브쿼리의 MySQL 풀스캔.** `message_id IN (SELECT MAX(...))`에서 MySQL이 PK 인덱스를 타지 않고 3.1M 행을 풀스캔했다. derived table JOIN으로 바꾸니 30.9배 빨라졌다. 단독 실행에서는 ~1초 차이지만, 동시 부하에서 30초 타임아웃까지 벌어진다.

둘째, **Redis Pub/Sub 디스패치 풀 포화.** 메시지 수신 후 STOMP 구독자에게 전달하는 ThreadPool이 32스레드로 고정되어 있었다. 극한 부하에서 큐가 꽉 차면 `RejectedExecutionException`으로 메시지를 드랍했다. `CallerRunsPolicy`로 전환하면 드랍 대신 호출 스레드가 직접 처리해 백프레셔가 걸린다.

Storage 추상화는 MySQL↔MongoDB 전환 가능성을 확보한 것이지, 현재 데이터 규모에서 MongoDB가 무조건 빠른 것은 아니다. 채팅방 목록처럼 aggregation이 필요한 쿼리는 MySQL의 derived JOIN이 더 효율적이었다.

---

## 수정 내역 요약

수정 1: `ChatMessageStoragePort` 인터페이스 + MySQL/MongoDB 어댑터로 Storage 전환 가능. 수정 2: `findLastMessagesByChatRoomIds`에서 WHERE IN을 derived JOIN으로 변경하여 채팅방 목록 p95 30.9배 개선. 수정 3: Redis Pub/Sub 풀을 8/32/1000에서 128/400/10000으로 확대하여 수신 처리량 2.5배 증가. 수정 4: 거부 정책을 AbortPolicy에서 CallerRunsPolicy로 변경하여 메시지 드랍 0건. 수정 5: Async 풀을 32/100에서 64/200으로 확대하고 CallerRunsPolicy를 적용하여 비동기 저장 안정화.

---

## 시리즈 탐색

**◀ 이전 글**
[teammates_clubs 쿼리 62배 개선 기록](/search-load-test-not-exists-optimization/)

**▶ 다음 글**
[멀티 도메인 리팩토링 — ErrorCode 분리부터 30파일 코드 품질 정리까지](/multi-domain-refactoring-errorcode-code-quality/)
