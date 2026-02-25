---
title: 스케줄 도메인 — 권한 체크 순서 하나가 자금 동결로 이어지는 구조
date: 2026-02-23
tags: [JPA, Spring, 동시성, 트러블슈팅, Java, 스케줄, 정산]
permalink: /schedule-permission-fund-freeze/
---

# 스케줄 도메인 — 권한 체크 순서 하나가 자금 동결로 이어지는 구조

## 개요

스케줄(일정) 코드를 다시 살펴보다가 `createSchedule()`에서 이상한 순서를 발견했다. DB에 먼저 저장하고, 그 다음에 권한을 확인하고 있었다. `@Transactional`로 롤백되니까 기능상 문제는 없지만, 권한이 없는 사용자의 요청마다 쓸데없이 INSERT가 나간다.

그런데 `deleteSchedule()`을 보다가 훨씬 심각한 걸 발견했다. 스케줄을 삭제할 때 참여자들의 지갑 홀드금을 해제하는 코드가 없었다. 스케줄이 삭제되면 그 시점에 지갑에 잡혀 있던 예약금이 영구적으로 동결된다. 사용자 자금이 묶이는 문제였다.

여기에 `joinSchedule()`의 동시성 구멍까지 있었다. 정원 체크와 중복 참여 체크 사이에 시간 갭이 있고, `user_schedule` 테이블에 Unique Constraint가 없어서 동시에 두 요청이 들어오면 같은 사용자가 2번 참여하고 예약금도 2배 빠져나간다.

---

## 시스템 구조

### 스케줄 참여 흐름

```
createSchedule()  → Settlement 생성 + 리더 UserSchedule 생성
    ↓
joinSchedule()    → 정원/중복 체크 → 지갑 홀드 → UserSchedule + UserSettlement 생성
    ↓
@Scheduled(자정)  → 지난 스케줄 READY → ENDED 상태 변경
    ↓
deleteSchedule()  → Schedule 삭제 + ChatRoom 삭제
                  (❌ 지갑 홀드 해제 없음, ❌ Settlement 정리 없음)
```

`joinSchedule()`이 지갑에 홀드를 걸고, `deleteSchedule()`이 그걸 풀어줘야 하는데 풀어주지 않았다. 생성과 해제가 비대칭이었다.

---

## 버그 상세

### `deleteSchedule()` — 스케줄 삭제 시 참여자 예약금 영구 동결

```java
// ScheduleService.java
public void deleteSchedule(Long clubId, Long scheduleId) {
    // ...
    club.getSchedules().remove(schedule);   // Schedule 삭제
    chatRoomRepository.delete(chatRoom);    // ChatRoom 삭제
    // ❌ walletHoldService.release() 없음
    // ❌ Settlement 삭제 없음
    // ❌ UserSettlement 삭제 없음
}
```

`joinSchedule()`에서는 `walletRepository.holdBalanceIfEnough()`로 참여자 지갑에 예약금을 홀드한다. 그런데 `deleteSchedule()`은 `UserClub`과 채팅방만 삭제하고 지갑 홀드를 풀지 않는다.

스케줄이 삭제되면 해당 시점에 참여자들의 지갑에 잡혀 있던 금액이 `pending_out` 상태로 영구 잠긴다. `Settlement` 레코드도 남아 있어 `Schedule`을 참조하는 FK가 남은 상태로 의도치 않게 유지된다.

---

### `joinSchedule()` Race Condition — 중복 참여 시 2배 과금

```java
// ScheduleService.java
int userCount = userScheduleRepository.countBySchedule(schedule);          // ① SELECT COUNT
if (userCount >= schedule.getUserLimit()) { throw ... }
if (userScheduleRepository.findByUserAndSchedule(user, schedule).isPresent()) { throw ... }  // ② SELECT
// ... 시간 갭
int flag = walletRepository.holdBalanceIfEnough(userId, schedule.getCost());  // ③ UPDATE
userScheduleRepository.save(userSchedule);  // ④ INSERT
```

