---
title: 피드 성능 — 99% 에러율을 만들어낸 N+1과 커버링 인덱스로 해결한 과정
date: 2026-02-19
tags: [JPA, QueryDSL, Redis, 성능최적화, 트러블슈팅, Java, 피드, k6]
---

# 피드 성능 — 99% 에러율을 만들어낸 N+1과 커버링 인덱스로 해결한 과정

## 개요

k6로 피드 API에 부하를 걸었더니 `popular_feed` 엔드포인트의 에러율이 **99.54%** 였다. 요청의 절반도 아니고 거의 전부가 실패하고 있었다. p95 응답시간은 1,800ms였고 그마저도 대부분 timeout이라 실제로 응답을 받은 요청이 거의 없었다.

원인은 단순했다. 피드 20건을 조회할 때 쿼리가 61개 나가고 있었다. 피드 1건에 댓글, 좋아요, 리포스트 컬렉션을 각각 따로 조회하는 전형적인 N+1이었다. 눈에 띄는 버그는 아니었지만 트래픽이 붙는 순간 DB 커넥션 풀을 잡아먹고 전체가 죽는 구조였다.

---

## 시스템 구조

### 피드 조회 흐름

```
클라이언트 → /feed/popular
    ↓
FeedService.getPopularFeed()
    ↓
feedRepository.findPopular(pageable)   → Feed N건 SELECT
    ↓
각 Feed에 대해
    feed.getComments().size()          → Feed마다 SELECT
    feed.getLikes().size()             → Feed마다 SELECT
    feed.getReposts().size()           → Feed마다 SELECT
    ↓
FeedResponseDto로 변환
```

20건을 조회하면 `1 + 3 × 20 = 61`개 쿼리가 나간다. 인기 피드는 좋아요와 댓글이 많을수록 느려지고, 느려질수록 커넥션을 오래 잡고, 커넥션 풀이 차면 다른 요청까지 대기한다.

---

## 문제 상세

### N+1 — 피드 20건에 쿼리 61개

```java
// FeedService.java
List<FeedItem> feeds = feedRepository.findPopular(pageable);  // 1번

for (FeedItem feed : feeds) {
    feed.getComments().size();   // N번 — 댓글 전체 로딩 후 .size()
    feed.getLikes().size();      // N번 — 좋아요 전체 로딩 후 .size()
    feed.getReposts().size();    // N번 — 리포스트 전체 로딩 후 .size()
}
```

카운트 하나를 얻으려고 컬렉션 전체를 메모리에 올리고 있었다. 인기 피드의 좋아요가 500개라면 500개의 `FeedLike` 엔티티를 영속화한 뒤 `.size()`만 쓰고 버린다. `COUNT(*)` 쿼리 하나면 될 것을 엔티티 전체 로딩으로 처리하고 있었던 것이다.

k6 측정 결과:

| 엔드포인트 | p95 | 에러율 |
|-----------|-----|--------|
| /feed/popular | timeout | 99.54% |
| /feed/timeline | 2,100ms | - |
| /feed/like | 850ms | - |
| /feed/comment | 920ms | - |

---

### 댓글 생성마다 `COUNT(*)` 재계산

```java
// CommentService.java
public void addComment(Long feedId, String text) {
    commentRepository.save(new Comment(feedId, text));
    // 댓글 수는 조회할 때마다 COUNT(*) 쿼리로 집계
}
```

댓글 수를 보여줄 때마다 `SELECT COUNT(*) FROM comment WHERE feed_id = ?`를 실행했다. 피드 목록 20건이면 20번의 COUNT 쿼리가 추가로 나간다. 비정규화 컬럼인 `commentCount`를 `FeedItem`에 두면 조회 시 COUNT 쿼리가 전혀 필요 없다.

---

### 인기 피드 정렬 인덱스 없음 — `created_at DESC` 풀스캔

```java
// FeedRepository.java
@Query("SELECT f FROM FeedItem f ORDER BY f.createdAt DESC, f.likeCount DESC")
List<FeedItem> findPopular(Pageable pageable);
```

`created_at DESC`로 정렬하면서 `like_count`까지 함께 내려받는데, 이를 커버하는 인덱스가 없었다. 데이터가 많아질수록 매번 풀스캔 + 정렬이 발생한다. 커버링 인덱스가 있으면 인덱스만으로 정렬과 데이터 조회가 완결되어 실제 테이블에 접근하지 않아도 된다.

---

### 좋아요 토글 Race Condition

```java
// LikeService.java
public void toggleLike(Long feedId, Long userId) {
    Optional<FeedLike> existing = likeRepository.findByFeedAndUser(feedId, userId);
    if (existing.isPresent()) {
        likeRepository.delete(existing.get());
    } else {
        likeRepository.save(new FeedLike(feedId, userId));
    }
}
```

`findByFeedAndUser`와 `save` 사이에 다른 요청이 끼어들면 중복 좋아요가 생긴다. 두 요청이 동시에 `Optional.empty()`를 받고 둘 다 `save`까지 진행하는 구조다. 좋아요 같은 토글 연산은 단일 왕복으로 처리해야 Race Condition이 없다.

---

## 해결

### Two-Pass Query — 61개 쿼리를 2개로

