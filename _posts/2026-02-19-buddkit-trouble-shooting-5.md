---
title: 모임 도메인 — 동시성 구멍과 유령 멤버
date: 2026-02-19
tags: [JPA, Spring, 동시성, 트러블슈팅, Java, 모임]
---

# 모임 도메인 — 동시성 구멍과 유령 멤버

## 개요

모임(Club) 코드를 다시 살펴보던 중, 멤버 수를 관리하는 방식에 심각한 동시성 문제가 있다는 걸 발견했다. `memberCount++`를 JPA dirty checking으로 처리하고 있어서 동시 요청 시 Lost Update가 발생한다. 거기에 `user_club` 테이블에 Unique Constraint가 없어서 동시에 두 요청이 들어오면 중복 가입이 가능했다.

더 황당한 건 `leaveClub()`이었다. 탈퇴 시 `UserClub`만 삭제하고 `UserChatRoom`은 삭제하지 않아서, 탈퇴한 사용자가 채팅방에 유령 멤버로 남아 메시지를 계속 수신했다. 가입할 때는 채팅방에 넣어주고 탈퇴할 때는 빼주지 않은 것이다.

---

## 시스템 구조

### 모임 가입/탈퇴 흐름

```
joinClub()
    ↓
중복 가입 여부 확인 (findByUserAndClub)
    ↓
정원 확인 (countByClub_ClubId)
    ↓
UserClub 저장 + memberCount 증가
    ↓
UserChatRoom 생성 (채팅방 자동 입장)
    ↓
Elasticsearch 인덱싱 (@Async)
```

탈퇴는 이 흐름의 역순이어야 한다. `leaveClub()`이 `UserClub` 삭제, `memberCount` 감소, ES 인덱싱까지는 하고 있었지만 `UserChatRoom` 삭제는 없었다.

---

## 버그 상세

### `memberCount` Race Condition — Lost Update

```java
// Club.java
public void incrementMemberCount() {
    this.memberCount++;  // Java 메모리상 ++, DB 락 없음
}
```

`memberCount++`는 JPA dirty checking을 통해 `UPDATE club SET member_count = ? WHERE club_id = ?`로 실행된다. `?`에 들어가는 값은 Java 메모리에서 계산한 값이다.

```
Thread A: SELECT → memberCount=5 → this.memberCount++ → 6
Thread B: SELECT → memberCount=5 → this.memberCount++ → 6
Thread A: UPDATE SET member_count=6 → 커밋
Thread B: UPDATE SET member_count=6 → 커밋 → Lost Update
```

실제로는 7이어야 하는데 6이 된다. `@Transactional`이 있어도 MySQL InnoDB의 기본 격리 수준(REPEATABLE READ)에서는 해결되지 않는다. 두 트랜잭션이 각자 읽은 값에서 계산하므로 서로의 변경을 덮어쓴다.

---

### `user_club` Unique Constraint 없음 — 중복 가입 가능

```java
// UserClub.java
@Table(name = "user_club", indexes = {
    @Index(name = "idx_user_club_user", columnList = "user_id"),
    @Index(name = "idx_user_club_club", columnList = "club_id"),
    @Index(name = "idx_user_club_user_club", columnList = "user_id, club_id")  // UNIQUE가 아닌 일반 인덱스
})
```

```java
// ClubService.java
if (userClubRepository.findByUserAndClub(user, club).isPresent()) {
    throw new CustomException(ErrorCode.ALREADY_JOINED_CLUB);
}
// ... save()
```

`findByUserAndClub`과 `save` 사이에 시간 갭이 존재한다. 두 요청이 동시에 들어오면 둘 다 `Optional.empty()`를 받고 `save`까지 진행해 중복 `UserClub` 레코드가 생긴다. `memberCount`도 2번 증가한다. DB 레벨의 제약이 없으니 애플리케이션 체크가 무의미해진다.

---

### `leaveClub()` — 탈퇴해도 채팅방에 유령 멤버로 잔류

```java
// ClubService.java
public void leaveClub(Long clubId) {
    // ...
    userClubRepository.delete(userClub);    // UserClub만 삭제
    club.decrementMemberCount();
    clubElasticsearchService.updateClub(club);
    // ❌ UserChatRoom 삭제 코드가 없음
}
```

