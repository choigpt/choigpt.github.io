---
title: 피드 도메인 — Storage 추상화, Lua Script 분석, 3-Backend 비교 테스트
date: 2026-03-03
tags: [Java, Spring, MongoDB, MySQL, Redis, Lua, 부하테스트, k6, 성능최적화, Port/Adapter]
permalink: /feed-storage-abstraction-lua-script-backend-comparison/
excerpt: "피드 도메인의 쿼리 최적화와 고부하 테스트를 마친 뒤, 다음 단계로 Storage를 교체 가능하게 만들었다. FeedStoragePort 인터페이스(11개 메서드 + 3개 DTO)를 설계하고 MySQL/MongoDB 어댑터를 구현했다. Lua Script 좋아요 토글은 conflict 0이라 손댈 필요가 없었지만, existsById MySQL 쿼리가 숨은 병목이었다. Redis 캐싱으로 throughput +82%. 3-Backend 비교에서 진짜 개선은 Storage 교체가 아니라 existsById 캐싱이었다."
---

## 개요

[피드 고부하 테스트 글](/feed-highload-force-index-caching-evolution/)에서 피드 도메인의 고부하 테스트를 통과시켰다. 여러 차례의 쿼리 최적화, FORCE INDEX 실패, 인메모리 캐싱 진화까지 거친 뒤 남은 과제는 두 가지였다.

1. Storage를 교체 가능하게 만들어서 MySQL 외 대안을 실측할 것
2. 좋아요 토글의 Lua Script 구조가 고부하에서도 버티는지 검증할 것

채팅 도메인에서 이미 [ChatMessageStoragePort 추상화](/chat-storage-websocket-extreme-test/)를 진행한 경험이 있었다. 같은 Port/Adapter 패턴을 피드 도메인에 적용하되, 피드 특유의 복잡성(이미지, 클럽, 좋아요 카운트, 리포스트)을 반영한 설계가 필요했다.

---

## Feed Storage Port/Adapter

### FeedStoragePort 인터페이스

`FeedStoragePort`를 11개 메서드 + 3개 record DTO로 설계했다. 채팅 도메인의 6개 메서드보다 규모가 크다. 피드는 조회 경로가 개인피드/인기피드/클럽피드/상세로 나뉘고, 쓰기도 생성/수정/삭제/카운트 증감이 별도이기 때문이다.

```java
public interface FeedStoragePort {
    // 조회
    FeedProjection findFeedById(Long feedId);
    List<FeedProjection> findClubFeeds(Long clubId, Pageable pageable);
    List<FeedProjection> findPersonalFeeds(List<Long> clubIds, Pageable pageable);
    List<FeedProjection> findPopularFeeds(Pageable pageable);
    List<FeedProjection> findFeedsByIds(List<Long> feedIds);

    // 쓰기
    Long saveFeed(FeedSaveDto dto);
    void updateFeed(Long feedId, FeedUpdateDto dto);
    void deleteFeed(Long feedId);

    // 카운트
    void incrementLikeCount(Long feedId);
    void decrementLikeCount(Long feedId);
    void incrementCommentCount(Long feedId);
}
```

DTO는 3개 record로 분리했다.

`FeedProjection`은 조회 결과(feedId, content, user, club, images, counts)를 담고, `FeedSaveDto`는 생성 입력(content, clubId, userId, imageUrls), `FeedUpdateDto`는 수정 입력(content, imageUrls)을 담는다.

### 어댑터 구현

`port/FeedStoragePort.java`는 저장소 추상화 인터페이스이고, `port/MysqlFeedStorageAdapter.java`는 MySQL 어댑터(`matchIfMissing=true` 기본값), `port/MongoFeedStorageAdapter.java`는 MongoDB 어댑터다. `entity/FeedDocument.java`는 MongoDB document이고, `config/MongoFeedConfig.java`는 MongoDB 설정이다.

`MysqlFeedStorageAdapter`는 기존 JPA Repository를 그대로 위임한다. QueryDSL 기반 피드 조회 로직이 이미 최적화되어 있으므로 어댑터 내부에서 추가 작업은 없다.

### FeedDocument: MongoDB 비정규화 설계

