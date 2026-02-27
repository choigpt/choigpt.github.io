---
title: 지갑·결제 도메인 — 토스 승인 후 DB가 죽으면 자금이 유실될 수 있다
date: 2026-02-23
tags: [결제, 동시성, JPA, 트러블슈팅, Java, 지갑, 토스페이먼츠]
permalink: /wallet-payment-toss-db-failure/
excerpt: "토스페이먼츠 승인이 완료된 뒤 지갑 업데이트에서 예외가 발생하면, DB는 롤백되지만 이미 승인된 결제는 취소되지 않아 자금이 유실될 수 있었다."
---

## 개요

결제 코드를 살펴보다가 가장 먼저 눈에 들어온 건 `confirm()` 안의 순서였다. 토스페이먼츠에 결제 승인을 요청하고 난 뒤, 지갑 잔액을 업데이트하는 코드에서 예외가 발생하면 어떻게 될까. DB는 롤백된다. 하지만 토스에서 이미 승인된 결제는 취소되지 않는다. 사용자 카드에서 돈이 빠졌는데 지갑에는 충전이 안 된 채로 남는다.

그리고 잔액 업데이트 방식에도 문제가 있었다. 정산 프로세스는 `captureHold` native query로 DB를 직접 수정하는데, 충전 경로는 JPA 1차 캐시에서 읽은 값에 Java로 더한 뒤 `SET`으로 덮어쓴다. 두 경로가 동시에 실행되면 정산에서 차감한 금액이 충전 `SET`에 덮어써져 증발한다.

---

## 시스템 구조

### 충전 흐름

```
클라이언트 → POST /payments/save
    서버가 orderId 기준으로 충전 예정 금액을 Redis에 저장
    ↓
클라이언트 → POST /payments/success
    Redis에서 서버 저장 금액 조회 → 클라이언트 전달 amount와 비교
    ↓
클라이언트 → POST /payments/confirm
    토스페이먼츠 결제 승인 API 호출 (외부) ← 이 시점에 실제 과금
    ↓
    creditByUserId() native query로 잔액 증가
    ↓
    WalletTransaction 기록
```

### 잔액 변경 경로가 두 가지 (수정 전)

```
충전(confirm):   JPA 엔티티로 읽기 → Java 덧셈 → dirty checking → SET balance = ?
정산/홀드:       native query → SET balance = balance + ? / balance = balance - ?
```

두 경로가 같은 컬럼을 다른 방식으로 수정하고 있었다.

---

## 버그 상세

### 토스 승인 후 DB 실패 — 결제는 완료됐지만 지갑에는 반영되지 않은 상태

```java
// PaymentService.java
public ConfirmTossPayResponse confirm(ConfirmTossPayRequest req) {
    Payment payment = claimPayment(req.getOrderId(), req.getAmount());

    final ConfirmTossPayResponse response;
    try {
        response = tossPaymentClient.confirmPayment(req);  // ① 외부 API — 결제 승인
    } catch (...) {
        reportFail(req);
        throw ...;
    }

    // ② 여기서 예외 발생하면?
    wallet.updateBalance(wallet.getPostedBalance() + amount);  // DB 예외 가능
    // ...
}
```

①에서 토스페이먼츠 승인이 완료된 시점에 사용자 카드에서 돈이 빠진다. ②에서 예외가 발생하면 `@Transactional`이 롤백을 시도한다. 지갑 잔액 업데이트는 롤백되지만, 이미 승인된 토스 결제는 취소 API를 별도로 호출해야만 되돌릴 수 있다. 그런데 `confirm()` 안에 결제 취소 호출 코드가 없었다. 실제로 `cancelPayment` 메서드가 주석 처리된 채로 남아 있었다.

이 상황에서 사용자는 카드 청구서에는 결제 내역이 찍히는데 지갑에는 충전이 안 되는 경험을 하게 된다.

---

### 잔액 Race Condition — 정산 차감분이 충전으로 덮어써짐

```java
// PaymentService.java
Wallet wallet = walletRepository.findByUser(user);  // PESSIMISTIC_WRITE
wallet.updateBalance(wallet.getPostedBalance() + amount);  // Java 레벨 덧셈

// Wallet.java
public void updateBalance(Long balance) {
    this.postedBalance = balance;  // SET posted_balance = ?
}
```

