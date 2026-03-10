---
title: 알림 부하 테스트 — AWS EC2에서 MySQL write 붕괴와 MongoDB 비교
date: 2026-03-07
tags: [AWS, EC2, MySQL, MongoDB, HikariCP, SSE, k6, 성능최적화, 부하테스트, Java, 알림]
permalink: /notification-aws-ec2-load-test/
excerpt: "EC2에 배포한 알림 API에서 MySQL write p95 1.2초, SSE 전달률 63.2%가 관측됐다. 2x VU(6,000 VU)로 스케일하면 MySQL write는 완전히 붕괴한다 — mark p50 29초, delete p50 59초. Port/Adapter 추상화로 MongoDB로 전환하여 비교하면 mark p50 30ms(952배), 총 iterations 10.5배, 6,000 VU Double Spike 100% 통과. document-level lock vs row lock의 구조적 차이를 확인했다."
---

## 개요

[로컬 부하 테스트](/notification-api-performance-improvement/)에서 알림 API 성공률을 13% → 99.9%로 개선하고, [SSE 시나리오 테스트](/sse-notification-load-test-analysis/)에서 7,500 VU까지 안정성을 확인했다. 이번에는 AWS EC2에 배포한 실제 환경에서 동일한 부하 테스트를 실행했다.

### EC2 인프라 구성

| 역할 | 인스턴스 타입 | 사양 | 비고 |
|------|-------------|------|------|
| Load Generator (k6) | c5.xlarge | 4 vCPU, 8GB | 부하 발생기 |
| App Server | c5.xlarge | 4 vCPU, 8GB | Spring Boot 앱 |
| Infra (MySQL, Redis 등) | c5.2xlarge | 8 vCPU, 16GB | DB·인프라 전용 분리 |

앱 서버와 인프라(DB, Redis)를 별도 인스턴스로 분리했다. 로컬 테스트에서는 앱·DB·k6가 전부 동일 머신에서 실행되어 네트워크 지연이 0이었지만, EC2에서는 인스턴스 간 네트워크 통신이 발생한다.

### 로컬 vs EC2 환경 비교

| 항목 | 로컬 | EC2 |
|------|------|-----|
| 네트워크 | localhost (지연 0ms) | EC2 인스턴스 간 네트워크 |
| HikariCP pool | 100 | 300 (ec2 프로필) |
| DB | 로컬 MySQL (앱과 동일 머신) | c5.2xlarge 별도 인스턴스 |
| CPU | 개발 장비 (제한 없음) | App 4 vCPU, Infra 8 vCPU |
| 메모리 | 개발 장비 (제한 없음) | App 8GB, Infra 16GB |
| 모니터링 | 로그 기반 | 앱 서버 + MySQL 서버 메트릭 수집 |

로컬에서는 네트워크 지연이 0이고 CPU/메모리가 충분해 병목이 드러나지 않는 구간이, EC2에서는 다른 양상을 보였다.

---

## 테스트 결과

### k6 전체

| 항목 | 결과 |
|------|------|
| 총 시간 | 18분 35초 |
| 총 iterations | 1,386,269 |
| 전 Phase 성공률 | 100% (P2~P10) |

### 엔드포인트별 응답 시간

| Endpoint | p50 | p95 | 비고 |
|----------|-----|-----|------|
| list (알림 목록) | 23.8ms | 949.5ms | 읽기, 고부하 구간에서 p95 상승 |
| unread (미읽음 수) | 13.8ms | 798.3ms | 서브쿼리 포함 |
| mark (읽음 처리) | 237.8ms | 1,210.9ms | write, row lock 발생 |
| delete (삭제) | 173.2ms | 1,092.0ms | write, 2단계 쿼리 |
| mark-all (전체 읽음) | 13.7ms | 1,323.7ms | write, 대량 UPDATE |
| deep-page (딥 페이징) | 22.4ms | 308.1ms | 커서 기반 |
| sse (SSE 연결) | 5,000ms | 5,001ms | timeout 기반 고정값 |

