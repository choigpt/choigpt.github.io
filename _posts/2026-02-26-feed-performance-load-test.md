---
title: 피드 도메인 — 부하 테스트 9라운드, 슬로우 쿼리 29,008건에서 전 구간 통과까지
date: 2026-02-26
tags: [MySQL, Redis, JPA, 성능최적화, 트러블슈팅, Java, 피드, k6, 캐싱, 인덱스]
permalink: /feed-performance-load-test/
excerpt: "피드 도메인 첫 부하 테스트에서 슬로우 쿼리가 29,008건 발생했다. 개인피드 p95 2,198ms, 피드상세 p95 2,725ms. N+1, 상관 서브쿼리, IN절 폭발, 댓글 전체 로드, 데드락 — 하나씩 고치며 9라운드 끝에 전 구간 통과했다."
---

## 개요

채팅 도메인 부하 테스트를 마친 뒤 피드 도메인으로 넘어왔다. 피드 도메인은 개인 타임라인·인기피드·클럽피드·피드상세·좋아요·댓글·리포스트가 얽혀 있는 구조라 쿼리 복잡도가 가장 높은 도메인이었다.

첫 k6 테스트에서 슬로우 쿼리가 29,008건 나왔다. 채팅 도메인의 1,332건 대비 21배다. 개인피드 p95는 2,198ms, 피드상세는 2,725ms였다. 9라운드 끝에 전 구간 threshold를 통과했다. 각 라운드에서 무엇을 발견하고 어떻게 고쳤는지를 기록한다.

---

## 시스템 구조

```
GET /api/v1/feeds/personal     → FeedQueryService.getPersonalFeed()
GET /api/v1/feeds/popular      → FeedQueryService.getPopularFeed()
GET /api/v1/clubs/{id}/feeds   → FeedQueryService.getClubFeed()
GET /api/v1/feeds/{id}         → FeedQueryService.getFeedDetail()
POST /api/v1/feeds/{id}/likes  → FeedCommandService.toggleLike()
POST /api/v1/feeds/{id}/comments → FeedCommentCommandService.createComment()
```

개인피드 조회는 3-pass로 동작한다. Pass1에서 유저가 속한 클럽들의 피드 ID + 비정규화 카운트를 가져오고, Pass2에서 ID 목록으로 피드 본문을 조회하며, Pass3에서 좋아요·부모·루트를 bulk 로딩한다. 인기피드는 `LOG(like_count + 1) + TIMESTAMPDIFF()` 인기도 알고리즘 정렬을 수행한다.

---

## 1라운드 — 첫 측정, 슬로우 쿼리 29,008건

데이터 규모: feed 100K, feed_like 500K, feed_comment 200K, user_club 50만(유저당 5개 클럽).

| Phase | 항목 | avg | p95 | max | 성공률 |
|-------|------|-----|-----|-----|--------|
| P1 Read (300VU) | 개인피드 | 716ms | 2,198ms | - | 100% |
| | 인기피드 | 321ms | 904ms | - | |
| | 클럽피드 | 239ms | 695ms | - | |
| P4 Detail (300VU) | 피드상세 | 652ms | 2,725ms | - | 99.9% |

주요 병목점:
1. 개인피드 p95 2,198ms — 3-pass 쿼리가 유저의 모든 클럽 피드를 스캔
2. 피드상세 p95 2,725ms — 댓글·이미지·좋아요 lazy loading + N+1
3. 인기피드 p95 904ms — `LOG()` + `TIMESTAMPDIFF()` 정렬
4. 슬로우 쿼리 29,008건 — 채팅(1,332건) 대비 21배

---

## 2라운드 — N+1, 상관 서브쿼리, 인덱스, 댓글 LIMIT

### N+1 — 피드상세 21쿼리 → 1쿼리

피드 상세 조회 시 댓글 목록을 `findByFeedIdWithUser(feedId)`로 가져올 때 댓글마다 user 연관이 LAZY 로딩됐다. 댓글 20개면 21쿼리다.

```java
// 수정 전
List<FeedComment> findByFeedIdWithUser(Long feedId);

// 수정 후 — JOIN FETCH + 페이지네이션
@Query("select c from FeedComment c join fetch c.user where c.feed.feedId = :feedId order by c.createdAt asc")
List<FeedComment> findByFeedIdWithUser(Long feedId, Pageable pageable);
```

`PageRequest.of(0, 20)`으로 최근 20개만 로드하도록 변경했다. 댓글 수천 개인 피드에서 전부 메모리에 올리던 문제도 함께 해결됐다.

### 상관 서브쿼리 — 클럽 피드 썸네일 쿼리

각 클럽의 최신 피드 썸네일을 가져오는 쿼리에 상관 서브쿼리가 있었다.

