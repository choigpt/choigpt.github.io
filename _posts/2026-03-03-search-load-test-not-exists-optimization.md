---
title: 검색 도메인 부하 테스트 — NOT IN 한 줄이 만든 62배 차이
date: 2026-03-03
tags: [Elasticsearch, MySQL, QueryDSL, 성능최적화, k6, 부하테스트, N+1, 인덱스, nori, Java]
permalink: /search-load-test-not-exists-optimization/
excerpt: "ES + MySQL 이중 구조 검색 도메인에 부하를 걸었다. 300VU에서 keyword_accuracy 79% FAIL, 내 모임 p95 760ms. 스트레스 테스트(500VU, user_club 280만건)에서 teammates_clubs p95가 11초까지 치솟았다. fetchJoin+groupBy 조합 실패, DISTINCT→groupBy 전환을 거쳐 NOT IN → NOT EXISTS 한 줄 변경으로 62배 개선. 18/18 전 구간 통과까지의 기록."
---
## 개요

검색 도메인은 Elasticsearch와 MySQL을 병행하는 이중 구조다.

| 역할 | 저장소 | 비고 |
|------|--------|------|
| 키워드 검색 | Elasticsearch | nori 한국어 형태소 분석기 + 동의어 + 불용어 |
| 필터 검색 | MySQL / QueryDSL | 관심사, 지역 조건 |
| 검색 대상 | Club | name, description 필드 |

엔드포인트는 6개다. 키워드 검색, 관심사/지역 필터, 추천, 내 모임(`my_clubs`), 팀원 클럽(`teammates_clubs`). ES 쪽은 빨랐다. 문제는 MySQL 쪽, 특히 `teammates_clubs`의 다단계 JOIN 쿼리에서 터졌다.

---

## 1차 테스트 — 300VU, 성능 PASS / 정확도 FAIL

Club 20만 건을 ES에 인덱싱한 뒤 첫 부하를 걸었다.

### 성능 지표

| 메트릭 | p50 | p95 | 임계값 | 결과 |
|--------|-----|-----|--------|------|
| 키워드 검색 (ES) | 27.7ms | 187ms | p95<500 | PASS |
| 필터 검색 (MySQL) | 13.98ms | 41ms | p95<500 | PASS |
| 복합 검색 | 12.39ms | 38ms | p95<800 | PASS |
| 관심사 검색 | 22.84ms | 227ms | p95<500 | PASS |
| 지역 검색 | 17.14ms | 180ms | p95<500 | PASS |
| 추천 | 41.05ms | 251ms | p95<800 | PASS |

성능은 전 항목 PASS. 하지만 두 가지가 걸렸다.

**병목: 내 모임 조회 (p95 760ms)**

| 메트릭 | p50 | p95 |
|--------|-----|-----|
| my_clubs | 315ms | 760ms |
| teammates_clubs | 42ms | 331ms |

**정확도: keyword_accuracy FAIL (79.13%)**

| 메트릭 | 결과 | 임계값 | 판정 |
|--------|------|--------|------|
| synonym_accuracy | 83.47% | >80% | PASS |
| keyword_accuracy | 79.13% | >90% | FAIL |
| filter_accuracy | 100% | >90% | PASS |

`'축구'` 검색 결과에 `'풋볼'`만 포함된 케이스가 21%였다. ES는 동의어 확장으로 정상 검색했지만, k6의 `containsAny` 매칭 로직이 원본 키워드만 검사해서 발생한 오탐이었다. 검색 자체는 작동하지만 **테스트 검증 로직이 엄격한 문제**였다.

---

## 내 모임 병목 분석

```
GET /api/v1/search/user
  └─ SearchService.getMyClubs()
       ├─ ① userService.getCurrentUser()              → SELECT user
       ├─ ② findMyClubsWithMemberCount(userId)        → SELECT uc JOIN club JOIN interest
       ├─ ③ stream.map → ClubResponseDto.from()       → club.getInterest().getCategory()
       └─ ④ existsByUserAndSettlementStatusNot()       → SELECT 1 FROM user_settlement
```

