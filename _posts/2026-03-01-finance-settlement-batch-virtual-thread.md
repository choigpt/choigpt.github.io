---
title: Finance 도메인 — 정산 p95 19.5초를 2.46초로, 배치 UPDATE와 Virtual Thread 병렬 Fallback
date: 2026-03-01
tags: [Java, Spring, JPA, k6, Kafka, VirtualThread, 성능최적화, 트러블슈팅, 결제, 정산, 동시성, 부하테스트]
permalink: /finance-settlement-batch-virtual-thread/
excerpt: "정산 p95가 19.5초였다. 배치 UPDATE가 부분 실패하면 10명을 순차로 재처리하는 fallback이 문제였다. Virtual Thread 병렬 실행으로 교체하고 Kafka 비재시도 예외를 등록한 뒤 2.46초로 떨어졌다."
---

## 개요

기존 Finance 부하 테스트에서 확인되지 않은 병목이 있었다. 이전 테스트에서는 정산 데이터가 소규모이고 VU도 낮아 발견되지 않았다. 이번에는 정산 데이터를 500건으로 늘리고 VU를 1,000까지 올려 전체 파이프라인을 다시 검증했다.

---

## 초기 확인 — Semaphore 50 제거

첫 번째 확인 대상은 `PaymentService`에 있는 `Semaphore(50)` 제한이었다.

```java
// PaymentService.java
private static final int MAX_CONCURRENT_CLAIMS = 50;
private final Semaphore claimSemaphore = new Semaphore(MAX_CONCURRENT_CLAIMS, true);
// "60개 이상의 동시 INSERT가 InnoDB에 진입하면 gap/row lock 경합이 기하급수적으로 증가"
```

주석에 근거가 기술돼 있으나 실제 코드 구조를 분석하면 불필요하다.

1. `INSERT IGNORE + unique 제약` — gap lock이 발생하지 않는다
2. `READ_COMMITTED` 격리 수준 — gap lock 자체가 없다
3. `Redis gate (SET NX)` — 동일 orderId 중복은 DB 도달 전에 이미 차단된다
4. 단일 JVM 한정 — 멀티 인스턴스 환경에서는 Semaphore가 의미 없다

51번째 요청부터 1초 대기 후 reject되는 인위적 병목이었다. Semaphore를 제거했다.

---

## 버그 수정 — ClassCastException

`SettlementRepository.existsScheduleInClub`에서 MySQL이 `Long`을 반환하는데 `boolean`으로 직접 캐스팅하고 있었다.

```java
// 수정 전
@Query(value = "SELECT COUNT(*) > 0 FROM settlement ...", nativeQuery = true)
boolean existsScheduleInClub(@Param("scheduleId") Long scheduleId);
// → MySQL에서 Long 반환 → boolean 캐스팅 실패 → ClassCastException

// 수정 후
@Query(value = "SELECT COUNT(*) FROM settlement ...", nativeQuery = true)
long existsScheduleInClub(@Param("scheduleId") Long scheduleId);
// 서비스에서 == 0 비교
```

---

## 첫 번째 전체 테스트 결과 (12/12 PASS)

Phase를 5단계로 구성했다.

| Phase | 시나리오 | VU | 결과 |
|-------|----------|-----|------|
| 1 | 결제 3-Phase 플로우 | 100 | 100% 성공, p95 438ms |
| 2 | 결제 멱등성 (동일 orderId) | 50 | Redis gate + CAS 이중 차단 검증 |
| 3 | 정산 E2E (Outbox→Kafka→완료) | 20건 | p95 55ms, Outbox→Kafka→완료 31초 |
| 4 | 고부하 혼합 | 300 | 99.98%, 5xx 0건, p95 796ms |
| 5 | 최종 검증 | - | 5/5 PASS |

12개 threshold 전부 통과했다. 여기서 VU를 2배 이상으로 늘려 재검증했다.

---

## 스케일업 후 유일한 병목 — 정산 대량 요청 p95 19.5초

VU를 전반적으로 두 배 이상 늘렸다. 결제(400 VU), 멱등성(500 VU), 정산(500 VU), 스파이크(1,000 VU).

21개 threshold 중 1개만 실패했다.

```
settle_request_duration p95 = 19.5s (threshold: < 5s) — FAIL
```

나머지 20개는 전부 통과, 5xx 0건, 성공률 99.9%+.

### 원인 분석

정산은 세 가지 문제가 복합적으로 작용했다.

