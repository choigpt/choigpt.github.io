---
layout: post
title: 피드 도메인 — 부하 테스트 9번 만에 슬로우 쿼리 29,008건에서 전 구간 통과까지
date: 2026-02-26
tags: [MySQL, Redis, JPA, 성능최적화, 트러블슈팅, Java, 피드, k6, 캐싱, 인덱스]
permalink: /feed-performance-load-test/
excerpt: "피드 도메인 첫 부하 테스트에서 슬로우 쿼리가 29,008건 발생했다. 개인피드 p95 2,198ms, 피드상세 p95 2,725ms. 인덱스 전무, 컬렉션 전체 로딩, 인기피드 전체 테이블 집계 — 하나씩 고치며 9번의 테스트 끝에 전 구간 통과했다."
---

## 개요

채팅 도메인 부하 테스트를 마친 뒤 피드 도메인으로 넘어왔다. 피드 도메인은 개인 타임라인·인기피드·클럽피드·피드상세·좋아요·댓글·리포스트가 연관된 구조로, 쿼리 복잡도가 높은 도메인이다.

첫 k6 테스트에서 슬로우 쿼리가 29,008건 나왔다. 채팅 도메인의 1,332건 대비 21배다. 개인피드 p95는 2,198ms, 피드상세는 2,725ms였다. 9번의 테스트 끝에 전 구간 threshold를 통과했다. 각 테스트에서 무엇을 발견하고 어떻게 고쳤는지를 기록한다.

---

## 원본 코드의 구조와 문제점

### 피드 조회 흐름 (수정 전)

```
GET /clubs/{id}/feeds → FeedService.getFeedList()
  → feedRepository.findByClubAndParentFeedIdIsNull(club, pageable)
  → 피드마다 feed.getFeedImages()           ← LAZY 컬렉션 전체 로드
  → 피드마다 feed.getFeedLikes().size()      ← LAZY 컬렉션 전체 로드 후 .size()
  → 피드마다 feed.getFeedComments().size()   ← 동일

GET /feeds/personal → FeedMainService.getPersonalFeed()
  → resolveAccessibleClubIds(userId)        ← "친구의 친구" 팬아웃
  → feedRepository.findByClubIds(clubIds)   ← JOIN FETCH 없음
  → 피드마다 컬렉션 전체 로드 + N+1

GET /feeds/popular → FeedMainService.getPopularFeed()
  → resolveAccessibleClubIds(userId)
  → feedRepository.findPopularByClubIds()   ← feed_like, feed_comment 전체 테이블 집계
```

### 문제 1: 인덱스가 하나도 없다

Feed 엔티티에 `uq_refeed_once_alive` (중복 리피드 방지용) 외에 조회용 인덱스가 전무했다.

```java
// Feed.java — 원본
@Table(name = "feed", uniqueConstraints = {
    @UniqueConstraint(name = "uq_refeed_once_alive", columnNames = {"user_id", "club_id", "active_parent"})
})
```

`club_id`, `parent_feed_id`, `created_at`, `feed_comment.feed_id`, `feed_image.feed_id` — 조회에 필요한 컬럼에 인덱스가 없었다. `findByClubAndParentFeedIdIsNull`은 풀스캔이었다.

### 문제 2: likeCount 비정규화 컬럼이 있는데 안 쓴다

```java
// Feed.java — likeCount 필드가 존재
@Column(name = "like_count")
@Builder.Default
private Long likeCount = 0L;

// FeedService.java, FeedMainService.java — 모든 조회 코드
feed.getFeedLikes().size()      // ← 전체 FeedLike 엔티티 로드 후 .size()
feed.getFeedComments().size()   // ← 전체 FeedComment 엔티티 로드 후 .size()
```

Redis Stream Consumer가 `UPDATE feed SET like_count = ...`로 비정규화 컬럼을 유지하고 있었지만, 조회 코드 어디서도 `feed.getLikeCount()`를 사용하지 않았다. `@BatchSize(100)`이 적용되어 순수 N+1은 아니었지만, 카운트 하나를 위해 FeedLike 엔티티 전체를 메모리에 올리는 구조였다.