| 원인 | 위치 | 영향도 |
|------|------|--------|
| `user_settlement` 인덱스 없음 | UserSettlement 엔티티 | 높음 |
| `user_club` 인덱스 방향 불일치 | `(club_id, user_id)` → user_id 선행 불가 | 중간 |
| 캐싱 미적용 | `getMyClubs()`만 캐싱 없음 | 중간 |

`user_settlement` 테이블에 인덱스가 전혀 없었다. `existsByUserAndSettlementStatusNot`이 `(user_id, status)` 조합으로 검색하는데 풀스캔이 발생했다.

Interest N+1은 아니었다. `findMyClubsWithInterest`에서 `join fetch c.interest`로 이미 EAGER 로딩하고 있어서 `club.getInterest().getCategory()` 호출 시 추가 쿼리는 없었다.

---

## 1차 수정 — 3건 일괄 적용

### my_clubs N+1 해결

`findMyClubsWithMemberCount`가 `select new` 방식을 써서 Interest를 별도 쿼리로 가져오고 있었다. JPQL `select new`에서는 `fetch join`이 불가능하다.

```java
// 수정 전 — select new, Interest N+1 발생
@Query("SELECT new ClubWithMemberCount(c, ...) FROM UserClub uc JOIN uc.club c ...")
List<ClubWithMemberCount> findMyClubsWithMemberCount(Long userId);

// 수정 후 — entity 반환 + join fetch
@Query("SELECT uc FROM UserClub uc JOIN FETCH uc.club c JOIN FETCH c.interest WHERE uc.user.userId = :userId")
List<UserClub> findMyClubsWithInterest(Long userId);
```

### teammates_clubs 쿼리 최적화 — 5쿼리 → 3쿼리

`findClubsByTeammates`에서 Step 3의 2개 쿼리(내 클럽 ID + 팀원 클럽 ID)와 in-memory 필터링을 NOT IN 서브쿼리 1개로 통합했다. 최종 클럽 조회에서 `join(club.interest).fetchJoin()`을 추가해 Interest N+1도 제거했다. `executeClubQuery()`에도 동일하게 적용해 추천/관심사/지역 검색도 개선했다.

### k6 정확도 로직 수정

동의어 맵을 구성해 원본 키워드의 동의어가 결과에 포함돼도 정확도 체크를 통과하도록 수정했다.

```js
const synonymMap = { '축구': ['풋볼', 'soccer', '축구'], ... };

function keywordOrSynonymFound(results, keyword) {
  const targets = synonymMap[keyword] ?? [keyword];
  return results.some(r => targets.some(t => r.name.includes(t)));
}
```

### 인덱스 추가

```java
// UserSettlement.java — settlement 존재 확인 풀스캔 제거
@Table(indexes = @Index(name = "idx_user_settlement_user_status", columnList = "user_id, status"))

// UserClub.java — user_id 단독 조회 커버링 인덱스
@Index(name = "idx_user_club_user", columnList = "user_id")
```

> `uk_user_club (user_id, club_id)` Unique Constraint가 이미 있어서 user_id 기반 조회 자체는 가능했다. 핵심 병목은 `user_settlement` 인덱스 부재였다.

### 1차 수정 결과

| 메트릭 | 수정 전 | 수정 후 | 변화 |
|--------|---------|---------|------|
| ES 키워드 p95 | 187ms | 115ms | -38% |
| 추천 p95 | 251ms | 150ms | -40% |
| 내 모임 p95 | 760ms | 494ms | -35% |
| Teammates p95 | 331ms | 195ms | -41% |
| keyword_accuracy | 79.13% (FAIL) | 94.8% (PASS) | 해결 |
| 전체 Threshold | — | **11/11 PASS** | — |

---

## 스트레스 테스트 — 500VU에서 터지다