| # | 원인 | 영향 |
|---|------|------|
| 1 | `batchCaptureHold` all-or-nothing: 10명 중 1명이라도 잔액 부족이면 전체 롤백 | 거의 항상 fallback 진입 |
| 2 | fallback이 순차 처리 (`for loop + REQUIRES_NEW × 10명`) | 10명 × 2~3s = 20~30s |
| 3 | Kafka `FixedBackOff(5s × 3회)`: `CustomException`에도 재시도 | 비즈니스 예외가 누출 시 +15s |

---

## 최적화 1 — 배치 UPDATE 도입

기존 구조는 참가자 1명 = 1개 `REQUIRES_NEW` 트랜잭션이었다. 100건 정산 × 10명 = 1,000개 독립 트랜잭션이 동시에 발생한다.

```
기존: SELECT × 3 + UPDATE(captureHold) + UPDATE(markCompleted) + INSERT(outbox) × 1,000회
개선: IN절 UPDATE × 100회
```

두 개의 배치 쿼리를 추가했다.

```java
// WalletRepository.java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query(value = """
  UPDATE wallet
     SET posted_balance = posted_balance - :amount,
         pending_out    = pending_out - :amount
   WHERE user_id IN (:userIds)
     AND pending_out    >= :amount
     AND posted_balance >= :amount
""", nativeQuery = true)
int batchCaptureHold(@Param("userIds") List<Long> userIds, @Param("amount") long amount);
```

```java
// UserSettlementRepository.java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query(value = """
    UPDATE user_settlement
       SET status = 'COMPLETED', completed_time = :now
     WHERE settlement_id = :settlementId
       AND user_id IN (:userIds)
       AND status = 'HOLD_ACTIVE'
""", nativeQuery = true)
int batchMarkCompleted(@Param("settlementId") Long settlementId,
                       @Param("userIds") List<Long> userIds,
                       @Param("now") java.time.LocalDateTime now);
```

| 항목 | 기존 (개별) | 개선 (배치) |
|------|-------------|-------------|
| 트랜잭션 수 | 1,000개 | 100개 |
| `captureHold` 실행 | UPDATE × 1,000회 | IN절 UPDATE × 100회 |
| `markCompleted` 실행 | UPDATE × 1,000회 | IN절 UPDATE × 100회 |
| Redis gate | 1,000회 acquire/release | 0회 |
| SELECT 쿼리 | 3,000회 | 0회 |

배치 실패 시 기존 개별 처리로 자동 fallback하는 안전망을 유지했다.

---

## 최적화 2 — Virtual Thread 병렬 Fallback

배치 UPDATE는 all-or-nothing이라 참가자 중 1명이라도 잔액이 부족하면 전체 롤백 후 fallback 경로로 들어간다. 기존 fallback은 순차 `for loop`였다.

```java
// 수정 전 — 순차 처리
for (Long participantId : targetUserIds) {
    processIndividual(participantId, settlement, amount);
}
// 10명 × 2~3s = 20~30s
```

```java
// 수정 후 — Virtual Thread 병렬 처리
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<Void>> futures = targetUserIds.stream()
            .map(participantId -> executor.submit(() -> {
                processIndividual(participantId, settlement, amount);
                return null;
            }))
            .toList();
    for (Future<Void> f : futures) {
        f.get(10, TimeUnit.SECONDS);
    }
}
// 가장 느린 1명 기준 = 2~3s
```

Java 21 Virtual Thread는 I/O 바운드 작업에 적합하다. DB 호출 중 스레드가 블로킹되면 carrier thread를 반환하고, 응답이 오면 다시 마운트된다. 별도 스레드풀 설정 없이 참가자를 병렬로 처리할 수 있다.

---

## 최적화 3 — Redis 설정 하드코딩 제거

`RedisConfig.java`에서 YAML 설정을 덮어쓰는 하드코딩이 있었다.

```java
// 수정 전 — yml의 max-active: 256을 무시하고 128로 덮어씀
poolConfig.setMaxTotal(128);
// commandTimeout: 15초 — 500 VU 스파이크 시 Tomcat 스레드 15초간 블로킹
```

```yaml
# 수정 후 — local 프로필
spring.data.redis.lettuce.pool:
  max-active: 256
  max-idle: 128
  min-idle: 32
  max-wait: 3s
lettuce.command-timeout: 5s  # 15s → 5s (빠른 실패)
```

`commandTimeout`을 15초에서 5초로 줄였다. Redis가 응답하지 않으면 15초간 스레드가 블로킹되면서 Tomcat 스레드 풀이 고갈되는 문제를 차단한다.

---

## 최적화 4 — Kafka ErrorHandler 비재시도 예외 등록