### 문제 3: resolveAccessibleClubIds — O(N×M) 팬아웃

[피드 도메인 — Redis 좋아요 파이프라인의 허점과 N+1](/feed-redis-like-n-plus-1/)에서 다룬 "친구의 친구" 범위 폭발이 모든 개인/인기 피드 요청마다 실행됐다.

```java
// FeedMainService.java
List<UserClub> myJoinClubs = userClubRepository.findByUserUserId(userId);
List<Long> myClubIds = myJoinClubs.stream()
    .map(uc -> uc.getClub().getClubId())   // ← N+1: Club LAZY 로딩
    .toList();

List<Long> memberIds = userClubRepository.findUserIdByClubIds(myClubIds);
// ↑ 내 클럽의 전체 멤버 ID (수백 명)

List<UserClub> friendMemberJoinClubs = userClubRepository.findByUserUserIdIn(memberIds);
// ↑ 그 멤버들의 전체 클럽 가입 내역 (수천 건)

List<Long> friendClubIds = friendMemberJoinClubs.stream()
    .map(uc -> uc.getClub().getClubId())   // ← N+1 다시
    .filter(id -> !myClubIds.contains(id)) // ← List.contains = O(n)
    .toList();
```

5개 클럽 × 100명 멤버 = 500명 → 500명의 전체 가입 내역 5000건 → 각각 Club LAZY 로딩. 매 요청마다 이 폭발이 일어났다.

### 문제 4: findPopularByClubIds — 전체 테이블 집계

```sql
-- FeedRepository.java — 인기피드 쿼리 (원본)
SELECT f.*
FROM feed f
LEFT JOIN (SELECT fl.feed_id, COUNT(*) AS cnt FROM feed_like fl GROUP BY fl.feed_id) l
    ON l.feed_id = f.feed_id
LEFT JOIN (SELECT fc.feed_id, COUNT(*) AS cnt FROM feed_comment fc GROUP BY fc.feed_id) c
    ON c.feed_id = f.feed_id
WHERE f.club_id IN (:clubIds) AND f.deleted = false
ORDER BY (LOG(GREATEST(COALESCE(l.cnt, 0) + COALESCE(c.cnt, 0) * 2 + ..., 1))
         - (TIMESTAMPDIFF(HOUR, f.created_at, NOW()) / 12.0)) DESC
```

서브쿼리가 **feed_like 전체**와 **feed_comment 전체**를 GROUP BY한다. 대상 피드만 집계하는 게 아니라 테이블 전체를 스캔한다. 거기에 `LOG() + TIMESTAMPDIFF()` 산술 정렬이 있어 인덱스를 탈 수 없다.

### 문제 5: 피드 상세 — JOIN FETCH 없음 + 좋아요 전체 로드

```java
// FeedService.getFeedDetail()
Feed feed = feedRepository.findByFeedIdAndClub(feedId, club);  // JOIN FETCH 없음
feed.getFeedImages().stream()...       // LAZY 로드
feed.getFeedLikes().stream()           // 전체 FeedLike + User LAZY 로드
    .anyMatch(l -> l.getUser().getUserId().equals(userId));  // isLiked 확인
feed.getFeedComments().stream()...     // 전체 FeedComment + User LAZY 로드
```

좋아요 확인을 위해 전체 FeedLike 컬렉션을 로드하고, 각 FeedLike의 User까지 LAZY 로딩했다. `EXISTS` 쿼리 하나면 될 일이다.

### 문제 6: isLiked 최적화 경로가 죽은 코드