데이터 규모를 키웠다. user_club을 유저당 20~50개로 확대해 전체 280만 건, user_settlement 5만 건+. 7-Phase, 500VU, 21분 스트레스 테스트를 설계했다.

### 결과 — 대량 FAIL

| 메트릭 | p50 | p95 | 임계값 | 결과 |
|--------|-----|-----|--------|------|
| teammates_clubs | 318ms | **11.08s** | <1,200ms | FAIL |
| recommend | 318ms | 4.06s | <1,200ms | FAIL |
| my_clubs | 638ms | 2.08s | <1,000ms | FAIL |
| keyword_search | 134ms | 1.05s | <800ms | FAIL |
| interest_search | 359ms | 1.48s | <800ms | FAIL |
| location_search | 344ms | 1.21s | <800ms | FAIL |
| spike | 428ms | 3.49s | <2,000ms | FAIL |
| peak | 439ms | 1.95s | <1,500ms | FAIL |
| 에러율 | — | 0.28% | <5% | PASS |

`teammates_clubs`가 p95 11초로 가장 심각했다. 성공률 80%, 5xx 에러 664건이 전부 이 API에 집중됐다.

---

## teammates_clubs 병목 분석

`ClubRepositoryImpl.findClubsByTeammates`의 쿼리 흐름을 분석했다.

```
Step 1: 내 최근 10개 모임               → 440건 중 10건 (OK)
Step 2: 10개 모임의 팀원 ID 조회         → 1,205명 중 LIMIT 100
Step 3: 100명의 모든 모임에서 내 모임 제외  → 13,333개 club_id (LIMIT 없음!)
Step 4: IN(13,333개) club JOIN interest  → 대량 IN절
```

**핵심: Step 3에 LIMIT이 없었다.** 100명 × 유저당 280개 모임 = ~28,000건을 스캔해서 DISTINCT 후 13,333개 club_id를 메모리에 올린 뒤, Step 4에서 `IN(13,333)` 대량 IN절을 날리고 있었다.

### 1차 시도 — Step 3+4 통합, fetchJoin+groupBy

Step 3과 Step 4를 단일 QueryDSL 쿼리로 합쳤다. `user_club JOIN club JOIN interest + NOT IN 서브쿼리 + GROUP BY + ORDER BY + LIMIT`을 한 번에 실행하는 구조.

```java
return queryFactory
    .select(club)
    .from(theirClubs)
    .join(club).on(club.clubId.eq(theirClubs.club.clubId))
    .join(club.interest).fetchJoin()
    .where(
        theirClubs.user.userId.in(teammateIds)
            .and(theirClubs.club.clubId.notIn(
                JPAExpressions.select(excl.club.clubId)
                    .from(excl).where(excl.user.userId.eq(userId))
            ))
    )
    .groupBy(club.clubId)
    .orderBy(club.memberCount.desc(), club.createdAt.desc())
    .offset(pageable.getOffset())
    .limit(pageable.getPageSize())
    .fetch();
```

**결과: p95 11.08s → 16.37s. 오히려 악화.**

`fetchJoin`과 `groupBy`를 함께 쓰면 Hibernate가 카르테시안 곱을 생성할 수 있다. `from(theirClubs).join(club)` 구조에서 `fetchJoin`이 중복 결과를 만들고, `groupBy`가 이를 다시 정리하느라 비효율적이었다.

### 2차 시도 — 2-step 유지, Step 3에 LIMIT + 정렬

fetchJoin+groupBy 통합을 포기하고, 기존 2-step 구조를 유지하되 Step 3에 `ORDER BY member_count DESC LIMIT 200`을 추가했다.

```java
// DISTINCT + ORDER BY member_count 조합에서 에러 발생
// MySQL: "ORDER BY not in SELECT list"
```

`DISTINCT`와 `ORDER BY member_count`를 함께 쓰면 MySQL에서 SELECT list에 없는 컬럼으로 정렬할 수 없다. `DISTINCT → groupBy`로 전환해 해결했다.

