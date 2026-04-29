---
layout: post
title: "Buddkit 백엔드 부하 테스트 & 성능 최적화 전체 보고서"
date: 2026-03-11
tags: [spring-boot, k6, mysql, mongodb, redis, kafka, elasticsearch, jvm, aws, 부하테스트, 성능최적화, Buddkit]
category: Backend
permalink: /buddkit-load-test-full-report/
redirect_from:
  - /onlyone-load-test-full-report/
excerpt: "Buddkit 백엔드 6개 도메인(피드, 채팅, 알림, 정산, 검색, 클럽-일정)의 부하 테스트와 성능 최적화 결과를 한 편으로 정리한 전체 보고서. 124M rows 규모의 DB, max 9,000 VU의 k6 부하, ZGC Generational 환경에서 인덱스/쿼리 최적화, 비동기 처리, 스토리지 추상화, 인프라 다운사이징(c5.xlarge → t3.medium)까지의 전 과정을 단일 글로 다룬다."
---

## 개요

| 항목 | 내용 |
|------|------|
| 프로젝트 | Buddkit (Spring Boot 백엔드, 레포: `OnlyOne-Back`) |
| 대상 도메인 | 피드, 채팅, 알림, 정산, 검색 (5개 EC2 부하) + 클럽-일정 (로컬 단독, 별도 글) |
| 테스트 도구 | k6 (Phase 기반 부하 테스트, max 9,000 VU) |
| 테스트 기간 | 2026-03-05 ~ 2026-03-11 |
| DB 규모 | ~124M rows (도메인당 10M+) |
| JVM | ZGC Generational, Heap 2~4GB |
| 인프라 초기 | App: c5.xlarge (4 vCPU, 8GB), Infra: c5.2xlarge (8 vCPU, 16GB) |
| 인프라 최종 | App: t3.medium (2 vCPU, 4GB), Infra: t3.large (2 vCPU, 8GB) |

> **베이스라인 출처에 대한 주의**: 본 글의 "최적화 전 → 후" 수치는 환경이 자주 바뀐 시리즈를 한 편으로 압축한 것이므로, 같은 도메인의 같은 API라도 베이스라인이 측정된 환경이 다음과 같이 다를 수 있다.
>
> - **로컬 (개발자 PC)**: 02-26 ~ 03-01 글들의 단계별 측정
> - **EC2 c5.xlarge**: 03-07/03-08 글의 EC2 부하 테스트
> - **EC2 t3.medium (다운사이징)**: 03-11 글들의 다운사이징 측정
>
> 도메인별 표의 "최적화 전" 수치는 가장 가깝게 매칭되는 단계의 베이스라인을 사용했고, 환경이 바뀌면서 새로 측정된 값이 다수다. 환경별 자세한 흐름은 [부하 테스트 시리즈 글들](/aws-ec2-load-test-review/)을 참조.
>
> 또한 검색 도메인의 "ES vs FULLTEXT 830배" 같은 수치는 같은 코드의 최적화가 아니라 **검색 엔진을 교체한 결과**다. 단일 "최적화 개선율"로 읽지 않도록 주의.

> **클럽-일정 도메인 안내**: 클럽-일정은 로컬 단독으로만 부하 테스트를 돌렸고 EC2 시리즈에서는 빠졌다. 결과는 별도 글 [클럽 도메인 — Virtual Thread Pinning 분석](/club-load-test-virtual-thread-pinning/)에서 확인할 수 있다. 본 글은 EC2 부하 테스트 5개 도메인을 다룬다.

---

## 1. 피드 도메인

### 1.1 테스트 환경

- DB: Feed 10.6M, Club 69K, User 100K
- 테스트: 10 Phase, max 1,300 VUs, 19분 50초
- 주요 API: 개인 피드, 인기 피드, 클럽 피드 목록, 피드 상세, 댓글, 좋아요, 피드/댓글 생성, 리피드

### 1.2 최적화 내용

#### 1.2.1 쿼리 최적화 — ORDER BY + 인덱스

**문제**: 개인 피드 `ORDER BY created_at DESC`가 IN절 복수 값에서 filesort 강제 발생.

**수정**:
- `ORDER BY created_at DESC` → `ORDER BY feed_id DESC` — `feed_id`는 auto_increment이므로 동일 정렬 보장
- covering index `(club_id, feed_id DESC, like_count, comment_count)` 활용 가능해짐
- 결과: 개인 피드 p95 4.12s → 1.84s (55% 개선)

#### 1.2.2 인기 피드 사전계산 스코어

**문제**: 매 쿼리마다 10.6M 행에 대해 실시간 스코어 계산 → filesort 강제 발생.

```sql
-- Before
ORDER BY LN(GREATEST(like_count + comment_count*2, 1))
         - (TIMESTAMPDIFF(SECOND, created_at, NOW()) / 43200.0) DESC
```

