---
title: EC2 다운사이징 후 최적화 (중) — 정산·검색 도메인과 공통 인프라 튜닝
date: 2026-03-11
tags: [AWS, EC2, MySQL, Elasticsearch, Kafka, HikariCP, k6, 성능최적화, 부하테스트, Java, ZGC, VirtualThread, 정산, 검색]
permalink: /ec2-downsizing-optimization-part2/
excerpt: "정산 도메인에서 R/W 풀 분리가 단일 MySQL에서 역효과를 일으킨 원인을 분석하고, 검색 도메인에서 ES vs MySQL FULLTEXT 830배 차이를 확인했다. JWT Parser ClassLoader lock(BLOCKED 388→0), 커널 TCP 튜닝(SSE p95 708→90ms), EC2 컨테이너 메모리 할당까지 공통 인프라 튜닝을 정리한다."
---

---

## 1. 정산 도메인

[이전 정산 EC2 테스트](/settlement-aws-ec2-load-test/)에서 배치 TX 통합과 5라운드 최적화를 다뤘다. 다운사이징 후 N+1 제거, R/W 풀 분리 역효과 분석을 추가로 수행했다.

### 1.1 최적화 내용

- **배치 TX 통합**: 참가자별 독립 TX를 정산당 1 TX로 전환(8TX x 5쿼리 → 1TX x 12쿼리). 750건 기준 2분에서 31.7초로 개선.
- **배치 조회 쿼리**: IN절 배치(WalletIds, UserSettlementIds)로 SELECT N+1 제거.
- **DB 인덱스 3종**: settlement(schedule_id), user_settlement(settlement_id, user_id), wallet_transaction(wallet_id, status, created_at) 추가로 full scan을 index scan으로 전환.
- **N+1 제거**: WalletTransaction → Payment JOIN FETCH로 추가 20개 쿼리 제거.
- **Kafka + Outbox 직접**: RabbitMQ/Redis Streams/Spring Events 어댑터 제거하여 코드 단순화.
- **Outbox 처리량 증가**: 200ms/200건에서 50ms/500건으로 처리 속도 5배 증가.
- **Stuck 복구 스케줄러**: 5분마다 IN_PROGRESS 5분 초과 건을 FAILED 처리하여 정산 hang 방지.

### 1.2 R/W 풀 분리 — 단일 MySQL에서 역효과

지갑 조회의 경우 단일 풀 400 p95는 1.10s였으나, R/W 분리(350+350) p95는 8.23s로 648% 악화되었다.

원인:
- write-350 + read-350 = 총 700 커넥션 → 단일 MySQL max_connections 초과
- `wallet_query_storm` (450 VUs, readOnly) → read pool 350개에 몰림 → 100 VU 대기
- 단일 풀 400에서는 50 VU만 대기 → R/W 분리로 대기 VU 2배

모니터링 데이터에서 확인한 진단:

모니터링 데이터로 확인한 결과, 유휴 시 threads_running 3, slow_queries +0/분, HikariCP pending 0이었지만, 부하 중에는 threads_running 349, slow_queries +100/분, HikariCP pending 238까지 치솟았다.

**결론**: 같은 쿼리가 유휴 시 빠르고 부하 시 느리면 → 인스턴스 한계 (자원 경합). 단일 MySQL에서 R/W 분리는 커넥션 수만 늘려 부하 증가. read replica 추가 시에만 유효.

### 1.3 Redis 사용 현황

Finance 도메인은 데이터 캐싱을 사용하지 않는다. Redis는 동시성 제어 전용:

- **결제 멱등성 게이트**: 키 패턴 `payment:gate:{orderId}`, TTL 5분
- **결제 정보 임시 저장**: 키 패턴 `payment:{orderId}`, TTL 30분
- **지갑 분산 락**: 키 패턴 `wallet:gate:{userId}:{op}`, TTL 10초

### 1.4 최종 결과

Round 1에서 Round 5로 정산 요청 p95는 18.64s에서 1.50s로, 지갑 조회 p95는 14.95s에서 1.24s로 개선되었다. 5xx Stress는 2.74%에서 0.00%로, Lock wait timeout은 175+건에서 0건으로, RPS는 600 VU 기준 약 550에서 약 2,000으로 향상되었다.

28 PASS / 0 FAIL, 전 Phase 성공률 100%, 5xx 0%.

---

## 2. 검색 도메인

[이전 검색 EC2 테스트](/search-aws-ec2-load-test/)에서 MySQL FULLTEXT ngram의 한계를 다뤘다.