```java
.select(theirClubs.club.clubId)
// .distinct() 제거
.groupBy(theirClubs.club.clubId)
.orderBy(club.memberCount.desc())
.limit(200)
```

**결과: 16/18 PASS. teammates_clubs p95 = 2,650ms (FAIL), my_clubs p95 = 1,050ms (FAIL).**

개선은 됐지만 여전히 threshold를 넘었다. 팀원 수 제한도 100 → 30으로 축소했지만 부족했다.

---

## 최종 수정 — Step 3 근본 재설계

EXPLAIN ANALYZE로 Step 3의 실행계획을 확인했다.

```sql
SELECT uc1_0.club_id FROM user_club uc1_0
JOIN club c1_0 ON c1_0.club_id = uc1_0.club_id
WHERE uc1_0.user_id IN (30명)
  AND uc1_0.club_id NOT IN (
    SELECT uc2_0.club_id FROM user_club uc2_0 WHERE uc2_0.user_id = ?
  )
GROUP BY uc1_0.club_id
ORDER BY c1_0.member_count DESC
LIMIT 200;
```

```
Using where; Using index; Using temporary; Using filesort
단일 실행: 37ms / 500 VU 동시: 타임아웃
```

세 가지 문제가 있었다.

1. **club 테이블 JOIN이 불필요했다.** `member_count` 정렬만을 위해 14,445번 nested loop를 돌았다. Step 4에서 이미 동일한 정렬을 수행하므로 Step 3의 정렬은 중복이었다.
2. **NOT IN 서브쿼리가 비효율적이었다.** 매 행마다 서브쿼리를 평가했다.
3. **GROUP BY + ORDER BY 조합이 filesort를 유발했다.** 단독 실행은 37ms지만 500 VU 동시에 temporary table과 filesort가 메모리/CPU를 서로 빼앗았다.

### 수정 내용

```java
// 수정 전 — club JOIN + NOT IN + GROUP BY + ORDER BY
List<Long> filteredClubIds = queryFactory
    .select(theirClubs.club.clubId)
    .from(theirClubs)
    .join(club).on(club.clubId.eq(theirClubs.club.clubId))  // ← 불필요
    .where(
        theirClubs.user.userId.in(teammateIds)
            .and(theirClubs.club.clubId.notIn(              // ← NOT IN
                JPAExpressions.select(excl.club.clubId)
                    .from(excl).where(excl.user.userId.eq(userId))
            ))
    )
    .groupBy(theirClubs.club.clubId)                         // ← filesort
    .orderBy(club.memberCount.desc())
    .limit(200)
    .fetch();

// 수정 후 — club JOIN 제거 + NOT EXISTS + DISTINCT
List<Long> filteredClubIds = queryFactory
    .select(theirClubs.club.clubId).distinct()
    .from(theirClubs)
    // club JOIN 제거 — 정렬은 Step 4에서 수행
    .where(
        theirClubs.user.userId.in(teammateIds)
            .and(JPAExpressions
                .selectOne().from(excl)
                .where(excl.user.userId.eq(userId)
                    .and(excl.club.clubId.eq(theirClubs.club.clubId)))
                .notExists())                                // ← NOT EXISTS
    )
    .limit(200)
    .fetch();
```

### 실행계획 비교

| 항목 | Before | After | 개선 |
|------|--------|-------|------|
| 실행 시간 | 37ms | 0.6ms | **62배** |
| 스캔 행 | 14,445 | 737 | -95% |
| filesort | O | X | 제거 |
| club 테이블 접근 | 14,445 loops | 0 | 제거 |

**DISTINCT vs GROUP BY**: MySQL에서 DISTINCT는 covering index만으로 중복 제거가 가능해 temporary table을 피한다. GROUP BY는 항상 temporary table을 생성한다. 정렬이 필요 없는 중복 제거라면 DISTINCT가 효율적이다.

