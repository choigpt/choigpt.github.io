---
title: 피드 도메인 — Redis를 도입했는데 조회는 여전히 JPA를 통하고 있었다
date: 2026-02-21
tags: [Redis, JPA, Spring, 트러블슈팅, Java, 피드]
permalink: /feed-redis-like-n-plus-1/
excerpt: "좋아요 수를 읽는 소스가 Redis, DB like_count 컬럼, feed_like 테이블 세 군데로 나뉘어 서로 다른 값을 반환할 수 있었다. Redis를 도입했는데 조회는 여전히 JPA 컬렉션을 통하고 있었다."
---

JWT/SSE 인증 인프라의 버그를 정리한 뒤([1편](/spring-security-jwt-structure-bugs/)~[4편](/sse-notification-chained-bugs/)), 이제 각 도메인의 비즈니스 로직을 점검한다.

> **비동기 파이프라인을 만들었다면 조회도 그 파이프라인의 결과물을 읽어야 한다.**
> 쓰기는 Redis를 통하고 읽기는 JPA 컬렉션을 통하면, 파이프라인이 실제로 활용되지 않는다.

---

## 개요

피드 코드를 점검한 결과, 좋아요 수를 표시하는 소스가 **세 군데**이며 각각 다른 값을 반환할 수 있는 상태였다. Redis 카운터, DB의 `like_count` 컬럼, `feed_like` 테이블의 row 수. 셋 중 아무것도 실제 화면에 표시되는 값의 정답이라고 보장할 수 없었다.

다른 팀원이 좋아요 성능을 높이려고 Redis + Stream + Consumer로 이어지는 비동기 파이프라인을 도입했다. 프로젝트 종료 후 이 코드를 읽으면서 각 단계에서 검증과 복구 로직이 빠져 데이터가 세 곳으로 분산되는 문제를 발견했다. 여기에 N+1 쿼리, 피드 접근 권한 구멍까지 겹쳐 있었다.

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

문제는 이 파이프라인의 끝에 있는 `feed.like_count`와 `feed_like` 테이블을 **조회 API가 전혀 사용하지 않는다**는 것이었다. 실제 코드를 보면 명확하다.

**쓰기 경로** — `FeedService.toggleLike()`는 Redis Lua 스크립트를 실행한다:

```java
// FeedService.toggleLike()
List<?> raw = redis.execute(likeToggleScript, keys, args);
// → feed:{feedId}:likers Set, feed:{feedId}:like_count 카운터 즉시 업데이트
// → Stream에 이벤트 발행 → Consumer가 비동기로 DB를 따라감
```

**읽기 경로** — Redis를 전혀 읽지 않는다. `FeedService.getFeedList()`는 이렇게 되어 있다:

```java
// FeedService.getFeedList()
return new FeedSummaryResponseDto(
        feed.getFeedId(),
        thumbnailUrl,
        feed.getFeedLikes().size(),      // ← JPA 컬렉션 전체 로딩 후 .size()
        feed.getFeedComments().size()    // ← 동일
);
```

`getFeedLikes()`는 `@OneToMany` 연관 컬렉션이므로 `feed_like` 테이블의 row 전체를 메모리에 올린 뒤 `.size()`를 부르는 것이다. Redis 카운터도, Consumer가 업데이트하는 `like_count` 컬럼도 사용하지 않는다.

`FeedMainService`도 동일하다:

```java
// FeedMainService.toOverviewDto(), toShallowDto()
.likeCount(safeSize(f.getFeedLikes()))       // ← 컬렉션 전체 로딩
.commentCount(safeSize(f.getFeedComments())) // ← 컬렉션 전체 로딩
```

"현재 사용자가 좋아요를 눌렀는지"를 판별하는 `isLiked()`는 더 심각하다:

```java
// FeedMainService.isLiked()
private boolean isLiked(Feed f, Long userId, Set<Long> likedFeedIds) {
    if (likedFeedIds != null && !likedFeedIds.isEmpty()) {
        return likedFeedIds.contains(f.getFeedId());
    }
    // ↑ 여기서 빠져나갈 수 있지만...

    List<FeedLike> likes = f.getFeedLikes();
    return likes.stream()
            .anyMatch(l -> l.getUser() != null
                    && Objects.equals(l.getUser().getUserId(), userId));
    // ↑ 모든 FeedLike 엔티티를 순회
}
```

