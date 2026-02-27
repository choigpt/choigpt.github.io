---
title: Finance 도메인 부하 테스트 — Mock 없이는 보이지 않던 버그
date: 2026-02-27
tags:
  - Java
  - Spring
  - JPA
  - k6
  - 성능최적화
  - 트러블슈팅
  - 결제
  - 동시성
  - 부하테스트
permalink: /finance-domain-load-test-mock-exposed-hidden-bug/
excerpt: Phase 3 파이프라인이 항상 500을 반환하고 있었다. 실제 Toss API가 없어서 Phase 2에서 일찍 실패했기 때문에
  이전에는 드러나지 않았다. MockTossPaymentClient를 주입하고 나서야 clearAutomatically=true로 인한
  detached entity 버그가 나타났다.
---

## 개요

Finance 도메인은 다른 팀원이 설계한 도메인이다. Payment·Settlement·Wallet 세 서브도메인으로 구성되며, 결제 흐름이 3-Phase로 나뉘어 있고, Kafka 기반 정산과 Redis + Pessimistic Lock 기반 지갑 연산이 동시에 움직인다. 전체 프로젝트를 통틀어 처음 부하 테스트를 돌렸을 때부터 지표가 가장 안정적으로 나온 도메인이었다.

성능보다 정확성이 중요한 도메인이라, 부하 테스트의 목적도 "얼마나 빠른가"보다 "몇 VU에서 무너지는가"에 가깝다. 다만 외부 API(Toss)를 포함한 결제 파이프라인을 어떻게 끝까지 테스트할 것인가가 처음부터 방법론적으로 막히는 지점이었다.

---

## 시스템 구조

| 서브도메인 | 핵심 기능 | 동시성 제어 |
|-----------|----------|------------|
| Payment | 토스 3-Phase 결제 (선점→외부API→결과반영) | Semaphore(50) + Redis Gate + CAS |
| Settlement | Kafka 기반 참여자별 병렬 정산 | Outbox Pattern + StructuredTaskScope + Semaphore(32) |
| Wallet | 잔액 충전/홀드/캡처/크레딧 | Redis Lua Gate + Pessimistic Lock + Native UPDATE |

---

## 1차 부하 테스트 — 성공률 97%의 의미

전체 성공률은 97%였다. 숫자만 보면 나쁘지 않다. 문제는 이 97%가 파이프라인의 가장 중요한 부분인 Phase 3를 한 번도 실행하지 않은 채 나온 수치라는 점이다.

### 테스트 설계

| Phase | 시나리오 | VU | 측정 대상 |
|-------|----------|-----|----------|
| P1 | Payment Redis (save→verify) | 200 | Redis SET/GET 레이턴시 |
| P2 | Payment Confirm CAS 경합 | 300 | CAS INSERT IGNORE + Semaphore |
| P3 | 혼합 (Payment+Settlement+Wallet) | 400 | 전체 API 조합 부하 |

### 1차 결과

| 항목 | 성공 | 실패 | 성공률 |
|------|------|------|--------|
| payment save | 64,363 | 5 | 99.99% |
| payment verify | 64,363 | 0 | 100% |
| payment confirm | 76,749 | 27 | 99.96% |
| settlement list | 29,228 | 139 | 99.53% |
| wallet list | 28,624 | 7,891 | **78.4%** |

`wallet list` 성공률이 78.4%였다. p95는 14ms로 빠른데 500 에러가 7,891건 발생한 이유를 추적했다.

원인은 두 가지였다. 첫째, `WalletService.convertToDto()`에서 `Payment`가 null인 경우 NPE가 발생하고 있었다. 아직 처리되지 않은 결제의 지갑 트랜잭션을 조회할 때 결제 내역이 없을 수 있는데 null 체크가 없었다. 둘째, 400VU 고부하 구간에서 HikariCP 커넥션 풀이 소진되는 일시적 5xx였다. NPE를 수정하고 3차 결과에서 `wallet list` 성공률 99.98%로 해결됐다.

---

## Mock이 없었던 문제 — Phase 3가 항상 실패

