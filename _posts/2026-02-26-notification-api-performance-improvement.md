---
layout: post
title: 알림 API 성능 — 13% 성공률에서 100%로, 9번의 시도 기록
date: 2026-02-26
tags: [MySQL, InnoDB, HikariCP, SSE, Redis, 성능최적화, 트러블슈팅, Java, 알림, k6, Virtual Thread]
permalink: /notification-api-performance-improvement/
excerpt: "첫 측정에서 API 성공률 13.6%, 목록 조회 p95 59초. 인덱스 누락부터 비현실적인 시드 데이터까지, 9번의 시도 끝에 성공률 99.9%·p95 21ms에 도달했다."
---

## 시작: 첫 측정값

[SSE 알림 편](/sse-notification-chained-bugs/)에서 재설계한 알림 아키텍처를 부하 테스트로 검증한다.

k6로 처음 부하를 걸었을 때 나온 숫자다.

```
API 성공률:       13.6%
목록조회 p95:     59,801ms
전체읽음 p95:     59,798ms
5xx 오류:         3,336건
MySQL 슬로우쿼리: 3,713건
InnoDB Lock Wait: 28회
```

성공률이 13.6%였고, 성공한 요청도 응답 시간이 약 1분이었다.

애플리케이션 로그에는 다음이 기록되어 있었다.

```
HikariPool-1 - Connection is not available, request timed out after 5010ms
(total=100, active=100, idle=0, waiting=13)
```

커넥션 100개가 전부 active였다. 특정 작업이 커넥션을 장시간 점유하고 있다는 의미다.

---

## k6 테스트 구성

병목이 SSE에 있는지, REST에 있는지, 둘이 경합할 때만 나타나는지 구분하기 위해 4단계로 나눴다.

테스트는 4단계로 구성했다. P1(0~2:40)은 SSE 연결 단독으로 최대 300 VU, P2(2:40~5:20)는 REST API 단독으로 최대 250 VU, P3(5:20~7:30)은 SSE + REST 혼합으로 최대 150 VU, P4(7:30~8:30)는 스파이크로 최대 400 VU를 부여했다.

---

## 1단계 — 커넥션 점유 원인 파악

### 문제 1: 인덱스가 있는데도 풀스캔이 발생

notification 테이블에는 `(user_id, created_at DESC, notification_id DESC)` 등 인덱스가 이미 존재했다. 그런데 EXPLAIN을 돌려보면:

```sql
-- QueryDSL이 실제로 생성하는 SQL
EXPLAIN SELECT ... FROM notification n
JOIN user u ON n.user_id = u.user_id   -- ← QueryDSL이 유발한 JOIN
WHERE u.user_id = 1 ORDER BY n.notification_id DESC LIMIT 20;
```

```
type: ALL   key: NULL   rows: 40,000,000
```

```
원인: notification.user.userId.eq(userId) ← QueryDSL 연관 엔티티 경유
  → User 테이블 JOIN이 발생
  → 옵티마이저가 notification 인덱스 대신 JOIN 기반 실행 계획을 선택
  → 결과적으로 40M행 풀스캔

인덱스가 있어도 쿼리가 JOIN을 포함하면 인덱스를 타지 못할 수 있다.
→ 9단계에서 네이티브 쿼리(WHERE user_id = :userId)로 교체해 해결
```

```
40M행 풀스캔의 영향:
  단독 실행: 수백 ms
  250명 동시 실행: 각 쿼리가 버퍼풀 페이지를 서로 빼앗음
                  → 수십 초씩 커넥션을 점유
```

### 문제 2: 전체읽음이 수만 행에 락을 건다

```java
queryFactory.update(notification)
    .set(notification.isRead, true)
    .where(notification.user.userId.eq(userId), notification.isRead.eq(false))
    .execute();
entityManager.flush();
entityManager.clear();
```

```
WHERE user_id=? AND is_read=0 범위에 InnoDB X-락 + 갭 락이 걸림
  인덱스 없이 풀스캔 → 사실상 테이블 전체를 잠근 것과 동일
  → 같은 유저의 다른 쿼리는 락이 풀릴 때까지 전부 대기
  → Lock Wait 28회의 원인
```