```sql
-- 수정 전: DEPENDENT SUBQUERY — 매 row마다 서브쿼리 실행
SELECT f.*, (SELECT fi.image_url FROM feed_image fi WHERE fi.feed_id = f.feed_id LIMIT 1) as thumbnail
FROM feed f WHERE f.club_id = ?

-- 수정 후: LEFT JOIN
SELECT f.*, fi.image_url as thumbnail
FROM feed f
LEFT JOIN feed_image fi ON fi.feed_id = f.feed_id
    AND fi.feed_image_id = (SELECT MIN(fi2.feed_image_id) FROM feed_image fi2 WHERE fi2.feed_id = f.feed_id)
WHERE f.club_id = ? AND f.deleted = false
```

### 복합 인덱스 추가

```sql
-- 개인/인기/클럽 피드 목록 쿼리 커버
CREATE INDEX idx_feed_club_deleted_created ON feed(club_id, deleted, created_at);

-- repost 카운트 쿼리 커버
CREATE INDEX idx_feed_parent_deleted ON feed(parent_feed_id, deleted);
```

### 2라운드 결과

| 항목 | 1라운드 | 2라운드 | 개선 |
|------|---------|---------|------|
| 개인피드 p95 | 2,198ms | 107ms | 20.5배 |
| 인기피드 p95 | 904ms | 65ms | 13.9배 |
| 클럽피드 p95 | 695ms | 53ms | 13.1배 |
| 피드상세 p95 | 2,725ms | 99ms | 27.5배 |

---

## 3라운드 — 데이터 증가 후 병목 특화 고부하

2라운드에서 수치가 너무 좋게 나왔다. 데이터 규모를 현실에 맞게 늘렸다.

| 테이블 | 기존 | 증가 후 |
|--------|------|---------|
| feed | 100K | 500K |
| feed_like | 500K | 2.5M |
| feed_comment | 200K | 1M |
| user_club | 50만 | 253만 (유저당 20+ 클럽) |

식별된 핵심 병목 후보: 인기피드 `LOG()+TIMESTAMPDIFF()` CPU 정렬, 개인피드 IN절 폭발 (유저당 20+ 클럽), 리포스트 카운트 GROUP BY, 피드상세 댓글 페이지네이션 없음.

### 3라운드 결과

| Phase | 항목 | p95 | 슬로우 쿼리 |
|-------|------|-----|------------|
| P1 (400VU) | 인기피드 정렬 | 567ms | 9,531건 |
| P2 (400VU) | 개인피드 IN절 | 919ms | **36,094건** |
| P3 (300VU) | 피드상세 | 129ms | - |
| P3 | 클럽피드 | 80ms | - |

P2 개인피드 IN절이 CRITICAL로 분류됐다. `resolveAccessibleClubIds`가 getPersonalFeed·getPopularFeed에서 각각 호출되어 2회 중복 실행되고, 유저당 20+ 클럽으로 IN clause가 대형화된 것이 원인이었다.

---

## 4~5라운드 — 피드상세 N+1 수정, IN절 1차 대응

### 피드상세 N+1 근본 수정

`findByIdsWithRelations` 쿼리에 Club까지 JOIN FETCH를 추가하지 않아 Club 연관이 LAZY 로딩됐다.

```java
// 수정 후
@Query("""
    select f from Feed f
    join fetch f.user
    join fetch f.club
    left join fetch f.feedImages
    where f.feedId in :ids and f.deleted = false
""")
List<Feed> findByIdsWithRelations(@Param("ids") List<Long> ids);
```

**결과**: P4 피드상세 p95 1,395ms → 41ms (97% 개선)

### IN절 1차 대응 — UNION ALL 청크

유저당 클럽을 5개씩 청크로 나눠 UNION ALL로 실행했다.

```java
private static final int CLUB_CHUNK_SIZE = 5;

// 5개씩 나눠서 쿼리 후 병합
List<Long> feedIds = Lists.partition(clubIds, CLUB_CHUNK_SIZE).stream()
    .flatMap(chunk -> feedRepository.findPersonalFeedIds(chunk, cursor, pageable).stream())
    .distinct()
    .sorted(Comparator.reverseOrder())
    .limit(pageSize)
    .collect(toList());
```

**5라운드 결과**:

| 항목 | 4라운드 | 5라운드 |
|------|---------|---------|
| 5xx 에러 | 19,152건 | 0건 |
| P2 개인피드 p95 | 29,917ms | 1,134ms (-96%) |
| P3 피드상세 p95 | 147ms | 172ms (유지) |

---

## 6라운드 — 커버링 인덱스 + Redis 캐싱

### 커버링 인덱스 추가

IN clause + ORDER BY + LIMIT 조합에서 MySQL 옵티마이저가 filesort를 선택하는 문제를 커버링 인덱스로 해결했다.

