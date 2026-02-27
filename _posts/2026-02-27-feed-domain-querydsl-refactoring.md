---
title: 피드 도메인 리팩토링 — 442줄 서비스를 4개로 쪼갠 기록
date: 2026-02-27
tags:
  - Spring
  - Java
  - 리팩토링
  - QueryDSL
  - JPA
  - 피드
  - 설계
  - 캐시
permalink: /feed-domain-442-line-service-split/
excerpt: FeedQueryService 442줄에 SQL, 캐시, 렌더링 로직이 섞여 있었다. 네이티브 쿼리 6개를
  JPQL/QueryDSL로 전환하고, 서비스를
  FeedCacheService·FeedRenderService·FeedLikeWarmupService로 분리한 기록.
---

## 개요

피드 도메인은 다른 팀원이 설계한 코드다. 성능 개선 작업을 병행하면서 구조를 파악해나간 부분이라, 원 설계 의도를 완전히 알지 못하는 상태에서 진행한 리팩토링이다.

부하 테스트를 통해 성능을 끌어올린 뒤, 코드를 다시 읽었다. `FeedQueryService`가 442줄이었고, 한 클래스 안에 클럽 피드 조회·전체 피드 조회·Redis 캐시·인메모리 캐시·렌더링·UNION ALL 동적 SQL이 뒤섞여 있었다.

문제는 코드 길이가 아니라 레이어 침범이었다. 서비스가 SQL을 직접 조합하고 있었고, Redis 직렬화 로직이 서비스에 인라인으로 있었다. 네이티브 쿼리는 컴파일 타임 검증이 없어서 리팩토링 중 실수를 잡기 어려웠다.

---

## 구조 분석 — 한 클래스에 4가지 책임

```
FeedQueryService (442줄)
├── 모임 피드 조회 (getFeedList, getFeedDetail)
├── 전체 피드 조회 (getPersonalFeed, getPopularFeed)
├── 캐시 레이어 (Redis pass1, ConcurrentHashMap result/detail cache)
│   └── ConcurrentHashMap static 2개, record 3개 인라인 정의
└── 렌더링 로직 (buildOverviewList, toOverviewDto, resolveImages)
    └── UNION ALL 동적 SQL (findPersonalFeedChunked)  ← 레이어 위반
```

서비스에서 `EntityManager`를 직접 받아 네이티브 SQL 문자열을 StringBuilder로 조합하고 있었다. SQL을 서비스에서 조합하는 건 Repository 레이어의 책임이다.

---

## Repository 레이어 정리 — 네이티브 → QueryDSL/JPQL

### 전환 판단 기준

| 쿼리 | 전환 방향 | 이유 |
|------|-----------|------|
| `findFeedSummaryProjectionsByClubId` | QueryDSL | 서브쿼리 + Projection |
| `findFeedIdsWithCountsByClubIds` | QueryDSL | 커버링 인덱스 유지 가능 |
| `findPopularFeedIdsWithCountsByClubIds` | 네이티브 유지 | `LOG()`, `TIMESTAMPDIFF()` — MySQL 전용 함수 |
| `countDirectRepostsIn` | JPQL | 단순 GROUP BY |
| `incrementCommentCount` | JPQL (CASE WHEN 대체) | `GREATEST()` → `CASE WHEN` 표준 SQL |
| `decrementCommentCount` | 네이티브 유지 | `GREATEST()` 언더플로 방지가 MySQL 전용 |
| `findPersonalFeedChunked` (서비스 내) | QueryDSL + 이동 | 서비스 → Repository |

`LOG()`를 `QueryDSL MathExpressions`로, `GREATEST(x, 0)`을 `CASE WHEN x > 0 THEN x ELSE 0 END`로, `DATE_SUB(NOW(), INTERVAL 7 DAY)`를 Java에서 `LocalDateTime.now().minusDays(7)` 파라미터로 전달하는 방식으로 MySQL 전용 함수를 제거했다.