```java
@Document(collection = "feeds")
public class FeedDocument {
    @Id
    private String id;
    private Long feedId;
    private String content;

    // 비정규화 — user, club, images 임베딩
    private EmbeddedUser user;      // userId, nickname, profileImageUrl
    private EmbeddedClub club;      // clubId, clubName
    private List<String> imageUrls;

    private int likeCount;
    private int commentCount;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

MySQL에서는 user, club, feed_image를 JOIN으로 가져오지만, MongoDB에서는 단일 document에 임베딩했다. 읽기 경로에서 JOIN이 사라지는 대신 user 프로필 변경 시 관련 document를 모두 업데이트해야 하는 트레이드오프가 있다.

### 런타임 전환

`application.yml`의 `app.feed.storage` 값으로 전환한다.

```yaml
app:
  feed:
    storage: mysql    # mysql | mongodb
```

`@ConditionalOnProperty`로 어댑터 빈이 하나만 등록된다. 변경 범위는 다음과 같다.

`FeedCacheService`는 `FeedRepository` 직접 참조를 제거하고 `FeedStoragePort`를 주입받도록 변경했다. `FeedQueryService`는 조회 로직 전체를 Port 경유로 변경했고, `FeedRenderService`는 DTO 변환 로직에서 Port의 `FeedProjection`을 사용하도록 변경했다. `build.gradle`에는 `spring-boot-starter-data-mongodb` 의존성을 추가했다.

---

## Like Toggle Architecture 분석

Storage 추상화를 진행하면서 좋아요 토글 구조도 재검토했다. 이 부분은 Lua Script 기반이라 Storage 교체 대상이 아니지만, 고부하에서의 동작을 정확히 이해할 필요가 있었다.

### Lua Script 구조

좋아요 토글은 단일 `EVAL`로 4개 연산을 atomic하게 수행한다.

```lua
-- like_toggle.lua
local key = KEYS[1]           -- likers:{feedId}
local countKey = KEYS[2]      -- like_count:{feedId}
local streamKey = KEYS[3]     -- like:events
local userId = ARGV[1]
local feedId = ARGV[2]

local isMember = redis.call('SISMEMBER', key, userId)

if isMember == 1 then
    redis.call('SREM', key, userId)
    redis.call('INCRBY', countKey, -1)
    redis.call('XADD', streamKey, '*', 'feedId', feedId, 'userId', userId, 'action', 'UNLIKE')
    return 0
else
    redis.call('SADD', key, userId)
    redis.call('INCRBY', countKey, 1)
    redis.call('XADD', streamKey, '*', 'feedId', feedId, 'userId', userId, 'action', 'LIKE')
    return 1
end
```

Lua Script는 4개 명령을 순서대로 실행한다. 먼저 `SISMEMBER`로 likers SET에서 유저 존재 여부를 확인하고, `SADD` 또는 `SREM`으로 likers SET에 추가하거나 제거한다. 그 다음 `INCRBY`로 like_count를 증감하고, 마지막으로 `XADD`로 like:events Stream에 이벤트를 발행한다.

4개 연산이 하나의 `EVAL`로 실행되므로 Redis의 single-threaded 특성상 다른 명령이 끼어들 수 없다. 락 없이 원자성을 보장한다.

### FeedLikeStreamConsumer

DB 반영은 비동기로 처리한다. `FeedLikeStreamConsumer`가 `XREADGROUP`으로 like:events Stream을 소비한다.

```
[Client] → EVAL (Lua Script) → Redis (SET + COUNT + Stream)
                                         ↓
[FeedLikeStreamConsumer] ← XREADGROUP ← like:events Stream
         ↓
    batch UPDATE feed.like_count
    batch INSERT/DELETE feed_like
         ↓
       XACK