### 문제 3: SSE 구독이 DB 커넥션을 점유하며 블로킹

```java
for (NotificationItemDto item : missed) {
    CompletableFuture<Boolean> result = sseEventSender.sendEvent(...);
    if (Boolean.TRUE.equals(result.join())) {  // 건마다 순차 대기
        sentIds.add(item.notificationId());
    }
}
```

```
미전송 알림 50건을 join()으로 순차 대기
  → SSE 응답이 50건 × 전송시간만큼 지연
  → 그 시간 동안 DB 커넥션도 계속 점유
```

---

## 2단계 — HikariCP 증설 + 즉시 개선 4개

### Fix 1: HikariCP 100 → 200

```yaml
hikari:
  maximum-pool-size: 200
  minimum-idle: 80
```

250 VU가 동시 접근하는데 풀이 100개였다. 일단 배수로 늘렸다.

### Fix 2: markAllAsRead 배치 분할 — 갭 락 범위 축소

```java
List<Long> ids = notificationRepository.findUnreadIdsByUserId(userId);
for (int i = 0; i < ids.size(); i += 500) {
    List<Long> batch = ids.subList(i, Math.min(i + 500, ids.size()));
    notificationRepository.markAsReadByIds(batch);  // WHERE id IN (...)
}
```

`WHERE user_id=? AND is_read=0` 범위락 대신 PK IN 리스트로 바꿨다. IN 절은 갭 락이 아닌 레코드 락만 걸린다.

### Fix 3: Redis 미읽음 카운터 — COUNT(*) 제거

```java
redisTemplate.opsForValue().increment("unread:" + userId);  // 알림 생성 시
redisTemplate.opsForValue().decrement("unread:" + userId);  // 읽음 처리 시
```

매 요청마다 40M행 `COUNT(*)`를 날리던 걸 Redis INCR/DECR으로 대체했다.

### Fix 4: sendMissedNotifications 병렬화

```java
// 전부 띄우고 한 번만 대기
List<CompletableFuture<Boolean>> futures = missed.stream()
    .map(item -> sseEventSender.sendEvent(userId, "notification", item))
    .collect(toList());
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
```

### 결과 — API 성공률이 오히려 떨어졌다

SSE 성공률은 94.2%에서 96.9%로 +2.7% 개선됐고, 전체읽음 p95는 59,798ms에서 39,555ms로 33% 감소했다. 그러나 목록조회 p95는 59,801ms에서 59,783ms로 거의 동일했고, **API 성공률은 13.6%에서 10.4%로 오히려 악화**됐다. 5xx 오류는 3,336건에서 2,770건으로 17% 감소했다.

```
결과 해석:
  전체읽음 -33%  ← 배치 분할 효과
  목록조회 그대로 ← 40M 풀스캔이 그대로니까
  API 성공률 오히려 하락 ← 커넥션 늘린 탓에 더 많은 요청이 동시에 풀스캔
                          → 경합이 오히려 심해짐
```

---

## 3단계 — 인덱스 재구성

원본 코드에도 인덱스가 있었지만, QueryDSL이 유발하는 JOIN 때문에 제대로 활용되지 않고 있었다. 접근 패턴에 맞게 인덱스를 재구성했다.

```sql
-- 목록 조회: WHERE user_id=? [AND id < cursor] ORDER BY id DESC
CREATE INDEX idx_notification_user_id_desc ON notification (user_id, id DESC);

-- 전체읽음 / 미읽음 필터: WHERE user_id=? AND is_read=0
CREATE INDEX idx_notification_user_unread ON notification (user_id, is_read, id);

-- SSE 복구: WHERE user_id=? AND sse_sent=false
CREATE INDEX idx_notification_user_sse_sent ON notification (user_id, sse_sent, id);
```