**수정**:
- `popularity_score DOUBLE` 컬럼 추가
- `FeedPopularityScheduler` — 5분 주기, 50,000건 배치 UPDATE (7일 이내 피드 대상)
- `ORDER BY popularity_score DESC` + `idx_feed_club_popularity(club_id, popularity_score)` 인덱스
- 결과: 인기 피드 p95 3.95s → 3.15s (20% 개선)

#### 1.2.3 커서 기반 페이지네이션

**문제**: `OFFSET 1000`은 1,000행 스킵 필요, 깊은 페이지일수록 느림.

**수정**:
- `WHERE feed_id < :cursor ORDER BY feed_id DESC LIMIT 20`
- 인덱스 `(club_id, feed_id)`에서 즉시 시작점 탐색, 일정한 성능

#### 1.2.4 캐시 레이어 키 분리

**문제**: pass1 캐시 키와 result 캐시 키가 동일 포맷으로 충돌.

**수정**:
- pass1 키: `pf:p1:{userId}:{size}` (Redis TTL 30초)
- result 키: `pf:{userId}:first:{size}` (in-memory TTL 10초)
- 첫 페이지만 집중 캐싱 — 커서 페이지는 인덱스 스캔으로 충분

#### 1.2.5 댓글 생성 비동기 카운터 — p95 5.75s → 260ms (95%)

**문제**: `incrementCommentCount`가 feed row에 exclusive lock → 동일 피드에 동시 댓글 시 직렬화 대기 5초+.

**최적화 과정**:

| 시도 | 접근 | 결과 |
|------|------|------|
| TX merge | INSERT+UPDATE를 단일 TX로 | 효과 미미 (5.59s) |
| Redis 버퍼링 | HINCRBY + @Scheduled 배치 플러시 | 역효과 — 배치 X-lock이 INSERT FK check와 충돌 |
| @Transactional 통합 | 메서드 전체를 단일 커넥션으로 | 피드 생성 3.18s→102ms (97%), 댓글은 미미 |
| **@Async 이벤트** | @TransactionalEventListener + @Async | **댓글 생성 5.75s→260ms (95%)** |
| 스레드풀 제한 | @Async 스레드 80→10개 | 읽기 성능 회복 (커넥션 소비 억제) |

**최종 구조**:
- `FeedCommentService.createComment()` — INSERT만 동기 처리, `CommentCountChangedEvent` 발행
- `CommentCountEventListener` — `@TransactionalEventListener(AFTER_COMMIT)` + `@Async("commentCountExecutor")` — 별도 TX에서 count 갱신
- 전용 스레드풀 `commentCountExecutor` max 10 — 커넥션 소비 억제

#### 1.2.6 FeedCommentService @Transactional 통합 — 커넥션 4x → 1x

**문제**: 댓글 생성 1건에 커넥션 4회 획득/반환.

```
findFeedInClub()     → connection #1
getCurrentUser()     → connection #2
validateMembership() → connection #3
runInTx(save)        → connection #4
```

**수정**: `@Transactional`로 전체 메서드를 감싸 1회 커넥션 사용.

- 피드 생성: 3.18s → 102ms (97% 개선)

#### 1.2.7 Kafka engagement 파이프라인

- 토픽: `feed.engagement.v1`
- 좋아요/댓글 이벤트 → 5초 윈도우 버퍼 → `popularity_score` 실시간 갱신
- 좋아요: Redis Lua 스크립트(원자적 SET + counter) → FeedLikeStreamConsumer 배치 DB 반영 → p95 205ms

#### 1.2.8 시드 데이터 정합성 수정

Feed가 참조하는 club_id 중 19,149개가 Club 테이블에 부재 → 개인 피드 API 500 에러, Phase 5 성공률 0%. 빠진 19,149개 Club INSERT로 FK 정합성 복구.

### 1.3 테스트 결과

| API | 최적화 전 p95 | 최적화 후 p95 | 개선율 |
|-----|-------------|-------------|--------|
| 피드 상세 | 1,395ms | 398ms | 71% |
| 댓글 목록 | 614ms | 15ms | 97% |
| 좋아요 토글 | 190ms | 205ms | - |
| 클럽피드 목록 | 440ms | 357ms | 19% |
| 개인 피드 | 4,120ms | 1,680ms | 59% |
| 인기 피드 | 3,950ms | 2,910ms | 26% |
| 피드 생성 | 3,180ms | 102ms | 97% |
| 댓글 생성 | 5,750ms | 260ms | 95% |
| 리피드 | - | 6ms | - |

| 지표 | 최적화 전 | 최적화 후 |
|------|-----------|-----------|
| 에러율 | 9.02% | 0.17% |
| Phase 5 (Personal) 성공률 | 0% | 100% |
| Phase 9 (Extreme) 성공률 | 71% | 99% |
| Threshold PASS | 8/19 | 16/19 |
| CPU | 100% (포화) | 3~30% |
| HikariCP pending | 병목 | 0 |

### 1.4 남은 병목

