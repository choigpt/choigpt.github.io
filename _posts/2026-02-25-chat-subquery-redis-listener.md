---
title: 채팅 도메인 — 상관 서브쿼리·Redis 리스너 미등록·N+1이 만든 성능 저하
date: 2026-02-25
tags: [MySQL, Redis, JPA, WebSocket, STOMP, 성능최적화, 트러블슈팅, Java, 채팅, k6]
permalink: /chat-subquery-redis-listener/
excerpt: "k6 첫 측정에서 메시지 전송 성공률 0.9%, WebSocket 왕복 시간 측정 불가. 원인은 테스트 데이터 불일치, 상관 서브쿼리, 그리고 Redis 리스너 미등록이었다."
---

## 개요

k6로 채팅 API에 부하를 걸었더니 처음부터 이상했다. 메시지 전송 성공률이 0.9%였고, WebSocket 메시지 왕복 시간은 아예 측정조차 되지 않았다. 에러 로그를 들여다보니 버그가 한 곳이 아니었다. 테스트 데이터 설계 결함, 쿼리 구조 문제, 아무도 구독하지 않는 Redis 채널, N+1 쿼리가 한꺼번에 나타났다.

---

## 시스템 구조

### 채팅 메시지 흐름

```
클라이언트 (REST / WebSocket STOMP)
    ↓
ChatWebSocketController (@MessageMapping)
    ↓
MessageCommandService.publishImmediately() → Redis Pub/Sub 발행
    ↓
AsyncMessageService.saveMessageAsync() (@Async) → DB INSERT
    ↓
ChatSubscriber.onMessage() → SimpMessagingTemplate.convertAndSend()
    ↓
WebSocket 구독자 브로드캐스트
```

REST 조회는 `MessageQueryService`가 `MessageRepository.findLatest()` / `findOlderThan()`으로 커서 기반 페이징을 처리한다.

---

## 문제 상세

### P2 메시지 전송 성공률 0.9% — 테스트 데이터 불일치

```
k6 VU: 유저 1~1000을 순환
VU 1번 → userId=1로 로그인 → roomId 랜덤 선택 (1~1000)
→ POST /api/v1/clubs/{clubId}/chat/{roomId}/messages
→ 403 Forbidden
```

유저 1은 방 `[1, 519, 769, 987]` 4개에만 참여 중인데, k6가 랜덤 방(1~1000)에 접근하니 대부분 403이 났다. 테스트 스크립트가 유저별 참여 방 목록을 모르고 무작위 roomId를 사용한 구조적 결함이었다.

초기 데이터셋도 문제가 있었다.

| 항목 | 기존 | 개선 |
|------|------|------|
| 채팅방 타입 | CLUB만 1,000개 | CLUB 800 + SCHEDULE 200 |
| 유저-방 매핑 | 유저당 1개 방 | 유저당 4~8개 방 |
| 메시지 분포 | 균일 (방당 ~2,200건) | 핫룸 100개 × 5,000건 + 일반 700개 × 500건 |
| text 컬럼 | VARCHAR(255) | VARCHAR(2000) (엔티티와 일치) |
| 커서 페이징 인덱스 | 없음 | `(chat_room_id, sent_at DESC, message_id DESC)` 추가 |

---

### 채팅방 목록 조회 슬로우 쿼리 — 상관 서브쿼리

채팅방 목록 API(GET /clubs/{clubId}/chat)에서 마지막 메시지를 가져오는 쿼리가 병목이었다.

```java
// MessageRepository.java (수정 전)
@Query("""
SELECT m FROM Message m
WHERE m.chatRoom.chatRoomId IN :chatRoomIds
  AND m.deleted = false
  AND m.sentAt = (
    SELECT MAX(m2.sentAt) FROM Message m2
    WHERE m2.chatRoom.chatRoomId = m.chatRoom.chatRoomId
      AND m2.deleted = false
  )
""")
List<Message> findLastMessagesByChatRoomIds(@Param("chatRoomIds") List<Long> chatRoomIds);
```

`EXPLAIN` 결과가 핵심이었다.

```
기존: DEPENDENT SUBQUERY → 매 row마다 438건씩 스캔 (20,236 × 438 = 약 887만 건)
최적화: Using index for group-by → 인덱스만으로 GROUP BY 처리, derived 21건
```

채팅방이 많아질수록 지수적으로 느려지는 구조였다.

**수정 후:**

```java
@Query(value = """
    SELECT m.* FROM message m
    INNER JOIN (
        SELECT chat_room_id, MAX(sent_at) AS max_sent_at
        FROM message
        WHERE chat_room_id IN :chatRoomIds AND deleted = 0
        GROUP BY chat_room_id
    ) latest ON m.chat_room_id = latest.chat_room_id AND m.sent_at = latest.max_sent_at
    WHERE m.deleted = 0
""", nativeQuery = true)
List<Message> findLastMessagesByChatRoomIds(@Param("chatRoomIds") List<Long> chatRoomIds);
```

---

### P4 WebSocket 메시지 왕복 측정 불가 — Redis 리스너 미등록

