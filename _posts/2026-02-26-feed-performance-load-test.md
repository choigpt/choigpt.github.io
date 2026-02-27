---
title: 피드 도메인 — 부하 테스트 9라운드, 슬로우 쿼리 29,008건에서 전 구간 통과까지
date: 2026-02-26
tags: [MySQL, JPA, Redis, Spring, 성능최적화, 트러블슈팅, Java, 피드, k6, 인덱스, 동시성]
permalink: /feed-performance-load-test/
excerpt: "첫 측정에서 개인피드 p95 2,198ms, 슬로우 쿼리 29,008건. 문제는 N+1, 대형 IN절, 데드락, Redis 워밍업 블로킹이 겹쳐 있었다. 9라운드에 걸쳐 p95 41ms, 전 구간 성공률 100%에 도달한 기록."
---

## 개요

피드 도메인에 k6 부하 테스트를 걸었다. 첫 측정부터 숫자가 심각했다. 개인피드 p95 2,198ms, 피드상세 p95 2,725ms, 슬로우 쿼리 29,008건. 좋아요·댓글 쓰기만 p95 20~24ms로 빠르고 나머지는 전부 병목이었다.

문제는 하나가 아니었다. N+1 쿼리, 상관 서브쿼리, 인덱스 부재, 데드락, IN절 폭발, Redis 워밍업 블로킹, 인메모리 캐시 부재가 겹쳐 있었다. 한 병목을 고치면 다음 병목이 드러나는 패턴이 반복됐다. 9라운드 끝에 피드상세 p95 41ms, 전 구간 성공률 100%에 도달했다.

---

## 테스트 환경

| 항목 | 설정 |
|------|------|
| 도구 | k6 (Docker) |
| 데이터 규모 | feed 50만, feed_comment 245만, feed_like 512만, feed_image 150만, user 100만 |
| 테스트 Phase | P1 읽기(300VU), P2 상호작용(200VU), P3 혼합(300VU), P4 상세(300VU), P5 스파이크(400VU) |
| Threshold | 읽기 p95 < 500ms, 쓰기 p95 < 300ms, 성공률 > 95% |

---

## 1차 측정 — 병목 전체 파악

### 최초 부하 테스트 결과

| Phase | 지표 | avg | p95 | 성공률 |
|-------|------|-----|-----|--------|
| P1 읽기 300VU | 개인피드 | 716ms | 2,198ms | 100% |
| | 인기피드 | 321ms | 904ms | |
| | 클럽피드 | 239ms | 695ms | |
| P2 상호작용 200VU | 좋아요 | 34ms | 24ms | 99.9% |
| | 댓글 | 34ms | 20ms | |
| P3 혼합 300VU | 읽기 | 492ms | 2,903ms | 99.9% |
| | 쓰기 | 344ms | 1,407ms | |
| P4 상세 300VU | 피드상세 | 652ms | 2,725ms | 99.9% |
| | 댓글목록 | 295ms | 947ms | |
| P5 스파이크 400VU | 읽기 | 1,539ms | 5,405ms | 99.4% |
| | 쓰기 | 1,184ms | 3,079ms | |
| 전체 | 슬로우 쿼리 | — | — | 29,008건 |

쓰기(좋아요·댓글)는 Redis Lua + 단순 INSERT 덕분에 빠른데, 읽기 경로가 핵심 병목이었다.

---

## 1차 측정에서 드러난 병목

### 병목 1. 피드상세 — p95 2,725ms

```java
// FeedQueryService.java (수정 전)
public FeedDetailResponseDto getFeedDetail(Long clubId, Long feedId) {
    Club club = clubRepository.findById(clubId);       // 1쿼리
    Feed feed = feedRepository.findByFeedIdAndClub();  // 2쿼리
    boolean isLiked = feedLikeRepository.existsByFeed_FeedIdAndUser_UserId();  // 3쿼리
    List<FeedComment> comments = feed.getFeedComments();  // N+1 발생 ← 핵심
    long repostCount = feedRepository.countByParentFeedId(); // 4+N쿼리
}
```

원인이 세 가지였다.

1. **댓글 전체 로드 N+1**: `feed.getFeedComments()`가 댓글 전체를 LAZY 로딩하고, 각 댓글의 User도 별도 SELECT. 댓글 20개 → 21쿼리 추가
2. **Club 별도 조회**: `clubRepository.findById()` → `feedRepository.findByFeedIdAndClub()` 순서로 2쿼리 소비
3. **JOIN FETCH 미사용**: 이미 `findByIdAndClubIdWithRelations()`라는 메서드를 만들어뒀는데 사용하지 않고 있었다

