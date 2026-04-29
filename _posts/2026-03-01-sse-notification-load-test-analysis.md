---
layout: post
title: SSE 알림 부하 테스트 — 7가지 시나리오로 찾은 연결 한계와 잔여 병목
date: 2026-03-01
tags: [SSE, MySQL, InnoDB, HikariCP, k6, 성능최적화, 트러블슈팅, Java, 알림]
permalink: /sse-notification-load-test-analysis/
excerpt: "SSE 연결은 7,500 VU까지 429 없이 안정적이었다. 7가지 시나리오를 분리해 테스트한 결과, REST와 혼합할 때만 성공률이 16%로 떨어졌고, markSseSentByIds flush/clear 누락 등 기존 개선에서 빠진 3가지 문제를 추가로 발견했다."
---

## 테스트 목적

[이전 글](/notification-api-performance-improvement/)에서 API 성공률을 13% → 99.9%로 개선한 후, SSE 동작을 시나리오별로 분리 검증했다. SSE 단독 부하의 한계, REST 병행 시 경합 지점, 재연결 시 누락 알림 복구 여부를 7가지 시나리오로 측정했다.

---

## 7가지 시나리오 결과

**시나리오 1 (conflict, 동시 쓰기 충돌):** VU 100, REST 성공률 99.95%, p95 29ms 미만. **시나리오 2 (delivery, SSE 전달 검증):** VU 200, SSE 성공률 99.57%, 평균 25.66개 이벤트/연결. **시나리오 3 (reconnect, 재연결 복구):** VU 200, SSE 성공률 100%, 재연결 성공률 100%. **시나리오 4 (mixed, 운영 시뮬레이션):** VU 500, SSE 성공률 100%, REST 성공률 18.69% — REST API 심각한 성능 저하. **시나리오 5 (batch, 배치 포화):** VU 500, SSE 성공률 86.13%, REST 성공률 16.01%, 36,381건 복구, REST 거의 불가. **시나리오 6 (lifecycle, 수명주기):** VU 1,000, SSE 성공률 100%, 15,898회 재연결 성공. **시나리오 7 (limit, 연결 한계):** VU 7,500, SSE 성공률 100%, 194,933 요청, 에러 0건.

SSE 연결 자체는 7,500 VU까지 429 없이 안정적이었다. 문제는 REST와 동시에 쓸 때였다. 500 VU 혼합 부하에서 REST 성공률이 16~18%로 떨어졌고, 알림 목록 조회 p95는 36,242ms였다.

---

## 현재 한계

SSE 최대 연결은 현재 7,000으로 설정되어 있으며, 테스트에서 7,500 VU까지 429 미발생으로 여유가 있다. HikariCP 풀은 100인데, 500 VU 혼합 부하에서 고갈되어 REST 성공률이 16%까지 떨어졌다. Tomcat 스레드는 200으로 500 VU에서 포화 상태에 도달했다.

SSE 전용 부하는 10,000 VU까지 확장 가능할 것으로 예상된다. SSE + REST 혼합 부하는 500 VU가 이미 한계다. [이전 글](/notification-api-performance-improvement/)에서 적용한 비동기 분리와 인덱스 추가 외에 아래 3가지가 추가로 빠져 있었다.

---

## 이전 개선에서 빠진 것들

### 1. markSseSentByIds() — flush/clear 누락 (성능 개선 중 추가한 메서드)

```java
// NotificationRepositoryImpl.java
@Transactional
public void markSseSentByIds(List<Long> notificationIds) {
    queryFactory
        .update(notification)
        .set(notification.sseSent, true)
        .where(notification.id.in(notificationIds))
        .execute();
    // ← flush/clear 없음. markAllAsReadByUserId()에는 있음
}
```

원본의 `markAllAsReadByUserId()`는 `entityManager.clear()`를 수행한다. [이전 성능 개선](/notification-api-performance-improvement/)에서 추가한 `markSseSentByIds()`는 같은 패턴을 따라야 하는데 빠뜨렸다. 같은 트랜잭션 내에서 `sseSent` 상태를 다시 조회하면 영속성 컨텍스트의 stale 데이터를 반환할 수 있다.

### 2. findUnsentNotificationsByUserId() — 인덱스 순서 불일치 (성능 개선 중 추가한 메서드)

```sql
-- 실행 쿼리
SELECT ... FROM notification WHERE user_id=? AND sse_sent=false
ORDER BY notification_id ASC LIMIT 50
```

인덱스는 `(user_id, sse_sent)` 순서인데 ORDER BY는 `notification_id ASC`다. 인덱스 순서와 정렬 기준이 달라 filesort가 발생할 수 있다. `idx_notification_user_sse_sent`를 `(user_id, sse_sent, id)`로 변경하면 커버링 인덱스로 처리된다.

### 3. deleteNotification() — SELECT + DELETE → 직접 DELETE

```java
// 현재: fetch join SELECT 후 삭제 (2단계)
Notification notification = findNotificationOrThrow(notificationId);
validateOwnership(notification, userId);
notificationRepository.delete(notification);

// 개선: WHERE에 userId 포함해 단일 쿼리로
@Transactional
public void deleteNotification(Long notificationId) {
    long deleted = queryFactory.delete(notification)
        .where(notification.id.eq(notificationId),
               notification.user.userId.eq(userId))
        .execute();
    if (deleted == 0) {
        throw new CustomException(ErrorCode.NOTIFICATION_NOT_FOUND);
    }
}
```

불필요한 fetch join SELECT를 제거하고 락 보유 시간을 줄인다.

---

## 정리하며

**시나리오를 나눠 테스트하면 SSE 문제와 REST 문제를 구분할 수 있다.**
단일 테스트로 실행했다면 SSE 성능 문제, REST 성능 문제, 혼합 시에만 발생하는 문제를 구분하기 어렵다. limit 시나리오에서 SSE 자체가 7,500 VU까지 안정적임을 확인했고, mixed의 16% REST 성공률은 혼합 시에만 발생하는 커넥션 경합 문제임을 확인했다.

**SSE 연결 수 한계와 REST 경합은 별개 문제다.**
SSE는 DB 커넥션 없이 연결을 유지한다. 연결 한계는 커넥션 풀이 아니라 파일 디스크립터와 Tomcat 설정이 결정한다. 7,500 VU에서 에러가 없었던 이유다. 반면 REST 성공률이 저하된 원인은 subscribe 시점에 `sendMissedNotifications()`가 커넥션을 점유하기 때문이며, 이는 [이전 글](/notification-api-performance-improvement/)에서 비동기 분리로 해결했다.

---

## 시리즈 탐색

**◀ 이전 글**
[Finance 도메인 부하 테스트 — Mock 도입 후 발견된 버그](/finance-domain-load-test-mock-exposed-hidden-bug/)

**▶ 다음 글**
[Finance 도메인 — 정산 p95 19.5초를 2.46초로, 배치 UPDATE와 Kafka 비재시도 예외 등록](/finance-settlement-batch-kafka-tuning/)