- 개인/인기 피드 p95 1.6~2.9s — IN절 20+개 클럽 range scan 병목. 해결 방안: UNION ALL 또는 Redis Sorted Set 타임라인
- 피드 생성 p95 1.26s — 8개 인덱스 INSERT 비용, 구조적 한계

---

## 2. 채팅 도메인

### 2.1 테스트 환경

- EC2 t3.medium (2 vCPU, 4GB RAM), Infra t3.large
- 테스트: 11 Phase, max 4,500 VU, 19분 05초
- DB: 1,300만 메시지, 5만 채팅방, 10만 유저

### 2.2 최적화 단계

#### Phase 1: 기본 상태 (v3) — 81,911 에러

**에러 1: 409 Conflict 90,168건** — k6 `TOTAL_CLUBS=5000` 기본값 vs EC2 DB 50,000개 → 존재하지 않는 chatRoom ID 참조.

**에러 2: 500 에러 10,728건** — `@Cacheable("chatRooms")` + `GenericJackson2JsonRedisSerializer`의 `DefaultTyping(NON_FINAL)` → Java `record` 타입 역직렬화 실패. `"Unexpected token (END_ARRAY), expected VALUE_STRING: need type id"` 에러.

#### Phase 2: 에러 수정 (v4) — 4 에러, p95 6~13s

- k6 환경변수 정합성 맞춤 (`TOTAL_CLUBS=50000`)
- `@Cacheable("chatRooms")` 제거 → 커스텀 `ChatRoomListCache` (수동 JSON 직렬화로 DefaultTyping 우회)

#### Phase 3: Redis 캐시 3종 + Redis Streams (v5) — p95 4~6s

**HikariCP 84,000 에러의 근본 원인**:

메시지 전송 1건의 커넥션 사용 패턴:
```
POST /api/v1/chat/{roomId}/messages
  ① getCurrentUser()          → SELECT * FROM user     [커넥션 #1]
  ② findUserInfoIfMember()    → SELECT nickname, ...    [커넥션 #2]
  ③ chatMessageStoragePort.save() → INSERT             [커넥션 #3]
```

요청당 3회 커넥션 획득. 1,500 VU × 초당 수십 요청 = 초당 수만 커넥션 요청 → 풀 포화 → cascading failure.

**Redis Streams로 DB 커넥션 의존 제거**:

```
[Before]
HTTP Thread → ① DB 유저 조회 → ② DB 멤버십 확인 → ③ DB INSERT → ④ Redis Pub/Sub → 응답
              커넥션 3회 획득, 풀 포화 시 5~30s 대기

[After]
HTTP Thread → ① Redis 멤버십 캐시 GET → ② Redis Pub/Sub → ③ Redis XADD → 응답 (< 5ms, DB 0)

Background  → ChatMessageStreamConsumer: XREADGROUP 100건 → DB INSERT 1회 → XACK
              100건당 커넥션 1회 (vs 이전 300회)
```

| | Before | After | 감소율 |
|---|---|---|---|
| HTTP 경로 커넥션 (1K건) | 3,000회 | 0회 | 100% |
| 총 커넥션 획득 (1K건) | 3,000회 | 10회 | 99.7% |

**Redis 캐시 3종**:

| 캐시 | Redis 구조 | TTL | 효과 |
|------|-----------|-----|------|
| ChatMembershipCache | STRING `chat:member:{uid}:{rid}` + 네거티브 캐시 `"NOT_MEMBER"` | 5분/30초 | 멤버십 DB 쿼리 제거 |
| ChatMessageCache | ZSET `chat:room:{rid}:recent` (50건) | 2시간 | 첫 페이지 DB 쿼리 완전 스킵 |
| ChatRoomListCache | STRING `chat:roomlist:{uid}:{cid}` + 수동 JSON 직렬화 | 30초 | DB 쿼리 3개 스킵 |

**SecurityContext 활용**:
```java
// Before: DB SELECT 1회
User user = userService.getCurrentUser();
// After: JWT에서 파싱된 userId, DB 0회
Long userId = userService.getCurrentUserId();
```

#### Phase 4: VU 스케일 축소 (v6) — 47 에러, p95 1.8~2s

에러 47건은 Redis client timeout (CPU 포화 구간).

#### Phase 5: 커넥션 풀 축소 (v7 최종) — 0 에러, p95 1.73~2.35s

Redis 캐시 도입 후 DB 트래픽 95% 감소에 맞춰 풀 축소:

| 리소스 | 이전 | 축소 후 | 절약 |
|--------|------|---------|------|
| HikariCP write | 200 | 30 | ~170MB + MySQL 스레드 170개 |
| HikariCP read | 150 | 30 | MySQL 스레드 120개 |
| Tomcat threads | 400 | 100 | ~300MB |
| Tomcat connections | 20,000 | 8,000 | fd 12,000개 |
| Redis Lettuce | 128 | 512 | DB→Redis 이동 대응 |

테스트 전체 HikariCP pending 0 — 풀 30개가 충분함을 증명.

### 2.3 MySQL vs MongoDB 비교