p50은 전 엔드포인트에서 정상 범위다. 문제는 p95 구간이다. write 작업(mark, delete, mark-all)의 p95가 1~1.3초로, 로컬 테스트에서는 관측되지 않던 수치다.

### SSE Delivery E2E

| 항목 | 결과 |
|------|------|
| 전달 성공률 | 63.2% |
| 전달 지연 p50 / p95 | 308ms / 650ms |
| 알림 생성 성공률 | 100% |
| 알림 생성 p50 / p95 | 4.9ms / 314.8ms |

---

## 로컬 테스트와의 비교

### 성공률

| 환경 | 테스트 | 성공률 |
|------|--------|--------|
| 로컬 | API 성능 최적화 후 ([9차 시도](/notification-api-performance-improvement/)) | 99.9% |
| 로컬 | SSE 시나리오 — delivery ([7가지 시나리오](/sse-notification-load-test-analysis/)) | 99.57% |
| **EC2** | **전 Phase** | **100%** |

EC2에서 성공률 100%를 달성한 이유는 두 가지다. HikariCP pool을 100 → 300으로 확대했고, 로컬 테스트 이후 적용한 비동기 분리와 인덱스 최적화가 반영되어 있다.

### SSE 전달률

| 환경 | 전달률 | 조건 |
|------|--------|------|
| 로컬 | 99.57% | 200 VU, SSE 단독 |
| 로컬 | 100% | 프론트엔드 SSE delivery 테스트 (200 receivers) |
| **EC2** | **63.2%** | 고부하 혼합 시나리오 (최대 1,500 VU) |

로컬에서 99%+ 전달률을 보이던 SSE가 EC2에서 63.2%로 하락했다. 원인은 세 가지다.

1. **SSE timeout 60초**: 고부하 구간에서 SSE 연결이 timeout으로 끊기고, 재연결 시 Tomcat 스레드가 포화 상태여서 새 연결이 지연된다.
2. **Recovery 동기 실행**: SSE subscribe 시 `missedNotificationRecovery.recover()`가 동기로 DB 쿼리 + SSE 전송을 수행하여 커넥션 점유 시간이 증가한다.
3. **네트워크 지연**: localhost에서는 SSE 이벤트 전송이 즉시 완료되지만, EC2 네트워크에서는 전송 지연이 발생하고 이것이 Extreme 구간(1,500 VU)에서 누적된다.

### write 응답 시간

| Endpoint | 로컬 p95 | EC2 p95 |
|----------|----------|---------|
| mark (읽음 처리) | < 100ms | 1,210.9ms |
| delete (삭제) | < 100ms | 1,092.0ms |
| mark-all (전체 읽음) | < 100ms | 1,323.7ms |

로컬에서는 HikariCP pool 100으로도 write p95가 100ms 미만이었다. EC2에서는 pool을 300으로 확대했음에도 1초 이상이다. 네트워크 왕복 시간이 각 DB 요청에 추가되고, 1,500 VU 최대 부하 구간에서 커넥션 경합이 집중적으로 발생한다.

---

## 앱 서버 모니터링

234 samples, 5초 간격으로 수집했다.

| 지표 | 평균 | 최대 |
|------|------|------|
| CPU | 35.3% | 98.2% |
| JVM Heap | 1,047MB | 2,048MB |
| HikariCP Active | 25.9 | 365 |
| HikariCP Pending | — | 275 |
| Threads | 621 | 899 |
| GC 횟수 | — | 1,755 |

HikariCP Active 최대 365는 pool size 300을 초과한다 (풀 확장 과정에서 일시적으로 설정값 초과 가능). Pending 최대 275는 Extreme/Spike 구간에서 커넥션 대기가 대량 발생했음을 의미한다.

---

## MySQL 모니터링