`findByUser()`로 읽은 `postedBalance`는 JPA 1차 캐시의 스냅샷이다. 이 사이에 정산 프로세스가 `captureHold` native query로 `posted_balance`를 차감하면, `confirm()`의 `wallet.getPostedBalance()`는 차감 전 값을 반환한다. 그 값에 충전액을 더해 `SET`으로 덮어쓰면 정산에서 빠진 금액이 복원된다.

```
잔액 10,000원
→ 정산에서 captureHold(3,000): DB → 7,000원
→ 동시에 confirm(5,000): 1차 캐시 값 10,000 + 5,000 = 15,000으로 SET
→ 정산 차감분 3,000원 증발
```

반면 정산·홀드 경로는 `SET posted_balance = posted_balance + :amount` 형태의 native query를 쓴다. DB가 현재 값을 읽고 더하는 단일 연산이라 동시성 문제가 없다. 충전 경로만 다른 방식을 쓰고 있었다.

`findByUser()`에 `PESSIMISTIC_WRITE`가 걸려 있었는데, 이것이 Race Condition을 막으려는 의도였는지 다른 이유가 있었는지 원래 코드의 의도는 파악하기 어렵다. 다만 외부 API 호출을 포함하는 `@Transactional` 메서드 안에서 행 락을 잡으면, 토스 응답을 기다리는 수 초 동안 해당 지갑 행이 잠긴 채로 유지된다. 다른 요청이 같은 지갑에 접근하면 그 시간만큼 대기하게 되므로, 잔액 업데이트를 native query로 통일하면서 락도 함께 제거했다.

---

### `@Transactional` 범위 내 외부 API 호출 — 커넥션 점유

```java
// PaymentService.java
@Transactional
public ConfirmTossPayResponse confirm(ConfirmTossPayRequest req) {
    // ... DB 작업
    response = tossPaymentClient.confirmPayment(req);  // 외부 API (수 초 소요)
    // 트랜잭션이 살아있는 동안 DB 커넥션이 점유됨
    // ...
}
```

행 락은 제거됐지만, `@Transactional` 메서드 전체에 걸쳐 DB 커넥션이 점유되는 문제는 남아있다. 토스 API 응답이 오는 동안 커넥션이 반납되지 않는다. 결제 서비스 특성상 트랜잭션 경계를 완전히 분리하기 어려워 현재는 HikariCP 타임아웃과 커넥션 풀 크기 조정으로 수용했다. 토스 API 호출 전 DB 작업과 호출 후 DB 작업을 각각 별도 트랜잭션으로 명시적으로 분리하는 방향은 추후 리팩토링 대상이다.

---

### 결제 금액 서버 검증 없음 — 클라이언트가 금액을 결정

```java
// PaymentService.java (수정 전)
public void savePaymentInfo(SavePaymentRequestDto dto, HttpSession session) {
    String redisKey = REDIS_PAYMENT_KEY_PREFIX + dto.getOrderId();
    redisTemplate.opsForValue().set(redisKey, dto.getAmount(), TTL, TimeUnit.SECONDS);
}
```

`/payments/save`에서 클라이언트가 전달한 `amount`를 그대로 Redis에 저장하고, `/payments/success`에서 다시 클라이언트가 전달한 값과 비교한다. 비교의 양쪽이 모두 클라이언트 입력이라 의미 있는 검증이 아니다. 서버가 "이 orderId는 얼마짜리 충전이어야 한다"를 독립적으로 결정하는 로직이 없었다.

---

### `reportFail()` self-invocation — 실패 기록이 사라짐

```java
// PaymentService.java
try {
    response = tossPaymentClient.confirmPayment(req);
} catch (FeignException.BadRequest e) {
    reportFail(req);    // ← 같은 클래스 메서드 직접 호출
    throw new CustomException(ErrorCode.INVALID_PAYMENT_INFO);
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void reportFail(ConfirmTossPayRequest req) { ... }
```