| API | MySQL p95 | MongoDB p95 | 결론 |
|-----|-----------|-------------|------|
| 메시지 전송 | 1.73s | 1.84s | MySQL 6% 빠름 |
| 메시지 목록 | 2.09s | 2.35s | MySQL 11% 빠름 |
| 채팅방 목록 | 1.80s | 2.31s | MySQL 28% 빠름 |
| WS 연결 | 68ms | 1.45s | MySQL 21배 빠름 |
| 처리량 | 583K | 511K | MySQL 14% 높음 |

**채택: MySQL** — Redis 캐시가 DB 접근 95% 차단하므로 DB 엔진 차이 미미. MongoDB 드라이버 오버헤드만 추가.

### 2.4 최종 결과

| 지표 | v3 (최적화 전) | v7 (최종) | 개선율 |
|------|--------------|-----------|--------|
| 총 에러 | 81,911 | 0 | 100% |
| 전 Phase 성공률 | 일부 실패 | 100% | - |
| 메시지 전송 p95 | 6~13s | 1.73s | 73~87% |
| 메시지 목록 p95 | 6~13s | 2.09s | 68~84% |
| 채팅방 목록 p95 | 6~13s | 1.80s | 72~86% |
| WS 연결 p95 | - | 68ms | - |
| HikariCP 에러 | 84,000+ | 0 | 100% |

VU 스케일 테스트 (에러 0건 유지):

| 지표 | 1x (1,500 VU) | 2x (3,000 VU) | 3x (4,500 VU) |
|------|--------------|--------------|--------------|
| 성공률 | 100% | 100% | 100% |
| 에러 | 0 | 0 | 0 |
| 메시지 전송 p95 | 1.08s | 1.86s | 2.51s |
| WS 전송/수신 | 62K/162K | 125K/317K | 187K/481K |

4코어 8GB 단일 서버에서 4,500 동시 유저, 에러 0건.

### 2.5 sendAndPublish() TX 분리

**문제**: Redis Pub/Sub `publish()`가 `@Transactional` 안에서 실행 → DB 커넥션을 Redis 응답까지 보유 → write-pool acquire max 24.7s.

**수정**: Redis publish를 TX 외부로 분리 → write-pool acquire max 1.16s (95% 개선).

---

## 3. 알림 도메인

### 3.1 MySQL-Only 초기 테스트

#### CRUD 테스트 (c5.xlarge, 10커넥션 풀)

| 엔드포인트 | p50 | p95 | 결과 |
|-----------|------|------|------|
| list | 54.6ms | 356.7ms | PASS (<500ms) |
| unread | 54.1ms | 357.1ms | FAIL (>200ms) |
| mark | 70.4ms | 359.8ms | FAIL (>200ms) |
| delete | 27.9ms | 205.2ms | FAIL (>200ms) |
| mark-all | 37.8ms | 356.7ms | PASS (<500ms) |
| deep-page | 6.1ms | 14.7ms | PASS (<1000ms) |

핵심 병목: HikariCP 풀 고갈 — 10개 커넥션, 대기 최대 187건.

#### Mark-all Contention 테스트

`UPDATE notification SET is_read = true WHERE user_id = ? AND is_read = false` — 같은 유저에 50 VU가 동시 요청 시 InnoDB row lock 대기 p50=6,171ms, p95=9,490ms 발생.

#### SSE 테스트

- SSE 500개 동시 연결에도 CRUD p95 14.1ms (거의 영향 없음)
- SSE 연결 성공률 100%

### 3.2 핵심 최적화

#### 3.2.1 워터마크 방식 mark-all — O(N) → O(1)

**문제**: `UPDATE notification SET is_read = true WHERE user_id = ? AND is_read = false` — 유저의 unread 알림 전부에 X-lock.

**수정**: `user_notification_state` 테이블 추가.

```sql
CREATE TABLE user_notification_state (
  user_id BIGINT NOT NULL PRIMARY KEY,
  read_all_upto_id BIGINT NOT NULL DEFAULT 0,
  updated_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
    ON UPDATE CURRENT_TIMESTAMP(6)
);
```

- mark-all: `INSERT INTO user_notification_state VALUES (:userId, :maxId) ON DUPLICATE KEY UPDATE read_all_upto_id = GREATEST(read_all_upto_id, VALUES(read_all_upto_id))`
- O(N) UPDATE → O(1) upsert
- 동시 호출에도 `GREATEST`로 안전 (A: maxId=120, B: maxId=130 → 최종 130)
- 결과: mark-all p95 8,228ms → 1,075ms (87% 개선)

#### 3.2.2 LEFT JOIN 제거 — 500 에러 완전 해소

**문제**: `countUnreadByUserId`와 `findNotificationsByUserId`가 `LEFT JOIN user_notification_state` 사용 → markAllAsRead의 write lock과 shared lock 충돌 → 1500 VU에서 500 에러 135건.

```sql
-- Before: LEFT JOIN으로 lock 경합
SELECT COUNT(*) FROM notification n
LEFT JOIN user_notification_state uns ON uns.user_id = n.user_id
WHERE n.user_id = :userId AND n.is_read = false
AND n.notification_id > COALESCE(uns.read_all_upto_id, 0)
```

