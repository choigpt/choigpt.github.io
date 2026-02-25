---
title: 알림 SSE 구현 중 발견한 연쇄 버그와 구조 개선
date: 2026-02-19
tags: [SSE, Spring, JPA, 트러블슈팅, Java, 알림]
---

# 알림 SSE 구현 중 발견한 연쇄 버그와 구조 개선

## 개요

알림 API와 SSE 실시간 전송 코드를 다시 살펴보다가 13개의 이슈를 발견했다. 단독 버그도 있었지만 가장 골치 아팠던 건 세 개의 버그가 맞물려 **오프라인 사용자의 알림이 전송된 것으로 기록되고, 나중에 재접속해도 복구되지 않는** 시나리오를 만들어낸 것이었다. 각각은 작아 보이는 실수인데, 조합하면 알림이 조용히 영구 유실된다.

---

## 시스템 구조

### 알림 전송 흐름

```
알림 이벤트 발생 (댓글, 리피드 등)
    ↓
NotificationService.createNotification() → DB 저장
    ↓
NotificationCreatedEvent 발행 (@TransactionalEventListener)
    ↓
handleNotificationCreated() (@Async)
    ↓
SseEmittersService.sendEvent() → 실시간 전송 시도
    ↓
sseSent 상태 업데이트
```

SSE 연결은 `SseEmittersService` 내부의 `ConcurrentHashMap<Long, SseConnection>`으로 관리된다. 사용자가 오프라인이면 `sendEvent()`는 예외 없이 `false`를 반환한다. 재연결 시에는 `sendMissedMessages()`가 놓친 알림을 복구하도록 설계되어 있었다. 문제는 이 설계의 각 단계가 제대로 연결되어 있지 않았다는 것이다.

---

## 버그 상세

### 연쇄 버그 — `sendEvent()` 반환값 무시 + 놓친 메시지 쿼리 불일치

가장 심각한 문제는 세 곳의 버그가 연쇄적으로 맞물린 것이었다.

**첫 번째: `sendEvent()` 반환값을 무시한다**

```java
// NotificationService.java
private void sendNotification(Notification notification) {
    Long userId = notification.getUser().getUserId();
    try {
        sseEmittersService.sendEvent(userId, "notification", notification);
        updateSseSentStatus(notification, true);  // ← 반환값과 무관하게 항상 호출
    } catch (Exception e) {
        updateSseSentStatus(notification, false);
    }
}
```

`sendEvent()`는 사용자가 오프라인이면 예외 없이 `false`를 반환한다. 그런데 `sendNotification()`은 이 반환값을 완전히 무시하고 바로 `updateSseSentStatus(true)`를 호출한다. 사용자가 오프라인이어도 `sseSent=true`로 기록된다.

**두 번째: 놓친 메시지 복구 쿼리가 `isRead` 기준이다**

```java
// NotificationRepositoryImpl.java
public List<Notification> findUnreadNotificationsByUserId(Long userId) {
    return queryFactory.selectFrom(notification)
        .where(
            notification.user.userId.eq(userId),
            notification.isRead.eq(false)  // ← isRead 기준!
        )...
}
```

재연결 시 "SSE로 전송되지 않은 알림"을 찾아야 하는데, "읽지 않은 알림"을 조회하고 있었다. `isRead=false`와 `sseSent=false`는 전혀 다른 의미다.

`sseSent` 필드와 `idx_notification_sse_failed` 인덱스가 이미 존재했지만 조회 조건에서 한 번도 사용되지 않고 있었다.

**결합하면 이렇게 동작한다:**

```
오프라인 사용자에게 알림 발생
  → sendEvent() returns false    (전송 실패)
  → 반환값 무시, sseSent=true 기록
  → 재연결 시 isRead=false 기준으로 조회
  → sseSent=true이므로 복구 대상으로 인식 안 됨
  → 알림 영구 유실
```