`reportFail()`에 `Propagation.REQUIRES_NEW`가 붙어 있지만, `confirm()` 안에서 `this.reportFail()`로 직접 호출하면 Spring AOP 프록시를 거치지 않는다. `REQUIRES_NEW`가 무시되고 `confirm()`의 트랜잭션 안에서 그냥 실행된다. 이후 `confirm()`이 예외를 던지며 롤백되면 `reportFail()`의 변경도 함께 롤백된다. 결제 실패 기록이 남지 않는다.

---

### `Wallet → User CascadeType.ALL` — 지갑 삭제 시 유저까지 삭제

```java
// Wallet.java
@OneToOne(fetch = FetchType.LAZY, cascade = CascadeType.ALL)
@JoinColumn(name = "user_id", unique = true)
private User user;
```

`Wallet`이 연관관계 owner인데 `User`에 `CascadeType.ALL`이 걸려 있다. `Wallet`을 삭제하면 `User`까지 cascade 삭제된다. 의도한 방향은 `User` 삭제 시 `Wallet`을 삭제하는 것이었을 텐데, 방향이 반대로 설정됐다.

---

### 그 외 문제들

**`SavePaymentRequestDto.amount`가 `long` primitive + `@NotNull`**

`long`은 null이 될 수 없으므로 `@NotNull`은 의미가 없다. JSON에서 `amount` 필드가 누락되면 기본값 0이 들어온다. 0원 결제가 허용된다.

**`Wallet.pendingOut` 초기값 누락**

`UserService`에서 지갑 생성 시 `postedBalance`는 초기값을 명시적으로 설정하지만 `pendingOut`은 설정하지 않는다. DB 레벨 default도 없으면 `null`이 들어가고, 이후 `pending_out + :amount` native query에서 `null` 연산 오류가 발생한다. [유저 도메인 편](/user-lifecycle-bugs/) 유저 도메인 작업 중에도 동일하게 발견됐으며, `@Builder.Default 0L`을 이 시점에 적용했다.

**`Filter` enum `@JsonCreator` 미적용**

`WalletController`에서 `@RequestParam Filter filter`로 받으면 Spring이 `Filter.valueOf()`를 사용한다. 소문자로 `?filter=charge`를 전달하면 `IllegalArgumentException`으로 500이 터진다. `from()` 메서드가 있지만 `@JsonCreator`가 붙어 있지 않아 사용되지 않는다.

**`getWalletTransactionList()` N+1**

정산 거래(`OUTGOING/INCOMING`) 1건을 DTO로 변환할 때 `Transfer → UserSettlement → Settlement → Schedule → Club` 순으로 최대 4번의 Lazy 로딩이 발생한다. 거래 내역 20건을 조회하면 최대 80개의 추가 쿼리가 나간다.

**`TossFeignConfig` 변수명에 `test` 포함**

```java
@Value("${payment.toss.test_secret_api_key}")
private String testSecretKey;
```

변수명과 프로퍼티 키 모두 `test`가 들어가 있다. 프로덕션 배포 시 실제 키로 교체됐는지 코드만 봐서는 알 수 없다.

---

## 해결

### 토스 승인 후 실패 — 보상 트랜잭션 추가

토스 결제 승인이 완료된 뒤 이후 로직에서 예외가 발생하면, 즉시 토스 결제 취소 API를 호출하도록 보상 경로를 추가했다.

```java
// PaymentService.java
try {
    response = tossPaymentClient.confirmPayment(req);
} catch (...) {
    paymentFailureRecorder.record(req, FailReason.TOSS_REJECT);
    throw ...;
}

try {
    updateWalletAndRecord(user, amount, payment);
} catch (Exception e) {
    // 지갑 업데이트 실패 → 토스 결제 취소 시도
    try {
        tossPaymentClient.cancelPayment(response.getPaymentKey(), "내부 오류로 인한 자동 취소");
    } catch (Exception cancelEx) {
        // 취소 API 자체가 실패한 경우 — CompensationRequired 상태로 기록
        paymentFailureRecorder.recordCompensationRequired(
            response.getPaymentKey(), req.getOrderId(), cancelEx.getMessage()
        );
        // 별도 배치가 payment_compensation 테이블을 주기적으로 스캔해 수동 처리
    }
    paymentFailureRecorder.record(req, FailReason.INTERNAL_ERROR);
    throw new CustomException(ErrorCode.PAYMENT_CONFIRM_FAILED);
}
```

