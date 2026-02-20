---
title: 정산 도메인 — Kafka·Outbox·StructuredTaskScope, 각 경계에서 무너진 것들
date: 2026-02-20
tags: [Kafka, 정산, 동시성, 트러블슈팅, Java, Outbox, 분산시스템]
---

# 정산 도메인 — Kafka·Outbox·StructuredTaskScope, 각 경계에서 무너진 것들

## 개요

정산 코드는 다른 도메인보다 훨씬 복잡한 구조였다. Spring 이벤트로 시작해서 Kafka로 비동기 처리하고, Outbox 패턴으로 재시도를 보장하며, StructuredTaskScope로 참가자를 병렬 처리한다. 컴포넌트 하나하나는 각자의 역할이 있었다.

문제는 컴포넌트 사이사이의 경계에서 터졌다. Spring 이벤트와 Kafka가 동시에 같은 처리를 하고 있었다. 참가자 한 명이 실패해도 나머지를 다 처리한 뒤 COMPLETED로 끝냈다. 실패를 기록하려고 catch 블록에서 status를 FAILED로 바꿨는데, 바깥 트랜잭션이 롤백되면서 그 변경도 함께 사라졌다. `@Transactional(REQUIRES_NEW)`를 붙였는데 같은 클래스 안에서 직접 호출하고 있어서 프록시를 거치지 않았다.

눈에 띄는 버그가 아니었다. 각자 맞아 보이는 코드들이 조합됐을 때 생기는 문제들이었다.

---

## 시스템 구조

### 정산 흐름

```
LEADER가 automaticSettlement() 호출
    ↓
markProcessing() — CAS로 중복 요청 차단
    ↓
SettlementProcessEvent 발행
    ↓
Outbox 테이블에 저장 → OutboxRelayService가 Kafka로 전송
    ↓
SettlementKafkaEventListener 수신
    ↓
StructuredTaskScope로 참가자 병렬 처리
    ├─ 각 참가자: captureHold() → WalletTransaction 기록 → Transfer 저장
    └─ 실패 시: UserSettlement FAILED 처리
    ↓
전원 성공 → Settlement COMPLETED + 리더 입금
실패자 있음 → Settlement FAILED (revertSettlementToFailed)
```

각 단계가 별도 컴포넌트이고, 단계 사이에 비동기 경계가 있다. 문제는 주로 그 경계에서 발생했다.

---

## 버그 상세

### 이중 처리 경로 — Spring 이벤트와 Kafka가 같은 정산을 동시에 처리

```java
// SettlementCommandService.java
eventPublisher.publishEvent(new SettlementProcessEvent(settlementId));  // Spring 이벤트
// 동시에 Outbox → Kafka로도 같은 이벤트 발행
```

`SettlementProcessEventListener`가 Spring 이벤트를 수신해 정산 로직을 실행하고, `SettlementKafkaEventListener`도 Kafka 메시지를 수신해 같은 로직을 실행하고 있었다. 정산 하나가 두 번 처리되는 구조였다.

Spring 이벤트는 같은 트랜잭션 안에서 동기로 실행되고, Kafka는 비동기로 나중에 실행된다. 정산이 이미 완료된 뒤 Kafka 리스너가 다시 처리를 시작하면 idempotency가 없으면 이중 지갑 차감이 발생한다.

해결은 단순했다. `SettlementProcessEventListener.java`를 삭제하고 Kafka 단일 경로만 남겼다.

---

### 부분 성공 시 COMPLETED — 한 명이 실패해도 정산이 끝난 것처럼 보임

```java
// 수정 전 SettlementKafkaEventListener.java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    for (Long participantId : participantIds) {
        scope.fork(() -> processParticipantSettlement(participantId));
    }
    scope.join();
    scope.throwIfFailed();  // 첫 실패에서 전체 중단
}
// 일부 처리 후 중단된 상태로 completeSettlement() 호출 → COMPLETED
```

`ShutdownOnFailure`는 첫 번째 실패가 발생하면 나머지 작업을 모두 중단한다. 참가자 10명 중 3번째가 실패하면 4~10번째는 처리조차 되지 않는다. 그런데 이후 로직이 이 상태를 확인하지 않고 그냥 COMPLETED로 마무리했다.

1번, 2번 참가자의 지갑에서 예약금이 차감됐는데 4~10번은 차감되지 않은 상태. 리더는 COMPLETED이므로 입금도 됐다. 정산이 완료된 것처럼 보이지만 실제로는 부분 처리 상태다.

```java
// 수정 후 SettlementKafkaEventListener.java
try (var scope = new StructuredTaskScope()) {
    for (Long participantId : participantIds) {
        scope.fork(() -> processParticipantSettlement(participantId));
    }
    scope.join();  // 전원 완료까지 대기
}
// 실패자가 한 명이라도 있으면
if (!failedParticipants.isEmpty()) {
    revertSettlementToFailed(settlementId);  // 전체 FAILED
}
```