```java
// FeedMainService.java
Set<Long> likedFeedIds = Collections.emptySet();  // ← 항상 빈 Set

// isLiked()
if (likedFeedIds != null && !likedFeedIds.isEmpty()) {
    return likedFeedIds.contains(f.getFeedId());  // 절대 실행 안 됨
}
// 항상 여기로 폴스루 → FeedLike 전체 로드 + User N+1
return likes.stream().anyMatch(l -> l.getUser() != null
    && Objects.equals(l.getUser().getUserId(), userId));
```

### 문제 7: 댓글 목록 — 전체 로드 + User N+1

```java
// FeedCommentRepository.java
findByFeedOrderByCreatedAt(Feed feed, Pageable pageable);
// → JOIN FETCH 없음. 댓글마다 comment.getUser() LAZY 로딩
```

댓글 20개면 1(목록) + 20(User) = 21쿼리.

---

## 테스트 1 — 첫 측정, 슬로우 쿼리 29,008건

데이터 규모: feed 100K, feed_like 500K, feed_comment 200K, user_club 50만(유저당 5개 클럽).

P1 Read(300VU) 구간에서 개인피드 avg 716ms / p95 2,198ms, 인기피드 avg 321ms / p95 904ms, 클럽피드 avg 239ms / p95 695ms이며 성공률 100%였다. P4 Detail(300VU) 구간에서 피드상세 avg 652ms / p95 2,725ms, 성공률 99.9%였다.

주요 병목점:
1. 개인피드 p95 2,198ms — `resolveAccessibleClubIds` 팬아웃 + 인덱스 없음
2. 피드상세 p95 2,725ms — 댓글·이미지·좋아요 LAZY 로딩 + N+1
3. 인기피드 p95 904ms — `feed_like`, `feed_comment` 전체 테이블 집계 + 산술 정렬
4. 슬로우 쿼리 29,008건 — 채팅(1,332건) 대비 21배

---

## 테스트 2 — 인덱스 추가, 댓글 JOIN FETCH + LIMIT

### 복합 인덱스 추가

```sql
CREATE INDEX idx_feed_club_deleted_created ON feed(club_id, deleted, created_at);
CREATE INDEX idx_feed_parent_deleted ON feed(parent_feed_id, deleted);
```

### 댓글 N+1 해결 — JOIN FETCH + 페이지네이션

```java
// 수정 후
@Query("select c from FeedComment c join fetch c.user where c.feed.feedId = :feedId order by c.createdAt asc")
List<FeedComment> findByFeedIdWithUser(Long feedId, Pageable pageable);
```

`PageRequest.of(0, 20)`으로 최근 20개만 로드. 댓글 수천 개인 피드에서 전부 메모리에 올리던 문제도 함께 해결됐다.

### 좋아요 카운트 — 컬렉션 로드에서 비정규화 컬럼으로

모든 조회 코드의 `feed.getFeedLikes().size()`를 `feed.getLikeCount()`로 교체했다. 이미 존재하지만 사용되지 않던 비정규화 컬럼을 활용하도록 변경했다.

### 테스트 2 결과

테스트 2 결과, 개인피드 p95가 2,198ms에서 107ms로 20.5배 개선됐다. 인기피드 p95는 904ms에서 65ms로 13.9배, 클럽피드 p95는 695ms에서 53ms로 13.1배, 피드상세 p95는 2,725ms에서 99ms로 27.5배 개선됐다.

---

## 테스트 3 — 데이터 증가 후 병목 특화 고부하

테스트 2에서 데이터 규모 대비 수치가 낙관적이었다. 데이터 규모를 현실적 수준으로 늘렸다.

데이터 규모를 현실적 수준으로 늘렸다. feed는 100K에서 500K으로, feed_like는 500K에서 2.5M으로, feed_comment는 200K에서 1M으로, user_club은 50만에서 253만(유저당 20+ 클럽)으로 증가시켰다.

### 테스트 3 결과

테스트 3 결과, P1(400VU)에서 인기피드 정렬 p95 567ms / 슬로우 쿼리 9,531건, P2(400VU)에서 개인피드 IN절 p95 919ms / 슬로우 쿼리 **36,094건**으로 CRITICAL이었다. P3(300VU)에서 피드상세 p95 129ms, 클럽피드 p95 80ms는 양호했다.