---

### 병목 2. 개인피드 IN절 — p95 2,198ms

```java
// FeedQueryService.java
public List<FeedOverviewDto> getPersonalFeed(Pageable pageable) {
    List<Long> clubIds = resolveAccessibleClubIds(userId);  // 매 요청마다 DB 쿼리
    feedRepository.findFeedIdsWithCountsByClubIds(clubIds, pageable);  // WHERE club_id IN (20+개)
}
```

유저가 20개 이상의 클럽에 가입한 경우 `WHERE club_id IN (:clubIds)` 절에 20개 이상의 값이 들어갔다. MySQL 옵티마이저가 대형 IN + ORDER BY + LIMIT 조합에서 filesort를 선택하며 인덱스 활용이 어려워졌다. 여기에 `resolveAccessibleClubIds()`가 매 요청마다 DB를 치는 구조도 겹쳤다.

---

### 병목 3. 댓글목록 N+1 — p95 947ms

```java
// FeedCommentService.java (수정 전)
public List<FeedCommentResponseDto> getCommentList(Long feedId, Pageable pageable) {
    Feed feed = feedRepository.findById(feedId);  // 불필요한 Feed 전체 조회
    return feedCommentRepository.findByFeedOrderByCreatedAt(feed, pageable)  // JOIN FETCH 없음
            .stream().map(c -> FeedCommentResponseDto.from(c, userId));  // User N+1 발생
}
```

`findByFeedIdWithUser()`라는 JOIN FETCH 메서드가 이미 있었지만 사용되지 않고 있었다. 댓글 20개 기준 21쿼리가 발생했다. `(feed_id, created_at)` 복합 인덱스도 없어서 ORDER BY에 별도 정렬이 추가됐다.

---

### 병목 4. 쓰기 경합 — 데드락

댓글 작성과 좋아요 동기화가 같은 feed 행의 X lock을 두고 경합했다.

```
댓글 작성 (FeedCommentService)
BEGIN TX
  INSERT feed_comment  ← feed에 S lock (FK 체크)
  UPDATE feed SET comment_count++  ← X lock 요청 → 대기
COMMIT

좋아요 동기화 (FeedLikeStreamConsumer)
BEGIN TX
  UPDATE feed SET like_count++  ← X lock 보유
COMMIT
```

두 트랜잭션이 S lock을 잡고 X lock을 대기하면 데드락이 됐다.

---

## 1~3라운드 수정 — JPA 최적화 + 인덱스

### 수정 1. 피드상세 쿼리 통합

```java
// FeedQueryService.java (수정 후)
public FeedDetailResponseDto getFeedDetail(Long clubId, Long feedId) {
    // Club + Feed + User + Images를 1쿼리로
    Feed feed = feedRepository.findByIdAndClubIdWithRelations(feedId, clubId)
            .orElseThrow(() -> new CustomException(ErrorCode.FEED_NOT_FOUND));
    
    boolean isLiked = feedLikeRepository.existsByFeed_FeedIdAndUser_UserId(feedId, currentUserId);
    
    // JOIN FETCH로 N+1 제거, 최근 20개만
    List<FeedCommentResponseDto> comments = feedCommentRepository
            .findByFeedIdWithUser(feedId, PageRequest.of(0, 20))
            .stream().map(c -> FeedCommentResponseDto.from(c, currentUserId))
            .toList();
    
    long repostCount = feedRepository.countByParentFeedId(feedId);
    // 2 + N+1 쿼리 → 4쿼리 고정
}
```

### 수정 2. 댓글목록 N+1 제거

```java
// FeedCommentService.java (수정 후)
return feedCommentRepository.findByFeedIdWithUser(feedId, pageable)
        .stream().map(c -> FeedCommentResponseDto.from(c, userId))
        .toList();
// 21쿼리 → 1쿼리
```

### 수정 3. 복합 인덱스 추가

```sql
-- 피드 목록·repost 쿼리 커버
CREATE INDEX idx_feed_club_deleted_created ON feed(club_id, deleted, created_at);
CREATE INDEX idx_feed_parent_deleted ON feed(parent_feed_id, deleted);

-- 댓글 정렬
CREATE INDEX idx_feed_comment_feed_created ON feed_comment(feed_id, created_at ASC);
```