```
왜 선두 컬럼이 user_id인가:
  user_id가 선두 → 해당 유저의 데이터가 인덱스에서 인접하게 모임
  → 탐색 범위가 유저 단위로 좁아짐
  → 40M행 풀스캔 → 유저당 수만 행 스캔

테이블 룩업 여부:
  countUnreadByUserId → SELECT 컬럼이 인덱스에 다 있음 → Index-Only Scan ✓
  목록 조회           → content, type 등 인덱스에 없는 컬럼 SELECT
                     → 인덱스로 위치 찾고 → 테이블 룩업 추가 발생
                     → 이 부분이 나중에 다시 발목을 잡음
```

### 결과 — MySQL 레벨 병목이 완전히 사라졌다

목록조회 p95는 59,783ms에서 17,017ms로 **71% 감소**했고, 전체읽음 p95는 39,555ms에서 28,834ms로 27% 감소했다. API 성공률은 10.4%에서 **50.8%**로 +40.4%p 상승했다. MySQL 슬로우쿼리는 3,713건에서 **0건**, InnoDB Lock Wait는 28회에서 **0회**로 완전히 해결됐다.

```
MySQL 레벨 병목은 완전히 사라졌다.
  슬로우쿼리: 3,713건 → 0건
  Lock Wait:  28회 → 0회

하지만 API 성공률은 50.8%에 머뭄.
  원인: SSE subscribe가 sendMissedNotifications를 동기로 호출
  → 300명 동시 SSE 구독 → 커넥션 300개가 복구 끝날 때까지 점유
  → 200개 풀이 넘어감
```

---

## 4단계 — SSE 복구를 비동기로 분리

SSE 연결은 DB 없이 즉시 반환하고, 미전송 알림 복구는 별도 스레드에서 처리하도록 바꿨다.

```java
CompletableFuture.runAsync(() -> {
    missedNotificationRecovery.recover(userId, emitter);
}, virtualThreadExecutor);
return emitter;  // 커넥션 없이 즉시 반환
```

> **Virtual Thread + HikariCP 주의점**: `virtualThreadExecutor`를 쓸 때 HikariCP 내부의 `lock`이 carrier thread를 pin시키는 문제가 Java 21에서 보고된 바 있다. 복구 작업 중 HikariCP 획득 구간에서 carrier thread가 블로킹되면 Virtual Thread의 이점이 상쇄될 수 있으므로, 해당 구간의 latency를 별도로 모니터링할 필요가 있다.

SSE 성공률이 97.5%로 올랐다. API 성공률은 크게 변하지 않았다.

`CompletableFuture.runAsync()`도 결국 같은 HikariCP 풀을 쓴다. 스레드를 분리했을 뿐, 커넥션 경합은 그대로였다.

여전히 REST API가 수 초씩 걸리는 이유를 더 파야 했다. k6 테스트 중에 MySQL processlist를 실시간으로 확인했다.

```sql
SELECT id, time, state, SUBSTRING(info, 1, 120) as query
FROM information_schema.processlist WHERE command != 'Sleep';
```

```
총 커넥션: 200   idle: 0   active: 200

user1 미읽음: 30,421건
user2 미읽음: 28,943건
user3 미읽음: 31,205건
```

유저당 미읽음이 3만 건이었다. 배치 분할(500건)을 적용했어도 30,000 ÷ 500 = 60번 UPDATE가 반복되고, 앞에 ID를 가져오는 SELECT 1번까지 합쳐 전체읽음 호출 하나가 60번 이상의 쿼리를 날리고 있었다.

---

## 5단계 — 미읽음 데이터 정규화

```sql
SELECT COUNT(*) FROM notification WHERE is_read = 0;
-- 25,438,201 (전체 미읽음 2,500만 건)

SELECT COUNT(DISTINCT user_id), COUNT(*) FROM notification;
-- unique_users: 10,000   total: 40,273,142   유저당 평균 미읽음 ~2,500건
```

시드 데이터가 비현실적이었다. 실 서비스에서 한 사용자한테 알림이 수천 건씩 쌓이는 일은 없다. 유저당 최신 100건만 미읽음으로 남기고 나머지를 읽음 처리했다. (10단계에서는 유저당 최신 200건만 남기고 나머지 행 자체를 삭제한다.)

> 미읽음 100건은 활성 알림의 현실적 상한이고, 보관 200건은 이미 읽은 알림을 포함한 히스토리 용도다.

