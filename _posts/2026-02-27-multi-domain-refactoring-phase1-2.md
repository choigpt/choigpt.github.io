---
title: 멀티 도메인 코드 정리 — 표면 정리부터 Semaphore 제거까지
date: 2026-02-27
tags:
  - Spring
  - Java
  - 리팩토링
  - 알림
  - SSE
  - Redis
  - 동시성
  - 설계
permalink: /multi-domain-cleanup-surface-to-semaphore/
excerpt: 성능 개선 사이클을 돌리면서 코드를 다시 보다 보니 가독성이 떨어지는 곳들이 눈에 걸렸다.
  IllegalArgumentException이 CustomException으로, ResponseEntity<?>가 구체 타입으로, 그리고
  NotificationService 안에 숨어있던 Redis 카운터 로직이 별도 Bean으로 분리됐다.
---

## 개요

목표는 전체 시스템의 성능과 품질을 함께 끌어올리는 것이었다. 사이클을 돌리면서 진행하다 보니 성능 작업 중에 코드를 다시 보게 되는 순간들이 생겼고, 읽기 어렵거나 책임이 뒤섞인 곳들이 자연스럽게 눈에 걸렸다.

체계적인 계획보다는 사이클 중에 눈에 걸리는 것을 고쳐나간 방식이라 도메인별 깊이가 고르지 않다. Phase 1은 Chat·Feed·Club·Notification 전 도메인을 대상으로 표면적인 문제를 쓸어냈고, Phase 2는 알림 모듈에 집중해 성능 작업 중 미뤄뒀던 구조 문제들을 정리했다. Redis 카운터 로직 분리, 병렬 SSE 전송, Executor 통합이 주요 내용이다.

---

## Phase 1 — 도메인별 표면 정리

### Chat

가장 먼저 눈에 띈 건 예외 처리 방식이었다. `IllegalArgumentException`을 그대로 던지고 있었고, `existsByXxx` 대신 `findById` + `.isPresent()` 조합을 쓰는 곳이 여러 군데였다.

```java
// 수정 전 — 도메인 맥락 없는 런타임 예외
if (chatRoom == null) throw new IllegalArgumentException("채팅방 없음");

// 수정 후 — 도메인 에러코드 부여
throw new CustomException(ErrorCode.CHAT_ROOM_NOT_FOUND);
```

`existsById`로 교체해 불필요한 엔티티 로딩을 제거했고, 사용하지 않는 `UserRepository` 의존성도 함께 삭제했다. Redis에 파이프 구분자(`|`)로 직렬화하던 방식은 Jackson JSON으로 통일했다. 파이프 기반 직렬화는 필드 순서가 바뀌거나 값에 구분자가 포함되면 파싱이 깨지는 문제가 있었다.

### Feed

서비스 곳곳에 흩어진 "클럽 조회 + 없으면 예외" 패턴을 `findClubOrThrow()`, "멤버십 확인 + 없으면 예외" 패턴을 `validateClubMembership()` 헬퍼로 추출했다. 같은 3줄이 4군데에 복붙돼 있었다.

`@PreDestroy`가 없어서 서버 종료 시 스레드풀이 정리되지 않던 문제도 추가했다. 숫자 상수가 흩어져 있던 것도 `CHUNK_SIZE = 5`, `MAX_CACHEABLE_PAGE = 5` 같은 이름 있는 상수로 교체했다.

```java
// 수정 전
private List<FeedRepository.FeedIdWithCounts> findPersonalFeedChunked(...) {
    if (clubIds.size() <= 5) { ... }  // 매직넘버
}

// 수정 후
private static final int CLUB_CHUNK_SIZE = 5;
```

`ResponseEntity<?>` 반환 타입을 구체 타입으로 교체했다. 와일드카드는 같은 코드베이스 내 호출부에서 컴파일 타임 타입 안전성을 해치고 가독성을 떨어뜨린다.

### Club

`withdraw()` 메서드명이 금융 용어처럼 읽혀서 `leaveClub()`으로 변경했다. `Optional.get()` 직접 호출을 `Optional.map().orElse()` 패턴으로 교체했다. null을 직접 다루지 않는 방향이다. 다른 도메인에 비해 구조적 문제가 적어 변경 범위가 작았다.

### Notification Controller

`ResponseEntity<?>` → 구체 타입 반환. 나머지 도메인과 동일한 패턴 적용.