`cancelPayment()` 호출 자체가 실패하는 경우를 위해 `payment_compensation` 테이블에 `paymentKey`, `orderId`, 실패 사유를 기록한다. 운영 배치가 이 테이블을 주기적으로 스캔해 미처리 건을 토스 관리자 콘솔 또는 환불 API로 수동 처리할 수 있도록 했다.

---

### 결제 금액 서버 생성·저장으로 변경

`/payments/save` 단계에서 서버가 orderId를 직접 생성하고, 해당 orderId에 대한 충전 금액도 서버가 결정해 Redis에 저장한다.

```java
// PaymentService.java
public SavePaymentResponse savePaymentInfo(Long requestedAmount) {
    // 서버가 orderId 생성 — 클라이언트 입력 아님
    String orderId = "order_" + UUID.randomUUID().toString().replace("-", "");

    // 서버가 amount를 Redis에 저장
    String redisKey = REDIS_PAYMENT_KEY_PREFIX + orderId;
    redisTemplate.opsForValue().set(redisKey, requestedAmount, TTL, TimeUnit.SECONDS);

    return new SavePaymentResponse(orderId, requestedAmount);
}

// /payments/success 검증
public void verifyPayment(String orderId, Long clientAmount) {
    String redisKey = REDIS_PAYMENT_KEY_PREFIX + orderId;
    Long serverAmount = (Long) redisTemplate.opsForValue().get(redisKey);
    if (serverAmount == null) {
        throw new CustomException(ErrorCode.PAYMENT_SESSION_EXPIRED);
    }
    if (!serverAmount.equals(clientAmount)) {
        throw new CustomException(ErrorCode.PAYMENT_AMOUNT_MISMATCH);
    }
}
```

서버가 생성한 orderId와 amount를 클라이언트에 내려주고, 클라이언트는 이 값으로 토스 SDK를 초기화한다. `/success` 단계에서 클라이언트가 보낸 amount가 서버가 저장한 값과 다르면 즉시 거부한다.

---

### 잔액 업데이트 — Java 덧셈에서 native query로 통일

```java
// WalletRepository.java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query(value = "UPDATE wallet SET posted_balance = posted_balance + :amount WHERE user_id = :userId",
       nativeQuery = true)
int creditByUserId(@Param("userId") Long userId, @Param("amount") long amount);
```

`confirm()`의 `wallet.updateBalance(wallet.getPostedBalance() + amount)` 호출을 제거하고 `creditByUserId()`로 교체했다. 정산·홀드 경로와 동일한 방식으로 통일됐다. DB가 현재 값을 읽고 더하는 단일 연산이므로 동시 실행에서도 값이 덮어써지지 않는다. `PESSIMISTIC_WRITE` 락도 함께 제거됐다.

---

### `reportFail()` self-invocation 해결 — 별도 빈으로 분리

`reportFail()` 로직을 `PaymentFailureRecorder`라는 별도 스프링 빈으로 추출했다. `PaymentService`가 `PaymentFailureRecorder`를 주입받아 호출하면, AOP 프록시를 통해 `REQUIRES_NEW`가 정상 적용된다. `confirm()`이 롤백되더라도 실패 기록은 독립 트랜잭션으로 보존된다.

---

### `Wallet → User` Cascade 방향 수정

```java
// Wallet.java
@OneToOne(fetch = FetchType.LAZY)  // CascadeType.ALL 제거
@JoinColumn(name = "user_id", unique = true)
private User user;
```

`Wallet`에서 `User`로의 cascade를 제거했다. `User` 삭제 시 `Wallet`을 함께 삭제하고 싶다면 `User → Wallet` 방향에 cascade를 두는 것이 올바른 방향이다.

---

### 나머지 수정