P2 개인피드가 CRITICAL이었다. `resolveAccessibleClubIds`의 "친구의 친구" 팬아웃을 제거하고 자기 소속 클럽만 반환하도록 변경했지만, 유저당 20+ 클럽의 IN절 자체가 대형화되어 옵티마이저가 filesort를 선택하는 문제가 남았다.

---

## 테스트 4~5 — 피드상세 JOIN FETCH, IN절 청크

### 피드상세 — User, Club까지 JOIN FETCH

원본의 `findByFeedIdAndClub`에 JOIN FETCH가 없어 User, Club, Image가 각각 LAZY 로딩됐다. 목록 조회용 쿼리도 마찬가지였다.

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

**결과**: 피드상세 p95 1,395ms → 41ms (97% 개선)

### IN절 — 클럽 5개씩 청크 분할

유저당 클럽을 5개씩 청크로 나눠 개별 실행 후 Java에서 병합했다.

```java
private static final int CLUB_CHUNK_SIZE = 5;

List<Long> feedIds = Lists.partition(clubIds, CLUB_CHUNK_SIZE).stream()
    .flatMap(chunk -> feedRepository.findPersonalFeedIds(chunk, cursor, pageable).stream())
    .distinct()
    .sorted(Comparator.reverseOrder())
    .limit(pageSize)
    .collect(toList());
```

> 참고: 청크 분할 방식은 첫 페이지에서는 정확하지만, 커서 기반 페이지네이션에서 2페이지 이후의 정합성은 보장되지 않는다. 각 청크 쿼리가 독립 실행되므로 한 청크의 커서가 다른 청크에 유효하지 않을 수 있다. 현재 구현에서는 `distinct().sorted().limit(pageSize)`로 Java에서 병합하되, 이 한계를 인지하고 있다.

**테스트 5 결과**:

테스트 5 결과, 5xx 에러는 19,152건에서 0건으로 완전히 해결됐다. P2 개인피드 p95는 29,917ms에서 1,134ms로 96% 개선됐고, P3 피드상세 p95는 147ms에서 172ms로 유지 수준이었다.

---

## 테스트 6 — 커버링 인덱스 + Redis 캐싱

### 커버링 인덱스

IN clause + ORDER BY + LIMIT 조합에서 filesort를 제거하기 위해, SELECT하는 컬럼까지 인덱스에 포함시켰다.

```sql
-- personal 피드용 (club_id 선행 — IN clause 최적화)
CREATE INDEX idx_feed_personal_cover ON feed(club_id, deleted, created_at DESC, comment_count, parent_feed_id, feed_id);

-- popular 피드용
CREATE INDEX idx_feed_popular_cover ON feed(deleted, club_id, like_count DESC, comment_count, parent_feed_id, feed_id);
```

Pass1 쿼리가 `feed_id`, `like_count`, `comment_count`만 조회하므로 인덱스만으로 응답 가능(Index-Only Scan).

### Redis 캐싱 (TTL 15초)

개인/인기 피드 결과를 유저 ID + 페이지 기준으로 Redis에 캐싱했다.

### 테스트 6 결과

테스트 6 결과, P1 인기피드 p95가 815ms에서 238ms로 65% 감소, P2 개인피드 p95가 986ms에서 547ms로 40% 감소했다. P1 슬로우 쿼리는 13,848건에서 69건으로 99.5% 감소했고, 전체 처리량은 50% 증가했다.

---

## 테스트 7~8 — 피드상세 캐싱, 댓글/좋아요 쿼리 최적화

### 피드상세 인메모리 캐싱

`getFeedDetail`에서 매 요청 4개 쿼리를 실행했다. 피드 데이터 자체를 공유 캐싱하고 유저별 필드(`isLiked`, `isFeedMine`)만 오버레이하는 방식을 사용했다.

