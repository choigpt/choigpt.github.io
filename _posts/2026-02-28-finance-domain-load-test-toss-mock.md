---
title: Finance 도메인 부하 테스트 — Mock 도입 후 발견된 버그
date: 2026-02-28
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
excerpt: Phase 3 파이프라인이 항상 500을 반환하고 있었다. 실제 Toss API가 없어서 Phase 2에서 실패했기 때문에
  이전에는 확인되지 않았다. MockTossPaymentClient를 주입한 후 clearAutomatically=true로 인한
  detached entity 버그가 발견됐다.
---

## 개요

Finance 도메인은 다른 팀원이 설계한 도메인이다. Payment·Settlement·Wallet 세 서브도메인으로 구성되며, 결제 흐름이 3-Phase로 나뉘어 있고, Kafka 기반 정산과 Redis + Pessimistic Lock 기반 지갑 연산이 동시에 움직인다. 전체 프로젝트에서 초기 부하 테스트 시점부터 지표가 가장 안정적이었던 도메인이다.

성능보다 정확성이 중요한 도메인이라, 부하 테스트의 목적도 "얼마나 빠른가"보다 "몇 VU에서 무너지는가"에 가깝다. 다만 외부 API(Toss)를 포함한 결제 파이프라인의 End-to-End 테스트 방법이 초기부터 과제였다.

---

## 시스템 구조

Finance 도메인은 세 개의 서브도메인으로 구성된다. **Payment**는 토스 3-Phase 결제(선점 → 외부 API → 결과 반영)를 담당하며, Semaphore(50) + Redis Gate + CAS로 동시성을 제어한다. **Settlement**는 Kafka 기반 참여자별 병렬 정산을 처리하고, Outbox Pattern + StructuredTaskScope + Semaphore(32)를 사용한다. **Wallet**은 잔액 충전/홀드/캡처/크레딧 연산을 수행하며, Redis Lua Gate + Pessimistic Lock + Native UPDATE로 동시성을 제어한다.

---

## 1차 부하 테스트 — 성공률 97%의 의미

전체 성공률은 97%였다. 그러나 이 97%는 파이프라인에서 가장 중요한 Phase 3를 한 번도 실행하지 않은 상태에서 산출된 수치다.

### 테스트 설계

테스트는 3단계로 구성했다. P1에서는 200 VU로 Payment Redis(save → verify)를 실행하여 Redis SET/GET 레이턴시를 측정했다. P2에서는 300 VU로 Payment Confirm CAS 경합 시나리오를 실행하여 CAS INSERT IGNORE + Semaphore를 검증했다. P3에서는 400 VU로 Payment + Settlement + Wallet 혼합 시나리오를 실행하여 전체 API 조합 부하를 측정했다.

### 1차 결과

1차 결과를 보면, payment save는 64,363건 성공 / 5건 실패(99.99%), payment verify는 64,363건 전량 성공(100%), payment confirm은 76,749건 성공 / 27건 실패(99.96%), settlement list는 29,228건 성공 / 139건 실패(99.53%)로 양호했다. 그러나 **wallet list는 28,624건 성공 / 7,891건 실패(78.4%)** 로 유일하게 문제가 있었다.

`wallet list` 성공률이 78.4%였다. p95는 14ms로 응답 속도는 정상이었으나 500 에러가 7,891건 발생했다. 원인을 추적했다.

원인은 두 가지였다. 첫째, `WalletService.convertToDto()`에서 `Payment`가 null인 경우 NPE가 발생하고 있었다. 아직 처리되지 않은 결제의 지갑 트랜잭션을 조회할 때 결제 내역이 없을 수 있는데 null 체크가 없었다. 둘째, 400VU 고부하 구간에서 HikariCP 커넥션 풀이 소진되는 일시적 5xx였다. NPE를 수정하고 3차 결과에서 `wallet list` 성공률 99.98%로 해결됐다.

---

## Mock이 없었던 문제 — Phase 3가 항상 실패

Toss API는 외부 API이므로 부하 테스트 환경에서 호출할 수 없다. Phase 2에서 연결 실패가 발생하면 Phase 3(Wallet 잔액 반영)에 도달하지 못하며, 파이프라인의 가장 중요한 부분이 테스트 범위 밖에 있었다.

이 상태에서 전체 성공률은 양호한 수치로 표시됐다. Phase 2에서 조기 실패하면 Phase 3 실패가 집계되지 않기 때문이다. 결제 파이프라인의 최종 단계인 지갑 잔액 반영이 테스트되지 않고 있었다.

---

## MockTossPaymentClient — @Profile("loadtest")

`loadtest` 프로필에서만 활성화되는 Mock 구현체를 작성했다. Feign 클라이언트 대신 항상 성공 응답을 반환하도록 `TossPaymentClient`를 구현했다.

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

Mock을 주입한 후 Phase 3의 거의 전체가 실패했다.

```
Phase 3 failed for orderId=order_187_333_1772089194726. Initiating Toss cancel compensation.
Caused by: org.hibernate.PersistentObjectException: detached entity passed to persist: Payment
```

이전에는 Toss API가 실패하여 Phase 3에 도달하지 못했기 때문에 발견되지 않았다.

### 원인