N+1의 근본 원인은 "피드를 먼저 가져오고, 각 피드에 딸린 것들을 나중에 가져오는" 흐름이었다. 이걸 "ID 목록만 먼저 가져오고, ID 기반으로 필요한 것을 한 번에 가져오는" 2-pass 구조로 바꿨다.

```java
// Pass 1: 정렬 기준에 맞는 피드 ID만 조회 (인덱스만 사용)
List<Long> ids = feedRepository.findPopularIds(pageable);

// Pass 2: 해당 ID들에 대해 JOIN FETCH로 한 번에
List<FeedItem> feeds = feedRepository.findByIdInWithRelations(ids);
// → Feed + Author + Images를 쿼리 1개로
```

총 쿼리 수가 `1 + 3N`에서 `2`로 줄었다. Pass 1은 인덱스만 읽고, Pass 2는 `JOIN FETCH`로 필요한 관계를 한 번에 가져온다. 컬렉션 전체 로딩이 사라지고 `commentCount`, `likeCount` 비정규화 컬럼에서 직접 읽는다.

---

### `commentCount` 비정규화 — COUNT 쿼리 제거

```java
// FeedItem.java
@Builder.Default
private int commentCount = 0;

// CommentService.java
public void addComment(Long feedId, String text) {
    commentRepository.save(new Comment(feedId, text));
    feedItemRepository.incrementCommentCount(feedId);  // atomic UPDATE
}
```

```java
// FeedItemRepository.java
@Modifying
@Query("UPDATE FeedItem f SET f.commentCount = f.commentCount + 1 WHERE f.id = :feedId")
void incrementCommentCount(@Param("feedId") Long feedId);
```

댓글 생성 시 `UPDATE feed_item SET comment_count = comment_count + 1`을 원자적으로 실행한다. 이후 조회 시 `COUNT(*)` 없이 컬럼 값을 바로 읽는다. 5화의 `memberCount`와 같은 방식이다.

---

### Redis Lua Script — 좋아요 토글 원자화

```lua
-- like_toggle.lua
local key = KEYS[1]      -- feed:{feedId}:likers
local userId = ARGV[1]

if redis.call('SISMEMBER', key, userId) == 1 then
    redis.call('SREM', key, userId)
    return 0  -- 좋아요 취소
else
    redis.call('SADD', key, userId)
    return 1  -- 좋아요
end
```

Lua 스크립트는 Redis에서 원자적으로 실행된다. 체크와 업데이트가 단일 연산이 되므로 Race Condition이 없다. 네트워크 왕복도 1회로 줄어든다. DB에는 Redis Stream을 통해 비동기로 반영한다.

---

### 커버링 인덱스 — 인기 피드 정렬 최적화

```sql
CREATE INDEX idx_feed_popular
    ON feed_item(created_at DESC, id, user_id, type, content, comment_count, like_count);
```

정렬 기준인 `created_at DESC`를 선두에 두고, 응답에 필요한 컬럼들을 인덱스에 포함했다. 인덱스만으로 정렬과 데이터 조회가 완결되어 실제 테이블 접근(랜덤 I/O)이 발생하지 않는다. Pass 1의 ID 조회가 Index-Only Scan으로 동작한다.

---

## k6 결과

1,000 VU × 3분 시나리오로 측정했다.

| 엔드포인트 | Before p95 | After p95 | 개선율 |
|-----------|-----------|----------|--------|
| /feed/popular | timeout | 1,006ms | -44% |
| /feed/timeline | 2,100ms | 1,180ms | -43.8% |
| /feed/like | 850ms | 320ms | -62.4% |
| /feed/comment | 920ms | 410ms | -55.4% |

에러율은 99.54%에서 0.61%로 내려왔다. Timeout이 완전히 사라졌다.

수치보다 더 중요한 건 왜 이 정도 차이가 났냐는 것이다. N+1은 트래픽이 없을 때는 티가 안 난다. 피드 1건만 조회하면 쿼리 4개고 200ms 안에 끝난다. 그런데 동시 요청이 100개가 들어오면 각자 61개씩 쿼리를 날리니 DB 커넥션 풀이 순식간에 찬다. 커넥션을 기다리는 요청들이 쌓이고 타임아웃이 연쇄적으로 터진다. 99% 에러율의 실체는 N+1 자체가 아니라 N+1이 만들어낸 커넥션 풀 고갈이었다.

---

## 교훈

N+1은 개발 환경에서 거의 보이지 않는다. 데이터가 적고 동시 요청이 없으니 61개 쿼리가 10ms 안에 다 끝난다. 로컬에서 잘 돌아가고 테스트도 통과한다. 문제는 프로덕션 트래픽 아래서 터지는데, 그때는 로그가 수천 줄씩 쏟아져서 원인을 찾기가 어렵다.

부하 테스트를 먼저 돌려보길 잘했다. 99% 에러율이라는 숫자가 없었다면 "응답이 조금 느린 것 같다" 정도로 넘어갔을 것이다. 숫자가 있어야 얼마나 심각한지 알고, 개선 후에 얼마나 나아졌는지도 알 수 있다.

> **N+1은 로컬에서 죽지 않는다. 부하 아래서 죽는다.**
> 기능 테스트만으로는 보이지 않는 문제가 있다. 실제 트래픽과 유사한 조건에서 한 번이라도 돌려봐야 숨어 있는 N+1과 커넥션 풀 고갈이 드러난다.
