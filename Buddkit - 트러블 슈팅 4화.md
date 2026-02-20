---
title: 피드 도메인 — Redis 좋아요 파이프라인의 허점과 N+1
date: 2026-02-19
tags: [Redis, JPA, Spring, 트러블슈팅, Java, 피드]
---

# 피드 도메인 — Redis 좋아요 파이프라인의 허점과 N+1

## 개요

피드 코드를 다시 살펴보던 중, 좋아요 수를 표시하는 소스가 **세 군데**인데 세 군데가 각각 다른 값을 반환할 수 있다는 걸 발견했다. Redis 카운터, DB의 `like_count` 컬럼, `feed_like` 테이블의 row 수. 셋 중 아무것도 실제 화면에 표시되는 값의 정답이라고 보장할 수 없었다.

좋아요 성능을 높이려고 Redis + Stream + Consumer로 이어지는 비동기 파이프라인을 도입했는데, 각 단계에서 검증과 복구 로직이 빠지면서 오히려 데이터가 세 곳으로 쪼개져버린 것이다. 여기에 N+1 쿼리, 피드 접근 권한 구멍까지 겹쳐 있었다.

---

## 시스템 구조

### 좋아요 파이프라인

```
클라이언트 → toggleLike()
    ↓
Redis Lua 스크립트 (feed:{feedId}:likers Set, feed:{feedId}:like_count 카운터 즉시 업데이트)
    ↓
Redis Stream에 이벤트 발행 (ON/OFF)
    ↓
FeedLikeStreamConsumer (비동기)
    ↓
like_applied 테이블 INSERT IGNORE (멱등성 보장)
    ↓
feed.like_count 업데이트 + feed_like INSERT/DELETE
```

문제는 이 파이프라인의 끝에 있는 `feed.like_count`와 `feed_like` 테이블을 **조회 API가 전혀 사용하지 않는다**는 것이었다.

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