Toss API는 외부 API라 부하 테스트 환경에서 호출할 수 없다. 처음에는 이 파이프라인을 어떻게 끝까지 테스트해야 할지 방법 자체가 막혔다. Phase 2에서 연결 실패가 나면 Phase 3(Wallet 잔액 반영)에는 도달하지 못했고, 결국 가장 중요한 부분이 테스트 사각지대에 놓여 있었다.

문제는 이 상태에서 전체 성공률 숫자가 그럭저럭 좋아 보인다는 것이었다. Phase 2에서 일찍 실패하면 Phase 3 실패가 카운트되지 않기 때문이다. 결제 파이프라인의 가장 중요한 부분, 즉 지갑 잔액 반영이 전혀 테스트되지 않고 있었다.

---

## MockTossPaymentClient — @Profile("loadtest")

해결책은 `loadtest` 프로필에서만 활성화되는 Mock 구현체를 만드는 것이었다. Feign 클라이언트 대신 항상 성공 응답을 반환하도록 `TossPaymentClient`를 구현했다.

```java
@Profile("loadtest")
@Primary
@Component
public class MockTossPaymentClient implements TossPaymentClient {

    @Override
    public ConfirmTossPayResponse confirmPayment(ConfirmTossPayRequest request) {
        return ConfirmTossPayResponse.builder()
                .paymentKey(request.paymentKey())
                .status("DONE")
                .method("카드")
                .totalAmount(request.amount())
                .build();
    }

    @Override
    public CancelTossPayResponse cancelPayment(String paymentKey, CancelRequest request) {
        return CancelTossPayResponse.builder()
                .paymentKey(paymentKey)
                .status("CANCELED")
                .build();
    }
}
```

`@Primary`로 Feign 빈보다 우선 주입된다. `loadtest` 프로필이 없으면 빈 자체가 생성되지 않으므로 프로덕션에 영향이 없다.

```bash
--spring.profiles.active=local,loadtest
```

Phase 1→2→3 전체 파이프라인이 처음으로 실제로 실행됐다.

---

## 프로덕션 버그 발견 — detached entity passed to persist

Mock을 주입하자마자 Phase 3가 거의 전부 실패하기 시작했다.

```
Phase 3 failed for orderId=order_187_333_1772089194726. Initiating Toss cancel compensation.
Caused by: org.hibernate.PersistentObjectException: detached entity passed to persist: Payment
```

이전에는 Toss API가 실패해서 Phase 3에 도달하지 못했기 때문에 완전히 숨어 있었다.

### 원인

`PaymentTransactionService.applyPaymentResult()` 안에서 문제가 되는 실행 순서는 다음과 같았다.

```java
// 문제가 있었던 순서
1. payment = paymentRepository.findById(orderId)   // managed 상태
2. walletRepository.creditByUserId(userId, amount) // @Modifying(clearAutomatically = true)
                                                   // → 영속성 컨텍스트 클리어
                                                   // → payment가 detached로 전환
3. walletTransaction = WalletTransaction.builder()
        .payment(payment)  // detached payment 연결
        .build();
4. walletTransactionRepository.save(walletTransaction)
   // cascade PERSIST → detached entity 에러
```

`creditByUserId`에 `@Modifying(clearAutomatically = true)`가 설정돼 있어서, 호출 후 영속성 컨텍스트가 클리어됐다. 이미 로드된 `payment`가 detached 상태가 되면서 `walletTransaction`에 연결할 때 오류가 발생했다.

### 수정

수정 방향은 두 가지를 고려했다. `clearAutomatically = false`로 바꾸는 방법과, `payment` 조회 순서를 `creditByUserId` 이후로 이동하는 방법이었다. 일단 후자를 적용했고, 전자는 다른 호출부에 미치는 영향을 파악한 뒤 별도로 검토하기로 했다.

```java
// 수정 후
1. walletRepository.creditByUserId(userId, amount)  // 영속성 컨텍스트 클리어
2. payment = paymentRepository.findById(orderId)    // 클리어 이후 조회 → managed 상태
3. walletTransaction = WalletTransaction.builder()
        .payment(payment)  // managed payment 연결
        .build();
```

