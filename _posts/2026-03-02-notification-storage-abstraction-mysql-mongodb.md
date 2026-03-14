---
title: 알림 도메인 — Port/Adapter Storage 추상화, MySQL의 벽, MongoDB 1,061배 차이
date: 2026-03-02
tags: [Java, Spring, MongoDB, MySQL, 부하테스트, k6, 성능최적화, Port/Adapter, 헥사고날, 알림]
permalink: /notification-storage-abstraction-mysql-mongodb/
excerpt: "알림 도메인의 Storage/Delivery 계층을 Port/Adapter 패턴으로 추상화했다. property 한 줄로 MySQL↔MongoDB, SSE↔WebSocket↔FCM 전환이 가능하다. 단계별 스트레스 테스트로 MySQL의 성능 한계를 찾았고, 14M 데이터에서 MongoDB가 Write Storm p95 1,061배, 처리량 21배 차이를 보였다."
---

## 개요

[이전 글](/sse-notification-load-test-analysis/)에서 SSE 시나리오별 부하 테스트를 마쳤다. 개별 도메인 최적화가 끝난 시점에서 다음 과제는 Storage와 Delivery 계층을 교체 가능하게 만드는 것이었다. 채팅 도메인에서 `ChatMessageStoragePort` 추상화를 먼저 진행한 경험이 있었고, 알림 도메인에도 동일한 Port/Adapter 패턴을 적용했다.

추상화를 완료한 뒤에는 property 하나만 바꿔 MySQL과 MongoDB를 동일 시나리오로 비교할 수 있었다. 단계별로 VU와 데이터를 늘려가며 MySQL이 어디서 무너지는지 찾았고, 14M 데이터에서 MongoDB와의 차이를 측정했다.

---

## Part 1: Port/Adapter Architecture

### 3개의 Port 인터페이스

알림 도메인의 의존성을 세 가지 Port로 분리했다.

| Port | 메서드 수 | 역할 |
|------|----------|------|
| `NotificationStoragePort` | 8 | 알림 CRUD, 읽음 처리, 미읽음 카운트, 미전송 조회 |
| `NotificationDeliveryPort` | 3 | 실시간 전달 (connect, send, disconnect) |
| `NotificationEventPublisher` | 1 | 알림 생성 이벤트 발행 |

### Adapter 구현

```
NotificationStoragePort
├── MysqlNotificationStorageAdapter   (matchIfMissing=true, 기본값)
└── MongoNotificationStorageAdapter

NotificationDeliveryPort
├── SseNotificationDeliveryAdapter    (matchIfMissing=true, 기본값)
├── WebSocketNotificationDeliveryAdapter
└── FcmNotificationDeliveryAdapter

NotificationEventPublisher
└── RedisNotificationEventPublisher   (추상화 없음)
```

### @ConditionalOnProperty 전환

```yaml
# application.yml
app:
  notification:
    storage: mysql      # mysql | mongodb
    delivery: sse       # sse | websocket | fcm
```

```java
@Component
@ConditionalOnProperty(name = "app.notification.storage", havingValue = "mysql", matchIfMissing = true)
public class MysqlNotificationStorageAdapter implements NotificationStoragePort {
    // 기존 QueryDSL + Native Query 구현
}

@Component
@ConditionalOnProperty(name = "app.notification.storage", havingValue = "mongodb")
public class MongoNotificationStorageAdapter implements NotificationStoragePort {
    // MongoTemplate 기반 구현
}
```

`matchIfMissing=true`는 MySQL과 SSE에만 설정했다. property를 명시하지 않으면 기존 동작 그대로 MySQL + SSE로 뜬다.

### 변경 규모

| 구분 | 파일 수 |
|------|---------|
| 신규 파일 | 6 (Port 3 + Adapter 3) |
| 수정 파일 | 10 (Service, Controller, Config 등) |

서비스 계층(`NotificationCommandService`, `NotificationQueryService`, `SseStreamController` 등)이 `NotificationRepository`, `SseEmitterManager` 같은 구체 클래스 대신 Port 인터페이스를 주입받도록 변경했다.

### Pub/Sub은 왜 추상화하지 않았나

Redis Pub/Sub은 추상화 대상에서 제외했다. 이유는 채팅 도메인과 동일하다.

- 알림 이벤트 발행은 fire-and-forget 패턴이다. 메시지 영속성은 Storage가 담당한다.
- Redis Pub/Sub latency는 ~1ms로, Kafka(10~50ms)나 RabbitMQ로 전환할 실익이 없다.
- 발행 메서드가 `publish()` 하나뿐이라 추상화 비용 대비 이득이 없다.

### Multi-instance 지원: Redis SET 기반 연결 레지스트리

