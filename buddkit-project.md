---
layout: page
title: "Buddkit 프로젝트 — 버그 60건 수정부터 프로덕션 하드닝까지"
permalink: /buddkit-project/
redirect_from:
  - /onlyone-project/
---

## 프로젝트 소개

Buddkit은 **모임 기반 소셜 플랫폼**이다. 모임 생성/가입, 피드, 채팅, 스케줄, 정산/결제, 알림, 검색 기능을 갖춘 Spring Boot 백엔드 프로젝트로, 부트캠프에서 5인 팀으로 개발했다. 나는 **알림(SSE) 도메인**을 맡았다. (원본 레포 이름은 `OnlyOne-Back`)

프로젝트가 끝난 뒤, 내가 다루지 못한 Redis·Kafka·Elasticsearch 등을 공부하려고 다른 팀원들의 코드를 읽기 시작했다. 그 과정에서 6개 도메인에 걸쳐 약 60건의 버그를 발견했고, 고치는 것에서 끝나지 않고 부하 테스트 → 리팩토링 → AWS 배포 → 프로덕션 하드닝까지 이어졌다. 이 시리즈는 그 전체 과정의 기록이다.

**기술 스택:** Spring Boot 3, JPA/Hibernate, MySQL 8, Redis (Lua Script, Stream, Pub/Sub), Kafka, MongoDB, Elasticsearch (nori), WebSocket/STOMP, SSE, Docker, k6, Prometheus/Grafana, Chaos Monkey, Unleash