`likedFeedIds`가 피드별로 미리 채워져 있으면 `contains()`로 O(1)에 끝나지만, 호출부에서 이 값이 `Collections.emptySet()`으로 하드코딩되어 있다:

```java
// FeedMainService.getFeedsCommon()
Set<Long> likedFeedIds = Collections.emptySet();   // ← 항상 빈 Set
```

빈 Set이므로 `isEmpty()` 분기를 탈 수 없고, 매번 `f.getFeedLikes().stream().anyMatch(...)`로 **전체 FeedLike 엔티티를 순회**한다. Redis의 `feed:{feedId}:likers` Set에 이미 정보가 있는데 읽지 않는다.

`getFeedDetail()`도 마찬가지다:

```java
// FeedService.getFeedDetail()
boolean isLiked = feed.getFeedLikes().stream()
        .anyMatch(like -> like.getUser().getUserId().equals(currentUserId));
// ← 좋아요 1000개면 FeedLike 1000개 + User 1000개를 로딩해서 순회
```

Redis + Stream + Consumer로 쓰기 파이프라인을 만들어두고, Consumer가 업데이트하는 `feed.like_count` 컬럼은 조회 코드 어디서도 읽지 않으며, 조회는 전부 JPA 컬렉션 `.size()`와 `.stream().anyMatch()`를 통하고 있었다.

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

`like_applied` 테이블을 제거했다. 이 테이블은 JPA 엔티티가 아니라 `schema.sql`에 raw DDL로 정의되어 있었다:

```sql
-- schema.sql (원본 레포지토리)
CREATE TABLE IF NOT EXISTS like_applied (
    req_id     VARCHAR(64) PRIMARY KEY,   -- Stream message ID를 PK로 사용
    feed_id    BIGINT NOT NULL,
    user_id    BIGINT NOT NULL,
    delta      INT NOT NULL,              -- +1(좋아요) 또는 -1(취소)
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

엔티티가 아니라 DDL로 만든 이유는 이 테이블의 성격 때문이다:

```
1. Consumer가 INSERT IGNORE로만 쓰는 write-only 테이블
   → SELECT 할 일이 없음 → JPA 영속성 컨텍스트의 이점이 없다

2. PK가 Stream message ID (VARCHAR)
   → JPA의 @GeneratedValue Long id 패턴과 안 맞음

3. 용도가 "이 이벤트를 이미 처리했는가"의 멱등성 추적만
   → JdbcTemplate으로 INSERT IGNORE 한 줄이면 끝
   → 엔티티로 매핑하면 오히려 복잡해짐
```

하지만 `feed_like` 테이블의 UNIQUE 제약으로 이미 같은 멱등성이 보장되므로 이 테이블은 불필요했다:

```
왜 멱등성이 보장되는가?

전제: feed_like 테이블에 (user_id, feed_id) UNIQUE 제약이 걸려있다.

═══ 좋아요 ON 이벤트가 2번 들어온 경우 (중복) ═══

  1번째 ON: INSERT INTO feed_like (user_id, feed_id) VALUES (1, 100);
            → 성공. 행 1개 삽입.

  2번째 ON: INSERT IGNORE INTO feed_like (user_id, feed_id) VALUES (1, 100);
            → UNIQUE 제약 위반 → INSERT IGNORE이므로 무시. 에러 안 남.
            → 행이 이미 있으므로 아무 일도 안 일어남.

  결과: 1번 실행하든 100번 실행하든 행은 1개. 멱등!

═══ 좋아요 OFF 이벤트가 2번 들어온 경우 (중복) ═══

  1번째 OFF: DELETE FROM feed_like WHERE user_id = 1 AND feed_id = 100;
             → 성공. 행 1개 삭제.

  2번째 OFF: DELETE FROM feed_like WHERE user_id = 1 AND feed_id = 100;
             → 이미 없음. 삭제 대상 0건. 에러 안 남.

  결과: 1번 실행하든 100번 실행하든 행은 0개. 멱등!

═══ ON → OFF → ON이 빠르게 들어온 경우 ═══

  Stream Consumer는 순서대로 처리하므로:
  ON  → INSERT (성공)
  OFF → DELETE (성공)
  ON  → INSERT (성공)

  최종 상태: 행 1개 (좋아요 ON). 올바른 결과.