```sql
-- popular 피드용 (deleted 선행 — 7일 범위 필터에 유리)
CREATE INDEX idx_feed_popular_cover ON feed(deleted, club_id, like_count DESC, comment_count, parent_feed_id, feed_id);

-- personal 피드용 (club_id 선행 — IN clause 최적화)
CREATE INDEX idx_feed_personal_cover ON feed(club_id, deleted, created_at DESC, comment_count, parent_feed_id, feed_id);
```

### 개인/인기 피드 Redis 캐싱 (TTL 15초)

`resolveAccessibleClubIds` 결과가 매 요청마다 DB를 쳤다. 유저 ID + 커서 기준으로 pass1 결과 전체를 Redis에 캐싱했다.

```java
// cache key: "feed:personal:{userId}:{cursor}"
private List<FeedOverviewResponseDto> getPersonalFeedCached(Long userId, Long cursor, int size) {
    String key = "feed:personal:" + userId + ":" + cursor;
    String cached = redisTemplate.opsForValue().get(key);
    if (cached != null) return deserialize(cached);
    
    List<FeedOverviewResponseDto> result = buildPersonalFeed(userId, cursor, size);
    redisTemplate.opsForValue().set(key, serialize(result), Duration.ofSeconds(15));
    return result;
}
```

인기피드도 동일 패턴으로 적용했다.

### 6라운드 결과

| 항목 | 5라운드 | 6라운드 | 변화 |
|------|---------|---------|------|
| P1 인기피드 p95 | 815ms | 238ms | -65% |
| P2 개인피드 p95 | 986ms | 547ms | -40% |
| P3 피드상세 p95 | 172ms | 243ms | 유지 |
| P1 슬로우 쿼리 | 13,848건 | 69건 | -99.5% |
| 전체 처리량 | 기준 | +50% | |

---

## 7~8라운드 — 피드상세 캐싱, 댓글/좋아요 쿼리 최적화

### 피드상세 인메모리 캐싱

`getFeedDetail`에서 매 요청 4개 쿼리를 실행했다. 피드 상세 데이터는 `isLiked`와 `isFeedMine` 외엔 유저 간 공통이므로, 피드 데이터 자체를 공유 캐싱하고 유저별 필드만 오버레이하는 방식을 사용했다.

```java
// TTL 5초 — 피드 데이터 + 댓글 + 리포스트 수 캐싱
private record FeedDetailCache(Feed feed, List<String> imageUrls,
                                List<CommentCache> comments, long repostCount) {}

// 캐시 히트 시: existsByFeed_FeedIdAndUser_UserId 1쿼리만 실행
// isLiked, isFeedMine, isCommentMine은 캐시 데이터 기반으로 재계산
```

캐시 히트 시 DB 4쿼리 → 1쿼리(유저별 좋아요 확인)로 줄었다.

### 댓글 생성 쿼리 최적화

댓글 생성 시 `clubRepository.findById` + `feedRepository.findByFeedIdAndClub` 2건이 실행되던 것을 단일 쿼리로 합쳤다.

```java
// 수정 전: 2 SELECT
Club club = clubRepository.findById(clubId).orElseThrow();
Feed feed = feedRepository.findByFeedIdAndClub(feedId, club).orElseThrow();

// 수정 후: 1 SELECT (JOIN)
Feed feed = feedRepository.findByFeedIdAndClubId(feedId, clubId).orElseThrow();
```

### findUserIdsByFeedId LIMIT 없음 — 인기 피드 warmup 병목

`FeedLikeRepository.findUserIdsByFeedId`가 LIMIT 없이 피드의 모든 좋아요 유저를 로딩했다. 인기 피드에 1,536개 좋아요가 있다면 1,536 row를 매번 읽는다. 200VU가 동시에 다른 피드를 warmup하면 DB가 마비됐다.

```java
// 수정 후: LIMIT 100
@Query("select fl.user.userId from FeedLike fl where fl.feed.feedId = :feedId")
List<Long> findUserIdsByFeedId(@Param("feedId") Long feedId, Pageable pageable);
```

---

## 9라운드 — 전 구간 통과

모든 수정을 적용한 최종 테스트 결과다.

| Phase | 항목 | 초기 p95 | 최종 p95 | 개선 |
|-------|------|---------|---------|------|
| P1 Read | 개인피드 | 2,198ms | <500ms ✓ | 4배+ |
| P1 Read | 인기피드 | 904ms | <500ms ✓ | |
| P1 Read | 클럽피드 | 695ms | <300ms ✓ | |
| P2 Write | 좋아요/댓글 | - | <500ms ✓ | |
| P3 Mixed | 성공률 | - | >90% ✓ | |
| P4 Detail | 피드상세 | 2,725ms | <500ms ✓ | 5.5배+ |
| P5 Spike | 성공률 | - | >90% ✓ | |

슬로우 쿼리는 29,008건에서 69건 수준으로 줄었다.