SSE 연결은 JVM 메모리에 `ConcurrentHashMap`으로 관리된다. 서버를 2대 이상 띄우면 유저가 어느 인스턴스에 연결되어 있는지 알 수 없다. 이를 위해 Redis SET으로 분산 연결 레지스트리를 구현했다.

```java
// SSE 연결 시
redisTemplate.opsForSet().add("sse:connections:" + userId, instanceId);

// 알림 발행 시 — 어느 인스턴스에 연결되어 있는지 조회
Set<String> instances = redisTemplate.opsForSet().members("sse:connections:" + userId);
```

단일 인스턴스에서는 로컬 `ConcurrentHashMap`으로 충분하고, 스케일아웃 시에만 Redis SET이 동작한다.

---

## Part 2: 단계별 스트레스 테스트 -- MySQL의 벽 찾기

Port/Adapter 추상화가 끝났으니, MySQL이 어느 지점에서 한계에 도달하는지 단계적으로 부하를 올려가며 측정했다. VU와 데이터를 함께 늘리는 것이 핵심이다. VU만 올리면 커넥션 경합, 데이터만 올리면 I/O 병목만 보인다. 둘 다 올려야 실제 운영 환경의 한계가 드러난다.

### Stage 1: 100VU, 5K 데이터

| 항목 | 결과 |
|------|------|
| 판정 | **12/12 PASS** |
| p95 | 전 구간 < 40ms |

워밍업 수준이다. 모든 API가 40ms 이하로 응답했다.

### Stage 2: 300VU, 571K 데이터

| 항목 | 결과 |
|------|------|
| 판정 | **22/22 PASS** |
| 처리량 | 875 req/s |

시나리오를 22개로 확장했다. 전 구간 통과. 아직 여유가 있다.

### Stage 3: 500VU, 1.02M 데이터

| 항목 | 결과 |
|------|------|
| 판정 | **16/16 PASS** |
| 처리량 | 1,283 req/s |

100만 건까지는 MySQL이 안정적이다.

### Stage 4: Endurance 테스트 -- 500VU, 1.41M 데이터, 80% 쓰기

쓰기 비율을 80%로 올린 내구성 테스트다.

| 항목 | 결과 |
|------|------|
| 판정 | **16/16 PASS** |
| Hot User p95 | 1.59s |

전 구간 통과했지만, 특정 유저에게 알림이 집중되는 Hot User 시나리오에서 p95가 1.59초까지 올라갔다. 경고 신호다.

### Stage 5: 14M 데이터 -- MySQL 붕괴

데이터를 14M으로 늘렸을 때 MySQL이 무너졌다.

| 항목 | 결과 |
|------|------|
| 판정 | **8/16 FAIL** |
| 처리량 | 1,090 → 79 req/s (**13.8배 하락**) |
| Write Storm p95 | 508ms → 10,190ms (**20배 악화**) |

| 시나리오 | Stage 4 (1.41M) | Stage 5 (14M) | 변화 |
|----------|-----------------|---------------|------|
| Write Storm p95 | 508ms | 10,190ms | 20배 |
| Hot User p95 | 1,590ms | 5,820ms | 3.7배 |
| 처리량 | 1,090 req/s | 79 req/s | 13.8배 하락 |
| 실패율 | 0% | 12.73% | -- |

### 근본 원인: markAllAsRead의 Row Lock 폭발

`markAllAsRead`가 14M 데이터에서 붕괴의 직접 원인이었다.

```sql
UPDATE notification SET is_read = true
WHERE user_id = ? AND is_read = false
```

유저당 미읽음 알림이 14,000건 이상이면 이 UPDATE가 14,000+ 행에 InnoDB X-lock을 건다. 같은 시점에 다른 트랜잭션이 해당 유저의 알림을 조회하거나 삽입하려 하면 lock wait가 발생한다. 500VU에서 여러 유저가 동시에 `markAllAsRead`를 호출하면 row lock contention이 연쇄적으로 확산된다.

여기에 buffer pool overflow가 겹친다. 14M 행의 테이블 크기가 buffer pool을 초과하면, UPDATE 대상 행을 디스크에서 읽어야 하고, 변경된 dirty page를 flush하는 과정에서 I/O 병목이 추가로 발생한다. lock을 잡은 채로 디스크 I/O를 기다리니 lock 보유 시간이 길어지고, 대기 중인 트랜잭션도 함께 느려지는 악순환이다.

---

## Part 3: MySQL vs MongoDB 비교 (14M 데이터)

Port/Adapter 추상화 덕분에 `app.notification.storage: mongodb`로 바꾸고 동일 시나리오를 실행했다.