### 수정 4. 데드락 수정 — 트랜잭션 순서 역전

```java
// FeedCommentService.createComment() (수정 후)
// X lock 선점 → INSERT 시 FK 체크에서 이미 보유한 X lock 재사용
feedRepository.incrementCommentCount(feedId);  // UPDATE 먼저
feedCommentRepository.save(feedComment);        // INSERT 나중
```

### 1~3라운드 후 결과

| 지표 | Before | After | 개선율 |
|------|--------|-------|--------|
| 개인피드 p95 | 2,198ms | 107ms | **-95%** |
| 인기피드 p95 | 904ms | 65ms | **-93%** |
| 클럽피드 p95 | 695ms | 53ms | **-92%** |
| 피드상세 p95 | 2,725ms | 99ms | **-96%** |
| 댓글목록 p95 | 947ms | 159ms | **-83%** |
| 슬로우 쿼리 | 29,008건 | 9,938건 | -66% |
| 5xx 에러 | 86건 | 10건 | -88% |
| 처리량 | 176 req/s | 388 req/s | **+2.2x** |

---

## 4라운드 — 데이터 5배 스케일업 후 고부하 재탐지

기존 데이터로는 병목이 드러나지 않을 수 있어서 데이터를 실서비스 수준으로 올렸다.

| 테이블 | Before | After |
|--------|--------|-------|
| feed | 100K | 500K |
| feed_comment | 531K | 2.4M |
| feed_like | 1M | 5.1M |
| feed_image | 300K | 1.5M |

고부하 탐지 결과, 새로운 병목이 드러났다.

| 순위 | 병목 | p95 | 원인 |
|------|------|-----|------|
| 1 | 개인피드 IN절 | 919ms | 유저당 20+개 클럽 → 대형 IN clause |
| 2 | 인기피드 정렬 | 567ms | `LOG() + TIMESTAMPDIFF()` CPU-bound 정렬 |
| 3 | 쓰기 경합 | 554ms | comment_count++와 like_count++ X lock 경합 |

---

## 5~7라운드 수정 — IN절 최적화 + 캐싱

### 수정 5. 개인피드 UNION ALL 청크 분할

대형 IN절을 5개씩 청크로 쪼개어 각 청크가 인덱스를 온전히 활용하도록 분해했다.

```java
// FeedQueryService.java
private static final int CLUB_CHUNK_SIZE = 5;

private List<FeedRepository.FeedIdWithCounts> findPersonalFeedChunked(
        List<Long> clubIds, Pageable pageable) {
    int limit = (int) pageable.getOffset() + pageable.getPageSize();

    StringBuilder sql = new StringBuilder("SELECT feedId, likeCount, commentCount FROM (");
    Map<String, Object> paramMap = new HashMap<>();

    // 청크별 UNION ALL — 각각 (club_id, deleted, created_at) 인덱스 활용
    List<List<Long>> chunks = partitionList(clubIds, CLUB_CHUNK_SIZE);
    for (int i = 0; i < chunks.size(); i++) {
        if (i > 0) sql.append(" UNION ALL ");
        String paramName = "c" + i;
        sql.append("(SELECT f.feed_id as feedId, f.like_count as likeCount, ")
           .append("f.comment_count as commentCount, f.created_at as createdAt ")
           .append("FROM feed f WHERE f.club_id IN (:").append(paramName)
           .append(") AND f.deleted = false ORDER BY f.created_at DESC LIMIT ").append(limit).append(")");
        paramMap.put(paramName, chunks.get(i));
    }
    sql.append(") t ORDER BY createdAt DESC LIMIT :offset, :pageSize");
    // ...
}
```

`clubIds.size() <= 5`이면 기존 단일 쿼리를 사용하고, 초과 시에만 청크 분할 경로로 진입한다.

### 수정 6. 전용 커버링 인덱스 추가

```sql
-- 개인피드 IN절 최적화: club_id가 선행 컬럼이어야 range scan 가능
CREATE INDEX idx_feed_personal_cover
    ON feed(club_id, deleted, created_at DESC, feed_id, like_count, comment_count);
```

기존 `idx_feed_popular_cover`는 `(deleted, club_id, ...)` 순서라 IN절에서 club_id가 두 번째 컬럼이었다. 새 인덱스는 club_id를 선행 컬럼으로 두어 각 청크가 인덱스 range scan 후 즉시 LIMIT 푸시다운이 가능하다.