```java
// 수정 후 — QueryDSL로 인기피드 스코어 계산 (LN, TIMESTAMPDIFF 잔류는 numberTemplate)
NumberExpression<Double> score = feed.likeCount
        .add(feed.commentCount.multiply(2))
        .castToNum(Double.class)
        .add(numberTemplate(Double.class, "1.0"))
        .as("score");
```

`decrementCommentCount`의 `GREATEST`만 네이티브로 남겼다. MySQL에서 음수 언더플로를 방지하는 가장 안전한 방법이고, 다른 표준 방법이 없다.

### FeedRepositoryCustomImpl 신규 생성

```java
// FeedRepositoryCustomImpl.java (신규)
@Repository
public class FeedRepositoryCustomImpl implements FeedRepositoryCustom {
    private final JPAQueryFactory queryFactory;

    @Override
    public List<FeedIdWithCounts> findPersonalFeedChunked(
            List<Long> clubIds, int limit) {
        // CLUB_CHUNK_SIZE=5개씩 청크 → 각 청크를 별도 QueryDSL 쿼리
        // Java에서 merge sort로 정렬 후 limit 적용
        return partitionList(clubIds, CLUB_CHUNK_SIZE).stream()
                .flatMap(chunk -> queryChunk(chunk, limit))
                .sorted(Comparator.comparing(FeedIdWithCounts::createdAt).reversed())
                .limit(limit)
                .toList();
    }
}
```

`EntityManager` 직접 사용이 서비스에서 사라졌다. `FeedRepository`가 `FeedRepositoryCustom`을 extends하고, Spring Data JPA가 `FeedRepositoryCustomImpl`을 자동으로 찾아 주입한다.

---

## getCurrentUser() → getCurrentUserId() — 불필요한 SELECT 제거

`AuthService.getCurrentUser()`는 SecurityContext에서 principal을 꺼낸 뒤 `userRepository.findById()`를 호출한다. `getCurrentUserId()`는 DB 조회 없이 principal에서 ID만 추출한다.

피드 서비스 전반에 `getCurrentUser().getUserId()`가 8곳 있었다.

| 서비스 | 호출 수 | User 엔티티 실제 필요 | ID만 필요 |
|--------|---------|---------------------|----------|
| FeedQueryService | 3 | 0 | 3 |
| FeedLikeService | 1 | 0 | 1 |
| FeedCommentService | 3 | 1 (createComment) | 2 |
| FeedCommandService | 4 | 3 (create/update/refeed) | 1 |

`User` 엔티티가 필요한 5곳은 유지, 나머지 `userId`만 쓰는 3곳은 `getCurrentUserId()`로 교체했다. 요청당 최대 3회의 불필요한 SELECT가 제거됐다.

---

## FeedCacheService 분리 — static 캐시를 서비스에서 제거

서비스에 `static ConcurrentHashMap` 2개와 record 3개가 인라인으로 정의돼 있었다.

```java
// 수정 전 — 서비스에 static 캐시 하드코딩
@Service
public class FeedQueryService {
    private record CachedResult(List<FeedOverviewDto> data, long expiresAt) { ... }
    private record DetailCacheEntry(Feed feed, ..., long expiresAt) { ... }

    private static final ConcurrentHashMap<String, CachedResult> resultCache = new ConcurrentHashMap<>();
    private static final ConcurrentHashMap<Long, DetailCacheEntry> detailCache = new ConcurrentHashMap<>();

    // Redis pass1 직렬화/역직렬화도 인라인
    private void cachePass1(String key, List<FeedIdWithCounts> data) {
        String value = data.stream()
                .map(f -> f.feedId() + ":" + f.likeCount() + ":" + f.commentCount())
                .collect(Collectors.joining(","));
        redisTemplate.opsForValue().set(key, value, PASS1_TTL_SECONDS, TimeUnit.SECONDS);
    }
}
```