`PaymentTransactionService.applyPaymentResult()` 안에서 문제가 되는 실행 순서는 다음과 같았다.

```java
// 문제가 발생한 순서
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

`creditByUserId`는 [지갑·결제 편](/wallet-payment-toss-db-failure/)에서 Java 덧셈 → native query로 전환하면서 내가 추가한 메서드다. 원본의 다른 native query(`holdBalanceIfEnough`, `captureHold` 등)에 `clearAutomatically = true`가 붙어있어서 같은 패턴으로 작성했는데, 기존 메서드들은 트랜잭션 안에서 이후에 managed 엔티티를 참조하는 경우가 없었기 때문에 문제가 없었다. `creditByUserId`는 호출 이전에 로드된 `payment`를 이후에도 사용하는 구조여서 detached entity 문제가 발생한 것이다.

### 수정

수정 방향은 두 가지를 고려했다. `clearAutomatically = false`로 변경하는 방법과, `payment` 조회 순서를 `creditByUserId` 이후로 이동하는 방법이다. 후자를 적용했고, 전자는 다른 호출부에 대한 영향 분석 후 별도 검토 예정이다.

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

수정 후 `http_req_failed`는 10.89%(75,851건)에서 **0.03%**(249건)으로 대폭 감소했다. `payment_confirm` p95는 327ms(Phase 3 실패 상태)에서 **187ms**(전체 성공)로 개선됐다. check 성공률은 99.97%에서 99.95%로 동등 수준이며, 5xx 에러율은 수정 전후 모두 0%였다.

`payment_confirm` p95가 327ms → 187ms로 감소한 것은 성능 개선이 아니다. 이전에는 Phase 3에서 즉시 실패하여 빠르게 종료됐지만, 수정 후에는 Wallet UPDATE까지 전체 파이프라인이 실행된다.

---

## 데이터 스케일업 후 재검증

데이터를 대폭 스케일업했다. schedule은 200건에서 5,000건(25배), settlement는 200건에서 15,000건(75배), user_settlement는 약 2,000건에서 약 150,000건(75배), wallet_transaction은 600건에서 약 433,000건(720배)으로 증가시켰다.

스케일업 후 결과를 보면, payment_confirm p95는 187ms에서 105ms로 44% 감소했는데 이는 payment 레코드 정리 효과다. settlement_list p95는 45ms에서 56ms로 25% 증가(데이터 75배 대비 양호)했고, wallet_list p95는 221ms에서 278ms로 26% 증가(데이터 720배 대비 양호)했다. check 성공률은 99.95%에서 99.98%로 동등 수준이며, Thresholds는 소규모/대규모 모두 8/8 통과했다.

`wallet_transaction`이 720배 늘었는데 p95가 25%만 증가했다. `(user_id, created_at)` 인덱스가 잘 동작하고 있다는 신호다.

---

## 발견 및 수정한 버그

발견 및 수정한 버그는 세 가지다. 첫째, `WalletService.convertToDto()`에서 `Payment`가 null인 케이스를 미처리하여 NPE가 발생했으며, null 체크를 추가해 수정했다. 둘째, `PaymentTransactionService`에서 `creditByUserId`의 `clearAutomatically=true`로 영속성 컨텍스트가 클리어된 후 payment가 detached 상태가 되는 문제가 있었으며, `payment` 조회를 `creditByUserId` 이후로 이동하여 해결했다. 셋째, `settlement list` 성공률이 0%였는데, k6 스크립트의 `SCHEDULE_ID_START` 범위가 실제 시드 데이터와 맞지 않아 전체 요청이 404 처리되고 있었으며, 해당 상수를 시드 데이터 범위에 맞게 수정했다.

버그 2번이 가장 심각했다. 실제 Toss API가 성공하더라도 Phase 3에서 항상 500을 반환하는 구조였다. Mock을 도입하여 전체 파이프라인을 실행하기 전까지 발견되지 않았다.

---

## 정리하며

`@Modifying(clearAutomatically = true)`는 영속성 컨텍스트를 클리어한다. 이 쿼리 이전에 로드된 모든 엔티티가 detached 상태가 된다. 같은 트랜잭션 안에서 벌크 업데이트를 사용할 때는 항상 조회 순서를 의식해야 한다. 이 버그가 운영 중에 발견되지 않은 이유는 외부 API 장벽 때문이다. 이는 Mock 없이는 파이프라인 End-to-End 테스트가 불완전하다는 것을 의미한다.

> **외부 API 의존이 있는 파이프라인에서 Mock 없이 통합 테스트를 했다고 말하기 어렵다.**
> Phase 2까지만 실행되는 테스트는 Phase 3의 버그를 숨긴다. 결제 파이프라인의 최종 단계인 지갑 잔액 반영이 항상 실패하는 버그가 운영 환경에서 발생했다면 자금 유실로 이어질 수 있었다.

---

## 시리즈 탐색

**◀ 이전 글**
[피드 도메인 리팩토링 — 서비스 책임 분리와 네이티브 쿼리 전환](/feed-domain-442-line-service-split/)

**▶ 다음 글**
[SSE 알림 부하 테스트 — 7가지 시나리오로 찾은 연결 한계와 잔여 병목](/sse-notification-load-test-analysis/)
