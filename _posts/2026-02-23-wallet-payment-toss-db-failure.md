---
title: 지갑·결제 도메인 — 토스 승인 후 DB가 죽으면 자금이 유실될 수 있다
date: 2026-02-23
tags: [결제, 동시성, JPA, 트러블슈팅, Java, 지갑, 토스페이먼츠]
permalink: /wallet-payment-toss-db-failure/
excerpt: "토스페이먼츠 승인이 완료된 뒤 지갑 업데이트에서 예외가 발생하면, DB는 롤백되지만 이미 승인된 결제는 취소되지 않아 자금이 유실될 수 있었다."
---

## 한눈에 보기

```
이 글에서 다루는 문제 (6건):

1. 토스 승인 후 DB 실패 → 결제 완료인데 지갑 미충전    ← 핵심 버그
2. 잔액 Race Condition → 정산 차감분이 충전으로 덮어써짐
3. @Transactional 범위 내 외부 API 호출 → 커넥션 점유
4. 결제 금액 서버 검증 없음 → 클라이언트가 금액 결정
5. reportFail() self-invocation → 실패 기록이 롤백됨
6. Wallet → User CascadeType.ALL → 지갑 삭제 시 유저까지 삭제
```

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

```
실패 시나리오:

① tossPaymentClient.confirmPayment()  → 성공. 사용자 카드에서 돈이 빠짐.
② wallet.updateBalance()              → DB 예외 발생!

@Transactional 롤백 → 지갑 업데이트는 되돌려짐
토스 결제는?       → 이미 승인 완료. 별도로 취소 API를 호출해야 되돌릴 수 있음.
취소 코드가 있나?  → 없음. cancelPayment 메서드는 주석 처리된 채 남아있었음.

결과: 카드 청구서에는 결제 내역이 찍히는데, 지갑에는 충전이 안 됨.
```

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

> 아래 시나리오는 두 경로가 같은 시점에 같은 행을 수정하는 것처럼 보이지만, `PESSIMISTIC_WRITE` 락 때문에 실제로 동시 실행되지는 않는다. 문제의 본질은 동시성이 아니라 **JPA 1차 캐시와 native query의 혼용**이다. 시나리오를 직렬화된 순서로 다시 보면:

```
동시 실행 시나리오:

  충전 경로 (confirm)                 정산 경로 (captureHold)
  ──────────────────                 ─────────────────────
  findByUser() → 잔액 10,000 읽음
  (JPA 1차 캐시에 10,000 저장)
                                      native query:
                                      SET posted_balance = posted_balance - 3000
                                      → DB 잔액: 7,000

  wallet.updateBalance(10,000 + 5,000)
  → SET posted_balance = 15,000      ← 1차 캐시의 옛날 값(10,000)에 더함
  → DB 잔액: 15,000                  ← 정산 차감분 3,000원이 증발!

  기대 결과: 10,000 - 3,000 + 5,000 = 12,000
  실제 결과: 15,000 (3,000원 어디로?)
```

즉, `PESSIMISTIC_WRITE`(SELECT FOR UPDATE)가 행을 잠그므로 정산 경로의 native query는 락이 해제될 때까지 대기한다. 충전 트랜잭션이 커밋된 뒤 정산이 실행되거나 그 반대 순서로 직렬화된다. 문제는 **JPA 1차 캐시가 트랜잭션 시작 시점의 값을 기억**하고 있다는 것이다. 충전 경로가 `wallet.getPostedBalance()`로 읽은 값은 1차 캐시의 스냅샷이고, 이후 dirty checking으로 `SET posted_balance = ?`를 실행하면 DB의 현재 값이 아닌 캐시의 옛날 값 기준으로 덮어쓴다. 이것이 native query(`SET posted_balance = posted_balance + ?`)와 JPA dirty checking(`SET posted_balance = ?`)을 혼용할 때 발생하는 근본 문제다.

```
원인: 잔액 변경 경로가 두 가지 방식을 섞어 씀

  충전: JPA 읽기 → Java 덧셈 → SET posted_balance = 15000  ← 덮어쓰기!
  정산: native query → SET posted_balance = posted_balance - 3000  ← 안전

  정산 경로는 DB가 현재 값을 읽고 더하는 단일 연산이라 동시성에 안전.
  충전 경로만 다른 방식을 쓰고 있었음.
```

```
추가 문제: PESSIMISTIC_WRITE 락 + 외부 API 호출

  findByUser()에 PESSIMISTIC_WRITE가 걸려 있었음.
  → 토스 API 응답을 기다리는 수 초 동안 해당 지갑 행이 잠김
  → 다른 요청이 같은 지갑에 접근하면 그 시간만큼 대기
  → 잔액 업데이트를 native query로 통일하면 락 자체가 불필요해짐
```

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