`joinClub()`에서는 `UserChatRoom`을 생성해 채팅방에 입장시키는데, `leaveClub()`에서는 삭제하지 않는다. 탈퇴한 사용자가 채팅방 멤버 목록에 계속 표시되고 메시지도 계속 수신한다.

---

### `@Async` + Lazy Proxy — ES 인덱싱이 매번 실패

```java
// ClubElasticsearchService.java
@Async
@Retryable
public void indexClub(Club club) {
    ClubDocument document = ClubDocument.from(club);  // 여기서 lazy 접근
}

// ClubDocument.java
Category category = club.getInterest().getCategory();  // Interest는 LAZY
```

`createClub()`에서 `indexClub(club)`을 호출한다. `@Async`이므로 별도 스레드에서 실행되는데, 이때 원래 트랜잭션은 이미 커밋된 상태다. `club.getInterest()`는 LAZY 프록시이므로 별도 스레드에서 접근하면 영속성 컨텍스트가 없어 `LazyInitializationException`이 발생한다.

`@Retryable`로 재시도를 3번 반복하지만 매번 같은 에러가 난다. 그리고 `@Async` 스레드에서 발생한 예외는 호출자에게 전파되지 않으므로 모임은 생성됐는데 ES 인덱싱은 조용히 실패한 상태가 된다.

---

### 정원 체크 Race Condition — 정원 초과 가입 가능

```java
// ClubService.java
public void joinClub(Long clubId) {
    Club club = clubRepository.findById(clubId).orElseThrow(...);
    int userCount = userClubRepository.countByClub_ClubId(club.getClubId());  // COUNT 쿼리
    if (userCount >= club.getUserLimit()) {
        throw new CustomException(ErrorCode.CLUB_NOT_ENTER);
    }
    // ... save()
}
```

`COUNT` 쿼리 후 `INSERT` 사이에 다른 사용자가 가입할 수 있다. 정원이 100명인 모임에 100번째와 101번째 요청이 동시에 들어오면 둘 다 `userCount=99`를 읽고 체크를 통과해 101명이 가입된다.

`memberCount` 컬럼과 `COUNT` 쿼리가 혼용되는 문제도 있었다. 정원 체크는 `countByClub_ClubId`로 실제 row 수를 세고, 화면에 표시하는 멤버 수는 `memberCount` 컬럼을 쓴다. Race Condition으로 두 값이 달라지면 같은 모임인데 API마다 멤버 수가 다르게 보인다.

---

### LEADER 이전 불가 + 모임 삭제 불가 — 구조적 막힘

```java
// ClubService.java
if (userClub.getClubRole() == ClubRole.LEADER) {
    throw new CustomException(ErrorCode.CLUB_LEADER_NOT_LEAVE);
}
```

LEADER는 탈퇴할 수 없다. 그런데 LEADER를 다른 사람에게 이전하는 기능도 없고, 모임을 삭제하거나 해산하는 기능도 없다. 모임을 한 번 만들면 LEADER는 영구적으로 해당 모임에 묶인다.

---

### `getCurrentUserId()`가 `kakaoId`를 반환 — `isJoined` 항상 false

1화에서 확인된 버그가 모임 검색에도 영향을 미치고 있었다.

```java
// SearchService.java
Long userId = userService.getCurrentUserId();  // 실제로는 kakaoId 반환
List<Long> joinedClubIds = userClubRepository.findByClubIdsByUserId(userId);  // userId(DB PK) 기준 조회
```

`getCurrentUserId()`는 SecurityContext에서 `kakaoId`를 반환하는데, `findByClubIdsByUserId`는 DB PK인 `userId` 기준으로 조회한다. `kakaoId ≠ userId`이므로 `joinedClubIds`는 항상 빈 리스트가 된다. 검색 결과의 모든 모임에 `isJoined=false`가 표시된다. `searchClubByInterest()`, `searchClubByLocation()`, `searchClubs()` 세 곳 모두 같은 문제였다.

---

### 그 외 문제들

**`memberCount` 초기값 null**

```java
// Club.java
private Long memberCount = 0L;  // 필드 초기화

// ClubRequestDto.java
public Club toEntity(Interest interest) {
    return Club.builder()
            .name(name)
            .userLimit(userLimit)
            // memberCount 없음 + @Builder.Default 없음
            .build();
}
```

