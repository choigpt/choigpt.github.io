---
title: 알림 SSE 구현 중 발견한 연쇄 버그와 구조 개선
date: 2026-02-20
tags: [SSE, Spring, JPA, 트러블슈팅, Java, 알림]
permalink: /sse-notification-chained-bugs/
excerpt: "sendEvent() 반환값 무시와 복구 쿼리 조건 불일치, 두 버그가 맞물려 오프라인 사용자의 알림이 전송된 것으로 기록되고 재연결해도 복구되지 않는 시나리오를 만들었다."
---

## 개요

알림 API와 SSE 실시간 전송 코드를 다시 살펴보다가 13개의 이슈를 발견했다. 단독 버그도 있었지만 가장 파악하기 어려웠던 건 세 개의 버그가 맞물려 **오프라인 사용자의 알림이 전송된 것으로 기록되고, 나중에 재접속해도 복구되지 않는** 시나리오를 만들어낸 것이었다. 각각은 작아 보이는 실수인데, 조합하면 알림이 조용히 누락될 수 있었다.

---

## 시스템 구조

### 알림 전송 흐름 (수정 전)

```
알림 이벤트 발생 (댓글, 리피드 등)
    ↓
NotificationService.createNotification() → DB 저장
    ↓
NotificationCreatedEvent 발행 (@TransactionalEventListener AFTER_COMMIT)
    ↓
handleNotificationCreated() (@Async — SimpleAsyncTaskExecutor로 폴백됨)
    ↓
SseEmittersService.sendEvent() → 실시간 전송 시도
    ↓
sseSent 상태 업데이트 (반환값 무시, 항상 true)
```

### 알림 전송 흐름 (수정 후)

```
알림 이벤트 발생 (댓글, 리피드 등)
    ↓
NotificationService.createNotification() → DB 저장
    ↓
NotificationCreatedEvent 발행 (@TransactionalEventListener AFTER_COMMIT)
    ↓
NotificationQueue (in-memory LinkedBlockingQueue)에 적재
    ↓
@Scheduled(fixedDelay=100ms) 배치 처리 — NotificationBatchProcessor
    ↓
SseEventSender.sendEvent() → 실시간 전송 시도
    ↓
전송 성공한 id만 markSseSentByIds() — sseSent=true 반영
```

SSE 연결은 `SseConnectionManager` 내부의 `ConcurrentHashMap<Long, SseConnection>`으로 관리된다. 사용자가 오프라인이면 `sendEvent()`는 예외 없이 `false`를 반환한다. 재연결 시에는 `sendMissedMessages()`가 `sseSent=false` 기준으로 DB를 조회해 놓친 알림을 재전송한다.

큐는 in-memory `LinkedBlockingQueue`를 사용한다. Redis 큐로 전환하면 서버 재시작 시 큐 유실 문제를 방지할 수 있지만, 현재 단일 서버 환경에서는 in-memory로 수용했다. 서버 스케일 아웃 시에는 Kafka pub/sub으로 전환 예정이다.

---

## 버그 상세

### 연쇄 버그 — `sendEvent()` 반환값 무시 + 놓친 메시지 쿼리 불일치

**첫 번째: `sendEvent()` 반환값을 무시한다**

```java
// NotificationService.java — 수정 전
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
// NotificationRepositoryImpl.java — 수정 전
public List<Notification> findUnreadNotificationsByUserId(Long userId) {
    return queryFactory.selectFrom(notification)
        .where(
            notification.user.userId.eq(userId),
            notification.isRead.eq(false)  // ← isRead 기준, sseSent가 아님
        )...
}
```

재연결 시 "SSE로 전송되지 않은 알림"을 찾아야 하는데, "읽지 않은 알림"을 조회하고 있었다. `sseSent` 필드와 `idx_notification_sse_failed` 인덱스가 이미 존재했지만 조회 조건에서 한 번도 사용되지 않고 있었다.

**결합하면 이렇게 동작한다:**

```
오프라인 사용자에게 알림 발생
  → sendEvent() returns false    (전송 실패)
  → 반환값 무시, sseSent=true 기록
  → 재연결 시 isRead=false 기준으로 조회
  → sseSent=true이므로 복구 대상으로 인식 안 됨
  → 알림 영구 유실
```

두 곳을 함께 고쳐야 비로소 동작하는 구조였다.

---

### JPA 엔티티를 SSE로 직접 전송 — 민감 정보 노출 위험