```java
// TTL 5초 — 피드 데이터 + 댓글 + 리포스트 수 캐싱
private record FeedDetailCache(Feed feed, List<String> imageUrls,
                                List<CommentCache> comments, long repostCount) {}

// 캐시 히트 시: existsByFeed_FeedIdAndUser_UserId 1쿼리만 실행
```

캐시 히트 시 DB 4쿼리 → 1쿼리(유저별 좋아요 확인)로 줄었다.

### isLiked — 전체 로드에서 EXISTS 쿼리로

원본의 `feed.getFeedLikes().stream().anyMatch(...)` 패턴을 `feedLikeRepository.existsByFeed_FeedIdAndUser_UserId(feedId, userId)` 단일 쿼리로 교체했다. 좋아요 수천 건을 메모리에 올리던 문제가 해결됐다.

### 댓글 생성 쿼리 최적화

```java
// 수정 전: 2 SELECT
Club club = clubRepository.findById(clubId).orElseThrow();
Feed feed = feedRepository.findByFeedIdAndClub(feedId, club).orElseThrow();

// 수정 후: 1 SELECT (JOIN)
Feed feed = feedRepository.findByFeedIdAndClubId(feedId, clubId).orElseThrow();
```

---

## 테스트 9 — 전 구간 통과

모든 수정을 적용한 최종 테스트 결과다.

최종 결과, P1 Read에서 개인피드 p95는 초기 2,198ms에서 500ms 미만으로(4배+ 개선), 인기피드 p95는 904ms에서 500ms 미만으로, 클럽피드 p95는 695ms에서 300ms 미만으로 개선됐다. P2 Write에서 좋아요/댓글은 500ms 미만, P3 Mixed 성공률은 90% 초과, P4 Detail에서 피드상세 p95는 초기 2,725ms에서 500ms 미만으로(5.5배+ 개선), P5 Spike 성공률은 90% 초과를 달성했다.

슬로우 쿼리는 29,008건에서 69건 수준으로 줄었다.

> 부하 테스트는 첫 페이지 조회만 측정했다. 청크 분할의 2페이지 이후 정합성 한계는 테스트 범위에 포함되지 않았다.

> 테스트 9는 threshold 통과 여부만 기록했다. 정확한 p95 수치는 threshold 내에 있었으나 개별 값을 기록하지 않았다.

---

## 캐싱 전략 — 개인 피드의 근본 한계

개인피드는 본질적으로 `다수 클럽 × 정렬 × 페이지네이션`이라 단일 쿼리로 최적화하기 어렵다. 15초 TTL 캐싱이 가장 효과적인 해법이었다. 15초면 사용자 체감상 거의 실시간이고, 좋아요/댓글 수가 15초 지연되어도 UX 허용 범위 내에 있다.

캐싱 없이 근본 해결하는 방법은 두 가지다. 첫째, 비정규화 테이블(`user_personal_feed`) — 피드 생성 시 해당 클럽 멤버 전원의 타임라인에 INSERT하는 fan-out-on-write 패턴. 트위터가 쓰는 방식이다. 둘째, Materialized View — 주기적으로 갱신되는 뷰 테이블. 둘 다 현재 규모에서는 오버엔지니어링이다.

---

## 데드락 — 좋아요 취소 + 피드 삭제 동시 실행

부하 테스트 중 드물게 데드락이 발생했다.

```
Deadlock found when trying to get lock
Transaction 1: DELETE FROM feed_like WHERE feed_id=? AND user_id=? → feed_like row에 X lock
               → 이후 UPDATE feed SET like_count=? WHERE feed_id=? 실행 시 feed row X lock 대기
Transaction 2: DELETE FROM feed WHERE feed_id=? → feed row에 X lock
               → cascade DELETE feed_like 실행 시 feed_like row X lock 대기
→ 순환 대기: T1(feed_like 잠금→feed 대기) ↔ T2(feed 잠금→feed_like 대기)
```