k6의 STOMP 테스트에서 메시지를 보내도 WebSocket 클라이언트가 응답을 받지 못했다. 원인은 단순했다.

```java
// ChatSubscriber.java
@Component
public class ChatSubscriber implements MessageListener {
    @Override
    public void onMessage(Message message, byte[] pattern) {
        // Redis 메시지를 받아서 WebSocket으로 브로드캐스트
        simpMessagingTemplate.convertAndSend("/topic/chat/" + roomId, chatMessage);
    }
}
```

`ChatSubscriber`는 `MessageListener`를 구현하고 있지만, `RedisMessageListenerContainer`에 이 리스너를 등록하는 코드가 프로젝트 어디에도 없었다. 테스트 코드에서 Mock으로만 존재했다.

```
메시지 전송 → Redis Publish → Subscribe하는 쪽이 없음 → 메시지 소실
WebSocket 클라이언트 → 3초 타임아웃 → 왕복 측정 불가
```

**수정 후:**

```java
// RedisChatPubSubConfig.java (신규)
@Bean
public RedisMessageListenerContainer redisMessageListenerContainer(
    RedisConnectionFactory connectionFactory,
    ChatSubscriber chatSubscriber
) {
    RedisMessageListenerContainer container = new RedisMessageListenerContainer();
    container.setConnectionFactory(connectionFactory);
    container.addMessageListener(chatSubscriber, new PatternTopic("chat.room.*"));
    container.setTaskExecutor(chatPubSubExecutor());  // 전용 스레드풀 (core=8, max=32)
    return container;
}
```

---

### `findLatest` N+1 쿼리

메시지 50건을 조회하면 각 메시지마다 `chatRoom`과 `user` 연관이 LAZY 로딩됐다.

```
findLatest() → 1 쿼리
→ m.chatRoom 접근 × 50 → 50 쿼리
→ m.user 접근 × 50 → 50 쿼리
합계: 101 쿼리
```

**수정 후:**

```java
@Query("""
   select m from Message m
   join fetch m.chatRoom
   join fetch m.user
   where m.chatRoom.chatRoomId = :roomId
     and m.deleted = false
   order by m.sentAt desc, m.messageId desc
""")
List<Message> findLatest(@Param("roomId") Long roomId, Pageable pageable);
```

`findOlderThan`도 동일하게 수정했다.

---

### CallerRunsPolicy — 비동기 큐 포화 시 WebSocket 스레드 블로킹

```java
// AsyncConfig.java (수정 전)
executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
```

비동기 DB 저장 큐가 가득 차면 `CallerRunsPolicy`가 작동해 WebSocket 인바운드 스레드가 직접 DB 저장을 실행했다. STOMP 메시지를 처리해야 하는 스레드가 DB I/O를 하는 동안 다른 WebSocket 요청이 전부 블로킹됐다.

**수정 후:**

```java
executor.setRejectedExecutionHandler((r, executor) -> {
    log.warn("[AsyncExecutor] task rejected, queue full. poolSize={}, queueSize={}",
            executor.getPoolSize(), executor.getQueue().size());
    throw new RejectedExecutionException("Async queue full");
});
```

거부된 태스크는 `@Recover`가 Redis 폴백으로 처리한다.

---

### 로그 I/O 병목

매 메시지마다 `INFO` 레벨 로그가 4~5줄씩 찍혔다.

```java
log.info("[WebSocket.Receive] chatRoomId={}, userId={}", ...);
log.info("[WebSocket.Publish] chatRoomId={}, userId={}", ...);
log.info("[Async.SaveMessage] completed: chatRoomId={}, userId={}", ...);
log.info("[Chat.Subscriber] message broadcast: roomId={}, text={}", ...);
```

200VU 기준 초당 수백 건의 메시지가 처리될 때 로그 I/O가 병목이 됐다.

**수정:** 4개 모두 `log.debug`로 변경.

---

## 부하 테스트 결과 (3차 최종)

### 3M+ 메시지, 800VU → 400VU로 스케일업 후 최종 수치

| Phase | 항목 | 1차 (before) | 3차 (after) | 개선율 |
|-------|------|-------------|------------|--------|
| P1 Read (400VU) | 메시지 조회 p95 | 319ms | 108ms | -66% |
| | 채팅방 목록 p95 | 48ms | 33ms | -31% |
| | 성공률 | 84.2% | 100% | |
| P2 Write (150VU) | 메시지 전송 p95 | 44ms | 35ms | -20% |
| | 성공률 | 100% | 100% | |
| P3 Mixed (200VU) | 메시지 조회 p95 | 50ms | 91ms | (VU 2배) |
| | 성공률 | 85.1% | 100% | |
| P4 WebSocket (150VU) | 메시지 왕복 p95 | 측정불가 | 26ms | **해결** |
| | 연결 성공률 | 99.6% | 99.3% | |
| 전체 | 5xx 에러 | 9,668건 | 0건 | |
| | 슬로우 쿼리 | 827건 | 46건 | -83% |

### Phase별 결과 요약