`@Builder`는 필드 초기화 값을 무시한다. `@Builder.Default`가 없으면 `toEntity()`로 생성한 `Club`의 `memberCount`는 `null`이 된다. 이후 `incrementMemberCount()`에서 `null++`로 `NullPointerException`이 발생한다. `chatRooms`, `feeds`, `schedules`에는 `@Builder.Default`가 붙어 있는데 `memberCount`에만 빠져 있었다.

**`findMyClubs` 상관 서브쿼리**

```java
// UserClubRepository.java
@Query("""
select c,
       (select count(uc2) from UserClub uc2 where uc2.club = c)  // 상관 서브쿼리
from UserClub uc join uc.club c
where uc.user.userId = :userId
""")
```

`memberCount` 컬럼이 있는데 매번 상관 서브쿼리로 COUNT를 계산한다. 가입한 모임이 N개면 N번의 COUNT 서브쿼리가 실행된다.

**`Object[]` 반환 타입 — 배열 구조도 메서드마다 다름**

`ClubRepositoryCustom`의 6개 메서드가 모두 `List<Object[]>`를 반환한다. 문제는 배열 구조가 메서드마다 달랐다는 것이다. `searchByKeywordWithFilter`는 7개의 원시 필드를, 나머지는 `[Club, Long]`을 담는다. `SearchService`에서 `(Club) result[0]`으로 캐스팅하므로 `searchByKeywordWithFilter` 결과를 넘기면 `ClassCastException`이 발생한다.

**잘못된 ErrorCode**

```java
// ClubService.java
if (userClub.getClubRole() != ClubRole.LEADER) {
    throw new CustomException(ErrorCode.MEMBER_CANNOT_MODIFY_SCHEDULE);  // 일정 에러코드
}
```

모임 수정 권한 체크에 일정 관련 에러코드를 사용한다. 프론트엔드에서 이 에러를 받으면 "리더만 정기 모임을 수정할 수 있습니다"라는 메시지가 뜬다.

**`ClubLike` 미사용 엔티티**

`ClubLike` 엔티티와 `club_like` 테이블이 존재하지만 참조하는 Repository, Service, Controller가 전무하다. Feed 도메인처럼 Redis + Lua로 구현하려다 미완성으로 남은 것으로 보인다.

---

## 해결

### `memberCount` 원자적 업데이트 — Java `++` 제거

`Club.java`의 `incrementMemberCount()`, `decrementMemberCount()` 메서드를 제거하고 DB 레벨의 원자적 업데이트 쿼리로 교체했다.

```java
// ClubRepository.java
@Modifying
@Query("UPDATE Club c SET c.memberCount = c.memberCount + 1 WHERE c.clubId = :clubId")
void incrementMemberCount(@Param("clubId") Long clubId);

@Modifying
@Query("UPDATE Club c SET c.memberCount = CASE WHEN c.memberCount > 0 THEN c.memberCount - 1 ELSE 0 END WHERE c.clubId = :clubId")
void decrementMemberCount(@Param("clubId") Long clubId);
```

`SET member_count = member_count + 1`은 DB가 현재 값을 읽고 더하는 단일 연산이므로 Lost Update가 발생하지 않는다.

---

### `user_club` Unique Constraint 추가 — DB 레벨 중복 차단

```java
// UserClub.java
@Table(name = "user_club",
    uniqueConstraints = @UniqueConstraint(name = "uk_user_club", columnNames = {"user_id", "club_id"}),
    indexes = { ... }
)
```

애플리케이션 레벨의 중복 체크는 유지하되, DB 레벨에서도 제약을 걸었다. 동시 요청이 체크를 동시에 통과해도 `save` 시점에 `DataIntegrityViolationException`이 발생해 중복 가입이 차단된다.

---

### `leaveClub()` — `UserChatRoom` 삭제 추가

```java
// ClubCommandService.java
public void leaveClub(Long clubId) {
    // ...
    userChatRoomRepository.deleteByUserAndChatRoom(user, chatRoom);  // 추가
    userClubRepository.delete(userClub);
    clubRepository.decrementMemberCount(clubId);
}
```