150 samples, 10초 간격으로 수집했다.

| 지표 | 평균 | 최대 |
|------|------|------|
| Connections | 243 | 401 |
| Running Threads | 20 | 330 |
| QPS | 5,089 | 21,492 |
| Slow Queries | — | 613 |
| Row Lock Waits | — | 32 |
| Buffer Pool Hit | — | 99.89% |

Running Threads 최대 330은 1,500 VU 최대 부하 구간에서 DB 스레드가 급증한 결과다. Slow Query 613건은 이 구간에 집중되어 있다. Buffer Pool Hit 99.89%는 데이터가 메모리에 충분히 적재되어 있어 디스크 I/O가 병목이 아님을 확인한다.

---

## 병목 분석

### 1. HikariCP Pending 275 — 커넥션 풀 일시 포화

Extreme/Spike 구간(1,500 VU)에서 write 작업이 동시 폭주한다. `markAsRead`, `deleteByIdAndUserId`는 각각 `@Transactional`으로 커넥션 1개를 점유한다.

`deleteByIdAndUserId`는 2회 UPDATE(unread 삭제 시도 → 실패 시 전체 삭제)를 수행하여 커넥션 점유 시간이 2배다. ec2 프로필의 `maximum-pool-size: 300`이지만 1,500 VU의 write가 동시에 몰리면 부족하다.

### 2. MySQL Running Threads 330, Slow Query 613

`countUnreadByUserId` 쿼리가 서브쿼리 + `COUNT(*)`를 사용한다.

```sql
SELECT COUNT(*) FROM notification
WHERE user_id = :userId AND is_read = false
  AND notification_id > (SELECT read_all_upto_id FROM user_notification_state ...)
```

인덱스 `(user_id, is_read, notification_id)`가 존재하지만, 서브쿼리 + 범위 스캔 비용이 있다. `markAsReadByIdAndUserId`와 `deleteByIdAndUserId`는 row-level lock을 유발하고, 같은 유저의 동시 mark/delete가 lock 경합을 발생시킨다. 1,500 VU에서 write 비율이 높아지면 InnoDB row lock waits 32건, lock time 2.7초가 기록됐다.

### 3. Read p95 상승 — write lock의 read 전파

p50은 list 23.8ms, unread 13.8ms로 정상 범위지만 p95가 각각 949.5ms, 798.3ms다. 고부하 구간에서 write lock이 read 대기를 유발한다. InnoDB의 MVCC에서도 고부하 lock 경합 시 read 지연이 발생한다.

`findNotificationsByUserId` 네이티브 쿼리에서 `user_notification_state` 서브쿼리 JOIN이 매 요청마다 실행되고, content 컬럼 조회를 위한 테이블 랜덤 I/O가 추가된다.

### 4. SSE 전달률 63.2%

k6의 SSE E2E 패턴(연결 → 알림 생성 → 수신 검증)에서 `sseTimeoutMillis: 60000`(1분)으로 설정되어 있다. 테스트 중 SSE 연결이 timeout으로 끊긴다. k6 시나리오 코드에서 타임아웃, 파라미터 범위, 검증 로직을 시드 데이터와 정확히 맞추는 작업은 단순하지 않다.

`SseEventSender.sendEvent()`는 `CompletableFuture.supplyAsync`로 비동기 처리하며, `sseEventExecutor`는 Virtual Thread 기반이라 스레드 수 자체는 병목이 아니다. SSE subscribe 시 `missedNotificationRecovery.recover()`의 동기 실행이 DB 쿼리 + SSE 전송을 포함하여 커넥션 점유 시간을 증가시킨다. 1,500 VU Extreme 구간에서 Tomcat 스레드 포화로 새 SSE 연결 자체가 지연된다.

---

## MongoDB 비교 테스트 — 19.7M 데이터, 6,000 VU

