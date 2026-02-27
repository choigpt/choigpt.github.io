---
title: 피드 도메인 — Redis 좋아요 파이프라인의 허점과 N+1
date: 2026-02-21
tags: [Redis, JPA, Spring, 트러블슈팅, Java, 피드]
permalink: /feed-redis-like-n-plus-1/
excerpt: "좋아요 수를 읽는 소스가 Redis, DB like_count 컬럼, feed_like 테이블 세 군데로 나뉘어 서로 다른 값을 반환할 수 있었다. Redis를 도입했는데 조회는 여전히 JPA 컬렉션을 통하고 있었다."
---

## 개요

피드 코드를 다시 살펴보던 중, 좋아요 수를 표시하는 소스가 **세 군데**인데 세 군데가 각각 다른 값을 반환할 수 있다는 걸 발견했다. Redis 카운터, DB의 `like_count` 컬럼, `feed_like` 테이블의 row 수. 셋 중 아무것도 실제 화면에 표시되는 값의 정답이라고 보장할 수 없었다.

좋아요 성능을 높이려고 Redis + Stream + Consumer로 이어지는 비동기 파이프라인을 도입했는데, 각 단계에서 검증과 복구 로직이 빠지면서 오히려 데이터가 세 곳으로 쪼개져버린 것이다. 여기에 N+1 쿼리, 피드 접근 권한 구멍까지 겹쳐 있었다.

---

## 시스템 구조

### 좋아요 파이프라인 (수정 전)

```
클라이언트 → toggleLike()
    ↓
Redis Lua 스크립트 (feed:{feedId}:likers Set, feed:{feedId}:like_count 카운터 즉시 업데이트)
    ↓
Redis Stream에 이벤트 발행 (ON/OFF)
    ↓
FeedLikeStreamConsumer (비동기)
    ↓
like_applied 테이블 INSERT IGNORE (멱등성 보장)  ← 이후 제거됨
    ↓
feed.like_count 업데이트 + feed_like INSERT/DELETE
```

문제는 이 파이프라인의 끝에 있는 `feed.like_count`와 `feed_like` 테이블을 **조회 API가 전혀 사용하지 않는다**는 것이었다.

### 좋아요 파이프라인 (수정 후)

```
클라이언트 → FeedLikeService.toggleLike()
    ↓
ensureLikeCacheWarmed(feedId)  ← 키 없으면 DB에서 워밍업
    ↓
Redis Lua 스크립트 (feed:{feedId}:likers Set, feed:{feedId}:like_count 카운터 즉시 업데이트)
    ↓
Redis Stream에 이벤트 발행 (ON/OFF)
    ↓
FeedLikeStreamConsumer (비동기)
    ↓
feed.like_count 업데이트 + feed_like INSERT/DELETE
```

`like_applied` 테이블을 제거했다. 멱등성은 `feed_like` 테이블의 `(user_id, feed_id)` Unique Constraint와 Consumer의 `INSERT IGNORE` / `DELETE WHERE EXISTS` 패턴으로 보장한다.

---

## 버그 상세

### 좋아요 수를 읽는 소스가 세 군데 — 셋 다 다른 값

모든 조회 API는 `feed.getFeedLikes().size()`를 사용했다. `feed_like` 테이블의 row 수를 컬렉션으로 전부 올려서 `.size()`를 부르는 방식이다.

```java
// FeedDetailResponseDto.from()
feed.getFeedLikes().size()      // ← feed_like 테이블 row 수

// FeedOverviewDto (FeedMainService)
safeSize(f.getFeedLikes())      // ← 동일
```

반면 Consumer는 `feed.like_count` 컬럼을 업데이트하고, Redis는 별도 카운터 `feed:{feedId}:like_count`를 관리한다.

```
Redis 카운터    → toggleLike() 즉시 반영
feed.like_count → Consumer가 비동기로 업데이트
feed_like rows  → Consumer가 비동기로 INSERT/DELETE
```

