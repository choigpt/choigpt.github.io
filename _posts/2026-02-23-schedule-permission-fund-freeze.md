---
layout: post
title: 스케줄 도메인 — 권한 체크 순서 하나가 자금 동결로 이어지는 구조
date: 2026-02-23
tags: [JPA, Spring, 동시성, 트러블슈팅, Java, 스케줄, 정산]
permalink: /schedule-permission-fund-freeze/
excerpt: "스케줄 삭제 시 참여자 지갑 홀드금을 해제하는 코드가 없어, 삭제된 스케줄의 예약금이 영구적으로 동결될 수 있는 구조였다."
---

## 한눈에 보기

참고: 이 프로젝트의 정산/지갑 시스템은 실제 결제가 아니라 **가입 시 가상 포인트(가짜 돈)를 지급**하는 방식으로 테스트용으로 운영되었다. 그래서인지 지갑 홀드/해제 같은 자금 관련 로직의 검증이 다소 느슨하게 남아있었던 것으로 보인다.

```
이 글에서 다루는 문제 (6건):

1. deleteSchedule() — 지갑 홀드 해제 없음 → 예약금 영구 동결  ← 핵심 버그
2. joinSchedule() — Unique 제약 없음 → 중복 참여 + 2배 과금
3. createSchedule() — 권한 체크 전에 DB 저장
4. updateSchedule() — Long != 참조 비교 → 비용 미변경인데 지갑 업데이트
5. clubId ↔ scheduleId 소속 관계 미검증 → 다른 모임 스케줄 조작 가능
6. @Scheduled — 자정 1회, 다중 서버 중복 실행
```

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
                  (❌ 지갑 홀드 해제 없음)
```

`joinSchedule()`이 지갑에 홀드를 걸고, `deleteSchedule()`이 이를 해제해야 하지만 해제하지 않았다. 생성과 해제가 비대칭 구조였다.

---

## 버그 상세

### `deleteSchedule()` — 스케줄 삭제 시 참여자 예약금 영구 동결

```java
// ScheduleService.java
public void deleteSchedule(Long clubId, Long scheduleId) {
    // ...
    club.getSchedules().remove(schedule);   // Schedule 삭제 (cascade로 UserSchedule, Settlement, UserSettlement도 삭제됨)
    chatRoomRepository.delete(chatRoom);    // ChatRoom 삭제
    // ❌ walletHoldService.release() 없음 — 지갑 홀드가 해제되지 않는다!
}
```

```
비대칭 구조:

joinSchedule()   → walletRepository.holdBalanceIfEnough()  ← 홀드를 건다
leaveSchedule()  → walletRepository.releaseHoldBalance()   ← 홀드를 푼다 ✓
deleteSchedule() → ❌ 해제 코드 없음!                       ← 홀드가 풀리지 않는다

결과:
  참여자 3명이 각각 5,000원 홀드된 상태에서 스케줄 삭제
  → Schedule, Settlement, UserSettlement 레코드는 JPA cascade로 삭제됨
  → 하지만 지갑의 pending_out은 별도 테이블이므로 cascade 대상이 아님
  → 15,000원이 pending_out 상태로 영구 잠김
```

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

```
동시 요청 시나리오 (중복 참여):

  요청 A                              요청 B
  ─────                              ─────
  countBySchedule() = 4  ← OK        countBySchedule() = 4  ← OK
  findByUser() = empty   ← OK        findByUser() = empty   ← OK
  holdBalance(5000)      ← 5000원 홀드 holdBalance(5000)     ← 또 5000원 홀드!
  save(UserSchedule)     ← INSERT    save(UserSchedule)     ← 또 INSERT!

  결과: 같은 사용자가 2번 참여, 예약금 10,000원 빠져나감
  원인: user_schedule에 (user_id, schedule_id) Unique 제약이 없음
```

정원 초과도 같은 구조로 발생할 수 있다:

```
  정원 5명, 현재 4명
  요청 A: countBySchedule() = 4 → OK
  요청 B: countBySchedule() = 4 → OK (아직 A가 INSERT 전)
  둘 다 통과 → 6명이 참여하게 됨
```

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

```
실행 순서:
  ① scheduleRepository.save(schedule)   ← INSERT 실행 (IDENTITY라 즉시)
  ② if (role != LEADER) throw ...        ← 권한 체크

  권한 없으면 → 예외 → @Transactional 롤백
  하지만 auto_increment 값은 이미 소모됨
  → 불필요한 DB I/O + 시퀀스 낭비

  해결: ①과 ②의 순서만 바꾸면 됨
```

---

### `updateSchedule()` — `Long !=` 참조 비교 + 비용 변경 부분 실패 시 정합성 문제

```java
// ScheduleService.java
if (schedule.getCost() != requestDto.getCost()) {  // Long 객체 참조 비교
    // 참여자 전원 지갑 업데이트 로직...
}
```

**문제 1: `Long !=` 참조 비교**

```
Long a = 1000L;
Long b = 1000L;
a != b  →  true!  (객체 주소가 다르니까)