### 수정 7. 인메모리 결과 캐시 — 직렬화 비용 0

Redis JSON 캐시를 시도했으나 400VU에서 ObjectMapper 리플렉션 기반 직렬화가 CPU 병목이 됐다. 대신 JVM 힙의 `ConcurrentHashMap`을 사용했다.

```java
// FeedQueryService.java
private static final long RESULT_CACHE_TTL_MS = 10_000;  // 10초
private static final int MAX_RESULT_CACHE_SIZE = 2000;

private record CachedResult(List<FeedOverviewDto> data, long expiresAt) {
    boolean isExpired() { return System.currentTimeMillis() > expiresAt; }
}

private static final ConcurrentHashMap<String, CachedResult> resultCache = new ConcurrentHashMap<>();
```

캐시 키는 `pf:{userId}:{page}:{pageSize}` (개인피드), `ppf:{userId}:{page}:{pageSize}` (인기피드). 딥 페이지(6페이지 이상)는 캐싱하지 않아 키 폭발을 방지했다.

캐시 히트 시 `resolveAccessibleClubIds → pass1 → findByIdsWithRelations → findLikedFeedIdsByUser → bulkLoadParents → bulkLoadRoots → countDirectReposts` 최대 8개 DB 쿼리를 전부 스킵한다.

---

## 8~9라운드 수정 — 쓰기 경합 해소 + 피드상세 캐시

### 수정 8. 좋아요 워밍업 비동기 전환

워밍업이 동기로 실행되면서 200VU가 동시에 서로 다른 피드의 `findUserIdsByFeedId(feedId)`를 실행하고 있었다. 인기 피드 좋아요가 1,500개면 워밍업 1회에 1,500 row를 읽는다.

```java
// FeedLikeService.java (수정 전)
private void ensureLikeCacheWarmed(long feedId) {
    // DB에서 전체 userId 로딩 → 현재 요청 블로킹
    List<Long> userIds = feedLikeRepository.findUserIdsByFeedId(feedId);
    redis.opsForSet().add(likersKey, ...);
}

// FeedLikeService.java (수정 후)
private static final ExecutorService warmupExecutor = Executors.newFixedThreadPool(2);
private static final Set<Long> warmingUp = ConcurrentHashMap.newKeySet();

private void triggerAsyncWarmup(long feedId) {
    if (Boolean.TRUE.equals(redis.hasKey("feed:" + feedId + ":like_count"))) return;
    if (!warmingUp.add(feedId)) return;  // 이미 진행 중

    warmupExecutor.submit(() -> {
        try {
            List<Long> userIds = feedLikeRepository.findUserIdsByFeedId(feedId);
            redis.opsForSet().add(likersKey, ...);
        } finally {
            warmingUp.remove(feedId);
        }
    });
    // 현재 요청은 바로 Lua 스크립트 실행 (블로킹 없음)
}
```

200VU에서 DB 커넥션 소비가 200개 → 최대 2개(워밍업 스레드)로 줄었다. Lua 스크립트는 SET이 없어도 SADD가 key를 자동 생성하므로 워밍업 완료 전 요청도 정상 동작한다.

### 수정 9. 댓글 트랜잭션 분리 — X lock 경합 최소화

댓글 저장/삭제와 count UPDATE를 `runInTx` / `updateCountSafely` 헬퍼로 분리했다. 두 작업이 각각 독립 트랜잭션으로 실행되어 comment_count X lock 보유 시간이 ~5ms → ~1ms로 줄었다.

```java
// FeedCommentService.java (수정 후)

// 댓글 작성
runInTx(() -> feedCommentRepository.save(feedComment));
updateCountSafely(
    () -> feedRepository.incrementCommentCount(feedId),
    "댓글 카운트 증가 실패 (댓글은 정상 저장됨)", feedId
);

// 댓글 삭제
runInTx(() -> feedCommentRepository.delete(feedComment));
updateCountSafely(
    () -> feedRepository.decrementCommentCount(feedId),
    "댓글 카운트 감소 실패 (댓글은 정상 삭제됨)", feedId
);

// updateCountSafely 내부
private void updateCountSafely(Runnable countUpdate, String warnMsg, Long feedId) {
    try {
        runInTx(countUpdate);
    } catch (Exception e) {
        // 댓글 데이터(INSERT/DELETE)는 이미 커밋됐지만 comment_count가 틀어진다.
        // like_count와 달리 comment_count는 Redis 키가 없고 DB 컬럼이 유일한 소스다.
        // FeedRenderService의 폴백도 f.getCommentCount()를 직접 읽으므로,
        // 실패 시 틀어진 값이 보정 없이 API 응답에 그대로 노출된다.
        // 실서비스라면 스케줄러 기반 재집계 또는 DB 트리거로 보정하는 구조가 필요하다.
        log.warn(warnMsg + ": feedId={}", feedId);
    }
}
```

