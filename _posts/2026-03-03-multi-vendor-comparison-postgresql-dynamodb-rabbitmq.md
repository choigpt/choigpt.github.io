---
title: 멀티 벤더 비교 — PostgreSQL Lock, DynamoDB 알림 어댑터, Kafka/RabbitMQ/Redis Streams 정산 MQ
date: 2026-03-03
tags: [PostgreSQL, MySQL, DynamoDB, MongoDB, Kafka, RabbitMQ, Redis, 성능비교, 부하테스트, Port/Adapter, Testcontainers]
permalink: /multi-vendor-comparison-postgresql-dynamodb-rabbitmq/
excerpt: "각 도메인 최적화와 Storage 추상화를 완료한 뒤, 마지막 단계로 대안 벤더를 직접 비교했다. MySQL vs PostgreSQL Lock 경합, MongoDB vs DynamoDB 알림 저장, Kafka vs RabbitMQ vs Redis Streams 정산 MQ — Port/Adapter 패턴 덕분에 비즈니스 로직 변경 없이 전부 교체·측정할 수 있었다."
---

## 개요

도메인별 최적화를 마친 뒤 남은 질문은 하나였다. 현재 벤더가 스케일링 한계에 도달하면 어디로 전환할 것인가. Storage와 MQ를 Port/Adapter로 추상화해 둔 덕분에 비즈니스 로직을 건드리지 않고 대안 벤더를 꽂아서 비교할 수 있는 상태였다. 이 글은 세 가지 비교 실험의 결과를 정리한다.

| 비교 대상 | 현재 | 대안 | 핵심 관심사 |
|-----------|------|------|------------|
| RDBMS Lock | MySQL 8.0 (InnoDB) | PostgreSQL 16 | batch write 경합, deadlock 빈도 |
| 알림 Storage | MongoDB | DynamoDB (LocalStack) | 무한 스케일 vs 유연한 쿼리 |
| 정산 MQ | Kafka | RabbitMQ, Redis Streams | DLQ, infra 오버헤드, 메시지 보장 |

---

## MySQL vs PostgreSQL Lock Comparison

### Lock 경합 심각도 분류

부하 테스트에서 식별한 Lock 경합을 심각도 순으로 정렬했다.

| 순위 | 시나리오 | Lock 범위 | 심각도 | 비고 |
|------|---------|-----------|--------|------|
| 1 | Settlement batch capture | multi-row | CRITICAL | 10+ wallet 동시 잠금 |
| 2 | Settlement markProcessing + batchMarkCompleted | multi-row | CRITICAL | 상태 전이 배치 |
| 3 | Wallet findByUser `@PESSIMISTIC_WRITE` | `FOR UPDATE` | HIGH | 단일 row지만 빈도 높음 |
| 4 | Club/Schedule join | row + gap lock | HIGH | user_limit 체크 + unique constraint |
| 5 | Notification markAllAsRead | batch row | MEDIUM | 사용자별 전체 읽음 처리 |
| 6 | Feed commentCount increment | single row | LOW | 단일 row 갱신 |

1번과 2번이 CRITICAL인 이유는 정산 파이프라인 전체가 멈추기 때문이다. 10명 이상의 wallet을 한 트랜잭션에서 잠그면 MySQL InnoDB의 gap lock이 연쇄적으로 deadlock을 유발한다.

---

### 테스트 구성

두 DB를 동일 조건에서 비교하기 위해 Testcontainers로 격리 환경을 구성했다.

```java
// MySQLContainer(mysql:8.0) + PostgreSQLContainer(postgres:16-alpine)
// Raw JDBC — JPA/Hibernate 오버헤드 제거
// Virtual Thread + CountDownLatch — 동시성 제어
// 6개 시나리오, 시나리오당 50~100 VT
```

JPA를 거치지 않고 Raw JDBC를 사용한 이유는 ORM의 쿼리 변환·캐싱이 Lock 동작 자체를 왜곡할 수 있기 때문이다. Virtual Thread와 `CountDownLatch`로 50~100개의 동시 요청을 정확히 같은 시점에 발사하여 경합을 극대화했다.

---

### 핵심 차이점

핵심 차이는 Lock 전략의 근본적인 차이에서 나온다.

| 항목 | MySQL InnoDB | PostgreSQL |
|------|-------------|------------|
| Lock 모델 | gap lock + next-key lock | MVCC + row-level lock |
| 범위 INSERT | gap lock이 범위를 직렬화 | MVCC가 병렬 INSERT 허용 |
| 경합 회피 | 대기 or deadlock | `FOR UPDATE SKIP LOCKED` |
| Deadlock 빈도 (batch) | 빈번 | 거의 0 |

시나리오별 비교 결과는 다음과 같다.

**Settlement batch (CRITICAL)**
MySQL에서 10+ row를 한 트랜잭션으로 `FOR UPDATE` 잡으면 next-key lock이 겹치면서 deadlock이 빈번하게 발생했다. PostgreSQL은 MVCC 기반이라 동일 시나리오에서 deadlock이 거의 0에 수렴했다.