---

## Phase 2 — 알림 모듈 후속 정리

성능 작업을 하다 보면 가독성이나 테스트 가능성을 챙기기가 어렵다. 알림 모듈이 딱 그랬다. 동작은 하는데 `NotificationService` 안에 Redis 로직이 섞여 있었고, 복구 로직에 Semaphore가 달려 있었고, Executor 설정이 두 파일에 나뉘어 있었다. 기능 추가도 아니고 새로운 개념도 아닌, 그냥 밀린 정리를 한 것이다.

### NotificationUnreadCounter 분리 (SRP)

`NotificationService`가 `StringRedisTemplate`을 직접 의존하며 Redis 미읽음 카운터 로직을 인라인으로 처리하고 있었다.

```java
// 수정 전 — 서비스가 Redis 키 조합부터 카운트 증감까지 직접 관리
@Service
public class NotificationService {
    private final StringRedisTemplate redisTemplate;

    public void markAsRead(Long userId, Long notificationId) {
        String key = "unread:" + userId;
        redisTemplate.opsForValue().decrement(key);  // Redis 직접 사용
        // DB 업데이트 로직 혼재
    }
}
```

`NotificationUnreadCounter`를 별도 Bean으로 분리했다. Redis 장애 시 DB fallback 후 캐시 갱신하는 로직도 이 Bean 안으로 응집됐다. `NotificationService`는 `NotificationUnreadCounter`에 위임만 한다.

```java
// 수정 후 — Redis 카운터는 별도 Bean이 관리
@Component
public class NotificationUnreadCounter {
    public void decrement(Long userId) { ... }  // Redis + DB fallback
    public long get(Long userId) { ... }
}
```

---

### SseMissedNotificationRecovery — 순차 → 병렬 전송

SSE 재연결 시 누락 알림을 복구하는 로직이 `.join()` 순차 처리 방식이었다. 알림이 10개면 10개를 차례로 전송하고 앞이 끝날 때까지 다음이 기다렸다.

```java
// 수정 전 — 순차 .join()
for (NotificationItemDto missed : missedList) {
    sendEvent(userId, missed).join();  // 앞이 끝날 때까지 블로킹
}
```

`CompletableFuture.allOf()`로 병렬 전송으로 전환했다. `orTimeout(5, TimeUnit.SECONDS)`를 걸어 개별 전송이 블로킹되는 경우에 스레드 점유를 방지했다.

```java
// 수정 후 — CompletableFuture.allOf() 병렬
List<CompletableFuture<Long>> futures = missed.stream()
        .map(n -> sendEventAsync(userId, n).orTimeout(5, TimeUnit.SECONDS))
        .toList();
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
```

Javadoc 주석에 "세마포어 고갈 방지"라는 문구가 있었는데, Semaphore 자체가 제거되면서 "스레드 점유 방지"로 정정했다.

---

### Semaphore(30) 제거 — 인위적 병목

`SseMissedNotificationRecovery`에 `Semaphore(30)`을 두고 동시 복구 건수를 제한하고 있었다. 알림 API 성능 개선 과정에서 300 VU 동시 접속 시 커넥션 풀 고갈을 막기 위해 도입한 설정이었는데, 실제로는 부작용이 더 컸다.

```java
// 수정 전
private static final int MAX_CONCURRENT_RECOVERY = 30;
private final Semaphore semaphore = new Semaphore(MAX_CONCURRENT_RECOVERY);

// sendAllInParallel()
if (!semaphore.tryAcquire()) {
    log.warn("세마포어 포화 — 복구 스킵");
    return Collections.emptyList();  // 알림을 영구히 sse_sent=false로 남김
}
```

제거 근거가 세 가지였다. 첫째, `BoundedVtExecutor`와 Semaphore 양쪽 모두 동시성 제어 역할이 중복되는 인위적 이중 제한이었다. 리팩토링에서 두 가지를 함께 제거하고 Virtual Thread 기반 Executor에 동시성 관리를 일원화했다. 둘째, 세마포어 30개는 서버 재시작 후 동시 재접속 시 대부분 유저의 복구를 무음으로 스킵해 `sse_sent=false`가 영구히 쌓이는 문제를 만들었다. 셋째, HikariCP가 DB 커넥션 풀 대기열을 이미 자체 관리하므로 추가 보호가 불필요했다. Semaphore, 상수, 관련 설정 프로퍼티까지 전부 삭제했다.