**NOT EXISTS vs NOT IN**: NOT IN은 서브쿼리를 매 행마다 평가한다. NOT EXISTS는 MySQL이 materialized subquery로 최적화해 antijoin 처리한다.

---

## 최종 결과

| 메트릭 | 스트레스 Before | 최종 After | 개선율 |
|--------|---------------|-----------|--------|
| teammates_clubs p95 | 2,650ms (FAIL) | 255ms | **90.4% 감소** |
| teammates_clubs p50 | 71ms | 20ms | 72% 감소 |
| teammates_clubs avg | 415ms | 57ms | 86% 감소 |
| my_clubs p95 | 1,050ms (FAIL) | 820ms (PASS) | 22% 감소 |
| recommend p95 | 744ms | 222ms | 70% 감소 |
| keyword p95 | 352ms | 291ms | 17% 감소 |

| 항목 | Before | After | 변화 |
|------|--------|-------|------|
| Threshold | 16/18 PASS | **18/18 PASS** | 완전 통과 |
| 총 요청 | 733,772건 | 940,356건 | +28% |
| 처리량 | 555 req/s | 707 req/s | +27% |
| 5xx 에러 | 129건 | **0건** | 완전 해소 |
| http_req p95 | 615ms | 387ms | 37% 감소 |

---

## 정리하며

**단독 실행이 빠르다고 부하 테스트를 건너뛰면 안 된다.** Step 3은 단독으로 37ms였다. 500 VU 동시 접근에서 타임아웃이 난다. `Using temporary; Using filesort`는 단독에서는 괜찮지만, 동시 접근이 늘면 메모리와 CPU를 서로 빼앗는다. 고부하 환경에서만 드러나는 경합이 있다.

**불필요한 정렬을 하고 있었다.** Step 3의 `ORDER BY member_count DESC`는 Step 4에서 이미 수행하는 정렬을 중복으로 하고 있었다. 그 정렬을 위해 club 테이블을 14,445번 JOIN하고 있었다. 정렬 결과가 Step 4에서 덮어씌워지니 처음부터 의미 없는 JOIN이었다.

**중간 결과셋 크기가 핵심이다.** 13,333개 club_id를 메모리에 올린 뒤 `IN(13,333)`을 날리는 구조였다. NOT EXISTS로 DB 레벨에서 antijoin 처리하고 DISTINCT + LIMIT으로 early termination을 걸자 0.6ms로 떨어졌다.

> **중간 결과셋이 크다는 건 DB가 할 일을 애플리케이션이 떠맡고 있다는 신호다.**

---

## 수정 내역 요약

| # | 변경 내용 | 효과 |
|---|----------|------|
| 1 | `findMyClubsWithMemberCount` → `findMyClubsWithInterest` (JOIN FETCH) | Interest N+1 제거 |
| 2 | `user_settlement` 인덱스 `(user_id, status)` 추가 | 풀스캔 제거 |
| 3 | `user_club` 인덱스 `(user_id)` 추가 | 단독 조회 커버링 |
| 4 | teammates_clubs Step 3+4 통합 → 5쿼리→3쿼리 | 쿼리 수 감소 |
| 5 | k6 동의어 맵 구축 | keyword_accuracy 79%→95% |
| 6 | 팀원 수 제한 100→30 | 스캔 범위 70% 감소 |
| 7 | Step 3: club JOIN 제거 + NOT IN→NOT EXISTS + GROUP BY→DISTINCT | **p95 2,650ms→255ms** |

---

## 시리즈 탐색

**◀ 이전 글**
[클럽 도메인 부하 테스트 — Virtual Thread Pinning이 JVM을 죽인 날](/club-load-test-virtual-thread-pinning/)

**▶ 다음 글**
[채팅 도메인 — Storage 추상화, WHERE IN 풀스캔, WebSocket 1,000VU 극한 테스트](/chat-storage-websocket-extreme-test/)
