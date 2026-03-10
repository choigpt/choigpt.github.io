---
title: 피드 부하 테스트 — AWS EC2에서 IN절 병목, covering index, X-lock 경합 해소
date: 2026-03-07
tags: [AWS, EC2, MySQL, Redis, HikariCP, k6, 성능최적화, 부하테스트, Java, 피드]
permalink: /feed-aws-ec2-load-test/
excerpt: "로컬에서 최적화를 마친 피드 도메인을 AWS EC2에 배포하고 3,000 VU까지 부하를 올렸다. 개인 피드 IN절 병목(p95 9.55s)을 covering index와 전 페이지 캐싱으로 2.00s까지 개선하고, comment_count 배치 버퍼링과 중복 인덱스 제거로 피드 생성 p95 14.21s → 2.88s를 달성했다."
---

## 개요

[로컬 피드 부하 테스트](/feed-storage-abstraction-lua-script-backend-comparison/)에서 storage abstraction, Lua script 토글, UNION ALL 인기 피드 등의 최적화를 적용했다. 이 글은 동일한 피드 도메인을 AWS EC2에 배포하고 부하 테스트를 수행한 결과를 다룬다.

### EC2 인프라 구성

| 역할 | 인스턴스 타입 | 사양 |
|------|-------------|------|
| Load Generator (k6) | c5.xlarge | 4 vCPU, 8GB |
| App Server | c5.xlarge | 4 vCPU, 8GB |
| Infra (MySQL, Redis) | c5.2xlarge | 8 vCPU, 16GB |

---

## 1차 테스트 — 초기 결과

### API 응답 시간

| API | p95 | avg | max |
|-----|-----|-----|-----|
| 클럽피드 목록 | 669ms | 234ms | 3.0s |
| 피드 상세 | 483ms | 73ms | 2.3s |
| 개인 피드 | 1,674ms | 507ms | 7.97s |
| 인기 피드 | 1,604ms | 475ms | 8.04s |
| 댓글 목록 | 50ms | 30ms | 2.1s |
| 좋아요 토글 | 297ms | 131ms | 2.1s |
| 리피드 | 8ms | 4ms | 238ms |

### Phase별 성공률

| Phase | 성공률 | VU |
|-------|--------|-----|
| Baseline | 66.1% | 300 |
| ClubList | 100% | 350 |
| Detail | 100% | 500 |
| Personal | 0% | 500 |
| Comments | 100% | 350 |
| Like | 100% | 700 |
| WriteMix | 100% | 350 |
| Extreme | 71.7% | 1,000 |
| Soak | 71.9% | 300 |

Thresholds: 8 PASS / 11 FAIL. 총 1,641,223 HTTP 요청, 에러율 20.4%.

### Phase 5 Personal 0% 원인

Phase 5 실패 원인은 쿼리 성능이 아니라 데이터 정합성 문제다. 시드 데이터에서 삭제된 Club을 참조하는 Feed가 존재하여 Club 엔티티 조회 시 500 에러가 발생했다.

### 개인 피드 IN절 병목

`WHERE club_id IN (20+개) ORDER BY feed_id DESC` 쿼리가 10.6M 행 테이블에서 filesort를 유발했다. 이것이 개인 피드 응답 시간 저하의 원인이다.

---

## 1차 최적화 — 복합 인덱스 + 커서 페이징 + popularity_score 사전 계산 + ZGC

### 적용 내용

1. **복합 인덱스 추가**
   - `idx_feed_club_feedid (club_id, feed_id)` — 개인 피드 커서 기반 쿼리용
   - `idx_feed_club_popularity (club_id, popularity_score)` — 인기 피드 인덱스 활용
2. **커서 기반 페이지네이션**: `WHERE feed_id < :cursor ORDER BY feed_id DESC LIMIT :limit` (OFFSET 제거)
3. **popularity_score 사전 계산**: per-row `LN(GREATEST(...))-TIMESTAMPDIFF(...)` 계산을 5분마다 배치 업데이트로 전환하여 인덱스 정렬 활용
4. **ZGC 적용**: G1GC에서 ZGC로 변경

### 결과 (Club 정합성 수정 포함)

| Phase | 1차 성공률 | 2차 성공률 |
|-------|-----------|-----------|
| Baseline | 66.33% | 100% |
| Personal | 0% | 100% |
| Extreme | 71.55% | 99.16% |
| Soak | 71.79% | 99.93% |