`user_schedule` 테이블에 `(user_id, schedule_id)` Unique Constraint가 없다. 같은 사용자가 동시에 두 번 요청을 보내면 둘 다 `findByUserAndSchedule()`에서 `Optional.empty()`를 받고 `save()`까지 진행한다. 중복 `UserSchedule`이 생기고 지갑에서 예약금이 2배 빠져나간다.

정원 초과도 마찬가지다. 정원 5명인 스케줄에 마지막 자리를 두고 두 요청이 동시에 들어오면 둘 다 `countBySchedule()=4`를 읽고 체크를 통과해 6명이 참여하게 된다.

---

### `createSchedule()` — 권한 체크 전에 DB 저장

```java
// ScheduleService.java
public ScheduleCreateResponseDto createSchedule(Long clubId, ScheduleRequestDto requestDto) {
    Club club = clubRepository.findById(clubId).orElseThrow(...);
    Schedule schedule = requestDto.toEntity(club);
    scheduleRepository.save(schedule);                          // ① 먼저 저장
    User user = userService.getCurrentUser();
    UserClub userClub = userClubRepository.findByUserAndClub(user, club).orElseThrow(...);
    if (userClub.getClubRole() != ClubRole.LEADER) {
        throw new CustomException(ErrorCode.MEMBER_CANNOT_CREATE_SCHEDULE);  // ② 나중에 체크
    }
}
```

`GenerationType.IDENTITY`를 쓰면 `save()` 시점에 즉시 INSERT가 나간다. `@Transactional`로 롤백되더라도 auto_increment 값은 소모된다. 권한 없는 사용자가 반복해서 요청을 보내면 불필요한 DB I/O와 시퀀스 낭비가 쌓인다. 체크 순서를 바꾸면 간단히 해결된다.

---

### `updateSchedule()` — `Long !=` 참조 비교 + 비용 변경 부분 실패 시 정합성 문제

```java
// ScheduleService.java
if (schedule.getCost() != requestDto.getCost()) {  // Long 객체 참조 비교
    // 참여자 전원 지갑 업데이트 로직...
}
```

`Long`은 `-128~127` 범위만 캐싱한다. 비용이 1000원이면 두 `Long` 객체의 주소가 달라 `!=`가 `true`를 반환하고, 비용 변경이 없는데도 참여자 전원의 지갑 업데이트 로직으로 진입한다. `equals()`로 바꿔야 한다.

비용 변경 자체도 문제였다. 참여자 N명의 지갑을 순차적으로 홀드하다가 중간에 한 명이 잔액 부족이면 예외를 던져 롤백을 시도한다. 하지만 이미 업데이트된 앞선 참여자들의 지갑 상태와 DB 트랜잭션 롤백 사이에 정합성 문제가 생길 수 있는 구조였다. `holdBalanceIfEnough`가 native query(`UPDATE ... WHERE ...`)로 실행되는 경우, 롤백이 되더라도 그 사이 다른 트랜잭션이 해당 지갑 잔액을 읽었다면 일시적으로 틀린 잔액을 보게 된다. 이 문제는 비용 변경 기능 자체를 설계 레벨에서 재검토해야 하는 성격이라 이번 화에서 완전히 해결하지 못했다. 현재 상태로는 비용 변경 기능을 운영 환경에서 활성화하지 않는 것으로 수용했다.

---

### IDOR — URL의 `clubId`와 `scheduleId` 소속 관계 미검증

```java
// ScheduleService.java
public void deleteSchedule(Long clubId, Long scheduleId) {
    Club club = clubRepository.findById(clubId).orElseThrow(...);
    Schedule schedule = scheduleRepository.findById(scheduleId).orElseThrow(...);
    // ❌ schedule.getClub().getClubId().equals(clubId) 체크 없음
}
```