---

## 수정 후 결과

| 항목 | 수정 전 | 수정 후 |
|------|---------|---------|
| `http_req_failed` | 10.89% (75,851건) | **0.03%** (249건) |
| `payment_confirm` p95 | 327ms (Phase 3 실패) | **187ms** (전체 성공) |
| check 성공률 | 99.97% | 99.95% |
| 5xx 에러율 | 0% | 0% |

`payment_confirm` p95가 327ms → 187ms로 줄어든 건 성능 개선이 아니다. 이전엔 Phase 3에서 바로 실패해서 빠르게 끝났지만, 이제는 Wallet UPDATE까지 전체 파이프라인이 실행되기 때문이다.

---

## 데이터 스케일업 후 재검증

| 테이블 | 기존 | 스케일업 | 배수 |
|--------|------|---------|------|
| schedule | 200 | 5,000 | 25x |
| settlement | 200 | 15,000 | 75x |
| user_settlement | ~2,000 | ~150,000 | 75x |
| wallet_transaction | 600 | ~433,000 | 720x |

| 메트릭 | 소규모 | 대규모 | 변화 |
|--------|--------|--------|------|
| payment_confirm p95 | 187ms | 105ms | -44% (payment 레코드 정리 효과) |
| settlement_list p95 | 45ms | 56ms | +25% (데이터 75배) |
| wallet_list p95 | 221ms | 278ms | +26% (데이터 720배 대비 양호) |
| check 성공률 | 99.95% | 99.98% | 동등 |
| Thresholds | 8/8 | 8/8 | |

`wallet_transaction`이 720배 늘었는데 p95가 25%만 증가했다. `(user_id, created_at)` 인덱스가 잘 동작하고 있다는 신호다.

---

## 발견 및 수정한 버그

| # | 버그 | 원인 | 수정 |
|---|------|------|------|
| 1 | `WalletService.convertToDto()` NPE | `Payment` null 케이스 미처리 | null 체크 추가 |
| 2 | `PaymentTransactionService` detached entity | `creditByUserId`의 `clearAutomatically=true`로 영속성 컨텍스트 클리어 후 payment detached | `payment` 조회를 `creditByUserId` 이후로 이동 |
| 3 | `settlement list` 성공률 0% | k6 스크립트의 `SCHEDULE_ID_START` 범위가 실제 시드 데이터와 맞지 않아 전체 요청이 404 처리 | `SCHEDULE_ID_START` 상수를 시드 데이터 범위에 맞게 수정 |

버그 2번이 핵심이었다. 실제 Toss API가 성공했어도 Phase 3에서 항상 500을 반환했을 것이다. Mock을 도입해 전체 파이프라인을 실제로 실행하기 전까지 완전히 숨어 있었다.

---

## 정리하며

`@Modifying(clearAutomatically = true)`는 영속성 컨텍스트를 클리어한다. 이 쿼리 이전에 로드된 모든 엔티티가 detached 상태가 된다. 같은 트랜잭션 안에서 벌크 업데이트를 사용할 때는 항상 조회 순서를 의식해야 한다. 이번 버그가 운영 중에 드러나지 않았던 건 외부 API 장벽 덕분이었는데, 뒤집어 보면 Mock 없이는 파이프라인 끝까지 테스트했다고 말하기 어렵다는 뜻이기도 하다.

> **외부 API 의존이 있는 파이프라인에서 Mock 없이 통합 테스트를 했다고 말하기 어렵다.**
> Phase 2까지만 실행되는 테스트는 Phase 3의 버그를 숨긴다. 결제 파이프라인의 최종 단계인 지갑 잔액 반영이 항상 실패하는 버그가 운영 중에 터졌다면 자금 유실로 이어질 수 있었다.

---

## 시리즈 탐색

**◀ 이전 글**  
[피드 도메인 리팩토링 — 442줄 서비스를 4개로 쪼갠 기록](/feed-domain-442-line-service-split/)