```sql
UPDATE notification n
INNER JOIN (
    SELECT user_id, MIN(id) as cutoff_id FROM (
        SELECT user_id, id,
               ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY id DESC) as rn
        FROM notification WHERE is_read = 0
    ) t WHERE rn > 100
) cutoff ON n.user_id = cutoff.user_id AND n.id < cutoff.cutoff_id
SET n.is_read = 1 WHERE n.is_read = 0;
```

> **5단계와 10단계의 차이**: 이 작업은 `is_read` 컬럼만 업데이트한다. 전체 행 수는 여전히 40M이다. 이후에 다시 목록조회가 느린 이유는 SELECT 대상 컬럼(`content`, `type` 등)이 인덱스에 없어 테이블 룩업이 발생하고, 40M행이 버퍼풀보다 크기 때문이다. 10단계에서 행 자체를 삭제하는 것과 별개의 작업이다.

### 결과 — SSE 성공률 처음으로 100%

SSE 성공률은 97.5%에서 **100%**로 +2.5% 올랐다. 목록조회 p95는 17,017ms에서 16,581ms로 3% 감소, 전체읽음 p95는 28,834ms에서 15,600ms로 46% 감소했다. API 성공률은 50.8%에서 **57.3%**로 +6.5%p 상승, 5xx 오류는 2,770건에서 1,825건으로 34% 감소했다.

전체읽음이 60번 반복에서 1~2번으로 줄어들면서 커넥션 점유 시간이 0.5초 이하로 떨어졌다. SSE 성공률이 처음으로 100%를 찍었다.

그래도 API 성공률은 57.3%에서 멈췄다. 아직 병목이 남아있었다.

---

## 6단계 — Buffer Pool이 너무 작았다

쿼리 자체 실행 시간을 MySQL에서 직접 재봤다.

```sql
SELECT notification_id, content, type, is_read, created_at
FROM notification WHERE user_id = 500 ORDER BY notification_id DESC LIMIT 20;
-- 실행 시간: 11,238ms
```

인덱스가 존재하는데 11초가 소요된 원인을 확인하기 위해 버퍼풀 상태를 조회했다.

```
innodb_buffer_pool_size: 768MB
테이블 + 인덱스 크기:    15,367MB (15GB)
```

```
innodb_buffer_pool_size: 768MB
테이블 + 인덱스 크기:    15,367MB (15GB)

→ 데이터의 5%만 메모리에 올라가는 상태
→ 인덱스로 위치를 찾아도 실제 행 데이터는 디스크에 있음
→ 250명 동시 요청 → 랜덤 디스크 I/O 250개 동시 발생
```

```sql
SET GLOBAL innodb_buffer_pool_size = 4294967296;  -- 768MB → 4GB
SET GLOBAL innodb_buffer_pool_instances = 4;       -- 단일 인스턴스 mutex 경합 분산
SET GLOBAL thread_cache_size = 200;
SET GLOBAL sort_buffer_size = 2097152;             -- 256KB → 2MB
SET GLOBAL join_buffer_size = 2097152;
```

### 결과 — 전체읽음 -56%, 목록조회는 여전히 14초

전체읽음 p95는 15,600ms에서 6,843ms로 **56% 감소**했고, 단건읽음 p95는 4,243ms에서 2,525ms로 40% 감소했다. 목록조회 p95는 16,581ms에서 14,930ms로 10% 감소에 그쳤고, API 성공률은 57.3%에서 59.4%로 +2.1% 상승했다.

```
전체읽음: -56% ← 버퍼풀 증설 효과 확실
목록조회: -10% ← 4GB로 올려도 테이블의 73%는 여전히 디스크
               → 버퍼풀만으로는 해결 안 됨
```

---

## 7단계 — HikariCP 300으로 올렸다가 즉시 복원

400 VU 대비 커넥션이 200개로 부족할 가능성을 검토했다.

```yaml
hikari:
  maximum-pool-size: 300
  minimum-idle: 100
```

### 결과 — 대폭 악화