**수정**: 워터마크를 별도 단건 조회 후 파라미터로 전달.

```java
// 1. 워터마크 단독 조회 (PK lookup, O(1))
Long watermark = getWatermark(userId);
// 2. notification 쿼리에서 LEFT JOIN 제거
countUnread(userId, watermark);
```

- 500 에러: 135건 → 4건 (97% 감소)
- SSE E2E 전달률: 0% → 56.6%

#### 3.2.3 markAllAsRead 단일 문장 통합

- 기존: `SELECT MAX()` → `INSERT ON DUPLICATE KEY UPDATE` (2회 round-trip, shared lock 유지)
- 변경: `INSERT INTO ... SELECT MAX() ... ON DUPLICATE KEY UPDATE` (1회, lock 최소화)

#### 3.2.4 CQRS 분리

`NotificationService` → `NotificationCommandService` + `NotificationQueryService` + Facade.

#### 3.2.5 Port/Adapter 추상화

`NotificationStoragePort` → MySQL 어댑터 / MongoDB 어댑터. 설정 1줄로 스토리지 교체.

#### 3.2.6 SSE 최적화

- SSE 타임아웃 30s → 300s (재연결 빈도 10배 감소)
- `SseMissedNotificationRecovery`: SSE 미연결 유저 복구 스킵 가드 추가
- SSE Recovery: `CompletableFuture.runAsync()` → 동기 `recover()` 호출 → delivery rate 53.7% → 59.8%
- `NotificationUndeliveredCache`: 오프라인 유저 알림을 Redis LIST에 캐싱 → SSE 재연결 시 DB 대신 Redis에서 복구

#### 3.2.7 R/W 풀 분리 효과 (MySQL-Only)

- Write pending 183까지 올라가도 Read pending=0 유지 (13회 중 10회)
- 조회 API가 쓰기 작업의 커넥션 부족에 영향받지 않음
- 실질적으로 커넥션 풀 용량 2배 (W 300 + R 300)

### 3.3 MySQL vs MongoDB 비교 (4000~9000 VU)

| 부하 | MySQL | MongoDB |
|------|-------|---------|
| 4000 VU | 전 Phase 100%, 에러 5건 | 전 Phase 100%, mark-all p95 1.0s |
| 6000 VU | 붕괴 (mark p50 29s) | 전 Phase 96~100% |
| 9000 VU (3x 시드) | 안정 (워터마크 적용 후) | 안정, 전 엔드포인트에서 더 빠름 |

> 9000 VU MySQL이 안정적으로 버틴 것은 mark-all을 워터마크 방식으로 O(N) → O(1)로 바꾼 뒤의 결과다 (3.2.1절 참조). 워터마크 적용 전에는 6000 VU에서 mark p50 29초로 붕괴했다.

**채택: MongoDB** — 같은 워터마크 적용 후에도 MongoDB가 전 엔드포인트에서 더 빠르고, 워터마크 이전의 MySQL 한계(O(N) UPDATE 시 row lock contention)가 구조적으로 재발할 수 있다는 판단.

### 3.4 모니터링 라운드 (c5.xlarge, max 2000 VU)

#### Round 1 — Heap 2GB

| 항목 | 결과 |
|------|------|
| CPU | 77~84% |
| GC Allocation Stall | 44회, 108.6초, max 3.4초 |
| 에러 | 17,422건 (TestNotificationController 미활성화 14,778건) |
| 결론 | Heap 2GB 부족, Round 2에서 4GB로 증설 |

#### Round 2 — Heap 4GB, 프로필 수정

| 항목 | Round 1 → Round 2 |
|------|-------------------|
| 앱 에러 | 17,422 → 0 (100% 감소) |
| CPU | 77~84% → 47% |
| GC Stall | 44회 → 0 |
| Max response | 4.5s → 2.6s |
| 성공률 | 일부 실패 → 전 Phase 100% |

Round 2 조치: Heap 2GB→4GB, TestNotificationController ec2 프로필 추가, MockTossPaymentClient ec2 활성화, RabbitMQ/MongoDB 의존성 제거.

#### Round 3 — 1500 VU, SSE Recovery 동기 실행

| 항목 | Round 2 → Round 3 |
|------|-------------------|
| Max VU | 1000 → 1500 |
| 에러 | 0 → 0 |
| CPU | 47% → 82% (VU 증가) |
| SSE delivery | 53.7% → 59.8% |
| DB lock waits | - → 0건 |
| GC pause | - → 0.008초 |

**결론**: c5.xlarge 단일 앱 서버에서 1500 동시 사용자, 에러 0건, 100% 성공률. CPU 포화(82%)가 유일한 병목.

### 3.5 최종 결과