```

기존 `like_applied` 테이블은 "이 이벤트를 이미 처리했는가"를 별도로 추적하는 용도였다. 하지만 `feed_like` 테이블의 UNIQUE 제약 + INSERT IGNORE/DELETE 패턴이 이미 동일한 멱등성을 제공하므로 중복 테이블이었다.

> `feed_like`의 멱등성은 보장되지만, Consumer가 `like_count`를 업데이트하는 로직에서도 INSERT IGNORE의 affected row count를 확인해야 한다. INSERT가 실제로 0행을 삽입했다면(중복) `like_count`를 증가시키면 안 된다. affected row count를 무시하고 무조건 `like_count + 1`을 하면 카운트가 틀어진다.

```java
// Consumer에서 like_count 업데이트 시
int inserted = feedLikeRepository.insertIgnore(userId, feedId);
if (inserted > 0) {
    feedRepository.incrementLikeCount(feedId);  // 실제 삽입된 경우에만
}
```

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

먼저 Redis Stream의 Consumer Group 동작을 알아야 이 버그가 이해된다.

```
Redis Stream + Consumer Group의 메시지 처리 흐름:

1. Producer가 Stream에 메시지를 추가
   → XADD feed-like-stream * feedId 100 userId 1 action ON

2. Consumer가 XREADGROUP으로 메시지를 읽음
   → 이 순간 메시지가 PEL(Pending Entry List)에 들어감
   → "이 Consumer가 읽었지만 아직 처리 완료 확인을 받지 못한 메시지"라는 뜻

3. Consumer가 처리에 성공하면 → XACK로 확인(acknowledge)을 보냄
   → PEL에서 제거됨 → 정상 완료

4. Consumer가 처리에 실패하면 → XACK를 보내지 못함
   → PEL에 계속 남아있게 됨
   → 다음 읽기 시 재시도 대상이 됨
```

```
PEL(Pending Entry List)이란?

"읽었는데 아직 XACK 안 된 메시지"의 목록이라고 보면 된다.
정상적으로는 읽고 → 처리하고 → XACK 하면 PEL에서 빠진다.

문제는 처리가 영구적으로 실패하는 메시지가 있을 때 생긴다:
  → 읽기 → 실패 → 재시도 → 실패 → 재시도 → 실패 → ...
  → XACK가 영원히 안 되니까
  → PEL에 영원히 남게 되고
  → PEL이 계속 커지면서 → 메모리 낭비 + 재시도 부하 증가로 이어질 수 있다
```

이 프로젝트에서 PEL 적체가 발생할 수 있는 시나리오:

```
존재하지 않는 feedId=999999에 좋아요 요청이 들어옴
  ↓
toggleLike()에서 feedId 존재 확인을 하지 않음 → 바로 Redis Lua 실행
  ↓
Redis에 feed:999999:likers, feed:999999:like_count 키가 생성됨
  ↓
Stream에 이벤트 발행: {feedId: 999999, userId: 1, action: ON}
  ↓
Consumer가 읽음 → PEL에 등록됨
  ↓
feed_like 테이블에 INSERT를 시도
  → feedId=999999에 해당하는 feed가 없음
  → FK 제약 위반으로 실패
  ↓
재시도 1회 → 실패 (피드가 없는 건 변하지 않으니까)
재시도 2회 → 실패
재시도 3회 → 실패 → 재시도 포기
  ↓
XACK를 보내지 못함 → PEL에 영구적으로 남게 됨