```java
// KafkaErrorConfig.java 수정 전
FixedBackOff backOff = new FixedBackOff(5_000L, 3);
// CustomException도 5초 × 3회 재시도 → 잔액 부족 같은 비즈니스 예외에도 +15s

// 수정 후
FixedBackOff backOff = new FixedBackOff(1_000L, 2);
handler.addNotRetryableExceptions(CustomException.class);
// 비즈니스 예외는 즉시 DLT 전송, 트랜지언트 에러만 1s × 2회 재시도
```

---

## 최종 결과 — 21/21 PASS

| 지표 | 최적화 전 | 최적화 후 |
|------|----------|----------|
| `settle_request p95` | 19.5s | **2.46s** |
| 개선율 | — | **7.9배** |
| 전체 threshold | 20/21 | **21/21** |

| Phase | 메트릭 | 결과 |
|-------|--------|------|
| P2 결제 폭풍 (400 VU) | pay_confirm p95 | 1.34s (< 3s) |
| P3 멱등성 (500 VU) | idemp_correct | 100% (> 95%) |
| P4 정산 대량 (500 VU) | settle_request p95 | 2.46s (< 5s) ✓ |
| P5 정산 조회 (300 VU) | settle_query p95 | 457ms (< 3s) |
| P6 지갑 조회 (300 VU) | wallet_query p95 | 1.98s (< 3s) |
| P7 복합 고부하 (600 VU) | stress p95 | 1.47s (< 5s) |
| P8 스파이크 (1,000 VU) | spike_success | 99.93% (> 80%) |
| P9 이중 스파이크 (800 VU) | dbl_spike_success | 99.97% (> 80%) |
| P10 내구 Soak (400 VU) | soak p95 | 1.08s (< 3s) |
| P11 최종 검증 | verify_pass | 100% |

전체 지표: 972K HTTP 요청, 979 req/s, 5xx 0건, HTTP 실패율 0.03%.

---

## 수정 및 발견 내역

| # | 항목 | 변경 |
|---|------|------|
| 1 | `PaymentService` Semaphore(50) | 제거 — Redis gate + READ_COMMITTED로 이미 처리됨 |
| 2 | `existsScheduleInClub` ClassCastException | `boolean` → `long`, 서비스에서 `== 0` 비교 |
| 3 | `WalletRepository.batchCaptureHold` | 신규 — IN절 배치 차감 |
| 4 | `UserSettlementRepository.batchMarkCompleted` | 신규 — IN절 배치 상태 변경 |
| 5 | `SettlementKafkaEventListener` fallback | 순차 for loop → Virtual Thread 병렬 실행 |
| 6 | `KafkaErrorConfig` | `FixedBackOff(5s×3)` → `(1s×2)` + `CustomException` 비재시도 등록 |
| 7 | `RedisConfig` 하드코딩 | 제거, YAML 설정 연동, `commandTimeout` 15s → 5s |

---

## 정리하며

정산 p95 19.5초는 단일 원인이 아니다. 배치 all-or-nothing 실패, 순차 fallback, Kafka 불필요 재시도가 복합적으로 작용한 결과다.

fallback에 Virtual Thread를 적용한 이유는 다음과 같다. 순차 처리를 병렬로 전환하되, I/O 바운드 작업이므로 별도 스레드풀 없이 JVM이 관리하도록 했다. Java 21의 Virtual Thread가 이를 지원한다.

`clearAutomatically = true`가 붙은 `@Modifying` 쿼리 이후에 같은 트랜잭션에서 엔티티를 다시 로드해야 하는 것처럼, 배치 UPDATE를 쓸 때는 영속성 컨텍스트 상태를 의식해야 한다. 이번에는 배치 UPDATE 이후 추가 로드가 없는 구조라 `clearAutomatically`로 충분했다.

> **p95가 평균보다 현저히 높을 때는 꼬리 경로(fallback, 재시도)를 우선 확인해야 한다.**
> 평균이 2초인데 p95가 19초라면, 대부분의 요청은 정상이나 특정 경로에서 지연이 발생한다는 의미다. 이 경우 해당 경로는 배치 실패 후 순차 fallback이었다.

---

## 시리즈 탐색

**◀ 이전 글**
[SSE 알림 부하 테스트 — 7가지 시나리오로 찾은 연결 한계와 잔여 병목](/sse-notification-load-test-analysis/)

**▶ 다음 글**
[피드 도메인 고부하 테스트 — FORCE INDEX 실패부터 인메모리 캐싱까지](/feed-highload-force-index-caching-evolution/)