| Phase | Round 1 → Round 3 |
|-------|-------------------|
| Baseline (300 VU) | 불안정 → 100% |
| Extreme (1000~1500 VU) | 71% → 99.16% |
| 앱 에러 | 17,422 → 0 |
| CPU | 77~84% → 82% (VU 1.5배인데 유사) |
| GC Stall | 44회/108.6s → 0 |

---

## 4. 정산 도메인

### 4.1 최적화 내용

| 최적화 | 내용 | 효과 |
|--------|------|------|
| 배치 TX 통합 | 참가자별 독립 TX → 정산당 1 TX (8TX×5쿼리 → 1TX×12쿼리) | 750건: 2분 → 31.7초 |
| 배치 조회 쿼리 | IN절 배치 (WalletIds, UserSettlementIds) | SELECT N+1 제거 |
| DB 인덱스 3종 | settlement(schedule_id), user_settlement(settlement_id, user_id), wallet_transaction(wallet_id, status, created_at) | full scan → index scan |
| N+1 제거 | WalletTransaction → Payment JOIN FETCH | 추가 20개 쿼리 제거 |
| Kafka + Outbox 직접 | RabbitMQ/Redis Streams/Spring Events 어댑터 제거 | 코드 단순화 |
| Outbox 처리량 증가 | 200ms/200건 → 50ms/500건 | 처리 속도 5배 |
| Stuck 복구 스케줄러 | 5분마다 IN_PROGRESS 5분 초과 → FAILED 전환 | 정산 hang 방지 |

### 4.2 R/W 풀 분리 — 단일 MySQL에서 역효과

| API | 단일 풀 400 p95 | R/W 분리 (350+350) p95 |
|-----|----------------|----------------------|
| 지갑 조회 | 1.10s | 8.23s (648% 악화) |

**원인**: write-350 + read-350 = 총 700 커넥션 → 단일 MySQL에 과부하. read replica 없이 R/W 분리는 커넥션 수만 늘려 MySQL 부하 증가.

**결론**: 단일 MySQL 인스턴스에서 R/W 분리는 불필요. read replica 추가 시에만 유효.

### 4.3 Redis 캐시 사용 현황

Finance 도메인은 데이터 캐싱을 사용하지 않음. Redis는 동시성 제어 전용:

| 용도 | 키 패턴 | TTL | 비고 |
|------|---------|-----|------|
| 결제 멱등성 게이트 | `payment:gate:{orderId}` | 5분 | 중복 결제 방지 |
| 결제 정보 임시 저장 | `payment:{orderId}` | 30분 | save→verify→confirm 3단계 검증 |
| 지갑 분산 락 | `wallet:gate:{userId}:{op}` | 10초 | 동시 출금 방지 (Lua 스크립트) |

### 4.4 최종 결과 (EC2 c5.xlarge 기준)

| 지표 | Round 1 | Round 5 |
|------|---------|---------|
| 정산 요청 p95 | 18.64s | 1.50s |
| 지갑 조회 p95 | 14.95s | 1.24s |
| 5xx Stress | 2.74% | 0.00% |
| Lock wait timeout | 175+ | 0 |
| RPS (600 VU) | ~550 | ~2,000 |

28 PASS / 0 FAIL, 전 Phase 성공률 100%, 5xx 0%.

> 이 표는 EC2 c5.xlarge에서 측정한 5라운드 결과다. 그 전에 로컬에서 별도로 19.5s → 2.46s (7.9배)의 1차 개선이 있었고 ([정산 배치 + Kafka 튜닝](/finance-settlement-batch-kafka-tuning/)), EC2로 옮긴 직후 다시 18.64s 베이스라인이 잡혔다. 즉 정산 도메인의 최적화는 두 단계를 거쳤고, "92% 개선"은 EC2 단독 측정만 반영한 수치다.

---

## 5. 검색 도메인

### 5.1 Port/Adapter 추상화

`SearchPort` → ES 어댑터 / MySQL FULLTEXT 어댑터. 설정으로 교체 가능.

| 최적화 | 내용 |
|--------|------|
| Port/Adapter | SearchPort → ES / MySQL FULLTEXT 어댑터 |
| MySQL FULLTEXT | ngram parser (token_size=2), ApplicationReadyEvent에서 자동 인덱스 생성 |
| ES nori 토크나이저 | 한국어 형태소 분석 |

### 5.2 ES vs MySQL FULLTEXT 비교

| 지표 | ES | MySQL FULLTEXT |
|------|-----|----------------|
| 단일 키워드 p95 | 29ms | 24.11s |
| 스파이크 p95 | 261ms | 45.40s |
| 5xx 에러 | 0 | 29,966 |
| 처리량 | 1,029K req | 366K req |

> 위 표는 같은 코드의 최적화 결과가 아니라 **검색 엔진을 교체한 결과**다. "830배 개선" "99.9% 감소" 같은 표현으로 단일 코드 변경의 효과처럼 읽히지 않도록 주의. ES는 역색인 + nori 토크나이저, MySQL FULLTEXT는 ngram parser(token_size=2). 자료구조와 알고리즘 자체가 다르기 때문에 차이가 크다.