#1만 고쳐도 `sseSent=false`인 알림이 생기기 시작하지만 쿼리가 여전히 `isRead` 기준이라 복구가 안 된다. #2만 고쳐도 `sseSent`가 항상 `true`라 복구할 대상이 없다. 두 곳을 함께 고쳐야 비로소 동작하는 구조였다.

---

### JPA 엔티티를 SSE로 직접 전송 — 민감 정보 노출 위험

```java
// SseEmittersService.java:346
connection.getEmitter().send(SseEmitter.event()
    .data(notification));  // ← Notification 엔티티 그대로 직렬화
```

`Notification` 엔티티에는 `User user` 필드가 LAZY로 연관되어 있고, `User`에는 `kakaoAccessToken`, `kakaoId` 같은 민감한 필드가 있다.

`@Async` + `AFTER_COMMIT` 시점은 영속성 컨텍스트가 이미 닫혀 있으므로, Jackson이 `user` getter를 호출하는 순간 `LazyInitializationException`이 터진다. 반대로 만약 직렬화가 성공하면 `kakaoAccessToken`이 SSE 스트림에 그대로 실려 나간다. 어느 쪽이든 좋지 않다.

---

### `@Async` executor 미지정 — `SimpleAsyncTaskExecutor` 폴백

```java
@Async  // ← executor 이름 미지정
public void handleNotificationCreated(NotificationCreatedEvent event) { ... }

// AsyncConfig.java
@Bean(name = "customAsyncExecutor")  // 이름이 "taskExecutor"가 아님
public Executor customAsyncExecutor() { ... }
```

`@Async`에 qualifier를 지정하지 않으면 Spring은 `taskExecutor`라는 이름의 빈을 찾고, 없으면 `SimpleAsyncTaskExecutor`로 폴백한다. `SimpleAsyncTaskExecutor`는 스레드풀 없이 요청마다 새 스레드를 생성한다. 알림이 폭발적으로 발생하면 무제한 스레드 생성으로 이어질 수 있었다.

---

### Event ID 타임존 불일치 — 놓친 메시지 누락/중복

```java
// 하트비트 Event ID 생성
private String generateHeartbeatEventId() {
    return String.format("heartbeat_%s",
        LocalDateTime.now().format(...));  // ← 시스템 타임존 (KST)
}

// Event ID 파싱 후 createdAt과 비교
.filter(notification -> notification.getCreatedAt().isAfter(lastEventTime))
// lastEventTime은 UTC 기준 파싱, createdAt은 KST 기준 저장
```

한국 서버(KST = UTC+9) 환경에서 두 값 사이에 9시간 차이가 발생한다. 이미 보낸 메시지를 다시 전송하거나, 반대로 놓친 메시지를 이미 전송된 것으로 판단하여 유실한다.

---

### 기타 문제들

**`IOException`만 catch — 좀비 연결**

`SseEmitter.send()`는 `IOException` 외에 이미 완료된 emitter에 재전송 시 `IllegalStateException`도 던진다. 이 경우 `cleanupConnection()`이 호출되지 않아 `activeConnections`에 죽은 연결이 남는다. 이후 해당 사용자에게 보내는 모든 알림이 같은 예외를 반복했다.

**트랜잭션 없는 상태 업데이트 + `entityManager.clear()`**

`updateSseSentStatus()`는 트랜잭션 컨텍스트 없이 실행됐다. 내부에서 `entityManager.clear()`가 매번 호출되어, 동일 스레드풀에서 다른 JPA 작업이 함께 실행 중이면 1차 캐시가 날아가는 사이드이펙트가 생길 수 있었다.

**놓친 메시지 조회 시 전체 스캔 + 메모리 필터링**

DB에서 읽지 않은 알림을 전부 가져온 후 자바 코드에서 시간 조건으로 필터링한다. 읽지 않은 알림이 수천 건인 사용자는 그 전부를 조회·영속화한 뒤 대부분을 버리는 셈이었다.

**`ConcurrentHashMap` 기반 SSE — 스케일 아웃 불가**