Long은 -128~127 범위만 같은 객체를 재사용한다.
1000은 범위 밖이라 매번 새 객체가 만들어짐
→ 비용이 변경되지 않았는데도 != 가 true를 반환
→ 참여자 전원의 지갑 업데이트 로직에 진입

해결: .equals()로 변경
```

**문제 2: 비용 변경 시 중간 실패**

```
참여자 5명의 지갑을 순차적으로 홀드 업데이트:
  1번 → 성공 (지갑 업데이트됨)
  2번 → 성공
  3번 → 잔액 부족! → 예외 → 롤백 시도

  하지만 1번, 2번의 지갑은 이미 UPDATE됨
  → 단일 @Transactional 안이라면 롤백 시 모든 변경이 취소되며,
    REPEATABLE READ에서는 커밋 전 변경이 다른 트랜잭션에 보이지 않는다 (dirty read 불가).
  → 실제 위험은 지갑 업데이트가 별도 트랜잭션이나 native query로 개별 커밋되는 경우다.
    이 경우 1번·2번은 커밋되고 3번에서 실패하면, 이미 커밋된 변경은 롤백되지 않는다.

  → 비용 변경 기능 자체를 설계 레벨에서 재검토해야 하는 문제
  → 현재는 운영 환경에서 비활성화로 수용
```

---

### URL 파라미터 조작으로 다른 모임 스케줄 조작 가능

```java
// ScheduleService.java
public void deleteSchedule(Long clubId, Long scheduleId) {
    Club club = clubRepository.findById(clubId).orElseThrow(...);
    Schedule schedule = scheduleRepository.findById(scheduleId).orElseThrow(...);
    // ❌ schedule이 정말 이 club에 속하는지 확인하지 않음
}
```

```
요청: DELETE /api/clubs/1/schedules/99

schedule 99는 실제로 club 2에 속하는 스케줄.
하지만 코드는 clubId=1, scheduleId=99를 각각 따로 조회할 뿐
"schedule 99가 club 1 소속인가?"를 확인하지 않음

→ club 1의 리더가 club 2의 스케줄을 삭제할 수 있음!

영향 범위: updateSchedule, joinSchedule, deleteSchedule 세 곳 모두 동일한 구멍
```

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

```
문제 1: 상태 변경이 자정까지 지연됨
  오후 2시에 끝난 스케줄 → 다음날 0시까지 READY 상태로 남음
  joinSchedule()에 scheduleTime.isBefore(now) 방어가 있지만
  체크와 홀드 사이의 시간 갭은 여전히 존재

문제 2: 다중 서버에서 중복 실행
  @SchedulerLock 미사용
  → 서버 2대면 자정에 같은 UPDATE가 2번 실행됨

문제 3: 관심사 혼재
  비즈니스 서비스(ScheduleService) 안에 배치 로직(@Scheduled)이 섞여 있음
```

---

### 그 외 문제들

- **`getScheduleList()` N+1**: 스케줄마다 `countBySchedule` + `findByUserAndSchedule` 따로 호출 → `1+2N` 쿼리. 페이지네이션도 없음.
- **`leaveSchedule()` 유령 참여자**: status가 HOLD_ACTIVE/FAILED가 아니면 early return → delete 안 되는데 200 반환 → 탈퇴된 줄 알지만 참여자로 남음.
- **`club.getSchedules().remove()`**: 1개 제거하려고 전체 컬렉션 로딩. 스케줄 100개면 100개 SELECT 후 1개 제거.
- **메서드명 불일치**: `findAllUserSettlementIds...`인데 실제로 userId 반환. 동일 쿼리의 `findActiveParticipantIds`도 중복 존재.
- **`@Builder.Default` 누락**: 컬렉션 필드가 null 초기화 → 빌더 생성 시 NPE 가능.
- **DTO status 필드 누락**: `ScheduleDetailResponseDto`에 `scheduleStatus`가 없어 클라이언트가 상태를 알 수 없었음.

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

```
삭제 순서:
  1. 참여자 지갑 홀드 일괄 해제 (batchRelease)
  2. Schedule + UserSchedule 삭제 (Cascade)
  3. Settlement + UserSettlement 삭제 (이벤트로 분리)

이벤트 리스너 실패 시:
  → 스케줄 삭제는 이미 커밋됨 (AFTER_COMMIT 기반)
  → Settlement 레코드가 고아로 남을 수 있음
  → 에러 로그 + 배치 스캔으로 정리하는 방식으로 수용