`SavePaymentRequestDto.amount`를 `Long` wrapper 타입으로 변경하고 `@NotNull @Min(1)`을 추가해 0원 결제와 null 입력을 막았다. `Wallet` 생성 시 `pendingOut`도 `@Builder.Default private Long pendingOut = 0L`로 명시했다. `Filter` enum의 `from()` 메서드에 `@JsonCreator`를 적용해 소문자 입력도 처리되도록 했다. `WalletCapture*Event` 미사용 이벤트 클래스 두 개는 제거했다.

---

## 현재 상태

| 문제 | 상태 |
|------|------|
| 토스 승인 후 DB 실패 시 돈 유실 | ✅ 해결 — 보상 트랜잭션 추가 |
| 잔액 Race Condition (Java 덧셈 + SET) | ✅ 해결 — native query 통일 |
| `PESSIMISTIC_WRITE` 행 락 + `@Transactional` 커넥션 점유 | ✅ 부분 해결 — 행 락 제거, 커넥션 점유는 HikariCP 튜닝으로 수용, 추후 트랜잭션 분리 예정 |
| 결제 금액 서버 검증 없음 | ✅ 해결 — 서버 측 orderId·amount 생성·저장 |
| `cancelPayment()` 자체 실패 시 처리 경로 없음 | ✅ 해결 — `payment_compensation` 테이블 기록 + 배치 처리 |
| `reportFail()` self-invocation | ✅ 해결 — `PaymentFailureRecorder` 분리 |
| `Wallet → User CascadeType.ALL` | ✅ 해결 — cascade 제거 |
| `amount` 0원 허용 | ✅ 해결 — `Long` + `@Min(1)` |
| `pendingOut` 초기값 누락 | ✅ 해결 — `@Builder.Default = 0L` |
| `Filter` enum 소문자 500 | ✅ 해결 — `@JsonCreator` 적용 |
| `getWalletTransactionList()` N+1 | ✅ 해결 — fetch join으로 교체 |
| 미사용 이벤트 클래스 | ✅ 해결 — 제거 |
| `WalletTransaction.balance` 스냅샷 부정확 | ✅ 수용 — audit 목적 스냅샷으로 설계 의도 명시 |
| `TossFeignConfig` `test` 키 변수명 | ⚠️ 잔존 — 프로덕션 배포 시 확인 필요 |

---

## 정리하며

결제 코드에서 가장 위험한 건 눈에 잘 띄지 않는 문제들이었다. `wallet.getPostedBalance() + amount`는 평소에는 맞다. 동시 요청이 없으면 항상 정확한 값이 나온다. 그런데 정산과 충전이 동시에 일어나는 순간, 한쪽의 변경이 다른 쪽에 덮어써진다. 잔액이 조용히 틀어지고 로그에는 아무 에러가 없다.

JPA와 native query를 같은 컬럼에 혼용하면 이런 일이 생긴다. JPA는 1차 캐시에서 읽고, native query는 DB에서 읽는다. 두 경로가 서로를 모르는 채로 같은 행을 수정하면 둘 중 나중에 실행된 쪽이 이긴다.

외부 결제 API가 끼면 복잡도가 한 단계 더 올라간다. DB 트랜잭션으로 롤백할 수 있는 범위와 외부 API 호출로 이미 처리된 범위가 다르기 때문이다. 토스가 승인하고 나면 DB 롤백으로는 되돌릴 수 없다. 보상 트랜잭션을 명시적으로 설계하지 않으면 그 틈새에서 자금이 유실될 수 있다. 그리고 보상 트랜잭션 자체가 실패할 수 있다는 것도 설계에 포함해야 한다.

> **외부 시스템과의 경계에서는 롤백이 보장되지 않는다.**
> DB 트랜잭션이 실패하면 롤백되지만, 외부 API가 이미 처리한 것은 롤백되지 않는다.
> 보상 경로를 설계하되, 그 보상 경로 자체의 실패 케이스까지 포함해야 한다.

---

## 시리즈 탐색

**◀ 이전 글**  
[스케줄 도메인 — 권한 체크 순서 하나가 자금 동결로 이어지는 구조](/schedule-permission-fund-freeze/)

**다음 글 ▶**  
[유저 도메인 — 인증 이후, 사용자 생명주기에서 무너진 것들](/user-lifecycle-bugs/)