SSE 연결을 JVM 메모리 내에서만 관리한다. 서버가 2대 이상이면 알림이 다른 서버에서 생성될 경우 전송에 실패한다. Kafka를 이미 사용 중이므로 알림 이벤트를 topic으로 발행·구독하는 방식으로 해결 가능하다.

**사용자당 SSE 연결 1개 제한**

같은 사용자가 폰+PC로 동시 접속하면 기존 연결이 강제 종료된다. 브라우저가 자동 재연결을 시도하면 두 기기가 서로의 연결을 끊는 무한 루프에 빠진다.

**FCM 미구현 + 알림 타입 3/5 미연동**

`FirebaseMessaging` 빈이 등록되어 있지만 주입받아 사용하는 코드가 없다. `NotificationType`에 5개 타입이 정의되어 있으나 실제로 알림 생성 코드와 연동된 것은 `COMMENT`, `REFEED` 2개뿐이다. 오프라인 사용자는 앱을 직접 열지 않으면 알림이 왔는지조차 알 수 없다.

---

## 해결

### `SseEmittersService` 분리 — 구조부터 바꿨다

버그를 하나씩 패치하기 전에 `SseEmittersService` 하나가 너무 많은 책임을 지고 있다는 게 눈에 걸렸다. 연결 관리, 이벤트 전송, 놓친 메시지 복구까지 한 클래스에 다 있었다. 이걸 `SseConnectionManager`, `SseEventSender`, `NotificationBatchProcessor` 세 클래스로 분리하면서 구조를 다시 잡았다.

---

### `NotificationSseDto` 도입 — 엔티티 직렬화 문제 해결

엔티티를 그대로 전송하는 대신 SSE 전용 DTO를 새로 만들었다.

```java
// NotificationSseDto.java
public record NotificationSseDto(
    Long notificationId,
    String type,
    String message,
    boolean isRead,
    LocalDateTime createdAt
) {
    public static NotificationSseDto from(Notification notification) {
        return new NotificationSseDto(
            notification.getId(),
            notification.getNotificationType().getType(),
            notification.getMessage(),
            notification.isRead(),
            notification.getCreatedAt()
        );
    }
}
```

`User` 필드를 포함하지 않으므로 `LazyInitializationException`과 민감 정보 노출 문제가 함께 해결된다.

---

### 반환값 체크 + 놓친 메시지 복구 — 연쇄 버그 해결

**반환값 체크**

```java
// NotificationBatchProcessor.java
sseEventSender.sendEvent(userId, "notification", dto)
    .thenApply(success -> success ? notification.getId() : null)  // 반환값 체크
```

전송에 성공한 건만 `markSseSentByIds()`로 DB에 반영한다. 오프라인 사용자의 알림은 `sseSent=false` 상태로 남는다.

**놓친 메시지 복구 쿼리**

기존에 선언만 되어 있던 `idx_notification_user_sse_sent` 인덱스를 드디어 활용했다.

```java
// NotificationRepositoryImpl.java
public List<NotificationSseDto> findUnsentNotificationsByUserId(Long userId, int limit) {
    return queryFactory
        .select(Projections.constructor(NotificationSseDto.class, ...))
        .from(notification)
        .where(
            notification.user.userId.eq(userId),
            notification.sseSent.eq(false)  // isRead → sseSent로 변경
        )
        .limit(limit)
        ...
}
```

SSE 연결 직후 미전송 알림을 최대 50건 복구하고, 전송 성공한 건만 `markSseSentByIds()`로 갱신한다.

---

### `@Async` 제거 + 배치 처리 전환 — executor 문제 근본 해결

`@Async` 핸들러 자체를 제거하고 `@TransactionalEventListener(AFTER_COMMIT)` → 큐 적재 → `@Scheduled` 배치 처리 방식으로 전환했다. SSE 전송은 `@Qualifier("sseEventExecutor")`로 명시적으로 지정된 Virtual Thread executor를 사용한다.