### 2.1 Port/Adapter 추상화

`SearchPort` → ES 어댑터 / MySQL FULLTEXT 어댑터. 설정으로 교체 가능.

- MySQL FULLTEXT: ngram parser (token_size=2), `ApplicationReadyEvent`에서 자동 인덱스 생성
- ES: nori 토크나이저 (한국어 형태소 분석)

### 2.2 ES vs MySQL FULLTEXT 비교

ES와 MySQL FULLTEXT를 비교한 결과, 단일 키워드 p95는 ES 29ms / MySQL 24.11s, 스파이크 p95는 ES 261ms / MySQL 45.40s, 5xx 에러는 ES 0건 / MySQL 29,966건, 처리량은 ES 1,029K req / MySQL 366K req로 ES가 압도적이었다.

**채택: ES** — 830배 차이. MySQL FULLTEXT는 ES 불가 시 폴백용으로 유지.

---

## 3. 공통 인프라 튜닝

### 3.1 JVM 설정

```
-Xms2g -Xmx2g                          # 4GB RAM에서 2g가 한계 (3g는 OOM)
-XX:+UseZGC -XX:+ZGenerational          # 저지연 GC
-XX:MaxDirectMemorySize=256m            # Netty/Lettuce 버퍼
-XX:+AlwaysPreTouch                     # 힙 사전할당 (GC stall 감소)
-Djdk.virtualThreadScheduler.parallelism=8
```

Virtual Threads `spring.threads.virtual.enabled: true` 활성화. JFR continuous, 500MB max.

### 3.2 JWT Parser 싱글턴 캐싱 — BLOCKED 388 → 0

9000 VU 알림 테스트 중 스레드 덤프에서 161~388개 BLOCKED 스레드 발견:

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

// After: @PostConstruct에서 1회 생성, 필드 캐싱 (JwtParser는 thread-safe)
private JwtParser jwtParser;

@PostConstruct
void init() {
    jwtParser = Jwts.parser().setSigningKey(key).build();
}
```

### 3.3 커널 튜닝 (EC2)

```
net.core.rmem_max = 16MB                # SSE/WebSocket 수신 버퍼
net.core.wmem_max = 16MB                # 송신 버퍼
net.ipv4.tcp_keepalive_time = 60        # 유휴 연결 빠른 감지
net.ipv4.tcp_slow_start_after_idle = 0  # 재사용 커넥션 cwnd 유지
net.ipv4.tcp_fin_timeout = 15           # TIME_WAIT 빠른 회수
net.core.somaxconn = 65535              # listen backlog 확대
```

효과: SSE 연결 p95 708ms → 90ms, 전체 처리량 20% 증가.

### 3.4 Virtual Thread 피닝

- `SseEmitter.send()` — Spring 내부 `synchronized` 블록 → Virtual Thread pinning
- MongoDB 드라이버 — `synchronized` 연결 풀
- 권장: `ReentrantLock` 전환 (Spring 6.2+에서 일부 해결)

### 3.5 @Cacheable 직렬화 이슈

`GenericJackson2JsonRedisSerializer` + `activateDefaultTyping(NON_FINAL)` 환경에서:

- `PageImpl` — 기본 생성자 없음 → 역직렬화 실패
- Java `record` 타입 — compact constructor와 DefaultTyping 호환 불가

해결: `@Cacheable` 대신 커스텀 캐시 클래스 + `StringRedisTemplate` + 수동 JSON 직렬화.

### 3.6 SecurityContext userId 추출

```java
// Before: DB SELECT 1회
User user = userService.getCurrentUser();
// After: JWT에서 파싱된 userId, DB 0회
Long userId = userService.getCurrentUserId();
```

모든 도메인에 적용 — 요청당 DB 쿼리 1건 절약.

> 이것은 성능 최적화라기보다 잘못된 구현의 정정이다. Spring Security에서 인증 완료 후 Principal에 저장된 정보를 활용하는 것은 기본 패턴이며, 매 요청마다 DB 조회를 하던 기존 방식이 비정상이었다.

---

---

## 시리즈 탐색

**◀ 이전 글**
[EC2 다운사이징 후 최적화 (상) — 피드·채팅·알림 도메인 재테스트](/ec2-downsizing-optimization-part1/)

**▶ 다음 글**
[EC2 다운사이징 후 최적화 (하) — 전체 통합 결과와 코드 리뷰](/ec2-downsizing-optimization-part3/)