---

## 캐싱 전략 — 개인 피드의 근본 한계

개인피드는 본질적으로 `다수 클럽 × 정렬 × 페이지네이션`이라 단일 쿼리로 최적화하기 어렵다. 15초 TTL 캐싱이 가장 효과적인 해법이었다. 15초면 사용자 체감상 거의 실시간이고, 좋아요/댓글 수가 15초 지연되어도 UX 허용 범위 내에 있다.

캐싱 없이 근본 해결하는 방법은 두 가지다. 첫째, 비정규화 테이블(`user_personal_feed`) — 피드 생성 시 해당 클럽 멤버 전원의 타임라인에 INSERT하는 fan-out-on-write 패턴. 트위터가 쓰는 방식이다. 둘째, Materialized View — 주기적으로 갱신되는 뷰 테이블. 둘 다 현재 규모에서는 오버엔지니어링이다. 400VU 동시 부하는 프로덕션에서 드문 극한 상황이기도 하다.

---

## 데드락 — 좋아요 취소 + 피드 삭제 동시 실행

부하 테스트 중 드물게 데드락이 발생했다.

```
Deadlock found when trying to get lock
Transaction 1: DELETE FROM feed_like WHERE feed_id=? AND user_id=? (좋아요 취소, X lock)
Transaction 2: UPDATE feed SET like_count=? WHERE feed_id=? (피드 삭제 시 cascade, X lock 대기)
```

좋아요 취소가 `feed_like` row에 X lock을 잡고, 피드 삭제 cascade가 같은 row의 X lock을 대기하는 구조였다. 피드 삭제 시 좋아요를 먼저 일괄 삭제하는 순서로 수정해 해결했다.

---

## 수정 내역 요약

| # | 변경 내용 | 효과 |
|---|----------|------|
| 1 | 피드상세 댓글 N+1 → JOIN FETCH + LIMIT 20 | 21쿼리 → 1쿼리 |
| 2 | 클럽 피드 썸네일: 상관 서브쿼리 → LEFT JOIN | 클럽피드 p95 -87% |
| 3 | 복합 인덱스 2개 추가 (personal/popular cover) | 슬로우 쿼리 -99.5% |
| 4 | `findByIdsWithRelations`: Club JOIN FETCH 추가 | N+1 제거 |
| 5 | 개인피드 IN절: UNION ALL 청크 (크기=5) | 29,917ms → 1,134ms |
| 6 | 개인/인기피드 pass1 Redis 캐싱 (TTL 15초) | p95 각각 -65%, -40% |
| 7 | getFeedDetail 인메모리 캐시 (TTL 5초) | 4쿼리 → 1쿼리 |
| 8 | 댓글 생성 2 SELECT → 1 SELECT (JOIN) | 쿼리 수 감소 |
| 9 | `findUserIdsByFeedId`: LIMIT 100 추가 | warmup 병목 제거 |
| 10 | 좋아요 취소 + 피드 삭제 lock 순서 통일 | 데드락 해결 |
| 11 | `resolveAccessibleClubIds` 2회 → 1회 통합 | 중복 쿼리 제거 |
| 12 | Redis max-idle 64 → 128 | 커넥션 풀 안정화 |

---

## 정리하며

슬로우 쿼리 29,008건이라는 숫자는 실제 문제를 숨기고 있었다. 피드 하나를 조회할 때마다 N+1이 터지고, 클럽마다 상관 서브쿼리가 실행되고, IN절에 20개 클럽이 들어가면서 옵티마이저가 filesort를 선택했다. 각각은 개별 쿼리 실행 시간으로는 잡히지 않았다. 300+VU 동시 부하가 걸려야 커넥션 풀이 포화되고, 그때야 비로소 어디서 터지는지 보였다.

> **슬로우 쿼리 로그는 단건 쿼리 성능을 보여준다. 동시 부하 테스트는 전체 파이프라인의 자원 경합을 보여준다. 둘 다 필요하다.**

인기피드 캐싱이 슬로우 쿼리를 13,848건에서 69건으로 줄였다. 인덱스로 해결할 수 없는 CPU-bound 정렬 문제는 캐싱이 가장 빠른 답이다. 다만 캐싱이 Redis 키 폭발로 이어지면 P6 딥페이지네이션에서 역으로 Redis 리소스 경합이 생겼다. 캐싱은 문제를 해결하지만 새로운 병목을 만들 수 있다. TTL과 키 설계를 신중하게 해야 한다.

---

## 시리즈 탐색

**◀ 이전 글**
[알림 API 성능 — 13% 성공률에서 100%로, 9번의 시도 기록](/notification-api-performance-improvement/)

**▶ 다음 글**
[멀티 도메인 코드 정리 — 표면 정리부터 Semaphore 제거까지](/multi-domain-cleanup-surface-to-semaphore/)