`joinClub()`과 `leaveClub()`이 대칭적으로 동작하게 됐다. 탈퇴 후 채팅방에 유령 멤버로 남는 문제가 해소됐다.

---

### `@Async` Lazy Proxy — DTO로 변환 후 전달

`indexClub(Club club)` 대신 필요한 데이터를 미리 추출한 DTO를 넘기도록 변경했다.

```java
// ClubCommandService.java (트랜잭션 안에서)
ClubIndexDto dto = ClubIndexDto.from(club);  // Lazy 로딩이 트랜잭션 안에서 발생
clubElasticsearchService.indexClub(dto);     // @Async 메서드는 DTO만 받음
```

트랜잭션이 살아 있는 시점에 `club.getInterest().getCategory()`가 실행되므로, `@Async` 스레드에서 Lazy 프록시에 접근할 일이 없다.

---

### 타입 안전성 확보 — `Object[]`에서 Record로

`ClubRepositoryCustom`의 모든 메서드 반환 타입을 `ClubWithMemberCount` record로 통일했다.

```java
record ClubWithMemberCount(Club club, Long memberCount) {}
```

인덱스 기반 캐스팅이 사라지고, 컴파일 타임에 타입이 보장된다. 메서드마다 배열 구조가 달라 `ClassCastException`이 날 수 있던 문제도 함께 해결됐다.

`findMyClubs`의 상관 서브쿼리도 `c.memberCount`를 직접 읽도록 변경해 N번의 COUNT 서브쿼리를 제거했다.

---

## 현재 상태

| 문제 | 상태 |
|------|------|
| `memberCount` Race Condition | ✅ 해결 — 원자적 UPDATE 쿼리로 교체 |
| `user_club` Unique Constraint 없음 | ✅ 해결 — DB 레벨 제약 추가 |
| `leaveClub()` `UserChatRoom` 미삭제 | ✅ 해결 — 삭제 코드 추가 |
| `@Async` Lazy Proxy | ✅ 해결 — DTO 변환 후 전달 |
| `memberCount` 초기값 null | ✅ 해결 — `@Builder.Default` 추가 |
| `getCurrentUserId()` → `kakaoId` 반환 | ✅ 해결 — `UserPrincipal.getUserId()`로 수정 |
| `findMyClubs` 상관 서브쿼리 | ✅ 해결 — `memberCount` 컬럼 직접 사용 |
| `Object[]` 반환 + 구조 불일치 | ✅ 해결 — `ClubWithMemberCount` record 통일 |
| 잘못된 ErrorCode | ✅ 해결 — `CLUB_MODIFY_FORBIDDEN`으로 변경 |
| 데드 코드 (`createOrderSpecifiers`) | ✅ 해결 — 제거 |
| 중복/미사용 import | ✅ 해결 — 정리 |
| 정원 체크 Race Condition | ⚠️ 잔존 — SELECT + INSERT 원자성 보장 필요 |
| LEADER 이전 / 모임 삭제 불가 | ⚠️ 잔존 — 기능 미구현 |
| `ClubLike` 미사용 엔티티 | ⚠️ 잔존 — 구현 또는 제거 결정 필요 |
| `List.contains()` O(n) | ⚠️ 잔존 — `Set`으로 변환 필요 |

---

## 교훈

`memberCount++`와 `user_club` Unique Constraint 누락은 따로 보면 작은 실수처럼 보인다. 그런데 둘이 겹치면 동시에 두 명이 가입할 때 중복 레코드가 생기고, 멤버 수도 2번 증가한다. 동시성 문제는 단독으로 보면 재현하기 어렵고, 조합되면 데이터가 조용히 틀어지기 때문에 테스트로 잡기가 가장 어렵다.

`leaveClub()`의 `UserChatRoom` 미삭제는 코드를 보면 바로 눈에 띄는 문제인데 오래 남아 있었다. `joinClub()`을 작성할 때 만든 짝을 `leaveClub()`에서 풀어주지 않은 것이다. 무언가를 만들었다면 그것을 해제하는 경로도 반드시 함께 만들어야 한다.

> **대칭을 지켜라.**
> 생성과 삭제, 가입과 탈퇴, 열기와 닫기는 항상 쌍으로 존재해야 한다. 한쪽만 구현하면 반드시 유령이 남는다.