서비스에 `static` 캐시가 있으면 테스트 격리가 어렵고, Spring 컨텍스트 외부에서 캐시 생명주기를 제어할 수 없다. `FeedCacheService`를 별도 Bean으로 분리했다.

```java
// FeedCacheService.java (신규)
@Component
public class FeedCacheService {
    // Redis pass1 캐시
    public Optional<List<FeedIdWithCounts>> getPass1(String key) { ... }
    public void putPass1(String key, List<FeedIdWithCounts> data) { ... }

    // 인메모리 result 캐시 (10초 TTL)
    public Optional<List<FeedOverviewDto>> getResult(String key) { ... }
    public void putResult(String key, List<FeedOverviewDto> data) { ... }

    // 인메모리 detail 캐시 (5초 TTL)
    public Optional<DetailCacheEntry> getDetail(Long feedId) { ... }
    public void putDetail(Long feedId, DetailCacheEntry entry) { ... }

    // eviction 통합
    @Scheduled(fixedDelay = 30_000)
    public void evictExpired() { ... }
}
```

`FeedQueryService`의 `StringRedisTemplate` 직접 의존이 제거됐다. 캐시 관련 record, 직렬화/역직렬화, eviction 로직이 모두 `FeedCacheService` 안으로 응집됐다.

---

## FeedRenderService 분리 — 렌더링 로직 추출

`FeedQueryService`의 절반은 DTO 변환과 bulk 로딩이었다.

```java
// 수정 전 — 서비스에 렌더링 로직 인라인
private List<FeedOverviewDto> buildOverviewList(
        List<FeedIdWithCounts> feedIds, Long userId) {
    List<Feed> feeds = feedRepository.findByIdsWithRelations(feedIds);
    Map<Long, Long> likedFeedIds = feedLikeRepository.findLikedFeedIdsByUser(userId, ...);
    Map<Long, Feed> parents = bulkLoadParents(feeds);
    Map<Long, Feed> roots = bulkLoadRoots(feeds);
    Map<Long, Long> repostCounts = countDirectReposts(feedIds);
    // ...
}

private Map<Long, Feed> bulkLoadParents(List<Feed> feeds) { ... }
private Map<Long, Feed> bulkLoadRoots(List<Feed> feeds) { ... }
```

`FeedRenderService`로 추출했다. `bulkLoadParents`와 `bulkLoadRoots`는 `bulkLoadByIds(Set<Long>)` 하나로 통합했다. 두 메서드가 동일한 로직이었다.

```java
// FeedRenderService.java (신규)
@Service
public class FeedRenderService {
    public List<FeedOverviewDto> buildOverviewList(
            List<FeedIdWithCounts> feedIds, Long userId) { ... }

    public FeedOverviewDto toOverviewDto(
            Feed feed, FeedRenderContext ctx) { ... }

    private Map<Long, Feed> bulkLoadByIds(Set<Long> ids) { ... }  // 2개 → 1개 통합
}
```

`getPersonalFeed`와 `getPopularFeed`가 거의 동일한 구조였는데, `loadFeed(pageable, cachePrefix, chronological)` 메서드 하나로 통합했다.

```java
// 수정 후 — 공통 로직 1개 메서드로
public List<FeedOverviewDto> getPersonalFeed(Pageable pageable) {
    return loadFeed(pageable, "pf", true);
}

public List<FeedOverviewDto> getPopularFeed(Pageable pageable) {
    return loadFeed(pageable, "ppf", false);
}

private List<FeedOverviewDto> loadFeed(
        Pageable pageable, String cachePrefix, boolean chronological) {
    Long userId = userService.getCurrentUserId();
    String resultKey = buildCacheKey(cachePrefix, userId, pageable);

    return feedCacheService.getResult(resultKey)
            .orElseGet(() -> {
                List<FeedIdWithCounts> ids = chronological
                        ? feedRepository.findPersonalFeedChunked(...)
                        : feedRepository.findPopularFeedIdsWithCountsByClubIds(...);
                List<FeedOverviewDto> result = feedRenderService.buildOverviewList(ids, userId);
                feedCacheService.putResult(resultKey, result);
                return result;
            });
}
```