```
현재 검증 흐름:

  /save:    클라이언트가 보낸 amount → Redis에 저장
  /success: 클라이언트가 보낸 amount ← Redis에서 꺼낸 amount와 비교

  비교의 양쪽이 모두 클라이언트 입력!
  → 클라이언트가 양쪽에 같은 값을 보내면 항상 통과
  → 서버가 "이 orderId는 얼마짜리 충전이어야 한다"를 독립적으로 결정하지 않음
```

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

```
의도:
  reportFail()은 REQUIRES_NEW → 별도 트랜잭션으로 실패 기록을 남기겠다

실제:
  confirm() 안에서 this.reportFail()로 직접 호출
  → Spring AOP 프록시를 거치지 않음
  → REQUIRES_NEW가 무시됨
  → confirm()의 트랜잭션 안에서 그냥 실행됨

결과:
  confirm()이 예외 → 롤백 → reportFail()의 변경도 함께 롤백
  → 결제 실패 기록이 남지 않음
```

---

### `Wallet → User CascadeType.ALL` — 지갑 삭제 시 유저까지 삭제

```java
// Wallet.java
@OneToOne(fetch = FetchType.LAZY, cascade = CascadeType.ALL)
@JoinColumn(name = "user_id", unique = true)
private User user;
```

```
의도: User 삭제 → Wallet도 삭제

실제:
  Wallet.java:
    @OneToOne(cascade = CascadeType.ALL)
    private User user;

  Wallet이 연관관계의 owner
  → Wallet을 삭제하면 CascadeType.ALL로 User까지 삭제됨!

  방향이 반대로 설정되어 있음
```

---

### 그 외 문제들

- **`long amount` + `@NotNull`**: primitive `long`은 null이 될 수 없어 `@NotNull`이 무의미. JSON에서 `amount` 누락 시 기본값 0 → 0원 결제 허용.
- **`pendingOut`·`postedBalance` 초기값 누락**: 두 필드 모두 `Long` 타입인데 초기값 없음. `@Builder.Default`도 없어 빌더로 생성 시 null. `null + :amount` native query에서 null 연산 오류 가능.
- **`Filter` enum `@JsonCreator` 미적용**: `?filter=charge` (소문자) → `Filter.valueOf()` 실패 → 500. `from()` 메서드 있지만 `@JsonCreator` 안 붙어서 미사용.
- **거래 내역 N+1**: 정산 거래 1건 → `Transfer → UserSettlement → Settlement → Schedule → Club` 최대 4회 Lazy 로딩. 20건이면 80쿼리.
- **`TossFeignConfig`에 `test`**: 변수명 `testSecretKey`, 프로퍼티 키 `test_secret_api_key`. 프로덕션 키로 교체됐는지 코드만으로는 알 수 없음.

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

```
보상 흐름:

토스 승인 성공 → 지갑 업데이트 시도
  ├─ 성공 → 정상 완료
  └─ 실패 → 토스 취소 API 호출
              ├─ 취소 성공 → 실패 기록 남기고 끝
              └─ 취소도 실패 → payment_compensation 테이블에 기록
                               → 배치가 주기적으로 스캔해 수동 처리
```

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

```
수정 전: 클라이언트가 orderId와 amount를 모두 결정
수정 후: 서버가 orderId를 생성하고, amount도 서버가 Redis에 저장
        → 클라이언트는 서버가 내려준 값으로 토스 SDK를 초기화
        → /success에서 클라이언트 값과 서버 값이 다르면 즉시 거부
```

> 현재 구현에서 `requestedAmount`는 여전히 클라이언트가 전달한 값이다. 완전한 서버 검증을 위해서는 서버 측 충전 티어(예: 5,000 / 10,000 / 50,000원)를 정의하고, 클라이언트는 티어 ID만 전달하는 방식이 필요하다. 현재 구조는 orderId를 서버가 생성하고 amount 위변조를 검증하는 수준이며, 이것만으로도 기존의 "양쪽 모두 클라이언트 입력" 문제는 해결된다.

---

### 잔액 업데이트 — Java 덧셈에서 native query로 통일

```java
// WalletRepository.java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query(value = "UPDATE wallet SET posted_balance = posted_balance + :amount WHERE user_id = :userId",
       nativeQuery = true)
int creditByUserId(@Param("userId") Long userId, @Param("amount") long amount);
```

```
수정 전: wallet.updateBalance(wallet.getPostedBalance() + amount)
         → JPA 읽기 + Java 덧셈 + SET = 덮어쓰기

수정 후: walletRepository.creditByUserId(userId, amount)
         → SET posted_balance = posted_balance + :amount
         → DB가 현재 값을 읽고 더하는 단일 연산
         → 동시 실행에도 안전. PESSIMISTIC_WRITE 락도 불필요해져 제거.
```

---

### `reportFail()` self-invocation 해결 — 별도 빈으로 분리

```
수정 전: confirm() 안에서 this.reportFail() → 프록시 우회 → REQUIRES_NEW 무시
수정 후: PaymentFailureRecorder (별도 빈) → 주입받아 호출 → 프록시 정상 통과
         → confirm()이 롤백되어도 실패 기록은 별도 트랜잭션으로 보존
```