```

Consumer는 이벤트를 배치로 묶어서 DB에 반영한다. `like_count`는 `UPDATE feed SET like_count = like_count + ? WHERE feed_id = ?`로 처리하고, `feed_like` 테이블은 action에 따라 `INSERT` 또는 `DELETE`한다. 처리 완료 후 `XACK`로 메시지를 확인한다.

### 검증 결과

1,000 VUs로 좋아요 토글을 집중 테스트했다. 결과는 conflict 0이었다. Lua Script가 SET 기반 멱등성을 보장하므로, 같은 유저가 동시에 토글해도 SISMEMBER 결과가 일관된다. 이 구조는 수정할 필요가 없었다.

---

## existsById 병목 발견

### 문제

Lua Script 자체는 완벽했지만, 그 앞단에 숨은 병목이 있었다. 좋아요 토글 API의 전체 흐름은 다음과 같다.

```
toggleLike(feedId, userId)
    ├── feedRepository.existsById(feedId)   ← MySQL 쿼리
    ├── luaScript.execute(...)               ← Redis EVAL
    └── return result
```

`existsById`는 피드가 실제 존재하는지 확인하는 validation이다. 문제는 이 쿼리가 **매 요청마다** MySQL에 `SELECT 1 FROM feed WHERE feed_id = ?`를 실행한다는 점이다.

1,000 VUs에서 초당 1,000회 이상의 `existsById` 쿼리가 HikariCP connection pool을 점유한다. Lua Script는 Redis에서 수 마이크로초 만에 끝나지만, 그 전에 MySQL connection을 잡고 기다리는 시간이 전체 latency를 지배했다.

### 수정

피드 존재 여부는 자주 변하지 않는 데이터다. Redis에 캐싱했다.

```java
public boolean existsFeed(Long feedId) {
    String key = "feed:exists:" + feedId;
    Boolean cached = redisTemplate.opsForValue().get(key);
    if (cached != null) {
        return cached;
    }
    boolean exists = feedRepository.existsById(feedId);
    redisTemplate.opsForValue().set(key, exists, Duration.ofMinutes(10));
    return exists;
}
```

피드 삭제 시 해당 캐시를 evict한다. TTL 10분으로 설정하여 삭제 후 최대 10분간 stale 데이터가 남을 수 있지만, 삭제된 피드에 좋아요를 시도해도 Lua Script에서 무해하게 처리되므로 실질적 문제는 없다.

### 결과

existsById Redis 캐싱 적용 후 p95는 951ms에서 514ms로 46% 감소, Throughput은 972 req/s에서 1,770 req/s로 82% 증가, Success Rate는 99.94%에서 99.96%로 유지됐다.

HikariCP connection 대기 시간이 사라지면서 throughput이 82% 증가했다. Lua Script 최적화가 아니라, **Lua Script 앞의 MySQL 쿼리**가 병목이었다.

---

## 3-Backend 비교 (Like Toggle, 200-1000 VUs)

existsById 캐싱과 MongoDB 어댑터가 준비된 상태에서, 3가지 구성의 좋아요 토글 성능을 비교했다.

MySQL existsById(original)는 1,000VU에서 p95 951ms, 972 req/s, 성공률 99.94%. MySQL + Redis cache는 p95 514ms, 1,770 req/s, 99.96%. MongoDB + Redis cache는 p95 467ms, 1,769 req/s, 99.97%.

### 분석

MySQL + Redis cache와 MongoDB + Redis cache의 throughput 차이는 1 req/s로 무의미하다. p95에서 MongoDB가 47ms 빠르지만, 이것은 existsById 캐싱의 효과(951ms -> 514ms)에 비하면 미미하다.

**결과: 유의미한 개선은 Storage backend 교체가 아니라 existsById의 Redis 캐싱이었다.**

다만 mixed read+write 시나리오에서는 MongoDB의 비정규화 구조가 유의미한 차이를 만들었다.

Mixed read+write 시나리오에서 MySQL + Redis cache는 p95 401ms, MongoDB + Redis cache는 p95 318ms로 21% 차이가 났다.

읽기 경로에서 JOIN 없이 단일 document를 가져오는 MongoDB의 구조적 이점이 반영된 결과다. 쓰기 비중이 높은 시나리오에서는 두 backend의 차이가 줄어든다.

---

## Feed 종합 테스트 (MongoDB)

MongoDB 어댑터로 피드 도메인 전체 시나리오를 테스트했다.

MongoDB 어댑터 기준으로, 클럽 피드 목록은 p95 99ms, 성공률 100%(PASS). 피드 상세는 p95 34ms, 100%(PASS). 개인 피드는 성공률 0%(FAIL). 인기 피드도 성공률 0%(FAIL).

클럽 피드와 피드 상세는 단일 클럽 또는 단일 document 조회라서 MongoDB에서도 잘 동작한다. 문제는 개인 피드와 인기 피드다.

개인 피드는 유저가 속한 복수 클럽의 피드를 합쳐서 정렬하는 쿼리이고, 인기 피드는 `LOG(like_count + 1) + TIMESTAMPDIFF()` 알고리즘으로 정렬한다. MySQL에서는 QueryDSL로 최적화한 쿼리가 잘 동작하지만, MongoDB aggregation pipeline으로 동일한 로직을 옮기면 호환되지 않는 부분이 있었다. `$lookup` + `$sort` + `$skip/$limit` 조합이 MySQL의 커버링 인덱스를 활용한 쿼리 성능을 따라가지 못했다.

**결론: MySQL이 피드 도메인의 primary storage로 유지된다.** MongoDB는 클럽 피드, 피드 상세 같은 특정 패턴에서 이점이 있지만, 개인/인기 피드의 복잡한 쿼리를 커버하지 못한다. Port/Adapter 추상화는 유지하되, 현 시점에서 MongoDB로의 전면 전환은 하지 않는다.

---

## 동시성 검증

### 종합 부하 테스트

500 VUs로 피드 도메인 전체 시나리오를 동시 실행했다.

종합 부하 테스트 결과 시나리오 11/11 ALL PASS, Throughput 1,063 req/s, 5xx 에러 0건.

### 좋아요 원자성 검증

200 VUs로 좋아요 토글을 집중 테스트했다. 동일 피드에 대해 다수 유저가 동시에 토글하는 시나리오다.

총 토글 횟수 129,719회, 일관성 100%, Conflict 0건.

129,719번의 토글에서 단 한 건의 conflict도 발생하지 않았다. Lua Script의 SET 기반 멱등성과 EVAL의 원자성이 실측으로 확인되었다.

### 3-Backend 동시 스트레스

MySQL, MongoDB, Redis를 모두 사용하는 구성에서 500 VUs 스트레스 테스트를 수행했다.

3-Backend 동시 스트레스 결과 Success Rate 99.96%, p95 718ms.

3개 backend가 동시에 동작하는 상황에서도 안정적이다. p95 718ms는 단일 backend 테스트보다 높지만, 3개 저장소의 connection pool과 네트워크 비용을 고려하면 합리적인 수치다.

---

## 정리

Storage 추상화는 FeedStoragePort 11개 메서드 + 3개 DTO에 MySQL/MongoDB 어댑터를 구현했다. Lua Script는 수정 불필요로 1,000 VUs에서 conflict 0이었다. 진짜 병목은 existsById MySQL 쿼리로, Redis 캐싱으로 throughput +82% 개선했다. MySQL과 MongoDB의 throughput은 동일하며, MongoDB는 mixed read+write p95에서 21% 빠르다. MongoDB의 한계로 개인/인기 피드 aggregation pipeline 호환이 실패했다. 최종 선택은 MySQL primary 유지, Port/Adapter 추상화 보존이다.

Storage backend를 바꾸면 성능이 나아질 것이라는 가설은 절반만 맞았다. 좋아요 토글에서 진짜 병목은 backend가 아니라 validation 쿼리였고, MongoDB의 이점은 비정규화 읽기에서만 유의미했다. 추상화 레이어를 만들어둔 것은 향후 특정 도메인만 MongoDB로 분리할 수 있는 선택지를 열어두는 의미가 있다.

---

## 시리즈 탐색

**◀ 이전 글**
[알림 도메인 — Port/Adapter Storage 추상화, MySQL의 벽, MongoDB 1,061배 차이](/notification-storage-abstraction-mysql-mongodb/)

**▶ 다음 글**
[검색 엔진 추상화 — ES vs MySQL FULLTEXT, 236배 차이의 기록](/search-engine-abstraction-es-vs-fulltext/)