- 총 에러: 122,456 → 2,426 (98% 감소)
- Thresholds: 8/19 → 12/19 PASS
- CPU (1,000 VU): 100% (G1GC) → 3~30% (ZGC)
- HikariCP Pending: 병목 상태 → 0

---

## 2차 최적화 — UNION ALL 시도와 롤백

개인 피드에 UNION ALL 방식(각 club_id별 개별 서브쿼리 → Java에서 병합)을 적용했다.

| 방안 | 개인 피드 p95 | 인기 피드 p95 |
|------|-------------|-------------|
| IN절+청크 (baseline) | ~2.4s | ~2.1s |
| UNION ALL (Pool 300) | 11.32s | 387ms |
| UNION ALL (Pool 400) | 8.52s | 457ms |

인기 피드는 UNION ALL로 2.1s → 387ms로 개선되었다. 개인 피드는 유저당 약 20개의 순차 쿼리가 커넥션 풀을 포화시켜 오히려 응답 시간이 악화되었다.

결론: 개인 피드는 IN절+청크로 롤백하고, 인기 피드만 UNION ALL을 유지했다.

---

## 3차 최적화 — covering index + 전 페이지 캐싱

1,000 VU 테스트에서 개인 피드 단독 집중 시나리오를 실행한 결과, p95가 9.55s로 측정됐다. 1차 테스트의 1,674ms는 혼합 시나리오(Baseline 300 VU)에서의 수치이며, 500 VU가 개인 피드에 집중되면 IN절 병목이 증폭된다.

### Covering Index

`(club_id, feed_id, like_count, comment_count)` 인덱스를 추가하여 테이블 랜덤 I/O를 제거했다.

| 인덱스 | 개인 피드 p95 | avg |
|--------|-------------|-----|
| 원본 (club_id, feed_id) | 9.55s | 2.47s |
| Covering (club_id, feed_id, like_count, comment_count) | 5.94s | 2.18s |

Covering index 효과: p95 9.55s → 5.94s (38% 개선).

### 전 페이지 Redis 캐싱

기존 첫 페이지만 캐싱하던 구조를 전체 0~5 페이지로 확장했다 (TTL 30초). 단, Redis 캐시를 적용하면 p95가 급격히 개선되지만 DB 앞단에서 병목을 가리는 효과가 있다. 캐시 eviction이나 cold start 시점에 실제 DB 병목이 그대로 드러나므로, 캐시 적용 전후를 모두 측정해야 실제 DB 성능을 파악할 수 있다.

### 최종 결과 (covering index + 전 페이지 캐싱)

| 지표 | 최적화 전 | 최적화 후 | 변화 |
|------|---------|---------|------|
| 개인 피드 p95 | 9.55s | 2.00s | 79% 개선 |
| 개인 피드 avg | 2.47s | 269ms | 89% 개선 |
| 인기 피드 p95 | — | 517ms | — |
| 총 iterations | ~300K | 474K | 58% 처리량 증가 |

---

## 최종 전체 테스트 — 879K iterations

20분간 최대 1,000 VU로 879K iterations를 처리한 최종 테스트 결과다.

### API 응답 시간

| API | p95 | avg |
|-----|-----|-----|
| 클럽피드 목록 | 1.31s | 774ms |
| 피드 상세 | 923ms | 260ms |
| 개인 피드 | 1.35s | 469ms |
| 인기 피드 | 1.98s | 762ms |
| 댓글 목록 | 577ms | 165ms |
| 좋아요 토글 | 426ms | 313ms |
| 피드 생성 | 444ms | 328ms |
| 댓글 생성 | 629ms | 480ms |
| 리피드 | 434ms | 95ms |

Phase 2~8, 10: 98.5~100% 성공. Phase 9 (Extreme 1,000 VU): 32.6%.

### 인프라 메트릭

| 지표 | 유휴 | 피크 (Phase 9) |
|------|------|---------------|
| HikariCP Active | 1 | 300 (풀 MAX) |
| HikariCP Pending | 0 | 268 |
| JVM Heap | 170MB | 1,055MB |
| CPU | 2~4% | 100% |
| GC (ZGC) | — | 865회, 총 18.8s (1.6% overhead) |

### MySQL

| 지표 | 값 |
|------|-----|
| Slow Queries | +2,524 (Phase 9에서 발생) |
| Row Lock Waits | +152 |
| Buffer Pool Free | 142K → 28K pages (3GB 풀, 테스트 종료 시 여유 감소) |