```
안읽음 카운트 p95: 543ms → 4,102ms  (약 8배)
전체읽음 p95:      6,240ms → 16,500ms  (약 3배)
```

```
성능이 오히려 악화된 이유:

  MySQL은 동시 커넥션이 늘수록 InnoDB 내부 mutex 경합이 심해진다:
    row_lock, buf_pool_mutex, log_sys mutex를 300개 커넥션이 동시에 경쟁
    → MySQL 자체가 병목

  최적 커넥션 수 = VU 수가 아니라 DB 처리 능력 기준
    일반적으로 CPU 코어 × 2~4
    이 환경: 4코어 → 200이 이미 상한에 가까움
    → 300으로 올리는 건 역효과. 즉시 200으로 복원.
```

> 커넥션 풀은 동시 사용자 수에 맞추는 게 아니다. 풀을 VU 수만큼 늘리면 DB 내부 경합이 심해져 역효과가 난다.

---

## 8단계 — MySQL 설정 영구화 + Tomcat 튜닝

Docker 재시작 후 버퍼풀이 768MB로 초기화된 것을 확인했다. `SET GLOBAL`은 재시작 시 초기화되므로 `my.cnf`에 영구 반영했다.

```ini
[mysqld]
innodb_buffer_pool_size = 4G
innodb_buffer_pool_instances = 4
innodb_io_capacity = 2000        # HDD 기본값 200 → SSD 2000
innodb_io_capacity_max = 10000
innodb_read_io_threads = 8       # 기본 4 → 8
innodb_write_io_threads = 8
sync_binlog = 0                  # 로컬 개발 환경 한정
innodb_flush_log_at_trx_commit = 2  # 로컬 개발 환경 한정
```

> **운영 환경 주의**: `sync_binlog = 0` + `innodb_flush_log_at_trx_commit = 2` 조합은 OS 크래시 시 최대 1초치 트랜잭션이 유실될 수 있다. 두 설정 모두 fsync를 생략하기 때문에 운영 환경에서는 각각 `1`로 유지해야 한다. 로컬 부하 테스트 전용 설정이다.

k6 진단 리포트에서 `BoundedVtExecutor 세마포어(200) 포화` 경고도 나왔다. 400 VU 스파이크 시 SSE 이벤트 전송이 세마포어에서 대기하면서 이벤트가 지연되고 있었다. `sse-executor-permits`를 200 → 500으로 올렸다.

```yaml
server:
  tomcat:
    max-connections: 10000    # SSE 장기 연결 수용
    accept-count: 200         # TCP 백로그 큐 확장
```

이 시점 베이스라인:

```
목록조회 p95:  14,049ms   (목표 500ms 대비 28배 초과)
안읽음 p95:    3,601ms    (목표 200ms 대비 18배 초과)
단건읽음 p95:  900ms      (목표 300ms 대비 3배 초과)
전체읽음 p95:  2,172ms    (목표 1,000ms 대비 2배 초과)
API 성공률:    59.5%
```

---

## 9단계 — QueryDSL이 불필요한 JOIN을 유발하고 있었다

인덱스 추가와 버퍼풀 증설 후에도 목록조회가 14초인 원인을 코드 레벨에서 재분석했다.

### 문제 1: QueryDSL 연관 엔티티 조건 → User JOIN 발생

```java
.where(notification.user.userId.eq(userId))
```

실제 실행되는 SQL이 이랬다.

```sql
SELECT ... FROM notification n
JOIN user u ON n.user_id = u.user_id   -- 불필요한 JOIN
WHERE u.user_id = ?
```

`user_id`는 `notification` 테이블에 FK 컬럼으로 직접 있는데, 연관 엔티티를 거쳐서 접근하니 User 테이블을 JOIN했다. 네이티브 쿼리로 교체했다.

```java
@Query(value = "SELECT notification_id, content, type, is_read, created_at " +
               "FROM notification WHERE user_id = :userId " +
               "AND (:cursor IS NULL OR notification_id < :cursor) " +
               "ORDER BY notification_id DESC LIMIT :size", nativeQuery = true)
```

`markAllAsReadByUserId`, `countUnreadByUserId`, `markSseSentByIds`도 모두 네이티브로 교체했다.