[로컬 테스트](/notification-storage-abstraction-mysql-mongodb/)에서 Port/Adapter 추상화로 MySQL↔MongoDB 전환이 가능한 구조를 만들었다. MySQL의 write 병목이 EC2에서도 동일하게 나타났으므로, 동일 EC2 환경에서 MongoDB로 전환하여 2x VU(최대 6,000 VU) 비교 테스트를 실행했다.

### MySQL vs MongoDB 응답 시간

| Endpoint | MySQL p50 | MySQL p95 | MongoDB p50 | MongoDB p95 |
|----------|-----------|-----------|-------------|-------------|
| list | 162ms | 387ms | 29.6ms | 1,663ms |
| unread | 147ms | 324ms | 31.4ms | 1,683ms |
| mark | 29,054ms | 60,001ms (타임아웃) | 30.5ms | 3,307ms |
| delete | 59,205ms | 60,000ms (타임아웃) | 16.5ms | 1,513ms |
| mark-all | 60,000ms (타임아웃) | 60,000ms | 13.9ms | 10,904ms |
| deep-page | 124ms | 282ms | 14.8ms | 493ms |

MySQL의 write 연산이 완전히 붕괴한다. mark p50 29초, delete p50 59초, mark-all은 p50부터 타임아웃이다. 19.7M row 테이블에 동시 UPDATE가 몰리면 row lock 경합으로 거의 모든 write가 타임아웃된다. read(list, unread)는 MySQL도 정상이지만, write가 DB 전체를 마비시킨다.

MongoDB는 mark p50 30.5ms(MySQL 대비 952배), delete p50 16.5ms(3,588배)로 개선됐다. mark-all은 MySQL에서 타임아웃이었으나 MongoDB에서는 p50 13.9ms로 처리 가능해졌다 (p95 10.9s는 19.7M 문서 중 해당 유저의 전체 미읽음을 갱신하는 비용).

### 핵심 비교

| 지표 | MySQL (2x VU) | MongoDB (2x VU) |
|------|---------------|-----------------|
| 총 iterations | 134,322 | 1,407,375 (10.5배) |
| Write Storm 성공률 | 1.5% | 100% |
| Read Storm 성공률 | 100% | 100% |
| Extreme 성공률 | 27.2% | 100% |
| Soak 성공률 | 24.8% | 100% |
| Spike 성공률 | ~40% | 96.8% |
| Double Spike (6,000 VU) 성공률 | FAIL | 100% |

MySQL Write Storm 성공률 1.5%가 이 결과를 요약한다. MySQL의 row lock contention이 MongoDB의 document-level short lock으로 대체되면서, 6,000 VU Double Spike에서도 에러 없이 통과했다.

다만 MongoDB는 row lock waits, slow query 같은 명시적 지표가 없어서 성능 한계 지점을 진단하기 어렵다. MySQL은 HikariCP Pending, Row Lock Waits, Slow Query 등으로 병목이 수치로 드러나지만, MongoDB는 동일한 수준의 가시성을 제공하지 않는다. write 성능은 압도적이지만, 한계에 도달했을 때 어디가 병목인지 파악하는 것은 MySQL보다 어렵다.

---

## 커널 튜닝 + JVM 최적화 — MongoDB 추가 개선

MongoDB 비교 테스트 후 OS 커널과 JVM 설정을 점검한 결과, 네트워크 버퍼와 GC에 튜닝 여지가 있었다. SSE는 장시간 유지되는 TCP 연결이라 버퍼 설정이 성능에 직접 영향을 준다.

### 문제점

| 설정 | 현재 | 변경 | 영향 |
|------|------|------|------|
| rmem_max / wmem_max | 212KB | 16MB | SSE 6,000 연결에서 버퍼 부족 |
| tcp_keepalive_time | 7,200초 | 60초 | 죽은 연결 감지가 2시간 지연 |
| tcp_slow_start_after_idle | 1 | 0 | idle 후 SSE 전송 시 slow start로 지연 |

