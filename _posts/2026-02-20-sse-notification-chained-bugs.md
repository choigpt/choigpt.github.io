---
title: 알림 SSE 구현 중 발견한 연쇄 버그와 구조 개선
date: 2026-02-20
tags: [SSE, Spring, JPA, 트러블슈팅, Java, 알림]
permalink: /sse-notification-chained-bugs/
excerpt: "sendEvent() 반환값 무시와 복구 쿼리 조건 불일치, 두 버그가 맞물려 오프라인 사용자의 알림이 전송된 것으로 기록되고 재연결해도 복구되지 않는 시나리오를 만들었다."
---

## 개요

알림 API와 SSE 실시간 전송 코드를 점검한 결과 12개의 이슈가 확인됐다. 단독 버그 외에, 두 개의 버그가 결합되어 **오프라인 사용자의 알림이 전송된 것으로 기록되고, 재접속 후에도 복구되지 않는** 시나리오가 발생할 수 있었다. 개별적으로는 경미한 문제지만, 조합 시 알림이 누락되는 결과를 초래한다.

이 외에도 JPA 엔티티를 SSE로 직접 직렬화하면서 `LazyInitializationException`과 민감 정보 노출 위험이 있었고, `@Async` executor가 지정되지 않아 `SimpleAsyncTaskExecutor`로 폴백되어 스레드가 무제한 생성될 수 있었다.

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

큐는 in-memory `LinkedBlockingQueue`를 사용한다.

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

문제가 두 가지다:

```
경로 1: LazyInitializationException
  @Async + AFTER_COMMIT 시점 → 영속성 컨텍스트 이미 닫힘
  → Jackson이 notification.getUser()를 호출
  → LAZY 프록시 초기화 시도 → 세션 없음 → 예외!

경로 2: 민감 정보 노출
  만약 영속성 컨텍스트가 살아있어서 직렬화가 성공하면?
  → User 엔티티 전체가 JSON으로 변환
  → kakaoAccessToken, kakaoId가 SSE 스트림에 그대로 노출!
```

어느 쪽이든 문제다. 엔티티를 직접 직렬화하면 안 된다.

---

### `@Async` executor 미지정 — `SimpleAsyncTaskExecutor` 폴백

```java
@Async  // ← executor 이름 미지정
public void handleNotificationCreated(NotificationCreatedEvent event) { ... }

// AsyncConfig.java
@Bean(name = "customAsyncExecutor")  // 이름이 "taskExecutor"가 아님
public Executor customAsyncExecutor() { ... }
```

```
@Async의 executor 결정 로직:

1. @Async("이름") → 해당 이름의 빈 사용
2. @Async (이름 없음) → "taskExecutor" 이름의 빈을 찾음
3. "taskExecutor"도 없으면 → SimpleAsyncTaskExecutor로 폴백

이 프로젝트:
  @Async                           ← 이름 없음
  @Bean(name = "customAsyncExecutor")  ← 이름이 "taskExecutor"가 아님
  → 매칭 실패 → SimpleAsyncTaskExecutor 폴백!

SimpleAsyncTaskExecutor의 문제:
  스레드풀이 없다. 요청마다 새 스레드를 생성한다.
  알림 1000건 동시 발생 → 스레드 1000개 생성 → OOM 위험
```

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

```
Event ID 생성: LocalDateTime.now() → KST 기준 (예: 2026-02-20 14:30:00)
Last-Event-ID 파싱: UTC 기준으로 해석 (예: 2026-02-20 05:30:00으로 인식)

차이: 9시간

재연결 시:
  "05:30:00 이후의 알림을 보내줘" (UTC로 해석)
  → 실제로는 14:30:00(KST) 이후를 원하는 것
  → 05:30:00~14:30:00 사이의 알림을 중복 전송!

또는 반대 방향이면 9시간치 알림이 누락.
```

---

### 기타 문제들

- **`IOException`만 catch**: 좀비 연결이 `activeConnections`에 남음. `IllegalStateException`을 안 잡아서 `cleanupConnection()` 미호출.
- **트랜잭션 없는 상태 업데이트**: 1차 캐시 사이드이펙트. `entityManager.clear()`를 트랜잭션 밖에서 호출.
- **놓친 메시지 전체 스캔**: 재연결 시 불필요한 DB 부하. 전부 가져온 후 자바에서 시간 필터링.
- **FCM 미사용 의존성**: 빌드 크기 불필요 증가. SSE로 전환 후 `firebase-admin:9.5.0` 미제거.
- **알림 타입 3, 5 미연동**: 정산 완료/모임 해산 시 알림 미생성. 이벤트 발행 코드 없음.