`clubId=1`, `scheduleId=99`로 요청을 보낼 때, `schedule 99`가 실제로 `club 2`에 속하더라도 삭제가 실행된다. `updateSchedule`, `joinSchedule`, `deleteSchedule` 세 곳 모두 `clubId`로 클럽을 조회하지만 해당 스케줄이 그 클럽 소속인지는 확인하지 않는다. URL 파라미터 조작으로 다른 모임의 스케줄을 조작할 수 있는 IDOR 취약점이었다.

---

### `@Scheduled` — 자정에만 상태 변경, 다중 서버 중복 실행

```java
// ScheduleService.java
@Scheduled(cron = "0 0 0 * * *")
@Transactional
public void updateScheduleStatus() {
    int updatedCount = scheduleRepository.updateExpiredSchedules(
            ScheduleStatus.ENDED, ScheduleStatus.READY, now);
    log.info("{}개의 스케줄 상태가 READY에서 ENDED로 변경되었습니다.", updatedCount);
}
```

오후 2시에 종료된 스케줄이 다음날 자정까지 `READY` 상태로 남는다. `joinSchedule()`에 `scheduleTime.isBefore(now)` 방어 로직이 있기는 하지만, 체크와 홀드 사이의 시간 갭은 여전히 존재한다.

`@SchedulerLock`을 사용하지 않아 서버가 2대 이상이면 모든 인스턴스가 동시에 같은 `UPDATE`를 실행한다. 비즈니스 서비스 클래스 안에 배치 로직이 있는 구조도 관심사 분리 원칙에 맞지 않는다.

---

### 그 외 문제들

**`getScheduleList()` N+1**

스케줄 목록 조회 시 각 스케줄마다 `countBySchedule`과 `findByUserAndSchedule`을 따로 호출해 `1 + 2N` 쿼리가 나간다. 페이지네이션도 없이 모임의 모든 스케줄을 전체 조회한다.

**`leaveSchedule()` 멱등 처리의 부작용**

`SettlementStatus`가 `HOLD_ACTIVE`나 `FAILED`가 아니면 아무것도 하지 않고 `return`한다. 그런데 이 경우 `userScheduleRepository.delete()`와 `userChatRoomRepository.delete()`가 실행되지 않은 채 HTTP 200을 반환한다. 탈퇴가 된 것처럼 보이지만 실제로는 참여자로 남아 있는 유령 상태가 된다.

**`club.getSchedules().remove(schedule)` — 전체 컬렉션 로딩**

`Club.schedules` 컬렉션에서 `remove()`를 호출하려면 JPA가 먼저 전체 `schedules`를 로딩해야 한다. 모임에 스케줄이 100개면 100개를 전부 SELECT한 뒤 1개를 제거한다.

**`UserSettlementRepository` 메서드명 불일치**

`findAllUserSettlementIdsBySettlementIdAndStatus`라는 이름인데 실제로는 `userId`를 반환한다. 이름이 암시하는 반환 타입과 실제가 다르다. 완전히 동일한 쿼리를 가진 `findActiveParticipantIds`도 중복 선언되어 있었다.

---

## 해결

### `deleteSchedule()` — 삭제 전 3단계 정리

`deleteSchedule()`에 지갑 해제와 정산 정리를 추가했다.

```java
// ScheduleCommandService.java
public void deleteSchedule(Long clubId, Long scheduleId) {
    // ...
    // 1. 참여자 지갑 홀드 일괄 해제
    walletHoldService.batchRelease(memberUserIds, schedule.getCost());

    // 2. Schedule + UserSchedule 삭제 (CascadeType.ALL + orphanRemoval)
    scheduleRepository.delete(schedule);

    // 3. Settlement + UserSettlement 삭제 (이벤트로 분리)
    eventPublisher.publishEvent(new ScheduleDeletedEvent(scheduleId));
}
```

`ScheduleDeletedEvent`를 수신한 `SettlementScheduleEventListener`가 `settlementRepository.deleteByScheduleId()`를 실행한다. 삭제 순서를 지갑 해제 → 스케줄 삭제 → 정산 정리로 잡았다.