Consumer가 1초만 지연되어도 세 값이 전부 달라진다. 그리고 `feed.like_count`는 Consumer가 업데이트하지만 조회 코드 어디서도 읽지 않았다. 만들어두고 아무도 쓰지 않는 컬럼이었다.

---

### 존재하지 않는 피드에도 좋아요가 가능 — Consumer PEL 적체로 이어짐

`toggleLike()`는 피드가 실제로 존재하는지 DB에서 확인하지 않았다.

```java
public boolean toggleLike(long clubId, long feedId) {
    Club club = clubRepository.findById(clubId)
            .orElseThrow(() -> new CustomException(ErrorCode.CLUB_NOT_FOUND));
    long userId = userService.getCurrentUser().getUserId();
    // feedId로 DB 조회 없음 — 바로 Redis Lua 실행
}
```

존재하지 않는 `feedId`에 요청이 오면 Redis에 `feed:{feedId}:likers`와 `feed:{feedId}:like_count` 키가 생성된다. 이후 Consumer가 Stream에서 이벤트를 읽고 `feed_like` 테이블에 INSERT를 시도하지만, `feedId`에 해당하는 피드가 없으므로 FK 제약 위반으로 실패한다. Consumer는 재시도를 3번 반복하고, 그래도 실패하면 PEL(Pending Entry List)에 남는다. 이 패턴이 반복되면 PEL이 무한 적체된다.

모임 멤버 검증도 비어 있었다. `club`을 조회해두고 이후 어디서도 사용하지 않는다. 모임에 가입하지 않은 사용자도 좋아요가 가능했고, `clubId=1`인 피드에 `clubId=999`로 요청해도 통과됐다.

---

### Redis 재시작 시 좋아요 상태 초기화 — 복구 경로 없음

좋아요 상태의 source of truth는 Redis의 `feed:{feedId}:likers` Set이었다. 그런데 Redis가 재시작되면 이 Set이 사라지고, DB의 `feed_like` 테이블에서 Redis로 복구하는 로직이 없었다.

```
유저 A가 피드에 좋아요 → Redis Set에 추가
→ Redis 재시작 → Set 사라짐
→ A가 다시 좋아요 → Lua: Set에 없으므로 ON 이벤트 발행
→ Consumer가 feed_like INSERT → 이미 존재
→ INSERT IGNORE → like_count +1
→ 카운트 이중 증가
```

좋아요를 취소한 것처럼 보이다가 다시 눌렀을 때 카운트가 두 배로 뛰는 상황이 생길 수 있었다.

---

### 피드 목록 조회 시 N+1 — 좋아요 1000개면 1000개 엔티티 로딩

```java
// FeedService.java
feed.getFeedLikes().size(),      // ← 전체 FeedLike 엔티티 로드 후 .size()
feed.getFeedComments().size()    // ← 전체 FeedComment 엔티티 로드 후 .size()
```

피드 20건을 조회하면 각 피드마다 `getFeedLikes()`와 `getFeedComments()`를 호출한다. 카운트 하나를 얻으려고 엔티티 전체를 메모리에 올리는 것이다. 인기 피드의 좋아요가 1000개라면 1000개의 `FeedLike` 엔티티를 영속화한 뒤 `.size()`만 쓰고 버린다.

`FeedMainService`는 더 심각했다. 원본 피드 외에도 parent와 root 피드를 추가로 로딩하고, 그것들의 likes와 comments 컬렉션까지 전부 올렸다.

---

### "친구의 친구" 피드 범위 폭발 — 비공개 모임까지 노출

```java
private List<Long> resolveAccessibleClubIds(Long userId) {
    List<Long> myClubIds = ...;
    List<Long> memberIds = userClubRepository.findUserIdByClubIds(myClubIds);
    List<UserClub> friendMemberJoinClubs = userClubRepository.findByUserUserIdIn(memberIds);
}
```

내가 모임 3개에 가입하고 각 모임에 멤버가 50명이면 `memberIds`가 최대 150명이다. 이 150명이 각각 모임 5개에 가입했다면 최대 750개의 `UserClub` 엔티티를 가져온다. 결과적으로 피드 조회 대상 모임이 수백 개로 불어난다.