타임존 불일치 문제는 하트비트/recovery Event ID 생성·파싱 로직 자체를 제거하는 것으로 해결했다. Event ID는 단순히 `"evt_" + millis + "_" + counter` 형식만 사용하고 서버에서 파싱·비교하지 않는다.

---

### 예외 처리 + 트랜잭션 정리

`SseEventSender.sendEventInternal()`에서 `IOException`, `IllegalStateException`, `Exception` 세 가지를 모두 catch하고 각각 `cleanupConnection()`을 호출한다. 좀비 연결이 남는 문제가 해결됐다.

상태 업데이트는 `TransactionTemplate`을 사용해 명확한 트랜잭션 컨텍스트 안에서 실행하도록 변경했다. `entityManager.clear()`도 제거했다.

---

### FCM 미사용 의존성 제거 + 기타 정리

실제 구현 없이 `build.gradle`에만 존재하던 `firebase-admin:9.5.0` 의존성을 제거했다. `StopWatch` `INFO` 로깅 코드와 파싱되지 않는 `recovery_` Event ID 처리 로직도 함께 정리됐다.

---

## 현재 상태

| 문제 | 상태 |
|------|------|
| `sendEvent()` 반환값 무시 → `sseSent` 항상 true | ✅ 해결 — 반환값 체크 후 조건부 갱신 |
| 놓친 메시지 복구 쿼리 `isRead` 기준 | ✅ 해결 — `sseSent=false` 기준으로 변경 |
| JPA 엔티티 직렬화 → LazyInit + 민감정보 노출 | ✅ 해결 — `NotificationSseDto` 도입 |
| `@Async` executor 미지정 → `SimpleAsyncTaskExecutor` | ✅ 해결 — `@Scheduled` 배치 처리로 전환 |
| Event ID 타임존 불일치 → 누락/중복 전송 | ✅ 해결 — Event ID 비교 로직 제거 |
| `IOException`만 catch → 좀비 연결 | ✅ 해결 — `IllegalStateException` 포함 전체 catch |
| 트랜잭션 없는 상태 업데이트 + `entityManager.clear()` | ✅ 해결 — `TransactionTemplate` 적용, `clear()` 제거 |
| 놓친 메시지 전체 스캔 + 메모리 필터링 | ✅ 해결 — `sseSent` 인덱스 활용 + `limit` 적용 |
| FCM 미사용 의존성 | ✅ 해결 — `firebase-admin` 제거 |
| `StopWatch` INFO 로깅 + `recovery_` Event ID 처리 | ✅ 해결 — 데드 코드 정리 |
| SSE 단일 서버 한정 — 스케일 아웃 불가 | ⏸️ 잔존 — Kafka pub/sub 전환 예정 |
| FCM 연동 + 알림 타입 3/5 미연동 | ⏸️ 잔존 — 기능 구현 필요 |
| 다중 기기 동시 접속 연결 충돌 | ⏸️ 잔존 — `userId → List<SseConnection>` 구조 변경 필요 |

---

## 교훈

`sendEvent()`의 반환값을 무시한 것처럼, **반환값이 있는 메서드에서 반환값을 쓰지 않는 것은 거의 항상 버그다.** 특히 성공/실패를 나타내는 `boolean`은 반드시 확인해야 한다.

`sseSent` 필드, `idx_notification_sse_failed` 인덱스, FCM 의존성처럼 만들어두고 실제로 쓰지 않는 코드도 문제다. 없는 것보다 더 나쁠 수 있다. 코드를 읽는 사람이 "이게 동작하는 건가?"를 계속 의심하게 만들고, 실제로 버그가 있어도 "어차피 여기는 연결이 안 됐으니까"라는 착각으로 넘어가게 된다.

> **인프라를 만들었다면 실제로 연결하자.**
> 필드, 인덱스, 의존성을 선언해두고 로직을 연결하지 않으면 없는 것과 다름없다. 오히려 "이미 고려한 것처럼 보이게" 만드는 함정이 된다.