좋아요 취소가 `feed_like` row에 X lock을 잡고 `feed` row의 X lock을 대기하고, 동시에 피드 삭제가 `feed` row에 X lock을 잡고 cascade로 `feed_like` row의 X lock을 대기하는 순환 대기 구조였다. 피드 삭제 시 좋아요를 먼저 일괄 삭제하는 순서로 수정해 해결했다.

> 데드락은 단순한 lock 대기가 아니라 **순환 대기**다. Transaction 1(좋아요 취소)이 `feed_like` 행에 X-lock을 잡고 `feed` 행의 X-lock을 대기하고, 동시에 Transaction 2(피드 삭제)가 `feed` 행에 X-lock을 잡고 cascade로 `feed_like` 행의 X-lock을 대기한다. 양쪽이 서로의 lock을 기다리므로 교착 상태에 빠진다.

---

## 수정 내역 요약

1. 인덱스 추가: `(club_id, deleted, created_at)`, `(parent_feed_id, deleted)` — 풀스캔 제거
2. `getFeedLikes().size()` → `getLikeCount()` 비정규화 컬럼 활용 — 컬렉션 전체 로드 제거
3. `resolveAccessibleClubIds` "친구의 친구" → 자기 소속 클럽만 — 팬아웃 폭발 제거
4. 댓글 N+1 → JOIN FETCH + LIMIT 20 — 21쿼리 → 1쿼리
5. 피드 상세/목록: User, Club JOIN FETCH 추가 — N+1 제거
6. `isLiked`: 컬렉션 전체 로드 → `EXISTS` 쿼리 — 좋아요 수천 건 로드 제거
7. 개인피드 IN절: 5개씩 청크 분할 — 29,917ms → 1,134ms
8. 커버링 인덱스(personal/popular cover) — filesort 제거, Index-Only Scan
9. 개인/인기피드 Redis 캐싱(TTL 15초) — 슬로우 쿼리 -99.5%
10. 피드상세 인메모리 캐시(TTL 5초) — 4쿼리 → 1쿼리
11. 인기피드: 전체 테이블 집계 → 비정규화 `popularity_score` 컬럼 — CPU-bound 정렬 제거
12. 좋아요 취소 + 피드 삭제 lock 순서 통일 — 데드락 해결

---

## 정리하며

슬로우 쿼리 29,008건의 이면에는 복합적인 문제가 있었다. 인덱스가 없어서 풀스캔, `likeCount` 컬럼이 있는데 안 쓰고 컬렉션 전체 로드, 인기피드 쿼리가 `feed_like`·`feed_comment` 전체를 집계, `resolveAccessibleClubIds`가 매 요청 수천 건의 LAZY 로딩을 유발. 각각은 로컬 환경에서 감지되지 않았지만, 300+VU 동시 부하에서 커넥션 풀이 포화되면서 한꺼번에 드러났다.

> **슬로우 쿼리 로그는 단건 쿼리 성능을 보여준다. 동시 부하 테스트는 전체 파이프라인의 자원 경합을 보여준다. 둘 다 필요하다.**

인기피드 캐싱이 슬로우 쿼리를 13,848건에서 69건으로 줄였다. 인덱스로 해결할 수 없는 CPU-bound 정렬 문제는 캐싱이 가장 빠른 답이다. 다만 캐싱이 Redis 키 폭발로 이어지면 역으로 Redis 리소스 경합이 생겼다. 캐싱은 문제를 해결하지만 새로운 병목을 만들 수 있다. TTL과 키 설계를 신중하게 해야 한다.

---

## 시리즈 탐색

**◀ 이전 글**
[알림 API 성능 — 13% 성공률에서 100%로, 9번의 시도 기록](/notification-api-performance-improvement/)

**▶ 다음 글**
[피드 도메인 리팩토링 — 서비스 책임 분리와 네이티브 쿼리 전환](/feed-domain-442-line-service-split/)