### 문제 2: findByIdWithFetchJoin — 단건 조회에서 User JOIN

```java
// 기존: User fetch join
notificationRepository.findByIdWithFetchJoin(id);

// 수정: userId 필드 직접 접근
@Column(name = "user_id", insertable = false, updatable = false)
private Long userId;  // Notification 엔티티에 추가

notificationRepository.findById(id);
n.getUserId();  // LAZY 프록시 초기화 없이 직접 접근
```

### 문제 3: SSE 복구 로직을 별도 컴포넌트로 분리 + Semaphore 도입

SSE 복구 로직이 컨트롤러 내부에 있어 책임이 혼재되어 있었다. 별도 `@Component`로 분리하면서 Semaphore도 함께 도입했다.

```java
@Component
@RequiredArgsConstructor
public class SseMissedNotificationRecovery {
    private final Semaphore recoverySemaphore = new Semaphore(30);

    public void recover(Long userId, SseEmitter emitter) { ... }
}
```

```
Semaphore(30)을 넣은 이유:
  Virtual Thread는 생성 비용이 낮지만
  수천 개가 동시에 HikariCP 대기열에 쌓이면 풀 경합이 심해짐
  → 동시 복구를 30개로 제한해 DB 커넥션 경합 억제

부작용:
  tryAcquire()가 false → 복구를 조용히 스킵
  → 서버 재시작 후 대규모 재접속 시 sse_sent=false 레코드가 무음으로 쌓임
  → 이 문제와 제거 과정은 이후 리팩토링 글에서 다룸
```

### 그 외: Redis fallback + connection-timeout 단축

```java
public long getUnreadCount(Long userId) {
    try {
        String cached = redisTemplate.opsForValue().get("unread:" + userId);
        if (cached != null) return Long.parseLong(cached);
    } catch (Exception e) {
        log.warn("Redis unavailable, falling back to DB");
    }
    return notificationRepository.countUnreadByUserId(userId);
}
```

```yaml
hikari:
  connection-timeout: 5000   # 10s → 5s
spring.jpa.properties:
  jakarta.persistence.query.timeout: 5000
```

풀이 포화됐을 때 10초 대기 후 실패보다, 5초에 빠르게 실패하는 게 낫다. 성공 요청의 latency가 줄어들고 실패 요청도 빠르게 자원을 반납한다.

### 결과 — 처리량 +30%, p95 전반 -45%

목록조회 p95는 14,049ms에서 7,689ms로 **45% 감소**, 안읽음 p95는 3,601ms에서 2,393ms로 34% 감소, 단건읽음 p95는 900ms에서 599ms로 33% 감소, 전체읽음 p95는 2,172ms에서 1,075ms로 **50% 감소**했다. 처리량은 8,453 iter에서 10,969 iter로 **30% 증가**했다.

코드 레벨 수정을 모두 적용한 후에도 목록조회 p95가 7초로 남아 있었다.

---

## 10단계 — 데이터 자체가 문제였다 (근본 원인)

EXPLAIN을 재확인했다.

```sql
EXPLAIN SELECT notification_id, content, type, is_read, created_at
FROM notification WHERE user_id = 1 ORDER BY notification_id DESC LIMIT 20;
```

```
type: ref   key: idx_notification_user_id_desc   rows: 17,314,476
```

```
인덱스는 타고 있었다. 문제는 데이터 자체:

  user_id=1에 10,000,000건 (전체 40M의 25%)
  5단계에서 미읽음 수는 줄였지만 행 자체는 40M 그대로

  목록 조회가 느린 이유:
    SELECT content, type ... → 인덱스에 없는 컬럼
    → 인덱스로 위치 찾고 → 테이블 룩업 (랜덤 I/O)
    → user_id=1의 데이터가 15GB에 퍼져있음
    → LIMIT 20이어도 20번의 랜덤 I/O가 전부 디스크
    → 버퍼풀 4GB로는 15GB 테이블을 다 올릴 수 없음

  해결: 시드 데이터를 현실적 규모로 줄이는 것
       실서비스에서 유저당 알림 1만 건이 쌓일 리 없음
```