```java
// SseEmittersService.java — 수정 전
connection.getEmitter().send(SseEmitter.event()
    .data(notification));  // ← Notification 엔티티 그대로 직렬화
```

`Notification` 엔티티에는 `User user` 필드가 LAZY로 연관되어 있고, `User`에는 `kakaoAccessToken`, `kakaoId` 같은 민감한 필드가 있다. `@Async` + `AFTER_COMMIT` 시점은 영속성 컨텍스트가 이미 닫혀 있으므로, Jackson이 `user` getter를 호출하는 순간 `LazyInitializationException`이 터진다. 반대로 만약 직렬화가 성공하면 `kakaoAccessToken`이 SSE 스트림에 그대로 실려 나간다.

---

### `@Async` executor 미지정 — `SimpleAsyncTaskExecutor` 폴백

```java
@Async  // ← executor 이름 미지정
public void handleNotificationCreated(NotificationCreatedEvent event) { ... }

// AsyncConfig.java
@Bean(name = "customAsyncExecutor")  // 이름이 "taskExecutor"가 아님
public Executor customAsyncExecutor() { ... }
```

`@Async`에 qualifier를 지정하지 않으면 Spring은 `taskExecutor`라는 이름의 빈을 찾고, 없으면 `SimpleAsyncTaskExecutor`로 폴백한다. `SimpleAsyncTaskExecutor`는 스레드풀 없이 요청마다 새 스레드를 생성해 알림 폭발 시 무제한 스레드 생성으로 이어질 수 있었다.

---

### Event ID 타임존 불일치 — 놓친 메시지 누락/중복

```java
// 수정 전
private String generateHeartbeatEventId() {
    return String.format("heartbeat_%s",
        LocalDateTime.now().format(...));  // ← 시스템 타임존 (KST)
}
// lastEventTime은 UTC 기준 파싱, createdAt은 KST 기준 저장
```

한국 서버(KST = UTC+9) 환경에서 두 값 사이에 9시간 차이가 발생한다. SSE 재연결 시 `Last-Event-ID` 헤더로 마지막 수신 시각을 복구하는 로직이 이 불일치로 인해 9시간치 알림을 중복 전송하거나 누락시킬 수 있었다.

---

### 기타 문제들

`IOException`만 catch하는 `SseEmitter.send()`는 `IllegalStateException` 발생 시 `cleanupConnection()`이 호출되지 않아 `activeConnections`에 죽은 연결이 남는다. `updateSseSentStatus()`는 트랜잭션 컨텍스트 없이 실행되며 내부의 `entityManager.clear()`가 1차 캐시 사이드이펙트를 유발했다. 놓친 메시지는 DB에서 전부 가져온 후 자바 코드에서 시간 조건으로 필터링하는 전체 스캔 구조였다.

**FCM 미사용 의존성**: 초기에 FCM으로 알림을 구현하다가 SSE로 방향을 바꿨는데, `firebase-admin:9.5.0` 의존성이 `build.gradle`에 제거되지 않은 채 남아 있었다.

**FCM 연동 + 알림 타입 미연동**: SSE로 전환하면서 FCM 연동이 빠진 상태다. 알림 타입 3(정산 완료)과 5(모임 해산)도 이벤트 발행 코드가 없어 해당 상황에서 알림이 생성되지 않는다.

---

## 해결

### 클래스 분리 및 구조 재정비

기존 `SseEmittersService`를 세 클래스로 분리했다. `SseConnectionManager`는 `ConcurrentHashMap<Long, SseConnection>` 관리만 담당하고, `SseEventSender`는 실제 전송 로직을 맡는다. `NotificationBatchProcessor`는 큐에서 꺼낸 알림을 배치로 처리하고, 전송 성공한 id에 한해 `markSseSentByIds()`를 호출한다.

### `NotificationSseDto` 도입

```java
public record NotificationSseDto(
    Long notificationId,
    String type,
    String message,
    boolean isRead,
    LocalDateTime createdAt
) {
    public static NotificationSseDto from(Notification notification) { ... }
}
```

`User` 필드를 포함하지 않으므로 `LazyInitializationException`과 민감 정보 노출 문제가 함께 해결된다.

### 반환값 체크 + 놓친 메시지 복구 — 연쇄 버그 해결

```java
// NotificationBatchProcessor.java — 수정 후
boolean success = sseEventSender.sendEvent(userId, "notification", dto);
if (success) {
    successIds.add(notification.getId());
}
// 루프 종료 후 일괄 반영
notificationRepository.markSseSentByIds(successIds);
```