이벤트 리스너가 실패하는 경우(`deleteByScheduleId()` 예외)는 Settlement 레코드가 고아로 남을 수 있다. 이 케이스는 `@TransactionalEventListener(AFTER_COMMIT)` 기반이라 스케줄 삭제 자체는 이미 커밋된 상태이므로 트랜잭션 롤백으로 되돌릴 수 없다. 현재는 리스너 실패 시 에러 로그를 남기고, 주기적으로 `settlement WHERE schedule_id NOT IN (SELECT id FROM schedule)`를 스캔하는 배치로 고아 레코드를 정리하는 방식으로 수용했다.

---

### `joinSchedule()` — 3중 방어

단일 방어로는 Race Condition을 막을 수 없어서 세 겹으로 잡았다.

```java
// 1. 비관적 락
Schedule schedule = scheduleRepository.findByIdWithLock(scheduleId);
// @Lock(LockModeType.PESSIMISTIC_WRITE)

// 2. DB Unique Constraint
@UniqueConstraint(name = "uk_user_schedule", columnNames = {"user_id", "schedule_id"})

// 3. 예외 복구
} catch (DataIntegrityViolationException e) {
    walletHoldService.releaseOrThrow(userId, schedule.getCost());
    throw new CustomException(ErrorCode.ALREADY_JOINED_SCHEDULE);
}
```

비관적 락으로 같은 스케줄에 대한 동시 요청을 직렬화하고, Unique Constraint로 DB 레벨에서 중복을 차단한다. 락을 뚫고 중복 INSERT가 발생하더라도 `DataIntegrityViolationException`을 잡아 지갑 홀드를 즉시 해제한다.

---

### `createSchedule()` — 권한 체크 먼저

```java
// ScheduleCommandService.java
public ScheduleCreateResponseDto createSchedule(Long clubId, ScheduleRequestDto requestDto) {
    Club club = findClubOrThrow(clubId);
    User user = userService.getCurrentUser();
    UserClub userClub = userClubRepository.findByUserAndClub(user, club)
            .orElseThrow(() -> new CustomException(ErrorCode.CLUB_NOT_JOIN));
    if (userClub.getClubRole() != ClubRole.LEADER) {
        throw new CustomException(ErrorCode.MEMBER_CANNOT_CREATE_SCHEDULE);  // 먼저
    }
    Schedule schedule = requestDto.toEntity(club);
    scheduleRepository.save(schedule);  // 통과 후 저장
    // ...
}
```

권한 체크를 `save()` 앞으로 옮겼다. 권한 없는 사용자의 요청은 DB에 닿기 전에 끊긴다.

---

### IDOR 차단 — `validateScheduleBelongsToClub()` 공통 검증

```java
// ScheduleCommandService.java
private void validateScheduleBelongsToClub(Schedule schedule, Long clubId) {
    if (!schedule.getClub().getClubId().equals(clubId)) {
        throw new CustomException(ErrorCode.SCHEDULE_NOT_FOUND);
    }
}
```

`updateSchedule`, `joinSchedule`, `deleteSchedule` 세 곳에 이 검증을 추가했다. 기존에 `leaveSchedule`, `getScheduleDetails`, `getScheduleUserList`에 개별적으로 적용되어 있던 것까지 합쳐 6개 메서드 전부 커버된다.

---

### N+1 제거 + `leaveSchedule()` 재구성

`getScheduleList()`는 `ScheduleQueryService`로 분리하고, `COUNT`와 `scheduleRole`을 서브쿼리로 포함한 단일 JPQL로 교체해 `1 + 2N` 쿼리를 1개로 줄였다.

`leaveSchedule()`은 early return 로직을 제거하고 정상 검증 흐름으로 재구성했다. `UserSchedule` 없으면 예외 → LEADER 탈퇴 차단 → 지갑 해제 → 삭제 → `ScheduleLeftEvent` 발행 순서로 흘러가며, HTTP 200을 받았는데 실제로는 탈퇴가 안 된 유령 상태가 발생하지 않는다.