```sql
-- 유저별 최신 200건만 남기고 나머지 삭제 (10,000 유저 배치 실행)
DELETE FROM notification
WHERE user_id = :userId
  AND notification_id NOT IN (
      SELECT notification_id FROM (
          SELECT notification_id FROM notification
          WHERE user_id = :userId
          ORDER BY notification_id DESC LIMIT 200
      ) t
  );
```

```
삭제 전: 40,273,142행   15,367MB
삭제 후:    470,832행      173MB
```

173MB는 버퍼풀 4GB에 완전히 적재된다. 삭제 후 쿼리 시간을 다시 쟀다.

```
user500 목록조회:  11,238ms → 6.3ms   (1,782배)
unread count:              → 1.1ms
```

### 최종 k6 결과

데이터 정리 후 최종 k6 결과: P2 목록조회 p95는 6,795ms에서 **21ms**(323배), P2 안읽음 p95는 548ms에서 **20ms**(27배), P2 단건읽음 p95는 729ms에서 **13ms**(56배), P2 전체읽음 p95는 1,558ms에서 **16ms**(97배)로 개선됐다. P2 성공률은 52.2%에서 **100%**, P3 혼합 성공률은 72.3%에서 **99.9%**, P4 스파이크 성공률은 50.8%에서 **99.9%**로 올랐다. 총 5xx 에러는 3,355건에서 **0건**으로 완전 해결됐고, 처리량은 10,969 iter에서 **96,163 iter**로 8.6배 증가했다.

---

## 처음과 끝

처음과 끝을 비교하면, API 성공률은 13.6%에서 **99.9%**로, SSE 성공률은 94.2%에서 **100%**로, 목록조회 p95는 59,801ms에서 **21ms**로, 전체읽음 p95는 59,798ms에서 **16ms**로, 5xx 오류는 3,336건에서 **0건**으로, 처리량은 약 11K iter에서 **약 96K iter**로 개선됐다.

---

## 정리하며

**커넥션 풀 고갈의 증상은 다양하다.**
5xx, 타임아웃뿐 아니라 언뜻 무관해 보이는 에러로도 나타날 수 있다. HikariCP의 `(total=N, active=N, idle=0, waiting=N)` 로그가 찍혀 있다면 다른 에러 원인을 찾기 전에 풀 고갈을 먼저 의심해야 한다.

**커넥션 풀을 VU 수에 맞게 늘리면 역효과가 난다.**
HikariCP 300으로 올렸을 때 p95가 8배 악화됐다. MySQL은 동시 커넥션이 늘수록 InnoDB 내부 mutex 경합이 심해진다. 풀 크기는 사용자 수가 아니라 DB의 처리 능력 기준이다.

**QueryDSL 연관 엔티티 조건은 JOIN을 유발한다.**
`notification.user.userId.eq(userId)`는 자연스러워 보이지만 User 테이블을 JOIN한다. FK 컬럼이 직접 있는데도. 네이티브 쿼리나 `@Column(name = "user_id")`로 직접 접근하면 JOIN이 사라진다.

> **인덱스로 해결 안 되는 느린 쿼리는 데이터 양을 의심해라.**
> 인덱스, 버퍼풀, 코드 최적화를 전부 했는데 여전히 7초가 나왔다. 원인은 40M행 비현실적 시드 데이터였다. 인덱스를 타도 SELECT 컬럼이 인덱스에 없으면 테이블 룩업이 발생하고, 그 데이터가 버퍼풀보다 크면 디스크 I/O가 터진다. 유저당 200건으로 줄이자 모든 API가 20ms 이하로 떨어졌다. 부하 테스트는 현실적인 데이터 규모로 해야 의미 있는 결과가 나온다.

---

## 시리즈 탐색

**◀ 이전 글**
[채팅 도메인 — 상관 서브쿼리·N+1이 만든 성능 저하](/chat-subquery-redis-listener/)

**▶ 다음 글**
[피드 도메인 — 부하 테스트 9번 만에 슬로우 쿼리 29,008건에서 전 구간 통과까지](/feed-performance-load-test/)