---

## FeedLikeWarmupService 분리 — 서비스에서 스레드풀 제거

`FeedLikeService`가 138줄이었고, `static ExecutorService`와 `static Set<Long>`을 직접 관리하고 있었다. `@PreDestroy`로 스레드풀 종료도 서비스에서 처리했다. 서비스가 인프라 생명주기까지 관리하는 구조였다.

```java
// 수정 전 — 서비스에 스레드풀 하드코딩
@Service
public class FeedLikeService {
    private static final ExecutorService warmupExecutor =
            Executors.newFixedThreadPool(2);
    private static final Set<Long> warmingUp = ConcurrentHashMap.newKeySet();

    @PreDestroy
    public void shutdown() { warmupExecutor.shutdown(); }

    private void triggerAsyncWarmup(long feedId) {
        if (!warmingUp.add(feedId)) return;
        warmupExecutor.submit(() -> {
            try {
                List<Long> userIds = feedLikeRepository.findUserIdsByFeedId(feedId);
                // Redis 워밍업
            } finally { warmingUp.remove(feedId); }
        });
    }
}
```

`FeedLikeWarmupService`를 분리했다. `FeedLikeService`는 비즈니스 로직(toggleLike, isLiked)만 담당하고, 스레드풀·워밍업 로직·DB 조회는 `FeedLikeWarmupService`가 관리한다.

```java
// FeedLikeWarmupService.java (신규)
@Component
public class FeedLikeWarmupService implements DisposableBean {
    private final ExecutorService warmupExecutor = Executors.newFixedThreadPool(2);
    private final Set<Long> warmingUp = ConcurrentHashMap.newKeySet();
    private final FeedLikeRepository feedLikeRepository;
    private final StringRedisTemplate redis;

    public void triggerAsync(long feedId) { ... }

    @Override
    public void destroy() { warmupExecutor.shutdown(); }
}
```

`FeedLikeService`가 138줄 → 67줄로 줄었고, 의존성도 1개 감소했다.

---

## FeedCommentService — 반복 패턴 헬퍼 추출

`createComment`와 `deleteComment`에서 같은 패턴이 4번 반복됐다.

```java
// 반복 패턴 1: 트랜잭션 안에서 실행
transactionTemplate.executeWithoutResult(status -> {
    feedCommentRepository.save(feedComment);
});

// 반복 패턴 2: count 업데이트 실패 허용
try {
    transactionTemplate.executeWithoutResult(status -> {
        feedRepository.incrementCommentCount(feedId);
    });
} catch (Exception e) {
    log.warn("카운트 증가 실패 (댓글은 정상 저장): feedId={}", feedId);
}
```

`runInTx(Runnable)`과 `updateCountSafely(Runnable, String, Long)` 헬퍼를 추출했다.

```java
private void runInTx(Runnable action) {
    transactionTemplate.executeWithoutResult(status -> action.run());
}

private void updateCountSafely(Runnable action, String msg, Long feedId) {
    try { runInTx(action); }
    catch (Exception e) { log.warn("{}: feedId={}", msg, feedId); }
}
```

`findFeedInClub()`, `validateMembership()` 헬퍼도 추출해 feed 조회 + 필터, 멤버십 확인 패턴을 하나로 묶었다. 108줄 → 88줄.

---

## Feed.createRefeed() — 엔티티 팩토리 메서드

`FeedCommandService.createRefeed()`에서 `rootId` 계산과 `Feed.builder()` 조립이 서비스에 인라인이었다.

```java
// 수정 전 — 도메인 규칙이 서비스에
Long rootId = (parent.getRootFeedId() != null)
        ? parent.getRootFeedId()
        : parent.getFeedId();

Feed reFeed = Feed.builder()
        .content(requestDto.content())
        .feedType(FeedType.REFEED)
        .parentFeedId(parentFeedId)
        .rootFeedId(rootId)
        .club(club)
        .user(user)
        .build();
```