`ShutdownOnFailure` 대신 기본 `StructuredTaskScope`로 교체해 전원이 처리될 때까지 기다린다. 실패자가 있으면 정산을 FAILED로 되돌리고 리더에게 입금하지 않는다.

---

### `@Transactional(REQUIRES_NEW)` — self-invocation으로 무효화

```java
// SettlementCommandService.java — 수정 전
@Transactional
public void processSettlement(Long settlementId) {
    // ...
    processParticipantSettlement(participantId);  // ← 직접 호출
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void processParticipantSettlement(Long participantId) {
    // 독립 트랜잭션이어야 함
}
```

`processParticipantSettlement()`에 `REQUIRES_NEW`를 붙였는데, `this.processParticipantSettlement()`로 직접 호출하면 Spring AOP 프록시를 거치지 않는다. 그냥 `processSettlement()`의 트랜잭션 안에서 평범하게 실행된다. 참가자별 독립 트랜잭션이 없으니 한 명이 실패하면 이전에 성공한 모든 참가자의 변경도 함께 롤백된다.

`processParticipantSettlement()`를 `UserSettlementService`라는 별도 스프링 빈으로 분리했다. `SettlementCommandService`가 `UserSettlementService`를 주입받아 호출하면 프록시를 통과하고 `REQUIRES_NEW`가 정상 동작한다. `creditToLeader()`도 같은 이유로 별도 메서드로 독립했다. 8화의 `reportFail()` 분리와 같은 패턴이다. Spring AOP의 self-invocation 한계는 항상 같은 방식으로 나타나고, 해결도 항상 같다 — 별도 빈으로 꺼내기.

---

### catch 블록의 FAILED 저장이 롤백됨 — 실패 기록이 사라짐

```java
// 수정 전
@Transactional
public void processParticipantSettlement(Long participantId) {
    try {
        captureHold(participantId);
        // ...
    } catch (Exception e) {
        userSettlement.updateStatus(SettlementStatus.FAILED);  // ← FAILED로 변경
        throw e;  // 예외를 다시 던짐
    }
}
```

`updateStatus(FAILED)` 호출 뒤 `throw e`로 예외를 다시 던지면, 바깥 `@Transactional`이 롤백을 시작한다. `updateStatus(FAILED)`도 같은 트랜잭션 안에 있으니 함께 롤백된다. 예외는 발생했는데 FAILED 상태 변경은 없던 일이 된다. 실패 기록이 남지 않는다.

```java
// FailedEventAppender.java (신규)
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void markFailed(Long userSettlementId) {
    userSettlement.updateStatus(SettlementStatus.FAILED);
    outboxRepository.save(FailedOutboxEvent.of(userSettlementId));
    // 부모 트랜잭션이 롤백되어도 이 변경은 독립적으로 커밋됨
}
```

`FailedEventAppender`를 별도 빈으로 분리하고 `REQUIRES_NEW`를 적용했다. catch 블록에서 이 빈을 호출하면, 프록시를 통해 독립 트랜잭션에서 FAILED 상태가 커밋된다. 바깥 트랜잭션이 롤백되어도 실패 기록은 살아남는다.

---

### `Transfer` 저장 실패 silent swallow

```java
// LedgerWriter.java — 수정 전
private void saveTransfers(List<Transfer> transfers) {
    try {
        transferRepository.saveAll(transfers);
    } catch (DataIntegrityViolationException e) {
        // 아무것도 안 함
    }
}
```

`saveWalletTransactions()`는 중복 발생 시 개별 재시도 로직이 있는데, `saveTransfers()`는 예외를 잡고 아무것도 하지 않는다. `WalletTransaction`은 저장됐는데 `Transfer`는 유실된다. 지갑 거래 내역에는 기록됐는데 송금/수금 상세는 없는 상태다. 로그에도 아무것도 찍히지 않아 장애 추적이 불가능하다.

`saveWalletTransactions()`와 동일한 패턴으로 `insertTransfersIndividually()`를 추가하고, 실패 시 `log.warn()`으로 최소한의 기록을 남기도록 했다.

---

### Kafka 설정 — `auto.offset.reset=latest`와 `replicas=1`

```java
// KafkaConsumerConfig.java
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "latest");
```

컨슈머 그룹의 오프셋이 유실되거나 새로 생성되면 `latest`는 이전 메시지를 전부 건너뛴다. 서버 재배포나 컨슈머 그룹 리셋 시 그 사이에 발행된 정산 메시지가 영구적으로 처리되지 않는다.

`"earliest"`로 변경했다. 처음부터 다시 읽더라도 `operationId` 기반 idempotency가 이미 구현되어 있어 중복 처리가 차단된다. Kafka 토픽 replicas도 외부 프로퍼티로 분리해 운영 환경에서 2~3으로 설정할 수 있게 했다.

---

### Outbox status 업데이트가 Kafka 트랜잭션 밖

```java
// OutboxRelayService.java — 수정 전
kafkaTemplate.send(topic, message);   // ① Kafka 전송
outbox.setStatus(PUBLISHED);          // ② 별도 코드 블록
```