보안 문제도 있었다. 비공개 모임의 피드도 "친구의 친구" 관계라면 노출됐다. 모임의 공개/비공개 설정을 확인하는 코드가 없었다.

---

### 클래스 레벨 `@Transactional` — Redis 작업에도 DB 커넥션 점유

```java
@Transactional  // ← 클래스 레벨: 모든 public 메서드에 적용
public class FeedService {
```

`toggleLike()`는 Redis만 사용하고 DB를 직접 수정하지 않는다. 하지만 클래스 레벨 `@Transactional` 때문에 Redis Lua 스크립트가 실행되는 내내 DB 커넥션이 열려 있었다. 좋아요 TPS가 높아질수록 DB 커넥션 풀이 소진될 수 있는 구조였다.

---

### 그 외 문제들

**이미지 URL 검증 없음**: `javascript:alert(1)`, `file:///etc/passwd` 같은 값이 그대로 저장될 수 있었다. presigned URL 기반 S3 업로드 구조인데 서버에서 S3 도메인 검증을 하지 않고 있었다.

**`likedFeedIds`가 항상 빈 Set**: `FeedMainService.java`에서 `Set<Long> likedFeedIds = Collections.emptySet()`으로 하드코딩되어 있었다. 빈 Set으로 남아 N+1 문제를 가중시켰다.

**리피드 시 원본 피드 접근 권한 미확인**: 리피드를 올릴 모임의 멤버십만 확인하고, 원본 피드가 속한 모임에 접근 권한이 있는지는 확인하지 않았다.

---

## 해결

### 좋아요 수 단일 소스로 통일 — `like_count` 비정규화 컬럼 활용

`getFeedLikes().size()` 호출을 전부 제거하고 `feed.getLikeCount()`로 통일했다. Consumer가 업데이트하지만 아무도 읽지 않던 `like_count` 컬럼을 드디어 사용하게 됐다. 조회 API는 `likeCountMap`을 배치로 한 번에 가져와서 각 피드에 매핑한다.

---

### 검증 추가 — 피드 존재 확인 + 모임 멤버십 확인

```java
// FeedLikeService.toggleLike()
public boolean toggleLike(long clubId, long feedId) {
    long userId = authService.getCurrentUserId();

    if (!feedRepository.existsById(feedId)) {
        throw new CustomException(ErrorCode.FEED_NOT_FOUND);
    }
    if (!userClubRepository.existsByUser_UserIdAndClub_ClubId(userId, clubId)) {
        throw new CustomException(ErrorCode.CLUB_NOT_JOIN);
    }

    ensureLikeCacheWarmed(feedId);  // Redis 키 없으면 DB에서 워밍업
    // 이후 Lua 스크립트 실행
}
```

검증을 Redis 작업 앞에 배치해 존재하지 않는 피드에 대한 Stream 이벤트가 발행되지 않는다. PEL 적체 문제의 근원이 차단된다.

---

### Redis 워밍업 — lazy warmup으로 재시작 복구

```java
private void ensureLikeCacheWarmed(Long feedId) {
    String setKey = "feed:" + feedId + ":likers";
    if (!redis.hasKey(setKey)) {
        // Redis 키가 없으면 DB에서 읽어 복구
        List<Long> userIds = feedLikeRepository.findUserIdsByFeedId(feedId);
        if (!userIds.isEmpty()) {
            redis.opsForSet().add(setKey, userIds.stream().map(Object.class::cast).toArray());
        }
        redis.opsForValue().set("feed:" + feedId + ":like_count", (long) userIds.size());
    }
}
```