```

> 참고: Schedule 삭제가 먼저 커밋되고 Settlement 삭제가 이벤트로 후처리되므로, Settlement → Schedule FK 제약이 있다면 순서를 조정하거나 FK를 제거해야 한다.

---

### `joinSchedule()` — 3중 방어

단일 방어로는 Race Condition을 막을 수 없으므로 3단계 방어를 적용했다.

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

```
방어 순서:
  1차: 비관적 락 → 같은 스케줄에 대한 동시 요청을 직렬화
  2차: Unique 제약 → 락을 뚫더라도 DB 레벨에서 중복 차단
  3차: 예외 복구 → 중복 INSERT 발생 시 지갑 홀드를 즉시 해제
```

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

```
수정 사항:

getScheduleList():
  1+2N 쿼리 → COUNT + scheduleRole을 서브쿼리로 포함한 단일 JPQL로 교체

leaveSchedule():
  early return 제거 → 정상 검증 흐름으로 재구성
  UserSchedule 없으면 예외 → LEADER 차단 → 지갑 해제 → 삭제
  → "200인데 실제로는 안 된" 유령 상태 방지

deleteSchedule():
  club.getSchedules().remove() → scheduleRepository.delete() 직접 삭제
  → 전체 컬렉션 로딩 제거

@Scheduled 배치:
  ScheduleService → ScheduleBatchService로 분리
```

---

## 현재 상태

- **`deleteSchedule()` 지갑/정산 미처리**: ✅ 해결 — 3단계 정리 추가
- **`joinSchedule()` Race Condition**: ✅ 해결 — 비관적 락 + Unique Constraint + 예외 복구
- **권한 체크 전 DB 저장**: ✅ 해결 — 체크 순서 반전
- **`Long !=` 참조 비교**: ✅ 해결 — `.equals()`로 변경
- **IDOR — 소속 검증 없음**: ✅ 해결 — 6개 메서드 전부 검증 추가
- **`getScheduleList()` N+1**: ✅ 해결 — 단일 JPQL로 교체
- **`leaveSchedule()` 유령 참여자**: ✅ 해결 — early return 제거 + 흐름 재구성
- **배치 로직 비즈니스 서비스 안에 혼재**: ✅ 해결 — `ScheduleBatchService` 분리
- **전체 컬렉션 로딩 삭제**: ✅ 해결 — 직접 삭제로 교체
- **메서드명 불일치 + 중복**: ✅ 해결 — 이름 수정 + 중복 제거
- **`@Builder.Default` 누락**: ✅ 해결 — 컬렉션 필드 초기화 추가
- **`ScheduleDetailResponseDto` status 누락**: ✅ 해결 — 필드 추가 및 매핑
- **`updateSchedule()` 비용 변경 부분 실패 정합성**: ⚠️ 잔존 — 비용 변경 기능 운영 환경 비활성화로 수용
- **`ScheduleDeletedEvent` 리스너 실패 시 고아 Settlement**: ⚠️ 잔존 — 배치 스캔으로 정리 예정
- **`@Scheduled` 다중 서버 중복 실행**: ⚠️ 잔존 — `@SchedulerLock` 미적용
- **목록 조회 페이지네이션 없음**: ⚠️ 잔존 — 비즈니스 정책 결정 필요

---

## 정리하며

`joinSchedule()`의 Race Condition은 로컬 환경에서 재현이 어렵다. 동시 요청을 시뮬레이션하기 어렵고, 단위 테스트는 순차적으로 실행되므로 드러나지 않는다. "조회하고 체크하고 저장한다"는 패턴 자체가 Race Condition의 원인이 된다. DB 레벨의 Unique Constraint가 없으면 애플리케이션 레벨의 체크는 동시 요청에 대해 무력하다.

`deleteSchedule()`의 지갑 동결 버그는 리소스 생성-해제 비대칭 문제에 해당한다. `joinSchedule()`이 홀드를 걸었으면 `deleteSchedule()`이 해제해야 하지만 해제하지 않았다. [모임 도메인 편](/club-concurrency-ghost-members/)의 `leaveClub()`에서 채팅방을 삭제하지 않은 것과 동일한 패턴이다.

> **생성과 해제는 항상 짝을 이뤄야 한다.**
> 지갑에 홀드를 걸었다면 해제하는 경로가 반드시 존재해야 한다. 정상 흐름뿐 아니라 삭제, 취소, 예외 상황에서도 빠짐없이.

지갑 잔액 업데이트의 native query 통일은 [지갑·결제 편](/wallet-payment-toss-db-failure/)에서 더 자세히 다룬다.

---

## 시리즈 탐색

**◀ 이전 글**
[모임 도메인 — 동시성 구멍과 유령 멤버](/club-concurrency-ghost-members/)

**▶ 다음 글**
[지갑 결제 도메인 — 토스 승인 후 DB가 죽으면 자금이 유실될 수 있다](/wallet-payment-toss-db-failure/)