---

### SseConfig 삭제 → AsyncConfig 통합

`SseConfig.java`와 `AsyncConfig.java` 두 파일에 Executor 설정이 분산돼 있었다. `sseEventExecutor` Bean을 `AsyncConfig`로 이동해 모든 Executor 설정을 한 곳에서 관리하게 됐다.

| Executor | 용도 | 타입 |
|----------|------|------|
| getAsyncExecutor() | 기본 @Async | 플랫폼 스레드 30~80 |
| customAsyncExecutor | DB 저장, 메일 등 | 플랫폼 스레드 32~100 |
| sseEventExecutor | SSE 이벤트 전송 | Virtual Thread + destroyMethod |

`BoundedVtExecutor` 내부 클래스도 제거했다. 37줄짜리 설정이 6줄로 줄었다. `DisposableBean` 수동 구현도 `destroyMethod = "close"`로 대체했다. `ExecutorService`는 `AutoCloseable`을 구현하고 있지만, Spring은 이를 자동으로 감지해 `close()`를 호출하지 않는다. `@Bean(destroyMethod = "close")`를 명시해야 Spring 컨텍스트 종료 시 `close()`가 호출된다.

---

## 수정 내역 요약

| # | 도메인 | 변경 내용 |
|---|--------|----------|
| 1 | Chat | `IllegalArgumentException` → `CustomException` 도메인 에러코드 |
| 2 | Chat | `existsById` 최적화, unused `UserRepository` 제거 |
| 3 | Chat | 파이프 구분자 직렬화 → Jackson JSON |
| 4 | Feed | `findClubOrThrow`, `validateClubMembership` 헬퍼 추출 |
| 5 | Feed | 매직넘버 상수화, `@PreDestroy` 추가 |
| 6 | Feed/Club/Notification | `ResponseEntity<?>` → 구체 타입 |
| 7 | Club | `withdraw()` → `leaveClub()`, `Optional.map().orElse()` 패턴 |
| 8 | Notification | `NotificationUnreadCounter` Bean 분리 (SRP) |
| 9 | Notification | `NotificationBatchProcessor` `@RequiredArgsConstructor` 전환, DB 실패 로깅 추가 |
| 10 | Notification | `SseMissedNotificationRecovery` 순차 `.join()` → `CompletableFuture.allOf()` 병렬 전송 |
| 11 | Notification | `Semaphore(30)` 및 `BoundedVtExecutor` 제거 — 이중 제한, 복구 스킵 문제 해결 |
| 12 | Config | `SseConfig.java` 삭제, `sseEventExecutor` → `AsyncConfig` 통합 |

---

## 정리하며

Phase 1에서 가장 많이 발견된 패턴은 "만들어뒀는데 안 쓰는 것"이었다. 이미 `existsById`가 있는데 `findById().isPresent()`를 쓰거나, `CustomException`이 있는데 `IllegalArgumentException`을 던지거나, 헬퍼 메서드가 있는데 인라인으로 반복하거나. 도메인마다 작업하는 사람이 달라서 생긴 관습 불일치였다.

Phase 2의 Semaphore 제거는 "보호 로직이 오히려 서비스를 해친다"는 사례였다. 재시작 후 대규모 재접속 시 복구 알림의 대부분이 무음으로 스킵되고 있었는데, 로그에 warn만 찍혀서 알아채기 어려웠다. 실제로 이 문제는 `sse_sent=false` 레코드가 누적되면서 다음 재연결 때마다 복구 대상이 점점 늘어나는 형태로 드러났을 것이다.

> **제한 로직을 추가하기 전에, 그 제한이 실패했을 때 어떻게 동작하는지를 먼저 확인해야 한다.**
> tryAcquire()가 false를 반환할 때 "그냥 넘어가는" 방식은 조용한 데이터 유실로 이어질 수 있다.

---

## 시리즈 탐색

**◀ 이전 글**  
[알림 API 성능 — 13% 성공률에서 100%로, 9번의 시도 기록](/notification-api-performance-improvement/)

**다음 글 ▶**  
[채팅 도메인 리팩토링 — WebSocket 핸들러부터 커서 페이징 DTO까지](/chat-domain-deep-refactoring/)