전송에 성공한 건만 `markSseSentByIds()`로 DB에 반영한다.

```java
// NotificationRepositoryImpl.java — 수정 후
.where(
    notification.user.userId.eq(userId),
    notification.sseSent.eq(false)  // isRead → sseSent로 변경
)
.orderBy(notification.createdAt.asc())
.limit(limit)
```

`sseSent` 인덱스가 실제로 사용되도록 조회 조건을 변경하고, 전체 스캔 대신 `limit`을 걸어 재연결 시 한 번에 처리할 양을 제한했다.

### `@Async` 제거 + 배치 처리 전환

`@Async` 핸들러 자체를 제거했다. `@TransactionalEventListener(AFTER_COMMIT)`에서 in-memory `LinkedBlockingQueue`에 적재하고, `@Scheduled(fixedDelay = 100)` 배치가 큐를 드레인한다. SSE 전송 스레드는 `@Qualifier("sseEventExecutor")`로 명시된 Virtual Thread executor를 사용한다.

Event ID는 `Last-Event-ID` 헤더 기반 시각 비교를 제거하고, `sseSent=false` DB 조회로만 재연결 복구를 처리해 타임존 불일치 문제의 근원 자체를 없앴다. Event ID 형식은 `"evt_" + millis + "_" + counter`로 단순화했다.

예외 처리는 `IOException`, `IllegalStateException`, `Exception` 세 가지를 모두 catch하도록 확장했다. 상태 업데이트는 `TransactionTemplate`으로 명확한 트랜잭션 컨텍스트 안에서 실행하고 `entityManager.clear()`는 제거했다. `firebase-admin:9.5.0` 미사용 의존성도 함께 제거했다.

---

## 현재 상태

| 문제 | 상태 |
|------|------|
| `sendEvent()` 반환값 무시 → `sseSent` 항상 true | ✅ 해결 — 반환값 체크 후 성공 id만 일괄 갱신 |
| 놓친 메시지 복구 쿼리 `isRead` 기준 | ✅ 해결 — `sseSent=false` 기준으로 변경 |
| JPA 엔티티 직렬화 → LazyInit + 민감정보 노출 | ✅ 해결 — `NotificationSseDto` 도입 |
| `@Async` executor 미지정 → `SimpleAsyncTaskExecutor` | ✅ 해결 — `@Scheduled` 배치 처리로 전환 |
| Event ID 타임존 불일치 → 누락/중복 전송 | ✅ 해결 — Last-Event-ID 비교 제거, sseSent 기준으로 통일 |
| `IOException`만 catch → 좀비 연결 | ✅ 해결 — `IllegalStateException` 포함 전체 catch |
| 트랜잭션 없는 상태 업데이트 + `entityManager.clear()` | ✅ 해결 — `TransactionTemplate` 적용, `clear()` 제거 |
| 놓친 메시지 전체 스캔 + 메모리 필터링 | ✅ 해결 — `sseSent` 인덱스 활용 + `limit` 적용 |
| FCM 미사용 의존성 | ✅ 해결 — `firebase-admin` 제거 |
| SSE 단일 서버 한정 — 스케일 아웃 불가 | ⏸️ 잔존 — Kafka pub/sub 전환 예정 |
| FCM 연동 + 알림 타입 3/5 미연동 | ⏸️ 잔존 — 기능 구현 필요 |
| 다중 기기 동시 접속 연결 충돌 | ⏸️ 잔존 — `userId → List<SseConnection>` 구조 변경 필요 |

---

## 정리하며

**반환값이 있는 메서드에서 반환값을 쓰지 않는 것은 거의 항상 버그다.** `sseSent` 필드, `idx_notification_sse_failed` 인덱스처럼 만들어두고 실제로 쓰지 않는 코드는 없는 것보다 더 나쁠 수 있다. "이미 고려한 것처럼 보이게" 만드는 함정이 된다.

> **인프라를 만들었다면 실제로 연결하자.**
> 필드, 인덱스, 의존성을 선언해두고 로직을 연결하지 않으면 없는 것과 다름없다.

---

## 시리즈 탐색

**◀ 이전 글**  
[SseAuthenticationFilter 제거 — SSE 인증을 필터 하나로 단일화](/sse-auth-filter-removal/)

**다음 글 ▶**  
[피드 도메인 — Redis 좋아요 파이프라인의 허점과 N+1](/feed-redis-like-n-plus-1/)