c5.xlarge(4 vCPU, 8GB)에서 JVM heap 4GB + direct memory + thread stack으로 메모리 99.9%를 사용 중이었다. 6,000 SSE 연결 시 direct buffer가 부족할 수 있는 구조였다. ZGenerational GC, AlwaysPreTouch, Virtual Thread parallelism을 함께 적용했다.

### 결과

| Endpoint | 이전 p50 | 최적화 p50 | 이전 p95 | 최적화 p95 |
|----------|----------|-----------|----------|-----------|
| list | 29.6ms | 16.8ms (-43%) | 1,663ms | 1,437ms (-14%) |
| unread | 31.4ms | 16.5ms (-47%) | 1,683ms | 1,444ms (-14%) |
| mark | 30.5ms | 20.3ms (-33%) | 3,307ms | 3,055ms (-8%) |
| delete | 16.5ms | 13.2ms (-20%) | 1,513ms | 1,544ms (동일) |
| mark-all | 13.9ms | 10.8ms (-22%) | 10,904ms | 7,582ms (-30%) |
| deep-page | 14.8ms | 11.0ms (-26%) | 493ms | 459ms (-7%) |
| SSE 연결 p95 | 708ms | 90ms (-87%) | — | — |

| 지표 | 이전 MongoDB | 최적화 MongoDB |
|------|-------------|---------------|
| 총 iterations | 1,407,375 | 1,687,923 (+20%) |
| interrupted | 4 | 0 |
| Baseline 성공률 | 98.9% | 100% |

p50이 전체적으로 20~47% 개선된 것은 ZGenerational GC + AlwaysPreTouch 효과다. SSE 연결 p95가 708ms → 90ms(-87%)로 개선된 것은 커널 TCP 버퍼 확대와 keepalive 튜닝 효과다. mark-all p95가 10.9s → 7.6s(-30%)로 감소했고, 처리량은 20% 증가했다. interrupted 0으로 안정성도 향상됐다.

---

## 개선 방안

### 즉시 적용 (코드 변경 최소)

| # | 병목 | 개선안 | 예상 효과 |
|---|------|--------|-----------|
| 1 | HikariCP Pending 275 | `deleteByIdAndUserId` 2단계 → 1단계 쿼리 통합 | 커넥션 점유 시간 50% 감소 |
| 2 | MySQL Running 330 | list/unread 서브쿼리를 LEFT JOIN으로 변환 | 쿼리 실행 비용 감소 |
| 3 | Read p95 949ms | Read Replica 라우팅 확인 (`DataSourceRoutingConfig` 활용) | write lock에서 읽기 격리 |
| 4 | SSE 63.2% | SSE timeout 60초 → 300초, 미연결 유저 Recovery 스킵 | 전달률 80%+ |

**delete 쿼리 통합**

```sql
-- Before: 2단계 (unread 삭제 시도 → 실패 시 전체 삭제)
-- After: 1단계 — is_read 사전 확인 후 단일 DELETE
DELETE FROM notification WHERE id = :id AND user_id = :userId
```

**서브쿼리 → LEFT JOIN 변환**

```sql
-- Before: 매 요청마다 서브쿼리
WHERE notification_id > (SELECT read_all_upto_id FROM user_notification_state ...)

-- After: LEFT JOIN
LEFT JOIN user_notification_state uns ON uns.user_id = n.user_id
WHERE n.notification_id > COALESCE(uns.read_all_upto_id, 0)
```

**Covering Index 추가**

`(user_id, notification_id DESC, content, type, is_read, created_at)` — list 쿼리에서 테이블 랜덤 I/O를 제거한다.

### 중기 개선 (아키텍처 변경)