원본 레포: [GoormOnlyOne/OnlyOne-Back](https://github.com/choigpt/OnlyOne-Back-Original)
최종 레포: [choigpt/OnlyOne-Back (feat/notification/haechang)](https://github.com/choigpt/OnlyOne-Back/tree/feat/notification/haechang)

---

## 시스템 아키텍처

<div class="mermaid">
flowchart LR
    subgraph CLIENT["Client"]
        WEB["React Web<br/>(React Query)"]
        MOB["Mobile"]
    end

    subgraph EXT["External"]
        KAKAO["Kakao OAuth"]
        TOSS["Toss Payments"]
    end

    subgraph AWS["AWS EC2 (t3.medium / t3.large)"]
        subgraph APP["Spring Boot 3 API (onlyone-api)"]
            direction TB
            FILTER["JWT Filter<br/>+ SecurityFilterChain"]
            subgraph DOMAIN["Domain Modules"]
                direction LR
                D1["user / club / feed"]
                D2["chat / schedule"]
                D3["wallet / payment<br/>settlement"]
                D4["notification / search"]
            end
            subgraph PROTO["Realtime"]
                WS["WebSocket / STOMP<br/>(Chat)"]
                SSE["SSE Emitter<br/>(Notification)"]
            end
            FF["Unleash Client<br/>(A/B, Kill-Switch)"]
            ACT["Actuator + Micrometer"]
        end

        subgraph DATA["Data Layer"]
            direction TB
            MYSQL[("MySQL 8<br/>InnoDB BP 512M<br/>R/W pool 분리")]
            REDIS[("Redis 7<br/>Cache · Pub/Sub<br/>Streams · Lua")]
            MONGO[("MongoDB 7<br/>Notification 대체")]
            ES[("Elasticsearch 8<br/>nori analyzer<br/>(Club 검색)")]
            KAFKA[["Kafka (KRaft)<br/>Settlement Outbox"]]
        end

        subgraph OBS["Observability / Hardening"]
            PROM["Prometheus<br/>+ Exporters"]
            GRAF["Grafana<br/>(11 Alerts)"]
            CHAOS["Chaos Monkey<br/>for Spring Boot"]
            UNLEASH["Unleash Server"]
        end
    end

    WEB -->|HTTPS<br/>JWT| FILTER
    MOB -->|HTTPS<br/>JWT| FILTER
    WEB <-.->|WSS| WS
    WEB <-.->|EventStream| SSE

    FILTER --> DOMAIN
    DOMAIN --> WS
    DOMAIN --> SSE

    DOMAIN -->|JPA/QueryDSL| MYSQL
    DOMAIN -->|RedisTemplate<br/>Lua Script| REDIS
    DOMAIN -.->|Port/Adapter| MONGO
    DOMAIN -.->|Port/Adapter| ES
    DOMAIN -->|Outbox Producer| KAFKA
    KAFKA -->|@KafkaListener<br/>정산 처리| DOMAIN

    DOMAIN -->|OAuth2| KAKAO
    DOMAIN -->|결제 승인<br/>타임아웃 + 재시도| TOSS

    FF --> UNLEASH
    ACT -->|/actuator/prometheus| PROM
    PROM --> GRAF
    CHAOS -.->|장애 주입| APP

    classDef client fill:#E3F2FD,stroke:#1976D2
    classDef ext fill:#FFF3E0,stroke:#E65100
    classDef app fill:#E8F5E9,stroke:#2E7D32
    classDef data fill:#F3E5F5,stroke:#6A1B9A
    classDef obs fill:#FCE4EC,stroke:#AD1457
    class WEB,MOB client
    class KAKAO,TOSS ext
    class FILTER,D1,D2,D3,D4,WS,SSE,FF,ACT app
    class MYSQL,REDIS,MONGO,ES,KAFKA data
    class PROM,GRAF,CHAOS,UNLEASH obs
</div>

**핵심 포인트**

- **Storage 추상화 (Port/Adapter):** 알림·채팅·피드·검색은 인터페이스 뒤에서 백엔드를 교체할 수 있다. 알림은 MySQL ↔ MongoDB, 검색은 ES ↔ MySQL FULLTEXT, 채팅은 MySQL ↔ MongoDB로 부하 테스트 비교를 했다 (Phase 3).
- **Realtime은 두 채널:** 채팅은 WebSocket/STOMP, 알림은 SSE Emitter. 인증 필터는 SseAuthenticationFilter를 제거하고 단일 JwtAuthenticationFilter로 통합했다 (#3).
- **Kafka는 정산 Outbox 전용:** 결제 도메인 → Kafka → @KafkaListener 정산 처리. 비재시도 예외(BusinessException)를 등록해 무한 retry를 막았다 (#15).
- **결제 외부 호출:** Toss는 타임아웃 + 보상 트랜잭션이 적용된 경계. 승인 후 DB 실패로 자금이 유실되던 버그를 보상 트랜잭션으로 해결했다 (#8).
- **운영:** Prometheus 11개 알림 룰, Unleash A/B 플래그, Chaos Monkey 장애 주입까지 포함된 프로덕션 하드닝 (#36).

---

## 도메인 관계도

<div class="mermaid">
erDiagram
    User ||--o{ UserClub : "가입"
    Club ||--o{ UserClub : "멤버"
    Club ||--o{ Feed : "소속"
    Club ||--o{ Schedule : "일정"
    Club ||--o{ ChatRoom : "채팅방"
    Feed ||--o{ FeedLike : "좋아요"
    Feed ||--o{ FeedComment : "댓글"
    Schedule ||--o{ UserSettlement : "참여/정산"
    UserSettlement }o--|| Settlement : "정산 단위"
    Settlement }o--|| Wallet : "자금"
    Wallet ||--|| User : "1:1"
    ChatRoom ||--o{ Message : "메시지"
    ChatRoom ||--o{ UserChatRoom : "참여"
    Notification }o--|| User : "수신"
</div>

**Notification**은 모든 도메인의 이벤트(댓글, 리피드, 모임 가입 등)를 수신하는 독립 도메인이다. **Search**는 Elasticsearch에 Club을 인덱싱하여 검색을 제공한다.

---

## 전체 성과

| 지표 | Before | After |
|------|--------|-------|
| 버그 | ~60건 미발견 | Phase 1에서 전수 수정 |
| 피드 슬로우 쿼리 | 29,008건 | 69건 (-99.8%) |
| 알림 API 성공률 | 13.6% | 99.9% |
| 알림 목록 p95 | 59,801ms | 21ms |
| 정산 p95 | 19,500ms | 2,460ms |
| 검색 쿼리 | 37ms (단건) | 0.6ms (62배) |
| 채팅 슬로우 쿼리 | 827건 | 46건 |
| EC2 인스턴스 | c5.xlarge + c5.2xlarge | t3.medium + t3.large (비용 70% 절감) |
| 보안/운영 이슈 | 미점검 | 115건 식별, 48건 수정 (Phase 5) |

---

## Phase 1 — 버그 발견 (6개 도메인, ~60건)

팀 프로젝트가 끝나고 다른 팀원의 코드를 읽기 시작했다. Redis, Kafka, Elasticsearch를 공부하려는 목적이었는데, 읽다 보니 버그가 보이기 시작했다.

| # | 글 | 도메인 | 핵심 |
|---|---|--------|------|
| 1 | [Spring Security + JWT 구조 버그](/spring-security-jwt-structure-bugs/) | 인증 | JwtException 패키지 불일치 → 500, jjwt 이중 선언, charset 불일치 |
| 2 | [JWT 인증 흐름 불일치로 SSE 인증 실패](/jwt-sse-auth-filter-mismatch/) | 인증 | subject/kakaoId 불일치 → SSE 전면 불통 |
| 3 | [SseAuthenticationFilter 제거 — 필터 단일화](/sse-auth-filter-removal/) | 인증 | 두 필터 구조 → 하나로 통합 |
| 4 | [알림 SSE 연쇄 버그와 구조 개선](/sse-notification-chained-bugs/) | 알림 | sendEvent 반환값 무시 + 복구 쿼리 조건 불일치 → 알림 유실 |
| 5 | [피드 — Redis 좋아요 파이프라인의 허점](/feed-redis-like-n-plus-1/) | 피드 | 좋아요 수 3곳 불일치, 컬렉션 전체 로딩, "친구의 친구" 범위 폭발 |
| 6 | [모임 — 동시성 구멍과 유령 멤버](/club-concurrency-ghost-members/) | 모임 | memberCount Lost Update, Unique 제약 없음, 탈퇴 시 유령 멤버 |
| 7 | [스케줄 — 권한 체크 순서가 자금 동결로](/schedule-permission-fund-freeze/) | 스케줄 | deleteSchedule 지갑 홀드 미해제, joinSchedule Race Condition |
| 8 | [지갑·결제 — 토스 승인 후 DB 실패 시 자금 유실](/wallet-payment-toss-db-failure/) | 결제 | 보상 트랜잭션 없음, 잔액 Race Condition, reportFail self-invocation |
| 9 | [유저 — 인증 이후 생명주기에서 무너진 것들](/user-lifecycle-bugs/) | 유저 | getCurrentUserId()가 kakaoId 반환 → 14개 서비스 오염 |

---

## Phase 2 — 로컬 부하 테스트 + 성능 개선

버그를 고치면서 N+1, 풀스캔, 컬렉션 전체 로딩 같은 성능 문제도 함께 보였다. 코드만 보면 "느릴 것 같다"는 감이 오는데, 실제로 얼마나 느린지 측정해봐야 했다. k6로 부하를 걸기 시작했다.

| # | 글 | 도메인 | 핵심 |
|---|---|--------|------|
| 10 | [채팅 — 상관 서브쿼리·N+1 성능 저하](/chat-subquery-redis-listener/) | 채팅 | 상관 서브쿼리 → GROUP BY+JOIN, findLatest JOIN FETCH |
| 11 | [알림 API — 13% → 100%, 9번의 시도](/notification-api-performance-improvement/) | 알림 | QueryDSL JOIN 제거, 인덱스 재구성, 비동기 분리, 40M→47만 행 |
| 12 | [피드 — 슬로우 쿼리 29,008건 → 전 구간 통과](/feed-performance-load-test/) | 피드 | 인덱스 추가, likeCount 활용, IN절 청크, 커버링 인덱스, 캐싱 |
| 13 | [SSE 알림 — 7가지 시나리오 부하 테스트](/sse-notification-load-test-analysis/) | 알림 | SSE 7,500 VU 안정, REST 혼합 시 16% 성공률 |
| 14 | [Finance — Mock 도입 후 detached entity 버그 발견](/finance-domain-load-test-mock-exposed-hidden-bug/) | 결제 | MockTossPaymentClient로 Phase 3 최초 실행, clearAutomatically 버그 |
| 15 | [정산 — p95 19.5초 → 2.46초](/finance-settlement-batch-kafka-tuning/) | 정산 | 배치 UPDATE, Kafka 비재시도 예외 등록 |
| 16 | [피드 고부하 — FORCE INDEX 실패부터 인메모리 캐싱까지](/feed-highload-force-index-caching-evolution/) | 피드 | UNION ALL 청크, 캐싱 3단 진화, warmup 비동기화 |
| 17 | [클럽 — Virtual Thread Pinning JVM 크래시](/club-load-test-virtual-thread-pinning/) | 모임 | VT+JDBC synchronized → carrier thread 고갈 |
| 18 | [검색 — teammates_clubs 쿼리 62배 개선](/search-load-test-not-exists-optimization/) | 검색 | NOT IN → NOT EXISTS, club JOIN 제거, DISTINCT로 filesort 제거 |

---

## Phase 3 — 리팩토링 + Storage 추상화

성능은 잡았지만 코드 구조가 아직 거칠었다. 288줄짜리 서비스, 네이티브 쿼리, try-catch 이중 처리 같은 것들을 정리했다. 그리고 "MySQL 말고 MongoDB를 쓰면 어떨까?"라는 질문에 답하기 위해 Storage를 교체 가능한 구조로 만들었다.

| # | 글 | 도메인 | 핵심 |
|---|---|--------|------|
| 19 | [채팅 리팩토링 — WebSocket 핸들러부터 커서 페이징까지](/chat-domain-deep-refactoring/) | 채팅 | MAX(sent_at)→MAX(messageId), try-catch 이중 제거, 35줄→18줄 |
| 20 | [피드 리팩토링 — 서비스 책임 분리와 네이티브 쿼리 전환](/feed-domain-442-line-service-split/) | 피드 | FeedCacheService·FeedRenderService 분리, QueryDSL 전환 |
| 21 | [멀티 도메인 코드 정리 — Semaphore 제거까지](/multi-domain-cleanup-surface-to-semaphore/) | 전체 | existsById 교체, Redis 카운터 분리, Semaphore(30) 제거 |
| 22 | [멀티 도메인 리팩토링 — ErrorCode 분리와 코드 품질](/multi-domain-refactoring-errorcode-code-quality/) | 전체 | ErrorCode 도메인별 분리, 30파일 정리 |
| 23 | [채팅 — Storage 추상화, WebSocket 1,000VU](/chat-storage-websocket-extreme-test/) | 채팅 | Port/Adapter MySQL↔MongoDB, WHERE IN 풀스캔 해결 |
| 24 | [알림 — Port/Adapter Storage 추상화, MySQL vs MongoDB](/notification-storage-abstraction-mysql-mongodb/) | 알림 | MySQL↔MongoDB 전환, 14M Write Storm에서 p95 1,061배 차이 |
| 25 | [피드 — Storage 추상화, Lua Script, 3-Backend 비교](/feed-storage-abstraction-lua-script-backend-comparison/) | 피드 | MySQL↔MongoDB↔Hybrid, Lua 스크립트 분석 |
| 26 | [검색 엔진 추상화 — ES vs MySQL FULLTEXT, 236배](/search-engine-abstraction-es-vs-fulltext/) | 검색 | ES↔MySQL FULLTEXT 전환, ngram 토큰 분석 |

---

## Phase 4 — AWS EC2 부하 테스트

로컬에서 잘 돌아가는 코드가 EC2에서도 잘 돌아갈까? c5.xlarge로 시작해 t3.medium까지 다운사이징하면서 각 도메인을 재테스트했다. 로컬에서는 보이지 않던 병목(Buffer Pool 부족, 커널 파라미터, ClassLoader lock)이 드러났다.

| # | 글 | 도메인 | 핵심 |
|---|---|--------|------|
| 27 | [알림 EC2 — MySQL write 붕괴와 MongoDB 비교](/notification-aws-ec2-load-test/) | 알림 | MySQL mark p50 29s 붕괴, MongoDB 6,000VU 100% 통과 |
| 28 | [채팅 EC2 — TX 분리, VT Pinning, 4,500VU](/chat-aws-ec2-load-test/) | 채팅 | write-pool 24.7s→1.16s, 4,500VU 전 Phase 100% |
| 29 | [피드 EC2 — IN절 병목, covering index, X-lock 해소](/feed-aws-ec2-load-test/) | 피드 | covering index, comment_count 배치 버퍼링, 3,000VU |
| 30 | [정산 EC2 — 시드 데이터 버그와 5라운드 최적화](/settlement-aws-ec2-load-test/) | 정산 | @Version NULL, pending_out=0, 배치 TX 통합 |
| 31 | [검색 EC2 — MySQL FULLTEXT ngram 성능 한계](/search-aws-ec2-load-test/) | 검색 | 시드 이중 인코딩, FULLTEXT p95 24s |
| 32 | [EC2 중간 점검 — 5개 도메인 인프라 진단](/aws-ec2-load-test-midpoint/) | 전체 | 로컬 vs EC2 병목 차이, Buffer Pool 9.4% |
| 33 | [EC2 다운사이징 (상) — 피드·채팅·알림](/ec2-downsizing-optimization-part1/) | 피드/채팅/알림 | 댓글 비동기 카운터, Redis Streams, 워터마크 |
| 34 | [EC2 다운사이징 (중) — 정산·검색·인프라](/ec2-downsizing-optimization-part2/) | 정산/검색 | R/W 풀 분리 역효과, JWT ClassLoader lock |
| 35 | [EC2 다운사이징 (하) — 전체 통합 결과](/ec2-downsizing-optimization-part3/) | 전체 | 통합 테스트, 코드 리뷰 |

---

## Phase 5 — 프로덕션 하드닝 + 카오스 엔지니어링

성능은 괜찮은데, 이 코드를 프로덕션에 배포해도 되는가? Phase 4까지는 "개발자" 한 가지 관점이었다. DBA·백엔드·성능·DevOps·유지보수 5개 역할의 관점으로 전수 분석했다.

| # | 글 | 도메인 | 핵심 |
|---|---|--------|------|
| 36 | [프로덕션 하드닝 — 48건 수정, Chaos Monkey, A/B 테스트](/production-hardening-chaos-engineering/) | 전체 | 115건 식별 → 48건 수정. Cascade 삭제 방지, JWT 블랙리스트, Toss 타임아웃, Dockerfile non-root, Correlation ID, Prometheus 알림 11개, Unleash A/B, Chaos Monkey 장애 주입 |

---

## 부록

| # | 글 | 핵심 |
|---|---|------|
| A1 | [React 코드 품질 — C→B, React Query 도입](/react-code-quality-c-to-b-react-query/) | 프론트엔드 코드 품질 68→82점 |
| A2 | [React 심화 — 토큰 레이스, SSE 누수, 보안 버그](/react-deep-review-token-race-sse-leak/) | 프론트엔드 버그 리뷰 |
| A3 | [React 리팩토링 — memo, 코드 스플리팅](/react-frontend-optimization-memo-splitting/) | 프론트엔드 최적화 |
| A4 | [k6 부하 테스트 코드 리뷰 — 10가지 함정](/k6-load-test-code-review/) | 테스트 코드 자체의 함정과 수정 |

---

## 도메인별 탐색

한 도메인의 글을 처음부터 끝까지 이어서 읽고 싶다면:

| 도메인 | Phase 1 (버그) | Phase 2 (성능) | Phase 3 (리팩토링) | Phase 4 (EC2) | Phase 5 |
|--------|---------------|---------------|-------------------|--------------|---------|
| **인증** | [#1](/spring-security-jwt-structure-bugs/) [#2](/jwt-sse-auth-filter-mismatch/) [#3](/sse-auth-filter-removal/) | | | | [#36](/production-hardening-chaos-engineering/) |
| **알림** | [#4](/sse-notification-chained-bugs/) | [#11](/notification-api-performance-improvement/) [#13](/sse-notification-load-test-analysis/) | [#24](/notification-storage-abstraction-mysql-mongodb/) | [#27](/notification-aws-ec2-load-test/) [#33](/ec2-downsizing-optimization-part1/) | [#36](/production-hardening-chaos-engineering/) |
| **피드** | [#5](/feed-redis-like-n-plus-1/) | [#12](/feed-performance-load-test/) [#16](/feed-highload-force-index-caching-evolution/) | [#20](/feed-domain-442-line-service-split/) [#25](/feed-storage-abstraction-lua-script-backend-comparison/) | [#29](/feed-aws-ec2-load-test/) [#33](/ec2-downsizing-optimization-part1/) | [#36](/production-hardening-chaos-engineering/) |
| **모임** | [#6](/club-concurrency-ghost-members/) | [#17](/club-load-test-virtual-thread-pinning/) | | | [#36](/production-hardening-chaos-engineering/) |
| **스케줄/정산** | [#7](/schedule-permission-fund-freeze/) [#8](/wallet-payment-toss-db-failure/) | [#14](/finance-domain-load-test-mock-exposed-hidden-bug/) [#15](/finance-settlement-batch-kafka-tuning/) | | [#30](/settlement-aws-ec2-load-test/) [#34](/ec2-downsizing-optimization-part2/) | [#36](/production-hardening-chaos-engineering/) |
| **채팅** | | [#10](/chat-subquery-redis-listener/) | [#19](/chat-domain-deep-refactoring/) [#23](/chat-storage-websocket-extreme-test/) | [#28](/chat-aws-ec2-load-test/) [#33](/ec2-downsizing-optimization-part1/) | |
| **검색** | | [#18](/search-load-test-not-exists-optimization/) | [#26](/search-engine-abstraction-es-vs-fulltext/) | [#31](/search-aws-ec2-load-test/) [#34](/ec2-downsizing-optimization-part2/) | |
| **유저** | [#9](/user-lifecycle-bugs/) | | | | |
| **전체** | | | [#21](/multi-domain-cleanup-surface-to-semaphore/) [#22](/multi-domain-refactoring-errorcode-code-quality/) | [#32](/aws-ec2-load-test-midpoint/) [#35](/ec2-downsizing-optimization-part3/) | [#36](/production-hardening-chaos-engineering/) |