`club.getSchedules().remove(schedule)`는 `scheduleRepository.delete(schedule)` 직접 삭제로 교체해 불필요한 컬렉션 전체 로딩을 없앴다.

`@Scheduled` 배치는 `ScheduleBatchService`로 분리해 비즈니스 서비스에서 꺼냈다.

---

## 현재 상태

| 문제 | 상태 |
|------|------|
| `deleteSchedule()` 지갑/정산 미처리 | ✅ 해결 — 3단계 정리 추가 |
| `joinSchedule()` Race Condition | ✅ 해결 — 비관적 락 + Unique Constraint + 예외 복구 |
| 권한 체크 전 DB 저장 | ✅ 해결 — 체크 순서 반전 |
| `Long !=` 참조 비교 | ✅ 해결 — `.equals()`로 변경 |
| IDOR — 소속 검증 없음 | ✅ 해결 — 6개 메서드 전부 검증 추가 |
| `getScheduleList()` N+1 | ✅ 해결 — 단일 JPQL로 교체 |
| `leaveSchedule()` 유령 참여자 | ✅ 해결 — early return 제거 + 흐름 재구성 |
| 배치 로직 비즈니스 서비스 안에 혼재 | ✅ 해결 — `ScheduleBatchService` 분리 |
| 전체 컬렉션 로딩 삭제 | ✅ 해결 — 직접 삭제로 교체 |
| 메서드명 불일치 + 중복 | ✅ 해결 — 이름 수정 + 중복 제거 |
| `@Builder.Default` 누락 | ✅ 해결 — 적용 |
| `ScheduleDetailResponseDto` status 누락 | ✅ 해결 — 필드 추가 |
| `updateSchedule()` 비용 변경 부분 실패 정합성 | ⚠️ 잔존 — 비용 변경 기능 운영 환경 비활성화로 수용 |
| `ScheduleDeletedEvent` 리스너 실패 시 고아 Settlement | ⚠️ 잔존 — 배치 스캔으로 정리 예정 |
| `@Scheduled` 다중 서버 중복 실행 | ⚠️ 잔존 — `@SchedulerLock` 미적용 |
| 목록 조회 페이지네이션 없음 | ⚠️ 잔존 — 비즈니스 정책 결정 필요 |

---

## 교훈

`joinSchedule()`의 Race Condition은 찾기 쉽지 않았다. 로컬에서는 동시 요청을 흉내 내기 어렵고, 단위 테스트는 순차적으로 실행되니 보이지 않는다. "조회하고 체크하고 저장한다"는 패턴 자체가 Race Condition의 씨앗이다. DB 레벨의 Unique Constraint가 없으면 애플리케이션 레벨의 체크는 동시 요청 앞에서 무력하다.

`deleteSchedule()`의 지갑 동결 버그는 더 고전적인 문제였다. `joinSchedule()`이 홀드를 걸었으면 `deleteSchedule()`이 풀어줘야 하는데, 만든 사람과 지운 사람이 달랐거나 순서가 뒤바뀌어 그냥 넘어간 것이다. [모임 도메인 편](/club-concurrency-ghost-members/)의 `leaveClub()`에서 채팅방을 지우지 않은 것과 같은 패턴이다.

> **생성과 해제는 항상 짝을 이뤄야 한다.**
> 지갑에 홀드를 걸었다면 해제하는 경로가 반드시 존재해야 한다. 정상 흐름뿐 아니라 삭제, 취소, 예외 상황에서도 빠짐없이.

---

## 시리즈 탐색

**◀ 이전 글**  
[모임 도메인 — 동시성 구멍과 유령 멤버](/club-concurrency-ghost-members/)

**다음 글 ▶**  
[지갑·결제 도메인 — 토스 승인 후 DB가 죽으면 자금이 유실될 수 있다](/wallet-payment-toss-db-failure/)