**채택: ES** — 키워드 검색에서 두 자릿수 ms와 두 자릿수 초의 차이는 다른 종류의 시스템이고, MySQL FULLTEXT는 ES 불가 시 폴백 용도로만 유지.

---

## 6. 공통 인프라 튜닝

### 6.1 JVM 설정

| 항목 | 설정 |
|------|------|
| GC | ZGC Generational (`-XX:+UseZGC -XX:+ZGenerational`) |
| Heap | 2~3GB (인스턴스 크기에 따라) |
| Virtual Threads | `spring.threads.virtual.enabled: true` |
| Direct Memory | `-XX:MaxDirectMemorySize=256m` |
| 사전할당 | `-XX:+AlwaysPreTouch` (GC stall 감소) |
| VT 스케줄러 | `-Djdk.virtualThreadScheduler.parallelism=8` |
| JFR | continuous, 500MB max, 1h max age |

### 6.2 JWT Parser 싱글턴 캐싱

**진단**: 9000 VU 알림 테스트 중 스레드 덤프에서 161~388개 BLOCKED 스레드 발견.

```
BLOCKED on TomcatEmbeddedWebappClassLoader.loadClass()
  ← Classes.newInstance()
  ← Jwts.parser()        ← 매 요청마다 호출
```

`Jwts.parser()`가 내부적으로 `ClassLoader.loadClass()` (synchronized) 호출 → 모든 API 요청이 하나의 lock에 직렬화.

```java
// Before: 매 요청마다 parser 생성
public Claims parse(String token) {
    return Jwts.parser().setSigningKey(key).parseClaimsJws(token).getBody();
}

// After: 1회 생성, 필드 캐싱 (JwtParser는 thread-safe)
private JwtParser jwtParser;

@PostConstruct
void init() {
    jwtParser = Jwts.parser().setSigningKey(key).build();
}

public Claims parse(String token) {
    return jwtParser.parseClaimsJws(token).getBody();
}
```

BLOCKED 스레드: 388 → 0.

### 6.3 커널 튜닝 (EC2)

```
net.core.rmem_max = 16MB                # SSE/WebSocket 수신 버퍼
net.core.wmem_max = 16MB                # 송신 버퍼
net.ipv4.tcp_keepalive_time = 60        # 유휴 연결 빠른 감지
net.ipv4.tcp_slow_start_after_idle = 0  # 재사용 커넥션 cwnd 유지
net.ipv4.tcp_fin_timeout = 15           # TIME_WAIT 빠른 회수
net.core.somaxconn = 65535              # listen backlog 확대
```

효과: SSE 연결 p95 708ms → 90ms, 전체 처리량 20% 증가.

### 6.4 Virtual Thread 피닝 분석

- `SseEmitter.send()` — Spring 내부 `synchronized` 블록 → Virtual Thread pinning
- MongoDB 드라이버 — `synchronized` 연결 풀
- 권장: `ReentrantLock` 전환

### 6.5 EC2 다운사이징

| | 이전 | 이후 | 비용 절감 |
|---|------|------|-----------|
| App 서버 | c5.xlarge (4 vCPU, 8GB) | t3.medium (2 vCPU, 4GB) | ~70% |
| Infra 서버 | c5.2xlarge (8 vCPU, 16GB) | t3.large (2 vCPU, 8GB) | ~70% |

인프라 컨테이너 축소:

| 컨테이너 | 이전 | 이후 |
|----------|------|------|
| MySQL | 4.5GB | 2.5GB (buffer_pool 1.5GB) |
| MongoDB | 3GB | 1GB (WiredTiger 0.5GB) |
| ES | 2GB | 1.28GB (heap 512MB) |
| Redis | 768MB | 512MB (maxmemory 384MB) |
| Kafka | 1GB | 768MB (heap 384MB) |
| RabbitMQ | 존재 | 제거 |
| Grafana Image Renderer | 존재 | 제거 |
| ES Exporter | 존재 | 제거 |

### 6.6 @Cacheable 직렬화 이슈

`GenericJackson2JsonRedisSerializer` + `activateDefaultTyping(NON_FINAL)` 환경에서:
- `PageImpl` — 기본 생성자 없음 → 역직렬화 실패
- Java `record` 타입 — compact constructor와 DefaultTyping 호환 불가

해결: `@Cacheable` 대신 커스텀 캐시 클래스 + `StringRedisTemplate` + 수동 JSON 직렬화.

### 6.7 SecurityContext userId 추출

```java
// Before: DB SELECT 1회
User user = userService.getCurrentUser();

// After: JWT에서 파싱된 userId, DB 0회
Long userId = userService.getCurrentUserId();
```

모든 도메인에 적용 — 요청당 DB 쿼리 1건 절약.

---

## 7. 도메인별 최종 기술 스택