| # | 병목 | 개선안 | 효과 |
|---|------|--------|------|
| 1 | write 경합 | mark/delete를 Redis 큐잉 → 배치 flush (좋아요 스트림 패턴 적용) | row lock 경합 제거 |
| 2 | read 지연 | Read Replica 분리 (`DataSourceRoutingConfig` 활용) | write lock 영향 완전 격리 |
| 3 | SSE 안정성 | SSE 별도 포트/서버 분리 또는 WebFlux SSE 전환 | REST 경합 없이 독립 스케일링 |

---

## 로컬 vs EC2 — 차이가 발생하는 구조적 원인

| 항목 | 로컬 | EC2 | 영향 |
|------|------|-----|------|
| 네트워크 RTT | 0ms | 수 ms | 각 DB 요청에 RTT 추가, 1,500 VU에서 누적 |
| CPU 경합 | 없음 (개발 장비 여유) | 인스턴스 사양 제한 | CPU 최대 98.2% 도달 |
| 커넥션 풀 | 100 | 300 | 3배 확대에도 Pending 275 발생 |
| SSE 연결 유지 | 안정 (localhost) | 네트워크 지연 + timeout | 전달률 99% → 63% |
| GC 압력 | 낮음 | GC 1,755회 | JVM Heap 2,048MB 상한 도달 |

로컬 테스트에서 100ms 미만이던 write p95가 EC2에서 1초 이상으로 상승한 원인은 네트워크 RTT 자체보다, **RTT가 lock 보유 시간을 연장하는 효과**에 있다. 로컬에서는 lock 획득 → DB 연산 → lock 해제가 수 ms 내에 완료되지만, EC2에서는 각 단계에 네트워크 왕복이 추가되어 lock 보유 시간이 길어진다. 1,500 VU가 동시에 write를 실행하면 이 차이가 HikariCP Pending 275로 누적된다.

---

## 정리

| 항목 | EC2 MySQL (1,500 VU) | EC2 MySQL (6,000 VU) | EC2 MongoDB (6,000 VU) | MongoDB + 커널/JVM 튜닝 |
|------|---------------------|---------------------|----------------------|------------------------|
| 성공률 | 100% | Write Storm 1.5% | Write Storm 100% | Baseline 100% |
| write p95 | 1,092~1,323ms | 타임아웃 (60s) | mark 3,307ms | mark 3,055ms |
| 총 iterations | 1,386,269 | 134,322 | 1,407,375 | 1,687,923 |
| SSE 연결 p95 | — | — | 708ms | 90ms |
| mark-all p95 | 1,323ms | 타임아웃 | 10,904ms | 7,582ms |

1,500 VU에서 MySQL은 전 Phase 100%를 달성했지만 write p95가 1초 이상이었다. 6,000 VU로 스케일하면 MySQL의 write는 완전히 붕괴한다. MongoDB로 전환하여 비교하면 동일 6,000 VU에서 write p50이 수십 ms로 처리되고, Double Spike까지 에러 없이 통과한다. 커널 TCP 버퍼 확대와 ZGenerational GC 적용으로 SSE 연결 p95가 708ms → 90ms, 처리량이 20% 추가 증가했다.

> **알림 도메인의 병목은 MySQL의 row lock contention이다.** 19.7M row에 동시 UPDATE가 몰리면 MySQL은 구조적으로 한계에 도달한다. MongoDB의 document-level short lock은 비교 테스트에서 이 문제를 해소했다. 다만 MongoDB는 row lock waits, slow query 같은 명시적 진단 지표가 없어서, 성능 한계에 도달했을 때 병목 지점을 파악하기 MySQL보다 어렵다.

---

## 시리즈 탐색

**◀ 이전 글**
[React 코드 품질 — C(68점)에서 B(82점)으로, React Query 도입과 컴포넌트 분리](/react-code-quality-c-to-b-react-query/)

**▶ 다음 글**
[채팅 부하 테스트 — AWS EC2에서 TX 분리, Virtual Thread Pinning 분석, 4,500 VU 달성](/chat-aws-ec2-load-test/)