Consumer가 1초만 지연되어도 세 값이 전부 달라진다. 그리고 `feed.like_count`는 Consumer가 열심히 업데이트하지만 조회 코드 어디서도 읽지 않았다. 만들어두고 아무도 쓰지 않는 컬럼이었다.

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
    // 1) 내가 가입한 모임
    List<Long> myClubIds = ...;

    // 2) 내 모임 멤버들의 userId
    List<Long> memberIds = userClubRepository.findUserIdByClubIds(myClubIds);

    // 3) 그 멤버들이 가입한 다른 모임 전부
    List<UserClub> friendMemberJoinClubs = userClubRepository.findByUserUserIdIn(memberIds);
    // ...
}
```

내가 모임 3개에 가입하고 각 모임에 멤버가 50명이면 `memberIds`가 최대 150명이다. 이 150명이 각각 모임 5개에 가입했다면 `findByUserUserIdIn(150명)`으로 최대 750개의 `UserClub` 엔티티를 가져온다. 결과적으로 피드 조회 대상 모임이 수백 개로 불어난다.

보안 문제도 있었다. 비공개 모임의 피드도 "친구의 친구" 관계라면 노출됐다. 모임의 공개/비공개 설정을 확인하는 코드가 없었다.

`myClubIds.contains(id)`는 `List.contains()`여서 O(n) 순회를 반복했다.

---

### 클래스 레벨 `@Transactional` — Redis 작업에도 DB 커넥션 점유

```java
@Transactional  // ← 클래스 레벨: 모든 public 메서드에 적용
public class FeedService {
```

`toggleLike()`는 Redis만 사용하고 DB를 직접 수정하지 않는다. 하지만 클래스 레벨 `@Transactional` 때문에 Redis Lua 스크립트가 실행되는 내내 DB 커넥션이 열려 있었다. 좋아요 TPS가 높아질수록 DB 커넥션 풀이 소진될 수 있는 구조였다.

---

### 그 외 문제들

**이미지 URL 검증 없음**

```java
// FeedRequestDto.java
@Size(min = 1, max = 5)
private List<String> feedUrls;  // 원소 자체에는 아무 검증 없음
```

`javascript:alert(1)`, `file:///etc/passwd`, `http://internal-server:8080/admin` 같은 값이 그대로 저장될 수 있었다. presigned URL 기반으로 S3에 업로드한 뒤 URL만 보내는 구조인데, 서버에서 S3 도메인 검증을 하지 않고 있었다.

**`likedFeedIds`가 항상 빈 Set**

```java
// FeedMainService.java
Set<Long> likedFeedIds = Collections.emptySet();  // 하드코딩
```

`isLiked()` 메서드는 `likedFeedIds`가 비어 있으면 매번 `getFeedLikes()` 컬렉션을 순회해서 확인한다. 배치로 좋아요 여부를 조회해서 채울 의도로 만든 필드인데 하드코딩된 빈 Set으로 남아 있었다. N+1 문제와 결합되어 피드 20건 조회 시 컬렉션 로딩이 2배로 늘어났다.

**리피드 시 원본 피드 접근 권한 미확인**

리피드를 올릴 모임의 멤버십만 확인하고, 원본 피드가 속한 모임에 접근 권한이 있는지는 확인하지 않았다. `feedId`만 알면 비공개 모임의 피드도 리피드할 수 있었다.

---

## 해결

### 좋아요 수 단일 소스로 통일 — `like_count` 비정규화 컬럼 활용

`getFeedLikes().size()` 호출을 전부 제거하고 `feed.getLikeCount()`로 통일했다. Consumer가 업데이트하지만 아무도 읽지 않던 `like_count` 컬럼을 드디어 사용하게 됐다. Redis 카운터, DB 컬럼, 컬렉션 row 수로 쪼개져 있던 값의 소스가 하나로 모였다.

조회 API는 `likeCountMap`을 배치로 한 번에 가져와서 각 피드에 매핑한다.

---

### 검증 추가 — 피드 존재 확인 + 모임 멤버십 확인

`toggleLike()`에 두 가지 검증을 추가했다.

```java
// FeedLikeService (toggleLike 로직 분리)
if (!feedRepository.existsById(feedId)) {
    throw new CustomException(ErrorCode.FEED_NOT_FOUND);
}
if (!userClubRepository.existsByUser_UserIdAndClub_ClubId(userId, clubId)) {
    throw new CustomException(ErrorCode.CLUB_NOT_JOIN);
}
```

피드가 존재하지 않거나 soft-delete된 경우, 해당 모임의 멤버가 아닌 경우 모두 예외를 던진다. Consumer가 FK 실패로 PEL을 쌓는 상황이 사라졌다. 기존에 조회만 하고 사용하지 않던 `club` 변수도 제거했다.

---

### Redis 워밍업 — 재시작 시 복구 경로 확보

`FeedLikeService.ensureLikeCacheWarmed()`를 추가했다. `toggleLike()` 호출 시 Redis Set이 없으면 DB의 `feed_like` 테이블에서 해당 피드의 좋아요 유저 목록을 조회해 Redis에 복구한다.

```java
public void ensureLikeCacheWarmed(Long feedId) {
    String setKey = "feed:" + feedId + ":likers";
    if (!redis.hasKey(setKey)) {
        // SETNX 락으로 동시 워밍업 방지
        List<Long> userIds = feedLikeRepository.findUserIdsByFeedId(feedId);
        redis.opsForSet().add(setKey, userIds.toArray());
        redis.opsForValue().set("feed:" + feedId + ":like_count", userIds.size());
    }
}
```

Redis가 재시작된 직후 첫 번째 좋아요 요청에서 워밍업이 일어나고, 이후 요청부터는 정상적으로 Lua 스크립트가 동작한다.

---

### 2-pass 쿼리로 N+1 제거

컬렉션 전체 로딩 방식을 2-pass 쿼리로 교체했다.

```
Pass 1: findFeedIdsWithCountsByClubIds
         → 비정규화 컬럼(like_count, comment_count)으로 카운트만 조회

Pass 2: findByIdsWithRelations
         → fetch join으로 이미지, 작성자 등 필요한 관계만 한 번에 조회
```

`getFeedLikes().size()`, `getFeedComments().size()` 호출이 전부 사라졌다. `likedFeedIds`도 `feedLikeRepository.findLikedFeedIdsByUser(feedIds, userId)` 배치 조회로 채워서, 좋아요 여부 확인 시 컬렉션을 순회하지 않는다.

---

### "친구의 친구" 범위 단순화

`resolveAccessibleClubIds()`를 `findAccessibleClubIds(userId)` 단일 네이티브 쿼리로 교체했다. 내가 소속된 모임만 반환하도록 범위를 줄였다. 쿼리 폭발과 비공개 모임 노출 문제가 함께 해결됐다. `IN` 절에 들어가는 `clubId` 수도 "내 소속 모임 수"로 제한되어 합리적인 범위 안으로 들어왔다.

---

### 클래스 레벨 `@Transactional` 제거 + 이미지 URL 검증 추가

좋아요 로직을 `FeedLikeService`로 분리하면서 해당 서비스에는 클래스 레벨 `@Transactional`을 두지 않았다. Redis 작업이 불필요하게 DB 커넥션을 점유하는 문제가 해소됐다.

이미지 URL 검증은 `FeedRequestDto`의 원소에 `@NotBlank`와 `@URL`을 추가했다. 빈 문자열과 `http://`, `https://` 외의 스킴을 가진 URL을 요청 수신 시점에 거부한다.

---

## 현재 상태

| 문제 | 상태 |
|------|------|
| 좋아요 수 3곳 불일치 | ✅ 해결 — `like_count` 단일 소스로 통일 |
| toggleLike 모임 참여 검증 없음 | ✅ 해결 — `FeedLikeService` 분리 + 멤버십 검증 |
| toggleLike 피드 존재 검증 없음 | ✅ 해결 — `existsById` 검증 추가 |
| like_applied 테이블 무한 증가 | ✅ 해결 — `FeedLikeApplied` 엔티티 제거 |
| Redis 재시작 시 유실 | ✅ 해결 — lazy warmup 구현 |
| N+1 + 컬렉션 전체 로딩 | ✅ 해결 — 2-pass 쿼리 + 비정규화 컬럼 |
| "친구의 친구" 범위 폭발 | ✅ 해결 — 자기 소속 모임으로 범위 축소 |
| 클래스 레벨 `@Transactional` | ✅ 해결 — `FeedLikeService` 분리 |
| 이미지 URL 검증 없음 | ✅ 해결 — `@NotBlank` + `@URL` 추가 |
| `likedFeedIds` 항상 빈 Set | ✅ 해결 — 배치 조회 구현 |
| 리피드 시 원본 피드 접근 권한 미확인 | ✅ 해결 — 모임 멤버십 검증 추가 |
| 댓글 삭제 권한에 모임 LEADER 미반영 | ⚠️ 잔존 — 설계 논의 필요 |

---

## 교훈

Redis를 도입해 성능을 높이려는 시도 자체는 옳았다. 문제는 Lua → Stream → Consumer → DB로 이어지는 파이프라인을 만들어두고, 각 단계의 검증을 챙기지 않은 것이다. Redis에서 비동기로 DB를 따라가게 해놓고 조회는 여전히 `getFeedLikes().size()`로 DB를 읽으니, Redis를 도입한 의미가 없었다. 파이프라인의 끝에서 만들어지는 `like_count` 컬럼을 아무도 읽지 않은 채로 방치한 것이 핵심 문제였다.

**만들어둔 파이프라인은 끝까지 연결해야 한다.** Redis로 빠르게 쓰고, DB로 비동기 동기화하고, 조회할 때는 다시 컬렉션으로 읽어오면 세 단계가 전부 따로 노는 것이다. 각 단계가 서로 신뢰할 수 있는 단일 소스를 바라봐야 파이프라인이 의미를 갖는다.

> **비동기 파이프라인을 만들었다면 조회도 그 파이프라인의 결과물을 읽어야 한다.**
> 쓰기는 Redis를 통하고 읽기는 JPA 컬렉션을 통하면, 파이프라인은 그림의 떡이다.