`like_count++` UPDATE와의 경합 시간이 80%+ 감소했다.

### 수정 10. 피드상세 공통 데이터 캐시

피드상세는 `isLiked`, `isFeedMine`, `isCommentMine`만 유저별로 달라지고 나머지는 공통이다.

```java
// FeedQueryService.java
private record DetailCacheEntry(
        Feed feed, List<String> imageUrls,
        List<FeedCommentResponseDto> comments, long repostCount,
        long expiresAt
) {
    boolean isExpired() { return System.currentTimeMillis() > expiresAt; }
}
private static final ConcurrentHashMap<Long, DetailCacheEntry> detailCache = new ConcurrentHashMap<>();
private static final long DETAIL_CACHE_TTL_MS = 5_000;

public FeedDetailResponseDto getFeedDetail(Long clubId, Long feedId) {
    Long currentUserId = userService.getCurrentUser().getUserId();

    DetailCacheEntry cached = detailCache.get(feedId);
    if (cached != null && !cached.isExpired()) {
        // 캐시 히트 시 좋아요 1쿼리만 실행
        boolean isLiked = feedLikeRepository.existsByFeed_FeedIdAndUser_UserId(feedId, currentUserId);
        boolean isMine = cached.feed().getUser().getUserId().equals(currentUserId);
        List<FeedCommentResponseDto> userComments = cached.comments().stream()
                .map(c -> new FeedCommentResponseDto(
                        c.commentId(), c.userId(), c.nickname(), c.profileImage(), c.content(),
                        c.createdAt(), c.userId().equals(currentUserId)))
                .toList();
        return FeedDetailResponseDto.from(cached.feed(), cached.imageUrls(), isLiked, isMine, userComments, cached.repostCount());
    }

    // 캐시 미스 — DB 4쿼리 실행 후 캐싱
    // ...
}
```

캐시 히트 시 4쿼리 → 1쿼리. 같은 피드를 여러 유저가 보는 상황에서 효과가 크다.

---

## 최종 결과 — 전체 최적화 Before vs After

| 지표 | Before (최적화 전) | After (최적화 후) | 총 개선율 |
|------|-------------------|------------------|-----------|
| 개인피드 p95 | 2,198ms | ~107ms | **-95%** |
| 인기피드 p95 | 904ms | ~65ms | **-93%** |
| 클럽피드 p95 | 695ms | ~53ms | **-92%** |
| 피드상세 p95 | 2,725ms | **41ms** | **-98%** |
| 댓글목록 p95 | 947ms | **34ms** | **-96%** |
| 좋아요 p95 | 24ms | 49ms | 유지 |
| 댓글작성 p95 | 20ms | 35ms | 유지 |
| 슬로우 쿼리 | 29,008건 | ~수십 건 | **-99%+** |
| 5xx 에러 | 86건 | 0건 | **-100%** |
| API 성공률 | 99.88% | **100%** | |
| 총 처리량 | 176 req/s | 529K iters | **+3x** |

---

## 수정 내역 요약