**Notification markAllAsRead (MEDIUM, 핵심 차별점)**
MySQL은 `UPDATE ... WHERE user_id = ? AND is_read = false`에서 gap lock이 걸려 동일 사용자의 다른 알림 INSERT까지 블로킹된다. PostgreSQL은 `FOR UPDATE SKIP LOCKED`로 경합 row를 건너뛰어 대기 자체가 발생하지 않는다. 이 시나리오가 두 DB의 가장 극명한 차이를 보여주는 케이스였다.

**Club/Schedule join (HIGH)**
MySQL은 unique constraint 체크 과정에서 gap lock이 INSERT를 직렬화한다. PostgreSQL은 MVCC가 병렬 INSERT를 허용하고 unique violation은 트랜잭션 커밋 시점에만 검사한다.

**Wallet FOR UPDATE / Feed comment_count (HIGH/LOW)**
단일 row 경합은 두 DB 모두 동일한 성능을 보였다. row-level lock의 동작이 본질적으로 같기 때문이다. 차이가 드러나는 것은 multi-row 또는 range 경합이다.

---

### k6 Lock 비교 테스트

실제 API 수준에서의 비교를 위해 k6 가중치 시나리오를 설계했다.

| Endpoint | 비중 | Lock 특성 |
|----------|------|-----------|
| Settlement (정산) | 25% | multi-row batch, deadlock 핵심 |
| Schedule join (일정 참가) | 20% | gap lock + unique constraint |
| Notification read-all (전체 읽음) | 20% | batch row, SKIP LOCKED 핵심 |
| Payment (결제) | 15% | wallet FOR UPDATE |
| Feed comments (댓글) | 10% | single row increment |
| Club (모임) | 10% | row + gap lock |

```javascript
// k6 설정
// Max 120 VUs, progressive stages
// 비중은 실제 트래픽 패턴 기반
```

예상대로 PostgreSQL은 batch write + markAllAsRead 시나리오에서 유의미한 우위를 보였고, single-row 경합 시나리오에서는 두 DB가 동등했다. 전체적으로 deadlock 발생 건수 자체가 PostgreSQL에서 급감했다.

---

## DynamoDB Notification Storage Adapter

알림 도메인은 이미 MySQL에서 MongoDB로 전환하여 47배 성능 향상을 확인한 바 있다. 다음 단계로 DynamoDB 어댑터를 구현해 serverless 스케일링 경로를 확보했다.

### 데이터 모델

```
DynamoNotificationItem
├── PK: userId (Partition Key)
├── SK: numericId (Sort Key, 시간순 정렬)
├── GSI-1: isRead (읽음 상태 필터링)
└── GSI-2: sseSent (SSE 미전송 조회)
```

`DynamoNotificationConfig`에서 테이블과 GSI를 자동 생성하도록 구성했다. 로컬 개발은 LocalStack으로 대체한다.

### 구현

`DynamoNotificationStorageAdapter`가 `NotificationStoragePort`의 8개 메서드를 전부 구현한다.

| 메서드 | DynamoDB 구현 방식 |
|--------|-------------------|
| save | PutItem |
| findByUser (페이징) | Query on PK + SK desc |
| markAsRead | UpdateItem (단건) |
| markAllAsRead | BatchWriteItem |
| countUnread | Query on GSI-1 with Count |
| findUnsentSse | Query on GSI-2 |
| markSseSent | UpdateItem |
| deleteByUser | BatchWriteItem |

### 동시성 비교 테스트

`DynamoVsMongoConcurrencyTest`를 작성했다. LocalStack(DynamoDB)과 Testcontainers(MongoDB)를 동시에 띄워 3가지 시나리오를 비교한다.

| 시나리오 | 목적 |
|---------|------|
| 대량 INSERT (1,000건) | write throughput |
| markAllAsRead (500건 batch) | batch update |
| 페이지네이션 조회 (커서 기반) | read latency |

### 트레이드오프

DynamoDB는 사실상 무한 스케일이 가능하지만, 유연한 쿼리가 불가능하다. GSI 설계가 곧 쿼리 능력을 결정한다. 현재 알림 도메인의 쿼리 패턴이 단순하기 때문에(userId 기반 조회, 읽음 필터, SSE 미전송 필터) DynamoDB로 커버 가능하지만, 복합 조건 검색이 추가되면 MongoDB가 유리하다.

---

## Settlement MQ Abstraction (Kafka vs RabbitMQ vs Redis Streams)

정산 이벤트 파이프라인의 MQ를 Port/Adapter로 추상화하고 세 가지 구현체를 비교했다.

### 추상화 구조

```
OutboxMessageRelay (port interface)
├── KafkaOutboxRelay
├── RabbitOutboxRelay
└── RedisStreamsOutboxRelay

SettlementEventProcessor — 비즈니스 로직 (MQ 무관)
├── *SettlementListener (MQ별 수신)
└── *LedgerListener (MQ별 수신)
```