| 도메인 | DB | 메시지 큐 | 캐시 |
|--------|-----|----------|------|
| 피드 | MySQL | Kafka (engagement) | Redis (첫 페이지, popularity) |
| 채팅 | MySQL | Redis Streams (비동기 저장) + Redis Pub/Sub (실시간) | Redis 캐시 3종 |
| 알림 | MongoDB | Spring Events | Redis (SSE 분산 레지스트리, 미전달 캐시) |
| 검색 | ES (MySQL 폴백) | - | Redis |
| 정산 | MySQL | Kafka + Outbox | - (분산 락/게이트만) |

---

## 8. 코드 리뷰 수정사항

2026-03-05 코드 리뷰에서 식별된 11개 이슈와 수정 내역.

### 수정 완료 (11건)

| # | 이슈 | 심각도 | 수정 |
|---|------|--------|------|
| 1 | NotificationService 모놀리식 | Critical | CQRS 분리 (Command + Query + Facade) |
| 2 | @Transactional 내 이벤트 발행 | Critical | `@TransactionalEventListener(AFTER_COMMIT)` 확인 — 이미 안전 |
| 3 | Pagination 무제한 파라미터 | High | `@Validated` + `@Min`/`@Max` 추가 |
| 4 | DataIntegrityViolation 미처리 | High | `@ExceptionHandler` 추가 → HTTP 409 |
| 5 | Notification userId 필드 중복 | Critical | 중복 `Long userId` 필드 제거 |
| 6 | RedisTemplate null 체크 | Medium | dead code 제거 |
| 7 | Native SQL → JPQL | High | 3개 쿼리 JPQL 전환 (`markAllAsRead`의 MySQL-specific `ON DUPLICATE KEY UPDATE` 제외) |
| 8 | SSE Delivery E2E 검증 미비 | Enhancement | Phase 6b 추가 — 알림 생성→SSE 실제 전달 검증 |
| 9 | Finance 시드 pending_out 불일치 | Critical | 참여자 wallet `pending_out` 정합성 복구 |
| 10 | AUTO_INCREMENT 드리프트 | High | `resolve_db_offsets()` — DB에서 실제 MIN/MAX 자동 감지 |
| 11 | k6 디렉토리 정리 | Enhancement | 불필요 파일 7개 제거 |

### 식별 but 미수정 (5건)

| # | 이슈 | 사유 |
|---|------|------|
| 1 | Toss API circuit breaker | Resilience4j 도입 필요, 아키텍처 논의 |
| 2 | Redis Lua가 DB TX 밖 | 설계상 의도 (Redis 원자적, DB 비동기) |
| 3 | @WebMvcTest 부재 | Security context 모킹 비용, 다음 스프린트 |
| 4 | onlyone-common 테스트 0 | 유틸 클래스, 로직 성장 시 추가 |
| 5 | Club 모듈 테스트 2건 | 통합 테스트로 커버 |

---

## 9. 전체 성과 요약

### API 성능 개선 (p95 기준)

| API | 최적화 전 | 최적화 후 | 개선율 |
|-----|-----------|-----------|--------|
| 피드 상세 | 1,395ms | 398ms | 71% |
| 댓글 목록 | 614ms | 15ms | 97% |
| 댓글 생성 | 5,750ms | 260ms | 95% |
| 피드 생성 | 3,180ms | 102ms | 97% |
| mark-all (알림) | 8,228ms | 1,075ms | 87% |
| 채팅 메시지 전송 | 6~13s | 1.73s | 73~87% |
| 정산 요청 | 18,640ms | 1,500ms | 92% |
| 지갑 조회 | 14,950ms | 1,240ms | 92% |
| 검색 (ES) | 24,110ms | 29ms | 99.9% |

### 인프라 개선

| 지표 | 이전 | 이후 |
|------|------|------|
| 알림 에러 | 17,422건 | 0건 |
| 채팅 에러 | 81,911건 | 0건 |
| 피드 에러율 | 9.02% | 0.17% |
| 정산 5xx | 2.74% | 0.00% |
| CPU | 100% (포화) | 3~30% (평상시) |
| GC Stall | 44회/108.6s | 0 |
| 채팅 DB 커넥션 (1K 메시지) | 3,000회 | 10회 |
| EC2 비용 | c5.xlarge + c5.2xlarge | t3.medium + t3.large (70% 절감) |

### 적용 기술 스택

- **JVM**: ZGC Generational (본 환경 — t3.medium, Heap 2~4GB, write-heavy 시나리오에서 G1GC 대비 CPU 70%p 절감 측정. 일반 룰이 아니며 워크로드/할당률에 따라 차이가 작거나 역전될 수 있다.)
- **DB 인덱스**: 복합 인덱스 10종+ 추가
- **캐시**: Redis 캐시 14종 + 커스텀 캐시 3종
- **비동기**: Redis Streams, Kafka, @Async 이벤트
- **사전계산**: popularity_score 5분 주기 배치 갱신
- **페이징**: 커서 기반 pagination (OFFSET 제거)
- **커널**: TCP 버퍼/keepalive/backlog 튜닝
- **보안**: JWT Parser 싱글턴 캐싱 (ClassLoader lock 제거)