이런 요청이 반복되면 → PEL이 무한히 적체될 수 있다
```

모임 멤버 검증도 비어 있었다. `club`을 조회해두고 이후 어디서도 사용하지 않는다. `clubId=1`인 피드에 `clubId=999`로 요청해도 통과되는 상태였다.

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

### 피드 목록 조회 시 전체 컬렉션 로딩 — 카운트 하나에 엔티티 수백 개

```java
// FeedService.java
feed.getFeedLikes().size(),      // ← 전체 FeedLike 엔티티 로드 후 .size()
feed.getFeedComments().size()    // ← 전체 FeedComment 엔티티 로드 후 .size()
```

피드 20건을 조회하면 각 피드마다 `getFeedLikes()`와 `getFeedComments()`를 호출한다. `@BatchSize(size = 100)`이 적용되어 있어 순수한 N+1은 아니지만, 카운트 하나를 얻으려고 엔티티 전체를 메모리에 올리는 구조 자체가 문제다. 인기 피드의 좋아요가 1000개라면 1000개의 `FeedLike` 엔티티를 영속화한 뒤 `.size()`만 쓰고 버린다. `SELECT COUNT(*)`나 비정규화 컬럼 `like_count`를 읽으면 될 일이다.

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

`getFeedLikes().size()` 호출을 전부 제거하고 `feed.getLikeCount()`로 통일했다. Consumer가 업데이트하지만 조회 코드에서 사용하지 않던 `like_count` 컬럼을 실제 조회에 활용하도록 변경했다. 조회 API는 `likeCountMap`을 배치로 한 번에 가져와서 각 피드에 매핑한다.

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

> 예외: 좋아요가 0인 피드에서는 `redis.opsForSet().add()`를 호출하지 않아 Set 키가 생성되지 않는다. 이 경우 다음 `toggleLike` 호출 시 `hasKey()`가 다시 false를 반환해 매번 DB를 조회하게 된다. 좋아요 0인 피드가 많다면 빈 Set 대신 sentinel 값을 넣거나 별도의 flag 키를 두는 방식으로 보완할 수 있다.

> 주의: 이 check-then-act 패턴에는 TOCTOU(Time-of-Check-Time-of-Use) 레이스 컨디션이 있다. 두 요청이 동시에 `hasKey() == false`를 보면 둘 다 DB를 조회하고 Redis에 쓴다. 두 번째 쓰기가 첫 번째를 덮어쓰면서, 그 사이에 추가된 좋아요가 유실될 수 있다. 완전한 원자성이 필요하면 `SETNX` 기반 guard나 Lua 스크립트로 워밍업 자체를 원자적으로 수행해야 한다. 정상 운영 시에는 워밍업이 toggleLike 직전에만 발생해 빈도가 낮지만, Redis 재시작 직후에는 모든 피드의 첫 요청이 동시에 워밍업을 트리거하므로 thundering herd 상황에서 이 레이스 컨디션이 실제로 발생할 수 있다.

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

- **좋아요 수 3곳 불일치**: ✅ 해결 — `like_count` 단일 소스로 통일
- **toggleLike 모임 참여 검증 없음**: ✅ 해결 — `FeedLikeService` 분리 + 멤버십 검증
- **toggleLike 피드 존재 검증 없음**: ✅ 해결 — `existsById` 검증 추가
- **like_applied 테이블 PEL 적체**: ✅ 해결 — `FeedLikeApplied` 엔티티 제거, Unique Constraint로 멱등성 보장
- **Redis 재시작 시 이중 카운트**: ✅ 해결 — lazy warmup, toggleLike 진입 시마다 키 존재 확인
- **컬렉션 전체 로딩 (카운트에 엔티티 수백 개 적재)**: ✅ 해결 — 2-pass 쿼리 + 비정규화 컬럼
- **"친구의 친구" 범위 폭발 + 비공개 모임 노출**: ✅ 해결 — 자기 소속 모임으로 범위 축소
- **클래스 레벨 `@Transactional`**: ✅ 해결 — `FeedLikeService` 분리
- **이미지 URL 검증 없음**: ✅ 해결 — S3 도메인 패턴 검증 추가
- **`likedFeedIds` 항상 빈 Set**: ✅ 해결 — 배치 조회 구현
- **리피드 시 원본 피드 접근 권한 미확인**: ✅ 해결 — 모임 멤버십 검증 추가

---

## 정리하며

팀원이 Redis를 도입해 성능을 높이려는 시도 자체는 옳았다. 내가 코드를 읽으며 발견한 문제는, Lua → Stream → Consumer → DB로 이어지는 파이프라인을 만들어두고 각 단계의 검증을 챙기지 않은 것이다. Redis에서 비동기로 DB를 따라가게 해놓고 조회는 여전히 `getFeedLikes().size()`로 DB를 읽으니, Redis를 도입한 의미가 없었다.

> **비동기 파이프라인을 만들었다면 조회도 그 파이프라인의 결과물을 읽어야 한다.**
> 쓰기는 Redis를 통하고 읽기는 JPA 컬렉션을 통하면, 파이프라인이 실제로 활용되지 않는다.

이 구조 변경의 성능 효과는 [피드 도메인 부하 테스트 편](/feed-performance-load-test/)에서 검증한다.

---

## 시리즈 탐색

**◀ 이전 글**
[알림 SSE 구현 중 발견한 연쇄 버그와 구조 개선](/sse-notification-chained-bugs/)

**▶ 다음 글**
[모임 도메인 — 동시성 구멍과 유령 멤버](/club-concurrency-ghost-members/)