①이 성공하고 ②로 넘어가는 사이에 크래시가 나면, 다음 배치가 실행될 때 이 이벤트를 다시 전송한다. `setStatus(PUBLISHED)`가 Kafka 전송 트랜잭션 밖에 있어 원자적으로 처리되지 않는다.

`setStatus(PUBLISHED)`를 `executeInTransaction` 콜백 안으로 이동했다. Kafka 전송이 실패하면 status 변경도 무효화된다.

---

### `automaticSettlement()` — LEADER 권한 미검증

```java
// SettlementCommandService.java — 수정 전
public void automaticSettlement(Long clubId, Long scheduleId) {
    User user = userService.getCurrentUser();
    Settlement settlement = settlementRepository.findByScheduleId(scheduleId);
    // ❌ user가 해당 클럽의 LEADER인지 확인 없음
    settlement.setReceiver(user);  // 요청한 사람이 리더가 됨
}
```

인증된 사용자라면 누구든 타인의 클럽 정산을 요청할 수 있었다. 자신을 리더(수취인)로 등록해 다른 참가자들의 예약금을 자기 지갑으로 받아갈 수 있는 구조였다.

`settlement.getReceiver().getUserId().equals(user.getUserId())` 검증을 추가했다. 요청자가 해당 정산의 수취인(리더)과 일치하지 않으면 `MEMBER_CANNOT_CREATE_SETTLEMENT` 예외를 던진다.

---

## 현재 상태

| 문제 | 상태 |
|------|------|
| Spring 이벤트 + Kafka 이중 처리 | ✅ 해결 — Spring 이벤트 경로 삭제 |
| 부분 성공 시 COMPLETED 처리 | ✅ 해결 — StructuredTaskScope 전원 대기 + FAILED 전환 |
| self-invocation으로 REQUIRES_NEW 무효 | ✅ 해결 — `UserSettlementService` 분리 |
| catch 블록 FAILED 저장 롤백 | ✅ 해결 — `FailedEventAppender` REQUIRES_NEW 분리 |
| `Transfer` silent swallow | ✅ 해결 — 개별 재시도 + 로깅 |
| LEADER 권한 미검증 | ✅ 해결 — 수취인 일치 검증 추가 |
| `auto.offset.reset=latest` | ✅ 해결 — `earliest`로 변경 |
| Kafka replicas 하드코딩 | ✅ 해결 — 프로퍼티 외부화 |
| Outbox status가 Kafka 트랜잭션 밖 | ✅ 해결 — 콜백 안으로 이동 |
| Outbox 테이블 무한 축적 | ✅ 해결 — 매일 새벽 3시 cleanup 배치 |
| `totalAmount` 이중 합산 | ✅ 해결 — CAS markProcessing() |
| `SettlementProcessEvent` mutable | ✅ 해결 — Java record 전환 |
| `ShutdownOnFailure` 나머지 중단 | ✅ 해결 — 기본 StructuredTaskScope 교체 |
| Schedule-Club IDOR | ✅ 해결 — existsScheduleInClub() 검증 |
| `WalletTransaction.update()` self-target | ✅ 해결 — targetWallet 파라미터 분리 |
| Outbox payload 필드 불일치 | ✅ 해결 — record에 필드 추가 + @JsonIgnoreProperties |
| `WalletTransaction.balance` 스냅샷 부정확 | ✅ 수용 — audit 목적 스냅샷으로 설계 명시 |
| Redis gate TTL 한계 | ✅ 수용 — CAS가 DB 레벨 이중 방어 역할 |

---

## 교훈

이번 도메인의 버그들은 단독으로 보면 다 합리적인 코드였다. `REQUIRES_NEW`를 붙인 건 올바른 의도였다. catch 블록에서 FAILED 상태를 기록하려 한 것도 맞는 방향이었다. 그런데 Spring AOP가 self-invocation에서는 프록시를 거치지 않는다는 것, 같은 트랜잭션 안의 변경은 함께 롤백된다는 것을 간과했다.

여러 컴포넌트를 조합할수록 각 컴포넌트의 경계 조건을 정확히 이해해야 한다. Spring AOP는 외부에서 빈을 주입받아 호출해야 프록시가 동작한다. 트랜잭션 분리가 필요하다면 코드를 별도 빈으로 꺼내야 한다. 이중 처리 경로가 있다면 어느 쪽이 정규 경로인지 명확히 해야 한다.

그리고 Kafka 설정 한 줄(`auto.offset.reset=latest`)이 서버 재배포 시 그 사이의 정산 이벤트를 전부 날릴 수 있다는 것도 놓치기 쉬운 부분이었다. 기능 구현에는 반영되지 않는 인프라 설정이 실제 데이터 유실을 만들어낸다.

> **비동기 경계를 넘나드는 코드는 각 경계에서 무슨 일이 벌어지는지 명확히 해야 한다.**
> 트랜잭션 경계, AOP 프록시 경계, 비동기 스레드 경계 — 어느 경계에서 무엇이 보장되고 무엇이 보장되지 않는지 이해하지 못하면, 맞아 보이는 코드들이 조합돼 틀린 동작을 만들어낸다.