---

## 해결

### 클래스 분리 및 구조 재정비

기존 `SseEmittersService` 하나에 몰려있던 책임을 분리:

```
수정 전: SseEmittersService (연결 관리 + 전송 + 상태 업데이트 전부)

수정 후:
  SseConnectionManager  — ConcurrentHashMap<Long, SseConnection> 관리만
  SseEventSender        — 실제 SSE 전송 로직
  NotificationBatchProcessor — 큐에서 알림을 꺼내 배치 전송 + 성공 id만 DB 반영
```

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

```
@Async 제거 → 배치 처리로 전환:

수정 전: @Async 핸들러가 알림마다 스레드 생성
수정 후:
  1. @TransactionalEventListener(AFTER_COMMIT) → LinkedBlockingQueue에 적재
  2. @Scheduled(fixedDelay=100ms) 배치가 큐를 drain
  3. SSE 전송은 @Qualifier("sseEventExecutor") Virtual Thread executor 사용
```

```
타임존 문제 해결:
  수정 전: Last-Event-ID 헤더로 시각 비교 → KST/UTC 불일치
  수정 후: Last-Event-ID 비교 자체를 제거. sseSent=false DB 조회만으로 복구.
          Event ID 형식: "evt_" + millis + "_" + counter (단순화)
```

```
기타 수정:
  - 예외: IOException만 → IOException + IllegalStateException + Exception 전부 catch
  - 상태 업데이트: TransactionTemplate으로 명시적 트랜잭션. entityManager.clear() 제거.
  - 미사용 의존성: firebase-admin:9.5.0 제거
```

---

## 현재 상태

- **`sendEvent()` 반환값 무시 → `sseSent` 항상 true**: ✅ 해결 — 반환값 체크 후 성공 id만 일괄 갱신
- **놓친 메시지 복구 쿼리 `isRead` 기준**: ✅ 해결 — `sseSent=false` 기준으로 변경
- **JPA 엔티티 직렬화 → LazyInit + 민감정보 노출**: ✅ 해결 — `NotificationSseDto` 도입
- **`@Async` executor 미지정 → `SimpleAsyncTaskExecutor`**: ✅ 해결 — `@Scheduled` 배치 처리로 전환
- **Event ID 타임존 불일치 → 누락/중복 전송**: ✅ 해결 — Last-Event-ID 비교 제거, sseSent 기준으로 통일
- **`IOException`만 catch → 좀비 연결**: ✅ 해결 — `IllegalStateException` 포함 전체 catch
- **트랜잭션 없는 상태 업데이트 + `entityManager.clear()`**: ✅ 해결 — `TransactionTemplate` 적용, `clear()` 제거
- **놓친 메시지 전체 스캔 + 메모리 필터링**: ✅ 해결 — `sseSent` 인덱스 활용 + `limit` 적용
- **FCM 미사용 의존성**: ✅ 해결 — `firebase-admin` 제거
- **SSE 단일 서버 한정 — 스케일 아웃 불가**: ⏸️ 잔존
- **FCM 연동 + 알림 타입 3/5 미연동**: ⏸️ 잔존 — 기능 구현 필요
- **다중 기기 동시 접속 연결 충돌**: ⏸️ 잔존 — `userId → List<SseConnection>` 구조 변경 필요

---

## 정리하며

**반환값이 있는 메서드에서 반환값을 쓰지 않는 것은 거의 항상 버그다.** `sseSent` 필드, `idx_notification_sse_failed` 인덱스처럼 만들어두고 실제로 쓰지 않는 코드는 없는 것보다 더 나쁠 수 있다. "이미 고려한 것처럼 보이게" 만드는 함정이 된다.

> **인프라를 만들었다면 실제로 연결하자.**
> 필드, 인덱스, 의존성을 선언해두고 로직을 연결하지 않으면 없는 것과 다름없다.

이 구조의 성능은 [알림 API 성능 편](/notification-api-performance-improvement/)에서 부하 테스트로 검증한다.

---

## 시리즈 탐색

**◀ 이전 글**
[SseAuthenticationFilter 제거 — SSE 인증을 필터 하나로 단일화](/sse-auth-filter-removal/)

**▶ 다음 글**
[피드 도메인 — Redis 좋아요 파이프라인의 허점과 N+1](/feed-redis-like-n-plus-1/)
