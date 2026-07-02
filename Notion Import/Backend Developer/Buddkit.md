---
title: "Buddkit"
source: "Notion local cache"
notion_url: "https://app.notion.com/p/358779d1d7da804bbd05f6b250c4987a"
imported_at: "2026-07-02"
tags:
  - notion-import
  - backend-developer
---

# Buddkit

## 목차

  - Buddkit 정리 기준
  - 관련 링크
  - 운영 원칙
  - 📍 배경
  - 🎯 본인 역할
  - ✨ Highlights
  - 🏗 시스템 아키텍처
  - 🔍 Case Studies
  - 📊 전체 성과 요약
  - ⚠️ 한계와 향후 검증
  - 🛠 Tech Stack
  - 🧩 도메인별 최종 스택 결정

## Buddkit 정리 기준

이 페이지는 Buddkit 프로젝트의 대표 페이지로 사용합니다.

## 관련 링크

- [[장애 분석|장애 분석]]
- [ADsP iPad 앱 502 분석](https://app.notion.com/p/390779d1d7da8134abd0cd73f2a196ad)
- [[Backend|Backend]]

## 운영 원칙

메인 포트폴리오에는 Buddkit 요약만 남기고, 상세 작업과 사례는 이 페이지 또는 하위 사례 페이지로 연결합니다.

> 부트캠프 종료 후 5주, 8개 도메인을 운영 가능 상태로 끌어올린 백엔드 리팩토링 프로젝트

> **에러율 9.02% → 0.17%**, **EC2 비용 70% 절감**, **댓글 생성 p95 5.75s → 260ms**

---

| 구분 | 내용 |
| --- | --- |
| 역할 | Backend |
| 담당 도메인 | 알림(SSE) |
| 팀 프로젝트 | 2025.07 ~ 08, 부트캠프 5인 팀 |
| 개인 리팩토링 | 2026.02 ~ 03, 단독 수행 |
| 검토 도메인 | User, Feed, Chat, Notification, Settlement, Search, Wallet |

---

> 📈
> **At a glance**
>
> 8개 도메인 개선 · **60+ 버그 수정** · **댓글 생성 p95 5.75s → 260ms (95%)** · **채팅 에러 81,911 → 0** · **EC2 비용 70% 절감** · **GC Stall 44회/108.6s → 0** · 41편 작업 일지 · 5주
>
> **측정 규모** — DB ~124M rows (도메인당 10M+, Feed 10.6M, Chat 13M) · max 9,000 VU · k6 기반 Phase 시나리오
>

---

## 📍 배경

부트캠프 5인 팀 프로젝트 Buddkit은 졸업 시점에 기능 구현은 완료된 상태였지만, 실제 운영 환경에서 발생할 결함과 성능 한계는 검증되지 않은 채 종료되었습니다. 졸업 후 5주간 개인 작업으로 다음 질문에 답하고자 했습니다.

> *"이 시스템은 실제 트래픽을 견딜 수 있는가, 그리고 그것을 어떻게 데이터로 증명할 수 있는가?"*

---

## 🎯 본인 역할

- **팀 프로젝트 단계** — 알림 도메인 백엔드 담당 (SSE 실시간)
- **리팩토링 단계** — 7개 도메인 전반의 코드 리뷰부터 결함 추적, 부하 테스트 (k6), 인프라 다운사이징, 프로덕션 하드닝까지 단독 수행
- 모든 개선 결정을 **부하 테스트 데이터**로 뒷받침 (Phase 기반, max 9,000 VU, DB ~124M rows)

---

## ✨ Highlights

- **60+ 버그 수정** — 인증 보안, 결제 트랜잭션, 동시성, N+1 등 8개 도메인 전반
- **모든 도메인 N+1 해소** — Redis 좋아요 패턴, 채팅 서브쿼리 등 구조적 N+1까지
- **고부하 시나리오 튜닝** — 가상 스레드 pinning, FORCE INDEX, NOT EXISTS, Kafka 정산 배치
- **스토리지 추상화 레이어 도입** — DB 교체 가능성 확보 (Notification: MySQL/MongoDB, Search: ES/FULLTEXT)
- **AWS EC2 부하 테스트 → 다운사이징** — c5.xlarge → t3.medium, **비용 70% 절감**
- **Chaos Engineering 인프라 구축** — 장애 주입 기반 회복력 검증, **48건 추가 개선**
> 📚
> **이 페이지는 요약입니다.** 8단계 챕터 · 시스템 아키텍처 SVG · 작업 일지 41편 · 부하 테스트 풀 리포트는 → **[리팩토링 & 성능 개선 상세 페이지](https://www.notion.so/353779d1d7da80b283edc50867f970a6)** 에서.
>

---

## 🏗 시스템 아키텍처

Buddkit은 8개 도메인이 서로 다른 저장소, 메시지 큐, 캐시를 사용하는 구조입니다. 이 프로젝트의 핵심은 각 도메인의 **access pattern**에 맞게 기술을 선택하는 것이었습니다.

![buddkit-architecture.svg](attachment:c0ead532-8d06-4ebf-8bd6-74e241690328:buddkit-architecture.svg)

**도메인별 데이터 흐름**

---

## 🔍 Case Studies

각 사례는 **문제 → 대안 비교 → 결정 → 결과 → 회고** 순서로 정리합니다. 모든 수치는 EC2에서 k6로 측정한 결과입니다.

---

### Case 1. 댓글 생성 Lock 직렬화 — p95 5.75s → 260ms (95%)

> **키워드** Row Lock Contention · 비동기 이벤트 분리

> **측정 환경** Feed 10.6M rows / 1,500 VU / EC2 c5.xlarge / k6

**문제**

`createComment()`는 댓글을 INSERT한 직후 같은 트랜잭션에서 `feed.comment_count`를 `UPDATE`했습니다. 이때 feed row에 X-Lock이 걸렸고, 동일 피드에 댓글이 동시에 들어오면 요청들이 lock을 기다리며 직렬화되었습니다.

그 결과 1,500 VU 부하에서 p95가 5.75초까지 상승했고, HikariCP 커넥션 풀도 포화되었습니다.

**Before — 동일 TX**

```
INSERT comment
  ↓  (같은 커넥션)
UPDATE feed.count
  ↓  X-Lock 유지
TX 종료까지 대기자 전부 직렬화
  ↓
p95 5,750ms · HikariCP 포화
```

**After — @Async 이벤트 분리**

```
INSERT comment
  ↓  publish
즉시 응답 (동기, lock 최소)
  ↓  AFTER_COMMIT
@Async 스레드풀 (max=10)
  ↓  별도 TX
UPDATE feed.count
  → p95 260ms · 포화 미발생
```

**대안 비교**

- 동기 카운트 업데이트 유지: 정합성은 단순하지만 lock 경합이 계속 발생합니다.
- 비동기 이벤트 분리: 댓글 생성 응답은 빠르게 반환하고, 카운트 갱신은 별도 트랜잭션에서 처리합니다.

**결정**

`@Async` 이벤트 분리. INSERT만 동기 처리하고, `CommentCountChangedEvent` 발행 → 별도 트랜잭션에서 count 갱신. 전용 스레드풀 `commentCountExecutor` max=10으로 제한해 커넥션 소비를 억제.

**결과**

- p95 응답 시간: 5,750ms → 260ms
- HikariCP 포화: 발생 → 미발생
- 댓글 생성 경로의 lock 보유 시간 감소

**트레이드오프**

- 댓글 수 갱신이 즉시 반영되지 않을 수 있습니다.
- 비동기 executor와 실패 재처리 정책을 별도로 관리해야 합니다.

📚 **레퍼런스** — [Spring Docs: TransactionalEventListener](https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html) · [Spring Docs: @Async](https://docs.spring.io/spring-framework/reference/integration/scheduling.html)

**회고**

> Lock 경합은 직관과 자주 어긋납니다. 단일 트랜잭션으로 묶는다고 안정적인 것이 아니라, 어떤 작업이 lock을 얼마나 오래 잡고 있는지가 핵심입니다. 이 사례에서는 `AFTER_COMMIT` + `@Async` 조합이 가장 단순하면서도 효과적이었습니다.

🔗 [상세 글: 피드 성능 부하 테스트 (Chapter 2)](https://www.notion.so/353779d1d7da81c29188e410996f8a0c)

---

### Case 2. 채팅 도메인 — Redis Streams로 DB 커넥션 의존 제거

> **키워드** 비동기 파이프라인 · HikariCP 포화

> **측정 환경** Chat 13M messages / 4,500 VU / EC2 t3.medium / k6

**문제**

채팅 메시지 1건을 전송할 때마다 DB 커넥션을 3회 획득했습니다: `getCurrentUser()`, `findUserInfoIfMember()`, `chatMessageStoragePort.save()`.

1,500 VU에서 초당 수십 건의 요청이 들어오자 커넥션 요청이 폭증했고, HikariCP 풀이 포화되었습니다. 그 결과 **84,000+ pending 상태**, 5~30초 응답 대기, 총 81,911건의 에러가 발생했습니다.

**대안 비교**

**결정**

Redis Streams로 **쓰기 경로 자체를 분리**했습니다. HTTP 스레드는 멤버십 캐시 조회, Pub/Sub 실시간 전파, XADD 영속화 큐 적재까지만 처리합니다.

실제 DB INSERT는 Consumer가 백그라운드에서 100건 단위 batch로 수행합니다.

**메시지 수명주기** — HTTP 응답과 실제 영속화는 분리된 경로.

**HTTP 경로 (동기 · 단일)**

1. `POST /chat`
1. Redis 멤버십 캐시 조회 → HIT
1. Redis PUBLISH → 실시간 프로파게이션
1. Redis XADD → 메시지 스트림
1. `200 OK` 응답 (<5ms, **DB 0회**)
**백그라운드 경로 (비동기 · 배치)**

1. Consumer XREADGROUP → 100건 읽기
1. MySQL batch INSERT 1회
1. Redis XACK → PEL 제거
1. (장애 시 PEL 적체 → 재시도)
→ 이 경로는 HTTP 스레드를 점유하지 않음

**Before — HTTP가 DB 직접 호출**

```
HTTP 스레드
  ↓  커넥션 #1
DB User 조회
  ↓  커넥션 #2
DB 멤버십 확인
  ↓  커넥션 #3
DB INSERT message
  ↓
Pub/Sub

→ 1건당 3회 획득
→ 1500 VU → HikariCP 포화
→ 응답 5\~30s 대기
```

**After — HTTP는 DB 0회**

```
HTTP 스레드
  ↓
Redis 멤버십 캐시
  ↓
Redis PUBLISH
  ↓
Redis XADD
  ↓
200 OK (\<5ms)

→ 백그라운드
Consumer XREADGROUP 100건
→ batch INSERT 1회
```

**결과**

**부수 검증** — MySQL vs MongoDB 비교

→ **MySQL 채택**. Redis 캐시가 DB 접근 95% 차단하므로 DB 엔진 차이 미미. MongoDB 드라이버 오버헤드만 추가되는 구조.

**트레이드오프**

- 메시지 영속성이 동기에서 준동기(<5ms latency)로 이동 — Consumer 장애 시 최대 수 초 지연 가능
- 운영 복잡성 +30% 수준 — PEL(Pending Entry List) 모니터링과 Consumer Group 관리 필요
- 반대급부 — 총 에러 81,911 → 0, 4500 VU 성공률 100%, HikariCP pending 0
📚 **레퍼런스** — [Redis Streams Tutorial](https://redis.io/docs/latest/develop/data-types/streams/) · [Redis Streams Consumer Group Patterns](https://redis.io/docs/latest/develop/data-types/streams-consumer-groups/)

> 🤔
> **직관과 다른 결과 — 읽어내기**
>
> "MongoDB가 더 빠를 줄 알았는데 왜 MySQL이 이겼을까?" → 테스트가 측정한 것은 *DB 엔진 성능*이 아니라 *드라이버 오버헤드*였기 때문.
>
> Redis Streams + 캐시 3종 구조가 DB 트래픽의 95%를 차단하면, 남은 5%에서는 엔진 성능 차이보다 드라이버 자체 비용(BSON 직렬화, topology discovery, connection 관리)이 더 지배적이 됨.
>
> **교차 검증** — 같은 두 DB를 알림 도메인에서도 비교했는데, 거기서는 결과가 정반대였습니다.
>
> 6,000 VU에서 MySQL은 `markAllAsRead`의 O(N) UPDATE로 row lock contention이 커졌고, MongoDB는 단일 document upsert라 안정적이었습니다. 그래서 알림 도메인은 MongoDB를 채택했습니다.
>
> **결론**: DB 선택은 *엔진 평판*이 아니라 *access pattern*으로 결정해야 하는 문제.
>
> - 채팅 = Redis가 대부분 막음 → 드라이버 가벼운 곳이 유리 → **MySQL**
> - 알림 = DB write contention 지배적 → lock-free 단일 도큐먼트 업데이트가 유리 → **MongoDB**

**회고**

> 커넥션 풀 포화는 *증설로 풀 수 있는 문제*가 아님. HTTP 경로에서 DB 의존을 끊어내는 것이 진짜 해법. Redis Streams의 Consumer Group이 PEL 관리만 잘하면 가장 가성비 좋은 비동기 구조.

---

### Case 3. AWS EC2 데이터 기반 다운사이징 — 비용 70% 절감

> **키워드** 부하 테스트 기반 사이징 · 비용 효율화

> **측정 환경** 5개 도메인 (알림 max 9,000 VU 포함) / Before: c5.xlarge + c5.2xlarge / After: t3.medium + t3.large

**문제**

본 프로젝트 종료 시점에는 안정성을 우선해 비교적 큰 EC2 인스턴스를 사용하고 있었습니다. App은 c5.xlarge(4vCPU/8GB), Infra는 c5.2xlarge(8vCPU/16GB)였습니다.

실제 도메인별 부하 한계를 알지 못한 상태였기 때문에, 먼저 성능을 확보한 뒤 측정 데이터를 기준으로 인스턴스 크기를 줄이는 접근이 필요했습니다.

**대안 비교**

**진행 흐름**

1. k6로 도메인별 시나리오 작성 (Feed · Chat · Notification · Settlement · Search)
1. EC2에서 실제 측정 → 도메인별 RPS·p95 한계 파악
1. 한계 대비 headroom 정해 인스턴스 사이즈 단계적 축소
1. 축소 후 재측정해 SLO 충족 여부 확인
**결과 — 인스턴스 / 컨테이너 / 풀 크기**

**부수 효과 — JVM/커널 튜닝까지 포함**

- **ZGC Generational** 도입 — write-heavy 시나리오에서 G1GC 대비 CPU 70%p 절감
- **JWT Parser 싱글턴 캐싱** — 9,000 VU 환경에서 BLOCKED 스레드 388 → 0 (ClassLoader lock 제거)
- **커널 튜닝** (TCP 버퍼·keepalive·backlog) — SSE 연결 p95 708ms → 90ms, 처리량 +20%
- **CPU** 100% 포화 → 평상시 3 ~ 30%
- **GC Allocation Stall** 44회 / 108.6초 → 0회
**회고**

> 성능 개선의 마지막 단계는 *비용으로 환수*. "더 빠르게"보다 "같은 일을 더 적은 자원으로"가 의사결정자에게 더 와닿는 메시지. 단, 다운사이징은 데이터 기반이어야 함 — 직감이나 비용 압박이 아니라 부하 테스트 결과로.

**트레이드오프**

- 고도 이벤트 대응력 감소 — t3.medium은 burstable, sustained 고부하를 우아하게 처리하지 못함. SLO 이내에서만 유효
- 풍부한 모니터링 필수 — 필요 시 즉시 스케일 업 경로 준비한 상태여야 함
- 반대급부 — 월 EC2 비용 70% 절감, CPU 100% 이하 안정구간 확보
📚 **레퍼런스** — [AWS EC2 Sizing Guide](https://docs.aws.amazon.com/whitepapers/latest/cost-optimization-right-sizing/cost-optimization-right-sizing.html) · [ZGC Generational 소개 (JEP 439)](https://openjdk.org/jeps/439)

🔗 [상세 글: 최종 리포트 — 인프라](https://www.notion.so/353779d1d7da811292c9c30670ca792e)

---

### Case 4 (요약). Chaos Engineering — 48건 추가 개선

> **키워드** 장애 주입 · 회복력 검증

성능과 비용까지 정리된 후, **Chaos Monkey + A/B 테스트 인프라** 도입. Latency / Exception / Memory Assault 시나리오로 의도적 장애 주입. 정상 트래픽에서 보이지 않던 **48건의 추가 결함** 발견 (무한 대기, 예외 처리 누락, 타임아웃 미설정 등).

> *"Chaos Engineering은 장애를 일으키는 실험이 아니라, 이미 있던 장애를 보이게 하는 실험."*

🔗 [상세 글: 프로덕션 하드닝 / Chaos](https://www.notion.so/353779d1d7da81c9a16cf9cbdffa1600)

---

## 📊 전체 성과 요약

| 영역 | 개선 결과 |
| --- | --- |
| 에러율 | 9.02% → 0.17% |
| 댓글 생성 | p95 5.75s → 260ms |
| 채팅 | 81,911 errors → 0 |
| 인프라 비용 | EC2 비용 70% 절감 |
| GC Stall | 44회 / 108.6초 → 0 |
| 작업 규모 | 8단계 챕터, 작업 일지 41편 |

---

## ⚠️ 한계와 향후 검증

- 리팩토링은 개인 작업으로 진행했기 때문에, 팀 단위 운영 프로세스와 코드 리뷰 체계까지 검증한 것은 아닙니다.
- 부하 테스트는 k6 시나리오 기반이므로 실제 사용자 행동을 완전히 대체하지는 않습니다.
- 다운사이징 결과는 현재 측정된 SLO 안에서 유효합니다. 장기 운영에서는 트래픽 증가, 이벤트성 피크, 장애 복구 상황을 추가로 검증해야 합니다.
- Chaos Engineering은 장애 주입 기반 검증까지 진행했지만, 실서비스 운영 장애 데이터와의 비교는 앞으로 보완할 영역입니다.

---

## 🛠 Tech Stack

| 영역 | 사용 기술 |
| --- | --- |
| Backend | Java, Spring Boot, Gradle multi-module |
| Database | MySQL, MongoDB |
| Cache / Queue | Redis, Redis Streams, Kafka |
| Test | k6, JUnit |
| Infra | AWS EC2, Docker, Nginx, Monitoring |

---

## 🧩 도메인별 최종 스택 결정

부하 테스트 데이터를 근거로 도메인별 저장소 / 메시지 큐 / 캐시를 다르게 채택.

---

## 🗂 작업 흐름

리팩토링 작업은 8단계로 진행되었습니다 (2026.02.17 ~ 03.20, 약 5주, 41편).

1. **[버그 수정](https://www.notion.so/353779d1d7da810c980fe5381435bb7a)** — 인증·실시간·결제·동시성 도메인별 결함 추적
1. **[부하 테스트 / 성능 개선](https://www.notion.so/353779d1d7da81dda6c4ea09518b6b36)** — N+1 해소 → 부하 측정 → 고부하 시나리오 튜닝
1. **[리팩토링 / 코드 품질](https://www.notion.so/353779d1d7da811c8c8ad9cead747186)** — 도메인 구조 재설계, ErrorCode 인터페이스 + 도메인별 enum 분리
1. **[스토리지 / 아키텍처 추상화](https://www.notion.so/353779d1d7da815c8a56dbf606de14b9)** — 도메인별 저장소 선택지를 추상화 레이어로 분리
1. **[프론트엔드 / React](https://www.notion.so/353779d1d7da81689a4ccd74c269e381)** — 코드 품질 + 메모이제이션·코드 스플리팅 최적화
1. **[AWS EC2 부하 테스트](https://www.notion.so/353779d1d7da81afb161f9a107dfe8bc)** — 로컬을 벗어난 실제 환경 검증
1. **[최종 리포트 / 인프라](https://www.notion.so/353779d1d7da811292c9c30670ca792e)** — 성능 확보 후 거꾸로 비용 줄이기 (다운사이징)
1. **[프로덕션 하드닝](https://www.notion.so/353779d1d7da81c9a16cf9cbdffa1600)** — Chaos Engineering 도입

---

## 📁 상세 페이지

> 📚
> **리팩토링 & 성능 개선 — 메인 작업물**
>
> - **8단계 챕터** + 작업 일지 **41편**
> - 시스템 아키텍처 **SVG 다이어그램**
> - **[부하 테스트 풀 리포트](https://www.notion.so/353779d1d7da815dadc6ee63ab944462)** (장문 종합)
> - 도메인별 최종 스택 결정 근거 데이터
> - [리팩토링 & 성능 개선 (2026.02~03)](https://app.notion.com/p/353779d1d7da80b283edc50867f970a6)

> 📝
> **온리원 — 팀 원본** (참고)
>
> - 부트캠프 본 과정 **5인 팀 작업**
> - 본인이 **알림 도메인 (SSE)** 담당
> - 리팩토링 결과와 비교/대조용
> - [온리원 (팀 원본)](https://app.notion.com/p/e69779d1d7da82e5847281d0461aa6bb)

---

## 🔗 Links

### Repositories

- 🛠 **리팩토링 레포** — [github.com/choigpt/OnlyOne-Back](http://github.com/choigpt/OnlyOne-Back)
*develop 브랜치 / 1,399 커밋 / Java 74.7% / multi-module Gradle*

- 📝 **대조군 (팀 원본)** — [github.com/choigpt/OnlyOne-Back-Original](http://github.com/choigpt/OnlyOne-Back-Original)
*GoormOnlyOne/OnlyOne-Back에서 fork / 1,233 커밋*

### 도메인별 모듈 (리팩토링 레포)

- [feed](https://github.com/choigpt/OnlyOne-Back/tree/develop/onlyone-domain-feed) · [chat](https://github.com/choigpt/OnlyOne-Back/tree/develop/onlyone-domain-chat) · [notification](https://github.com/choigpt/OnlyOne-Back/tree/develop/onlyone-domain-notification) · [finance](https://github.com/choigpt/OnlyOne-Back/tree/develop/onlyone-domain-finance) · [search](https://github.com/choigpt/OnlyOne-Back/tree/develop/onlyone-domain-search) · [club](https://github.com/choigpt/OnlyOne-Back/tree/develop/onlyone-domain-club) · [user](https://github.com/choigpt/OnlyOne-Back/tree/develop/onlyone-domain-user)
- [k6-tests](https://github.com/choigpt/OnlyOne-Back/tree/develop/k6-tests) · [monitoring](https://github.com/choigpt/OnlyOne-Back/tree/develop/monitoring) · [docker](https://github.com/choigpt/OnlyOne-Back/tree/develop/docker)

### 상세 리포트

- 📊 **부하 테스트 풀 리포트** — [최종 리포트 페이지](https://www.notion.so/353779d1d7da815dadc6ee63ab944462)
- 📝 **작업 회고 블로그** — *링크 추가 예정*

---

<details>
<summary>💭 면접 키 메시지 (스스로 채우기)</summary>

*면접 빈출 질문에 대한 본인의 답변을 1~2줄로 정리.*

- **"가장 어려웠던 결정은?"** — ...
- **"다시 한다면 무엇을 바꿀 것인가?"** — ...
- **"이 프로젝트에서 배운 가장 중요한 한 가지는?"** — ...
- **"운영 환경에서 추가로 검증하고 싶은 것은?"** — ...
</details>