---

### `Wallet → User` Cascade 방향 수정

```java
// Wallet.java
@OneToOne(fetch = FetchType.LAZY)  // CascadeType.ALL 제거
@JoinColumn(name = "user_id", unique = true)
private User user;
```

```
수정: CascadeType.ALL 제거.
User 삭제 → Wallet 삭제를 원한다면, User 쪽에 cascade를 두는 것이 올바른 방향.
```

---

### 나머지 수정

- **`long amount` + `@NotNull`**: `Long` wrapper로 변경 + `@NotNull @Min(1)`
- **`pendingOut` 초기값 누락**: `@Builder.Default private Long pendingOut = 0L`
- **`Filter` enum 소문자 → 500**: `from()` 메서드에 `@JsonCreator` 적용
- **미사용 `WalletCapture*Event`**: 2개 제거

---

## 현재 상태

- **토스 승인 후 DB 실패 시 돈 유실**: ✅ 해결 — 보상 트랜잭션 추가
- **잔액 Race Condition (Java 덧셈 + SET)**: ✅ 해결 — native query 통일
- **`PESSIMISTIC_WRITE` 행 락 + `@Transactional` 커넥션 점유**: ✅ 부분 해결 — 행 락 제거, 커넥션 점유는 HikariCP 튜닝으로 수용, 추후 트랜잭션 분리 예정
- **결제 금액 서버 검증 없음**: ⚠️ 부분 해결 — 서버 측 orderId 생성 + amount 위변조 검증. 금액 자체는 여전히 클라이언트 입력이며, 완전한 서버 검증은 충전 티어 방식 도입 필요
- **`cancelPayment()` 자체 실패 시 처리 경로 없음**: ✅ 해결 — `payment_compensation` 테이블 기록 + 배치 처리
- **`reportFail()` self-invocation**: ✅ 해결 — `PaymentFailureRecorder` 분리
- **`Wallet → User CascadeType.ALL`**: ⚠️ 식별 — cascade 제거 방향 확인. 최종 수정은 [Phase 5 프로덕션 하드닝](/production-hardening-chaos-engineering/)에서 적용
- **`amount` 0원 허용**: ✅ 해결 — `Long` + `@Min(1)`
- **`pendingOut`·`postedBalance` 초기값 누락**: ✅ 해결 — 두 필드 모두 `@Builder.Default = 0L`
- **`Filter` enum 소문자 500**: ✅ 해결 — `@JsonCreator` 적용
- **`getWalletTransactionList()` N+1**: ✅ 해결 — fetch join으로 교체
- **미사용 이벤트 클래스**: ✅ 해결 — 제거
- **`WalletTransaction.balance` 스냅샷 부정확**: ✅ 수용 — audit 목적 스냅샷으로 설계 의도 명시
- **`TossFeignConfig` `test` 키 변수명**: ⚠️ 잔존 — 프로덕션 배포 시 확인 필요

---

## 정리하며

결제 코드의 주요 위험은 단일 요청에서는 드러나지 않는 동시성 문제에 있었다. `wallet.getPostedBalance() + amount`는 동시 요청이 없으면 정확한 값을 반환한다. 그러나 정산과 충전이 동시에 실행되면, 한쪽의 변경이 다른 쪽에 덮어써진다. 잔액이 틀어지지만 로그에는 에러가 기록되지 않는다.

JPA와 native query를 같은 컬럼에 혼용하면 이런 일이 생긴다. JPA는 1차 캐시에서 읽고, native query는 DB에서 읽는다. 두 경로가 서로를 모르는 채로 같은 행을 수정하면 둘 중 나중에 실행된 쪽이 이긴다.

외부 결제 API가 끼면 복잡도가 한 단계 더 올라간다. DB 트랜잭션으로 롤백할 수 있는 범위와 외부 API 호출로 이미 처리된 범위가 다르기 때문이다. 토스가 승인하고 나면 DB 롤백으로는 되돌릴 수 없다. 보상 트랜잭션을 명시적으로 설계하지 않으면 그 틈새에서 자금이 유실될 수 있다. 그리고 보상 트랜잭션 자체가 실패할 수 있다는 것도 설계에 포함해야 한다.

> **외부 시스템과의 경계에서는 롤백이 보장되지 않는다.**
> DB 트랜잭션이 실패하면 롤백되지만, 외부 API가 이미 처리한 것은 롤백되지 않는다.
> 보상 경로를 설계하되, 그 보상 경로 자체의 실패 케이스까지 포함해야 한다.

---

## 시리즈 탐색

**◀ 이전 글**
[스케줄 도메인 — 권한 체크 순서 하나가 자금 동결로 이어지는 구조](/schedule-permission-fund-freeze/)

**▶ 다음 글**
[유저 도메인 — 인증 이후, 사용자 생명주기에서 무너진 것들](/user-lifecycle-bugs/)