### 전체 결과

| 항목 | MySQL | MongoDB | 차이 |
|------|-------|---------|------|
| 판정 | 8/16 FAIL | **16/16 ALL PASS** | -- |
| Write Storm p95 | 10,190ms | **9.61ms** | **1,061배** |
| Hot User p95 | 5,820ms | **12.08ms** | **482배** |
| 처리량 | 79 req/s | **1,686 req/s** | **21배** |
| 실패율 | 12.73% | **0.03%** | -- |

1,061배는 MySQL이 14M 데이터에서 markAllAsRead의 O(N) row lock으로 사실상 응답 불능 상태에 빠진 것과 MongoDB를 비교한 수치다. 양쪽 모두 정상 작동하는 Stage 4(500 VU, 1.41M)에서는 MySQL과 MongoDB 모두 16/16 PASS로 유의미한 차이가 없었다. 의미 있는 해석은 '1,061배 빠르다'가 아니라 'MySQL은 14M 이상의 markAllAsRead 부하에서 구조적 한계에 도달한다'는 것이다.

### markAllAsRead: 왜 이렇게 차이가 나는가

| | MySQL | MongoDB |
|---|-------|---------|
| 실행 방식 | `UPDATE ... WHERE user_id=? AND is_read=false` | `updateMulti(query, update)` |
| Lock 모델 | 14,000+ 행에 개별 row lock | Document-level lock, 단일 명령 |
| 동시성 영향 | 같은 유저의 다른 쿼리 전부 대기 | 다른 document 접근에 영향 없음 |

MySQL의 `UPDATE`는 조건에 매칭되는 모든 행에 하나씩 X-lock을 건다. 14,000행이면 14,000개의 lock이다. 이 lock이 풀릴 때까지 같은 유저의 SELECT, INSERT 모두 대기한다.

MongoDB의 `updateMulti`는 내부적으로 각 document를 순차 갱신하지만, document 단위로 lock을 짧게 잡고 놓는다. 전체 범위를 한 번에 잠그는 것이 아니라 건별로 짧은 lock을 반복하기 때문에, 다른 연산이 끼어들 수 있고 contention이 거의 발생하지 않는다.

### MongoDB 스케일 테스트: 600VU

MongoDB가 14M에서 안정적이었으므로 VU를 추가로 올려 테스트했다.

| 항목 | 500VU | 600VU |
|------|-------|-------|
| 판정 | 16/16 PASS | **16/16 PASS** |
| 처리량 | 1,686 req/s | **2,471 req/s** |

600VU에서도 전 구간 통과. 처리량이 VU에 비례해 증가했다. MySQL이 79 req/s로 붕괴한 동일 데이터에서 MongoDB는 2,471 req/s를 처리했다.

---

## 정리하며

**Port/Adapter 추상화가 비교를 가능하게 했다.** property 한 줄 변경으로 동일 시나리오를 MySQL과 MongoDB에서 실행할 수 있었다. 추상화 없이 이 비교를 하려면 서비스 코드를 전부 고쳐야 했을 것이다. 6개 신규 파일과 10개 수정으로 Storage/Delivery 계층 전체가 교체 가능해졌다.

**MySQL은 ~1M 알림까지 안정적이다.** 100VU~500VU, 5K~1.41M 데이터 범위에서 16/16 전 구간 통과했다. 대부분의 서비스에서 알림이 수백만 건을 넘기는 일은 드물다. 일반적인 규모에서는 MySQL로 충분하다.

**14M 데이터에서 MySQL이 무너지는 정확한 지점은 `markAllAsRead`다.** 유저당 14,000+ 미읽음 알림에 UPDATE를 걸면 row lock contention + buffer pool overflow로 처리량이 13.8배 하락한다. 전체 시스템이 아니라 이 단일 연산이 병목이다. 쓰기 비율이 높은 알림 시스템에서 데이터가 수백만 건을 넘길 때 MongoDB 전환을 검토해야 한다.

> **MySQL의 성능 한계는 데이터 규모와 쓰기 패턴의 조합으로 결정된다.** 읽기 위주라면 14M에서도 버틸 수 있지만, markAllAsRead 같은 대량 UPDATE가 빈번하면 1M을 넘기는 시점부터 경고 신호가 나타난다.

---

## 시리즈 탐색

**◀ 이전 글**
[멀티 도메인 리팩토링 — ErrorCode 분리부터 29파일 코드 품질 정리까지](/multi-domain-refactoring-errorcode-code-quality/)

**▶ 다음 글**
[피드 도메인 — Storage 추상화, Lua Script 분석, 3-Backend 비교 테스트](/feed-storage-abstraction-lua-script-backend-comparison/)