### Redis

| 지표 | 값 |
|------|-----|
| Hit Rate | 97.8% |
| Peak OPS | 39,472/s |
| Memory | 225 → 302MB (+77MB) |

---

## 병목 분석 — Phase 9 (1,000 VU) 한계

Phase 9에서 성공률이 32.6%로 하락한 원인은 다음과 같다.

1. HikariCP 300 커넥션 풀 전부 소진, Pending 268
2. CPU 4코어 100% sustained
3. 커넥션 타임아웃이 cascade되어 60초 max 응답 발생

이는 코드 최적화로 해결할 수 없는 인프라 스케일링 한계다. 수평 확장(다중 앱 인스턴스) 또는 인스턴스 업그레이드(c5.2xlarge)가 필요하다.

---

## 4차 최적화 — X-lock 경합 해소와 3x 스케일

3차까지의 테스트는 단일 HikariCP 풀(300)로 실행했다. 4차부터 HikariCP를 write/read 풀로 분리하고, MySQL `max_connections`를 800으로 확대했다.

1,000 VU 테스트에서 확인된 병목을 분석한 결과, feed row에 대한 X-lock 경합이 쓰기 성능 저하의 원인이었다.

### 병목 진단

| 지표 | 값 |
|------|-----|
| Lock wait timeout (5s 초과) | 35,659건 |
| InnoDB row lock waits | 25,745건, 평균 5초 |
| Slow queries (10s+) | 2,636건 |
| read-pool 포화 | active=160, waiting=234 |
| feed 테이블 인덱스 수 | 10개 |

`UPDATE feed SET like_count = GREATEST(like_count + 1, 0)` 쿼리가 feed row에 X-lock을 건다. 좋아요 배치 consumer와 댓글 count 비동기 업데이트가 동일 feed row에 동시 접근하면서 직렬화가 발생했다.

### 적용 내용

**A. comment_count 배치 버퍼링**

기존: 댓글 생성마다 `@Async`로 즉시 `UPDATE feed SET comment_count = comment_count + 1` 실행 → 개별 X-lock.

변경: Redis `HINCRBY`로 델타를 축적하고 3초 주기로 배치 flush. 단일 `UPDATE ... CASE WHEN`으로 여러 feed를 1회에 갱신한다.

**B. MySQL 튜닝**

| 설정 | 변경 전 | 변경 후 |
|------|--------|--------|
| innodb_redo_log_capacity | 48MB | 512MB |
| innodb_lock_wait_timeout | 5s | 3s |
| innodb_flush_method | fsync | O_DIRECT |

redo log 48MB는 쓰기 부하 시 체크포인트를 빈번하게 유발한다. 512MB로 확대하여 I/O 경합을 줄였다.

**C. HikariCP 풀 조정**

MySQL `max_connections=800` 내에서 write 300 + read 300 = 600으로 분배했다. 기존 write 500 + read 500 = 1,000은 MySQL이 수용할 수 없는 크기였다.

**D. 중복 인덱스 제거**

covering index 추가 후 기존 인덱스 3개가 중복 상태였다.

| 제거 인덱스 | 대체 인덱스 |
|------------|-----------|
| `idx_feed_club_deleted_created` | `idx_feed_club_del_feedid_covering` |
| `idx_feed_club_feedid` | `idx_feed_club_feedid_covering`의 부분집합 |
| `idx_feed_club_popularity` | `idx_feed_club_del_popularity` |

인덱스 10개 → 7개로 축소. covering index를 포함해 인덱스를 적극적으로 추가한 결과 조회 성능은 개선됐지만, INSERT/UPDATE마다 인덱스 갱신 비용이 누적되어 쓰기 성능이 저하됐다. 피드 생성 p95가 14.21s까지 상승한 원인 중 하나다. 인덱스는 읽기와 쓰기의 트레이드오프이며, 조회 최적화만 측정하면 이 부작용을 놓친다.

### 3x 스케일 테스트 결과 (최대 3,000 VU)

11 Phase, 19분 54초, 1,213,438 iterations.

| 지표 | 최적화 전 (3x) | 최적화 후 (3x) | 변화 |
|------|--------------|--------------|------|
| 댓글 생성 p95 | 11.29s | 5.30s | 53% 개선 |
| 피드 생성 p95 | 14.21s | 2.88s | 80% 개선 |
| 피드 상세 p95 | — | 1.76s | — |
| 개인 피드 p95 | — | 2.86s | — |
| 인기 피드 p95 | — | 2.46s | — |
| 댓글 목록 p95 | — | 952ms | — |
| 좋아요 토글 p95 | — | 708ms | — |