| # | 파일 | 변경 내용 |
|---|------|----------|
| 1 | `FeedQueryService.java` | `getFeedDetail`: 2+N+1 → 4쿼리 고정 (JOIN FETCH, 댓글 LIMIT 20) |
| 2 | `FeedCommentService.java` | `getCommentList`: `findByFeedIdWithUser` 교체, N+1 제거 |
| 3 | `schema.sql` | `idx_feed_club_deleted_created`, `idx_feed_parent_deleted`, `idx_feed_comment_feed_created` 추가 |
| 4 | `FeedCommentService.java` | `createComment`: `incrementCommentCount` → `save` 순서 역전 (데드락 해소) |
| 5 | `FeedQueryService.java` | `findPersonalFeedChunked`: UNION ALL 청크 분할 (EntityManager 기반) |
| 6 | `schema.sql` | `idx_feed_personal_cover(club_id, deleted, created_at DESC, ...)` 추가 |
| 7 | `FeedQueryService.java` | 개인/인기피드 전체 결과 인메모리 캐시 (10초 TTL, 앞쪽 5페이지만) |
| 8 | `FeedLikeService.java` | `ensureLikeCacheWarmed` → `triggerAsyncWarmup` 비동기 전환 (스레드풀 2) |
| 9 | `FeedCommentService.java` | 댓글 INSERT/DELETE와 count UPDATE를 별도 트랜잭션으로 분리 (`updateCountSafely`) |
| 10 | `FeedLikeStreamConsumer.java` | `BATCH_COUNT` 8 → 1 (X lock 범위 최소화) |
| 11 | `FeedQueryService.java` | `getFeedDetail` 공통 데이터 인메모리 캐시 (5초 TTL) |

---

## 시행착오

### FORCE INDEX 역효과

커버링 인덱스를 schema.sql에 추가했지만 서버 재시작 전까지 DB에 반영이 안 된 상태에서 `FORCE INDEX`를 쿼리에 걸었다. 존재하지 않는 인덱스를 FORCE INDEX로 지정하면 MySQL이 에러를 반환한다. 개인피드 성공률이 0%로 떨어지는 이유를 찾는 데 시간을 썼다.

> **FORCE INDEX는 반드시 해당 인덱스가 DB에 실제로 존재한 후에 적용해야 한다.** `SHOW INDEX FROM feed`로 먼저 확인하자.

### ObjectMapper JSON 캐싱 실패

Redis에 JSON 직렬화로 전체 결과를 캐싱하려 했는데, 400VU에서 ObjectMapper의 리플렉션 오버헤드가 오히려 CPU 병목이 됐다. Redis I/O보다 직렬화 자체가 더 느렸다.

> **역직렬화 비용이 DB 쿼리보다 크다면 캐시가 역효과다.** JVM 힙 인메모리 캐시는 직렬화 비용이 0이고 네트워크 I/O도 없어서 훨씬 효율적이다.

### 캐시 키 폭발 — 딥 페이지 제외

1000명 유저 × 200페이지 조합으로 캐시 키가 최대 20만 개까지 생성됐다. Redis 메모리 압박과 P6 딥 페이지네이션 구간 성능 저하로 이어졌다. 앞쪽 5페이지만 캐싱하도록 범위를 제한하니 최대 12,000키로 줄었다.

### 인메모리 캐시는 근본 해결이 아니다

캐시는 증상을 완화할 뿐, 쿼리 자체가 느린 건 해결되지 않는다. 개인피드는 구조적으로 fan-out-on-write(트위터 방식) 없이는 IN절 폭발 문제를 근본적으로 해결하기 어렵다. 실서비스라면 피드 생성 시 해당 클럽 멤버 전원의 타임라인 테이블에 INSERT하는 방식이 올바른 해결이다. 현재 프로젝트 규모에서는 15초 TTL 캐시가 현실적 절충안이다.

---

## 정리하며

N+1을 고치면 IN절 폭발이 나타나고, IN절을 고치면 캐시 키 폭발이 나타나고, 캐시를 추가하면 직렬화 병목이 생겼다. 병목은 항상 다음 병목을 가리고 있었다.

가장 힘든 부분은 FORCE INDEX 실수나 ObjectMapper 선택처럼 방향 자체가 틀린 시도를 알아채는 데 드는 시간이었다. 성공률이 0%로 떨어지면 "왜"를 찾아야 하는데, 원인이 코드가 아니라 인덱스 존재 여부일 거라고 생각하기 쉽지 않았다.

> **부하 테스트는 "빠른가"를 측정하는 게 아니라 "어디서 무너지는가"를 드러내는 도구다.**
> 단건 API가 200ms여도 300VU에서 30초가 될 수 있다. 코드가 아닌 커넥션 풀, 인덱스, 트랜잭션 범위가 병목일 때가 많다.

---

## 시리즈 탐색

**◀ 이전 글**  
[채팅 도메인 — 상관 서브쿼리·Redis 리스너 미등록·N+1이 만든 성능 저하](/chat-subquery-redis-listener/)

**다음 글 ▶**  
[알림 API 성능 — 13% 성공률에서 100%로, 9번의 시도 기록](/notification-api-performance-improvement/)