`SettlementEventProcessor`에 비즈니스 로직을 추출하여 MQ 종류와 무관하게 동작하도록 만들었다. 각 MQ별 Listener는 메시지 수신과 ACK만 담당한다.

---

### Kafka (default, matchIfMissing=true)

```java
// KafkaOutboxRelay — Outbox 테이블 → Kafka 전송
// KafkaSettlementListener — settlement.events 토픽 수신
// KafkaLedgerListener — ledger 이벤트 수신
// Topic: settlement.events
// Partition key: settlementId (같은 정산은 같은 파티션)
```

기본 구현체다. `@ConditionalOnProperty(matchIfMissing = true)`로 설정 없이도 Kafka가 선택된다. `settlementId` 기반 파티셔닝으로 같은 정산 건의 이벤트 순서를 보장한다.

---

### RabbitMQ

```java
// RabbitOutboxRelay — Outbox 테이블 → RabbitMQ 전송
// RabbitSettlementListener — settlement queue 수신
// RabbitLedgerListener — ledger queue 수신
// Exchange: settlement.exchange (direct routing)
// DLQ: settlement.dlq (built-in)
```

RabbitMQ의 장점은 설정의 단순함이다. Exchange → Queue 바인딩만으로 라우팅이 완성되고, DLQ와 message TTL이 내장되어 있어 실패 메시지 처리에 별도 코드가 필요 없다. Kafka처럼 Consumer Group 오프셋을 관리할 필요도 없다.

---

### Redis Streams

```java
// RedisStreamsOutboxRelay — XADD로 스트림에 추가
// RedisStreamsSettlementListener — XREADGROUP + XACK
// RedisStreamsLedgerListener — XREADGROUP + XACK
// Consumer Group 기반 at-least-once delivery
```

Redis Streams의 최대 장점은 추가 인프라가 0이라는 점이다. 이미 캐시와 Pub/Sub으로 Redis를 사용하고 있으므로, MQ를 위한 별도 클러스터가 불필요하다. `XREADGROUP` + `XACK` 조합으로 Consumer Group 기반의 at-least-once 전송을 보장한다.

---

### MQ 비교 테스트

세 MQ를 동일 조건에서 비교하기 위해 Testcontainers로 Kafka, RabbitMQ, Redis를 동시에 띄웠다.

```yaml
# Property 전환으로 MQ 선택
app:
  settlement:
    message-broker: kafka | rabbitmq | redis-streams
```

| 항목 | 테스트 조건 |
|------|-----------|
| 이벤트 수 | MQ당 1,000건 |
| 동시 Producer | 10 threads |
| Consumer | 각 MQ별 SettlementListener + LedgerListener |
| 측정 | throughput, latency p50/p95/p99, 메시지 유실 여부 |

총 21개 파일을 생성/수정했다. Port 인터페이스 2개, MQ별 구현체 9개(3 MQ x 3 컴포넌트), 설정 클래스 3개, 테스트 3개, 기존 서비스 수정 4개.

---

## 결론

### PostgreSQL

batch write 경합에서 명확한 우위를 보인다. `FOR UPDATE SKIP LOCKED`는 MySQL에 없는 기능이며, 정산과 알림의 multi-row 갱신 시나리오에서 deadlock을 원천 차단한다. 정산/알림 트래픽이 현재의 10배 이상으로 증가하면 마이그레이션을 검토할 가치가 있다.

### DynamoDB

현재 규모에서는 과잉이다. MongoDB가 충분히 처리하고 있고, DynamoDB의 GSI 제약은 쿼리 유연성을 희생한다. 다만 어댑터가 준비되어 있으므로 serverless 스케일링이 필요한 시점에 즉시 전환 가능하다.

### MQ

Kafka가 기본값으로 유지된다. 이미 배포되어 있고 파티셔닝 기반 순서 보장이 정산에 적합하다. RabbitMQ는 DLQ가 중요한 신규 도메인에 고려할 수 있고, Redis Streams는 인프라 추가 없이 경량 이벤트 파이프라인이 필요할 때 선택지가 된다.

### Port/Adapter 패턴의 효과

이 모든 비교가 가능했던 이유는 Port/Adapter 패턴으로 Storage와 MQ를 추상화해 두었기 때문이다. 비즈니스 로직 변경 0, Property 한 줄 전환으로 벤더를 교체하고 동일 테스트를 돌릴 수 있었다. 최적화 시리즈의 마지막 단계로서, 현재 성능을 개선하는 것뿐 아니라 미래 확장 경로를 확보하는 작업이었다.

---

## 시리즈 탐색

**◀ 이전 글**
[검색 엔진 추상화 — ES vs MySQL FULLTEXT, 236배 차이의 기록](/search-engine-abstraction-es-vs-fulltext/)

**▶ 다음 글**
[React 프론트엔드 리팩토링 — memo 0건, 코드 스플리팅 0건에서 시작한 최적화](/react-frontend-optimization-memo-splitting/)