- Phase 2~7: 99.94~100% 성공률
- Phase 9 Extreme (3,000 VU): 46.88%
- Phase 10 Soak: 94.50%

댓글 생성의 53% 개선은 comment_count 배치 버퍼링 효과이며, 피드 생성의 80% 개선은 중복 인덱스 제거로 INSERT 비용이 감소한 결과다. Phase 9(3,000 VU) 46.88%는 단일 인스턴스의 처리 한계이며, Phase 2~8까지 안정적으로 동작한다.

---

## 개선 방안

### 적용 완료

| # | 변경 | 효과 |
|---|------|------|
| 1 | covering index (club_id, feed_id, like_count, comment_count) | 개인 피드 p95 9.55s → 5.94s |
| 2 | 전 페이지 Redis 캐싱 (0~5 페이지, TTL 30초) | 개인 피드 p95 → 2.00s, avg 269ms |
| 3 | 인기 피드 UNION ALL | 인기 피드 p95 2.1s → 387ms (UNION ALL 단독 테스트 기준, 최종 혼합 테스트에서는 1.98s) |
| 4 | popularity_score 사전 계산 + 인덱스 | per-row 계산 제거, 인덱스 정렬 |
| 5 | ZGC 적용 | CPU 100% → 3~30% |
| 6 | 커서 기반 페이지네이션 | OFFSET 제거, deep page 성능 일정 |
| 7 | comment_count 배치 버퍼링 | 댓글 생성 p95 11.29s → 5.30s |
| 8 | 중복 인덱스 제거 (10개 → 7개) | 피드 생성 p95 14.21s → 2.88s (3x 스케일 최적화 전 기준) |
| 9 | MySQL 튜닝 (redo log 512MB, O_DIRECT) | I/O 경합 감소 |

### 추가 개선 후보

| # | 병목 | 개선안 | 효과 |
|---|------|--------|------|
| 1 | 3,000 VU 인프라 한계 | 수평 확장 또는 c5.2xlarge | 고부하 처리 |
| 2 | Pass2 렌더링 5회 쿼리 | parent/root 단일 쿼리 병합 | 목록 API p95 감소 |
| 3 | 개인 피드 IN절 한계 | Redis ZSET fan-out-on-write | DB 쿼리 0 |
| 4 | 댓글 생성 X-lock 잔여 | feed_comment FK의 S-lock 제거 | 쓰기 경합 추가 감소 |

---

## 정리

| 항목 | 1차 (최적화 전) | 1,000 VU 최적화 후 | 3,000 VU 최적화 후 |
|------|---------------|-------------------|-------------------|
| Phase 5 Personal 성공률 | 0% | 100% | 100% |
| Baseline 성공률 | 66.1% | 100% | 100% |
| 개인 피드 p95 | 1,674ms (혼합) / 9.55s (집중) | 2.00s | 2.86s |
| 인기 피드 p95 | 2.1s | 1.98s | 2.46s |
| 피드 생성 p95 | — | 444ms | 2.88s (14.21s에서 80% 개선, 3x 스케일 최적화 전 기준) |
| 댓글 생성 p95 | — | 629ms | 5.30s (11.29s에서 53% 개선) |
| CPU (1,000 VU) | 100% (G1GC) | 3~30% (ZGC) | — |
| Redis Hit Rate | — | 97.8% | — |

> **피드 도메인의 병목은 읽기(IN절)와 쓰기(X-lock)에서 각각 다르게 나타난다.** 읽기 병목은 covering index와 Redis 캐싱으로, 쓰기 병목은 배치 버퍼링과 중복 인덱스 제거로 해소된다. 3,000 VU 고부하에서의 한계는 단일 인스턴스의 물리적 제약이며, 수평 확장이 필요하다.

---

## 시리즈 탐색

**◀ 이전 글**
[채팅 부하 테스트 — AWS EC2에서 TX 분리, Virtual Thread Pinning 분석, 4,500 VU 달성](/chat-aws-ec2-load-test/)

**▶ 다음 글**
[정산 부하 테스트 — AWS EC2에서 시드 데이터 버그 수정과 5라운드 최적화](/settlement-aws-ec2-load-test/)
