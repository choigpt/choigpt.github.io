---
title: 스케줄 도메인 — 권한 체크 순서 하나가 자금 동결로 이어지는 구조
date: 2026-02-19
tags: [JPA, Spring, 동시성, 트러블슈팅, Java, 스케줄, 정산]
---

# 스케줄 도메인 — 권한 체크 순서 하나가 자금 동결로 이어지는 구조

## 개요

스케줄(일정) 코드를 다시 살펴보다가 `createSchedule()`에서 이상한 순서를 발견했다. DB에 먼저 저장하고, 그 다음에 권한을 확인하고 있었다. `@Transactional`로 롤백되니까 기능상 문제는 없지만, 권한이 없는 사용자의 요청마다 쓸데없이 INSERT가 나간다.

그런데 `deleteSchedule()`을 보다가 훨씬 심각한 걸 발견했다. 스케줄을 삭제할 때 참여자들의 지갑 홀드금을 해제하는 코드가 없었다. 스케줄이 삭제되면 그 시점에 지갑에 잡혀 있던 예약금이 영구적으로 동결된다. 사용자 자금이 묶이는 문제였다.

여기에 `joinSchedule()`의 동시성 구멍까지 있었다. 정원 체크와 중복 참여 체크 사이에 시간 갭이 있고, `user_schedule` 테이블에 Unique Constraint가 없어서 동시에 두 요청이 들어오면 같은 사용자가 2번 참여하고 예약금도 2배 빠져나간다.

---

## 버그 상세

### `deleteSchedule()` — 스케줄 삭제 시 참여자 예약금 영구 동결

```java
public void deleteSchedule(Long clubId, Long scheduleId) {
    club.getSchedules().remove(schedule);   // Schedule 삭제
    chatRoomRepository.delete(chatRoom);    // ChatRoom 삭제
    // ❌ walletHoldService.release() 없음
    // ❌ Settlement 삭제 없음
}
```

`joinSchedule()`에서는 지갑에 예약금을 홀드하지만, `deleteSchedule()`은 홀드를 풀지 않는다. 스케줄이 삭제되면 참여자들의 지갑에 잡힌 금액이 `pending_out` 상태로 영구 잠긴다.

### `joinSchedule()` Race Condition — 중복 참여 시 2배 과금

```java
int userCount = userScheduleRepository.countBySchedule(schedule);          // ① SELECT COUNT
if (userCount >= schedule.getUserLimit()) { throw ... }
if (userScheduleRepository.findByUserAndSchedule(user, schedule).isPresent()) { throw ... }  // ② SELECT
// ... 시간 갭
int flag = walletRepository.holdBalanceIfEnough(userId, schedule.getCost());  // ③ UPDATE
userScheduleRepository.save(userSchedule);  // ④ INSERT
```

`user_schedule` 테이블에 `(user_id, schedule_id)` Unique Constraint가 없다. 같은 사용자가 동시에 두 번 요청을 보내면 둘 다 `findByUserAndSchedule()`에서 `Optional.empty()`를 받고 `save()`까지 진행한다. 중복 `UserSchedule`이 생기고 지갑에서 예약금이 2배 빠져나간다.

### `createSchedule()` — 권한 체크 전에 DB 저장

```java
scheduleRepository.save(schedule);                          // ① 먼저 저장
User user = userService.getCurrentUser();
if (userClub.getClubRole() != ClubRole.LEADER) {
    throw new CustomException(ErrorCode.MEMBER_CANNOT_CREATE_SCHEDULE);  // ② 나중에 체크
}
```

### IDOR — URL의 `clubId`와 `scheduleId` 소속 관계 미검증

`clubId=1`, `scheduleId=99`로 요청을 보낼 때, `schedule 99`가 실제로 `club 2`에 속하더라도 삭제가 실행된다. URL 파라미터 조작으로 다른 모임의 스케줄을 조작할 수 있는 IDOR 취약점이었다.

---

## 해결

### `deleteSchedule()` — 삭제 전 3단계 정리

```java
public void deleteSchedule(Long clubId, Long scheduleId) {
    // 1. 참여자 지갑 홀드 일괄 해제
    walletHoldService.batchRelease(memberUserIds, schedule.getCost());
    // 2. Schedule + UserSchedule 삭제
    scheduleRepository.delete(schedule);
    // 3. Settlement + UserSettlement 삭제 (이벤트로 분리)
    eventPublisher.publishEvent(new ScheduleDeletedEvent(scheduleId));
}
```

### `joinSchedule()` — 3중 방어

```java
// 1. 비관적 락
Schedule schedule = scheduleRepository.findByIdWithLock(scheduleId);

// 2. DB Unique Constraint
@UniqueConstraint(name = "uk_user_schedule", columnNames = {"user_id", "schedule_id"})

// 3. 예외 복구
} catch (DataIntegrityViolationException e) {
    walletHoldService.releaseOrThrow(userId, schedule.getCost());
    throw new CustomException(ErrorCode.ALREADY_JOINED_SCHEDULE);
}
```

### IDOR 차단

```java
private void validateScheduleBelongsToClub(Schedule schedule, Long clubId) {
    if (!schedule.getClub().getClubId().equals(clubId)) {
        throw new CustomException(ErrorCode.SCHEDULE_NOT_FOUND);
    }
}
```

6개 메서드 전부 커버.

---

## 교훈

> **생성과 해제는 항상 짝을 이뤄야 한다.**
> 지갑에 홀드를 걸었다면 해제하는 경로가 반드시 존재해야 한다. 정상 흐름뿐 아니라 삭제, 취소, 예외 상황에서도 빠짐없이.