`toggleLike()` 진입 시마다 `ensureLikeCacheWarmed(feedId)`를 호출한다. `redis.hasKey(setKey)`가 Redis에 O(1) 네트워크 요청 하나로 처리되므로 정상 동작 시 오버헤드는 미미하다. Redis 재시작 후 첫 번째 toggleLike 요청이 들어오는 시점에 해당 피드의 좋아요 상태가 DB 기준으로 복구된다. 재시작 직후 아무도 누르지 않은 피드는 다음 요청 전까지 캐시가 비어있지만, 그 상태에서 toggleLike가 오면 워밍업이 일어나므로 이중 카운트 문제는 발생하지 않는다.

---

### 2-pass 쿼리로 N+1 제거

컬렉션 전체 로딩 방식을 2-pass 쿼리로 교체했다. Pass 1은 비정규화 컬럼(like_count, comment_count)으로 카운트만 조회하고, Pass 2는 fetch join으로 이미지·작성자 등 필요한 관계만 한 번에 조회한다. `likedFeedIds`도 배치 조회로 채워 컬렉션 순회를 제거했다.

---

### "친구의 친구" 범위 단순화

`resolveAccessibleClubIds()`를 `findAccessibleClubIds(userId)` 단일 네이티브 쿼리로 교체해 내가 소속된 모임만 반환하도록 범위를 줄였다. 쿼리 폭발과 비공개 모임 노출 문제가 함께 해결됐다.

---

### 클래스 레벨 `@Transactional` 제거 + 이미지 URL 검증 추가

좋아요 로직을 `FeedLikeService`로 분리해 클래스 레벨 `@Transactional`을 제거했다. 이미지 URL 검증은 `FeedRequestDto` 원소에 `@NotBlank`와 `@Pattern`으로 S3 도메인 접두사를 검증하도록 추가했다.

---

## 현재 상태

| 문제 | 상태 |
|------|------|
| 좋아요 수 3곳 불일치 | ✅ 해결 — `like_count` 단일 소스로 통일 |
| toggleLike 모임 참여 검증 없음 | ✅ 해결 — `FeedLikeService` 분리 + 멤버십 검증 |
| toggleLike 피드 존재 검증 없음 | ✅ 해결 — `existsById` 검증 추가 |
| like_applied 테이블 PEL 적체 | ✅ 해결 — `FeedLikeApplied` 엔티티 제거, Unique Constraint로 멱등성 보장 |
| Redis 재시작 시 이중 카운트 | ✅ 해결 — lazy warmup, toggleLike 진입 시마다 키 존재 확인 |
| N+1 + 컬렉션 전체 로딩 | ✅ 해결 — 2-pass 쿼리 + 비정규화 컬럼 |
| "친구의 친구" 범위 폭발 + 비공개 모임 노출 | ✅ 해결 — 자기 소속 모임으로 범위 축소 |
| 클래스 레벨 `@Transactional` | ✅ 해결 — `FeedLikeService` 분리 |
| 이미지 URL 검증 없음 | ✅ 해결 — S3 도메인 패턴 검증 추가 |
| `likedFeedIds` 항상 빈 Set | ✅ 해결 — 배치 조회 구현 |
| 리피드 시 원본 피드 접근 권한 미확인 | ✅ 해결 — 모임 멤버십 검증 추가 |

---

## 정리하며

Redis를 도입해 성능을 높이려는 시도 자체는 옳았다. 문제는 Lua → Stream → Consumer → DB로 이어지는 파이프라인을 만들어두고, 각 단계의 검증을 챙기지 않은 것이다. Redis에서 비동기로 DB를 따라가게 해놓고 조회는 여전히 `getFeedLikes().size()`로 DB를 읽으니, Redis를 도입한 의미가 없었다.

> **비동기 파이프라인을 만들었다면 조회도 그 파이프라인의 결과물을 읽어야 한다.**
> 쓰기는 Redis를 통하고 읽기는 JPA 컬렉션을 통하면, 파이프라인은 그림의 떡이다.

---

## 시리즈 탐색

**◀ 이전 글**  
[알림 SSE 구현 중 발견한 연쇄 버그와 구조 개선](/sse-notification-chained-bugs/)

**다음 글 ▶**  
[모임 도메인 — 동시성 구멍과 유령 멤버](/club-concurrency-ghost-members/)