`Feed.createRefeed()`를 엔티티 팩토리 메서드로 추출했다. "리피드의 rootId는 원본 피드의 rootId를 따라가고, rootId가 없으면 원본 feedId가 root다"는 규칙은 `Feed` 엔티티가 알고 있어야 한다.

```java
// Feed.java (팩토리 메서드 추가)
public Feed createRefeed(String content, Club targetClub, User user) {
    Long rootId = (this.rootFeedId != null) ? this.rootFeedId : this.feedId;
    return Feed.builder()
            .content(content)
            .feedType(FeedType.REFEED)
            .parentFeedId(this.feedId)
            .rootFeedId(rootId)
            .club(targetClub)
            .user(user)
            .build();
}
```

서비스에서 `FeedType` import가 사라졌다. `createRefeed` 호출부가 32줄 → 16줄.

---

## 최종 구조 변화

| 클래스 | Before | After |
|--------|--------|-------|
| `FeedQueryService` | 442줄, 4가지 책임 | ~120줄, 조회 흐름 제어만 |
| `FeedCacheService` | 없음 | 신규 — Redis + 인메모리 캐시 통합 |
| `FeedRenderService` | 없음 | 신규 — DTO 변환 + bulk 로딩 |
| `FeedLikeService` | 138줄, static 스레드풀 | 67줄, 비즈니스 로직만 |
| `FeedLikeWarmupService` | 없음 | 신규 — 워밍업 스레드풀 관리 |
| `FeedCommentService` | 108줄 | 88줄 |
| `FeedCommandService` | 126줄 | ~100줄 |
| `FeedRepositoryCustomImpl` | 없음 | 신규 — QueryDSL 5개 메서드 |
| `FeedRepository` | 네이티브 6개 | JPQL 5개 + Custom 위임 |

---

## 정리하며

가장 큰 문제는 SQL이 서비스에 있었다는 것이다. `EntityManager`를 직접 받아 `StringBuilder`로 문자열 조합하는 코드가 `FeedQueryService`에 있었다. 레이어 침범이고, 컴파일 타임 검증도 없다. QueryDSL로 옮기면서 문자열 SQL이 타입 안전 코드로 바뀌었고, IDE가 오류를 잡을 수 있게 됐다.

`static ConcurrentHashMap`이 서비스에 있을 때의 문제는 테스트에서 처음 드러났다. 테스트 케이스 간 캐시 상태가 공유돼 순서에 따라 통과/실패가 달라지는 flaky test가 생겼다. Bean으로 분리하고 `@BeforeEach`에서 mock을 주입하면 격리가 보장된다.

`FeedLikeService`의 `static ExecutorService`는 단위 테스트에서 실제 스레드가 실행돼 비동기 워밍업 로직이 테스트 도중 DB를 치는 문제를 만들었다. 인프라 생명주기(스레드풀 시작/종료)는 Spring Bean의 `@PostConstruct`/`DisposableBean`이 관리하는 게 맞다.

> **도메인 규칙은 엔티티 안에 있어야 한다.**
> `rootId = parent.getRootFeedId() != null ? parent.getRootFeedId() : parent.getFeedId()`는 리피드 체인의 루트를 결정하는 도메인 규칙이다. 서비스가 이 계산을 직접 하면 같은 규칙이 여러 곳에 복붙될 수 있다.

---

## 시리즈 탐색

**◀ 이전 글**  
[채팅 도메인 리팩토링 — WebSocket 핸들러부터 커서 페이징 DTO까지](/chat-domain-deep-refactoring/)

**다음 글 ▶**  
[Finance 도메인 부하 테스트 — Mock 없이는 보이지 않던 버그](/finance-domain-load-test-mock-exposed-hidden-bug/)