**P1 (조회 단독 400VU)**: 메시지 조회 p95 108ms, 채팅방 목록 p95 33ms. 모두 임계값(300ms) 이내.

**P2 (전송 단독 150VU)**: 메시지 전송 p95 35ms. 성공률 100%.

**P3 (혼합 200VU)**: 조회/전송/삭제 모두 50~91ms. 성공률 100%. 조회 p95가 1차 대비 91ms로 올라간 건 VU가 두 배(100 → 200)로 늘었기 때문이며, 임계값(300ms) 대비로는 충분히 여유 있는 수치다.

**P4 (WebSocket 150VU)**: 연결 p95 15ms, **메시지 왕복 p95 26ms**. Redis 리스너 등록 후 처음으로 측정 가능해졌다.

**P5 (스파이크 500VU)**: p95 929ms. DB 커넥션 풀(HikariCP 400) 포화 구간으로, 이 수준의 스파이크는 인프라 확장이 필요한 로컬 환경 한계.

---

## 남은 병목점

### P5 스파이크 (800VU) — DB 커넥션 풀 포화

단일 쿼리 실행 시간은 1.2ms지만 400VU가 동시에 접근하면 1,411ms로 늘어난다. HikariCP 400 커넥션이 전부 사용 중일 때 나머지 요청이 큐에 대기하는 시간이 쿼리 시간에 더해지는 것이다.

실서버 환경에서는 DB 복제(Read Replica)와 커넥션 풀 수직 확장으로 대응 가능하다.

### WebSocket 메시지 왕복 1.5% 실패

STOMP SUBSCRIBE 후 MESSAGE 프레임이 3초 타임아웃 내에 도달하지 못하는 케이스가 1.5% 수준으로 남아 있다. Redis Pub/Sub → STOMP 브로드캐스트 사이의 지연으로, 300VU 이상의 고부하 구간에서 발생한다.

---

## 수정 내역 요약

| # | 파일 | 변경 내용 |
|---|------|----------|
| 1 | `RedisChatPubSubConfig.java` (신규) | `RedisMessageListenerContainer` 등록, `chat.room.*` 패턴 구독, 전용 스레드풀 |
| 2 | `MessageRepository.java` | `findLastMessagesByChatRoomIds`: 상관 서브쿼리 → GROUP BY + JOIN |
| 3 | `MessageRepository.java` | `findLatest`, `findOlderThan`: `JOIN FETCH m.chatRoom, m.user` 추가 |
| 4 | `ChatSubscriber.java` | `log.info` → `log.debug` |
| 5 | `AsyncMessageService.java` | 지수 백오프로 변경 (100ms / 300ms / 900ms) |
| 6 | `WebSocketConfig.java` | STOMP 아웃바운드 큐 10,000 → 2,000 |
| 7 | `AsyncConfig.java` | `CallerRunsPolicy` → 로깅 + `AbortPolicy` |
| 8 | `ChatWebSocketController.java` | `log.info` → `log.debug` |
| 9 | 테스트 데이터 | CLUB/SCHEDULE 혼합, 유저당 4~8개 방, 핫룸/일반/저활동 분포 |
| 10 | 커서 페이징 인덱스 | `idx_message_room_deleted_sent`, `idx_message_room_sent_id` 추가 |

---

## 정리하며

가장 오래 걸린 건 P4 WebSocket 왕복 시간이 "측정불가"인 원인을 찾는 것이었다. k6가 타임아웃을 내뱉는 게 서버가 느린 건지, 구조가 연결이 안 된 건지 구분이 안 됐다. Redis 채널에 아무도 Subscribe하지 않았다는 사실은 테스트 코드에 Mock이 있어서 단위 테스트에서 전혀 드러나지 않았다.

`ChatSubscriber`가 만들어져 있고 `MessageListener` 인터페이스도 구현하고 있어서 "연결된 것 같아 보였다". 실제로 `RedisMessageListenerContainer`에 등록된 적이 없었는데. 컴파일이 되고 단위 테스트도 통과하는데 실제 메시지 흐름은 처음부터 끊겨 있었다.

부하 테스트는 이런 종류의 "연결 안 된 파이프라인"을 잡아내는 데 효과적이다. 단위 테스트가 컴포넌트 단독 동작을 검증한다면, 부하 테스트는 컴포넌트들이 실제로 연결되어 기대한 흐름으로 동작하는지를 확인한다.

> **만들어둔 파이프라인은 실제로 연결되어 있는지 엔드-투-엔드로 확인해야 한다.**
> 인터페이스를 구현했다고 연결된 게 아니다. 컨테이너에 등록하고, 실제 메시지가 끝까지 흘러가는지 통합 테스트로 검증해야 한다.

---

## 시리즈 탐색

**◀ 이전 글**  
[유저 도메인 — 인증 이후, 사용자 생명주기에서 무너진 것들](/user-lifecycle-bugs/)

**다음 글 ▶**  
[피드 도메인 — 부하 테스트 9라운드, 슬로우 쿼리 29,008건에서 전 구간 통과까지](/feed-performance-load-test/)
