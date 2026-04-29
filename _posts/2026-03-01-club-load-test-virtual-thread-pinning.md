---
layout: post
title: 클럽 도메인 부하 테스트 — Virtual Thread Pinning에 의한 JVM 크래시 분석
date: 2026-03-01
tags: [Java, VirtualThread, k6, 부하테스트, MySQL, Spring, HikariCP, 데드락, 성능최적화, JVM]
permalink: /club-load-test-virtual-thread-pinning/
excerpt: "클럽 도메인 부하 테스트에서 100VU만으로 JVM이 크래시했다. 원인은 Virtual Thread + JDBC synchronized 조합의 carrier thread pinning. 피드 테스트(읽기 위주)에서는 발생하지 않았지만, 클럽 테스트(100% 쓰기)에서 발생했다. Pinning 제거 후 처리량 48배 증가, 데드락 수정, 이벤트 리스너 비동기화를 거쳐 400VU에서 전 구간 통과까지의 과정을 정리한다."
---

## 개요

피드 도메인 부하 테스트를 마치고 클럽 도메인으로 넘어왔다. 피드가 클럽에 종속되어 있고, 피드 최적화에서 `UserClubRepository` 캐싱을 건드렸으니 클럽 테스트로 영향 확인이 필요했다.

클럽 도메인은 API 5개로 단순하지만, 가입/탈퇴가 `memberCount` 갱신 + 이벤트 발행 + 캐시 eviction을 동반하는 쓰기 중심 도메인이다. 100VU 테스트에서 JVM이 크래시했으며, 이는 Virtual Thread의 구조적 한계에 해당한다.

---

## 도메인 구조 분석

**POST /clubs** — 쿼리 5개, 이벤트 발행 오버헤드가 주요 위험. **PATCH /clubs/{id}** — 쿼리 5개, 권한 체크 쿼리 다수. **GET /clubs/{id}** — 쿼리 4개, `countByClub_ClubId()`가 memberCount 필드가 이미 있음에도 매번 COUNT를 실행. **POST /clubs/{id}/join** — 쿼리 6개, memberCount 경합과 비관적 락 미사용이 위험. **DELETE /clubs/{id}/leave** — 쿼리 6개, memberCount 경합.

주요 병목 후보 3개:
1. `getClubDetail()` — 비정규화 필드가 있는데 매번 COUNT 쿼리 실행
2. `joinClub()` — 동시 가입 시 memberCount 경합 + userLimit 초과 가능
3. `accessibleClubIds` 캐시 — TTL 미설정 (기본 10초 fallback)

---

## 테스트 설계

3-Phase 병목 특화 시나리오를 설계했다.

**P1 (Hot Club, 10개 집중):** 200 VU, 동일 클럽에 동시 join/leave로 memberCount 경합을 유발. **P2 (Wide Club, 10,000개 분산):** 300 VU, 범용 부하. **P3 (고부하 혼합):** 400 VU, 최대 부하.

테스트 스크립트에는 상태 추적 로직을 넣었다:
- VU별 고유 유저 할당 (동일 유저 동시 접근 방지)
- `detectMembership()` — detail API로 현재 멤버 여부 1회 감지
- `toggleMembership()` — 멤버면 leave, 비멤버면 join (4xx 최소화)
- LEADER 보호 — LEADER 역할이면 leave 스킵

---

## JVM 크래시 — 100VU에서 서버 중단

첫 테스트에서 100VU만으로 JVM이 크래시했다. 피드 테스트에서는 400VU까지 정상 동작했으나, 클럽 테스트에서는 100VU에서 크래시가 발생했다.

### 1차 분석: 이벤트 리스너의 커넥션 이중 점유

`leaveClub()` 호출 시 `ClubLeftEvent`가 발행되고, `ClubEventListener.handleClubLeftEvent()`가 `REQUIRES_NEW` 트랜잭션으로 `userChatRoomRepository.deleteByUserIdAndClubId()`를 실행한다.

```
leaveClub() 본 트랜잭션 → DB 커넥션 1개 점유
  └→ ClubLeftEvent 발행
      └→ handleClubLeftEvent(@REQUIRES_NEW) → DB 커넥션 1개 추가 점유
```

leave 1건 = 트랜잭션 2개, DB 커넥션 2개 동시 점유. 100VU가 0.2초마다 toggle하면 초당 ~250회 leave, ~500개 트랜잭션이 커넥션 풀을 고갈시킨다.

**수정**: `@Async` 추가. 비동기 스레드에서 실행되므로 원래 트랜잭션의 커넥션은 이미 반환된 상태.

```java
@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void handleClubLeftEvent(ClubLeftEvent event) {
    // 본 트랜잭션 커밋 후 → 비동기 스레드에서 → 새 커넥션으로 실행
}
```

`@Async`와 `REQUIRES_NEW` 조합이 이 수정의 요점이다. `@TransactionalEventListener(AFTER_COMMIT)`은 Spring 6.2+에서 `REQUIRES_NEW` 또는 `NOT_SUPPORTED`만 허용하고, `@Async`가 있으면 별도 스레드라 커넥션 1개만 사용한다. AsyncConfig에 bounded executor(max 80, queue 500)가 설정되어 있어 스레드 폭발도 없다.

> `@Async` 환경에서는 새 스레드에 기존 트랜잭션이 없으므로 `REQUIRES_NEW`와 `REQUIRED`가 동일하게 동작한다(둘 다 새 트랜잭션 생성). `REQUIRES_NEW`를 명시하는 이유는 Spring 6.2+에서 `@TransactionalEventListener(AFTER_COMMIT)`에 `REQUIRES_NEW` 또는 `NOT_SUPPORTED`를 강제하기 때문이다.

### 2차 분석: Virtual Thread Pinning

`@Async`로 커넥션 이중 점유를 해결했으나 여전히 크래시가 발생했다. 설정을 확인한 결과, 근본 원인은 다음과 같다.

```java
// SseConfig.java:32
protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
```

Tomcat의 모든 요청이 Virtual Thread로 처리되고 있었다. Virtual Thread + JDBC(synchronized) 조합에서 carrier thread pinning이 발생한 것이다.

```
Virtual Thread → JDBC 호출 → synchronized 블록 진입 → carrier thread에 pin
→ pinned virtual thread는 carrier thread를 놓지 않음
→ carrier thread pool(ForkJoinPool)은 bounded (기본 parallelism = availableProcessors(), 상한 256)
→ 모든 carrier가 pinned → 새 virtual thread는 실행 대기
→ 그러나 Tomcat은 요청마다 새 virtual thread를 계속 생성 (backpressure 없음)
→ 실행되지 못하는 virtual thread가 무한 축적 → 각각 스택 메모리 점유
→ 메모리 초과 → JVM 크래시
```

**피드 테스트가 괜찮았던 이유**: 대부분 읽기(SELECT)라 synchronized 블록 체류 시간이 짧았다.

**클럽 테스트가 죽는 이유**: 100% 쓰기(INSERT/DELETE + UPDATE memberCount + 이벤트 발행) → synchronized 블록에 오래 머무름 → pinning 폭발.

### 수정: Tomcat Virtual Thread 제거

```java
// 제거
protocolHandlerVirtualThreadExecutorCustomizer  // Tomcat 전체 요청용
applicationTaskExecutor                         // Spring MVC 비동기 처리용

// 유지
sseEventExecutor  // SSE 이벤트 전송 전용 (JDBC 호출 없음, pinning 무관)
```

변경 전에는 Tomcat 요청 처리가 Virtual Thread로 동작해 pinning 위험이 있었으며, JDBC 쓰기 부하에서 carrier thread가 고갈되어 JVM 크래시가 발생했다. 변경 후에는 Tomcat 요청 처리를 Platform Thread(NIO, 200 threads)로 전환하여 안정화했다. SSE 이벤트 전송은 JDBC 호출이 없어 pinning과 무관하므로 Virtual Thread를 유지했다.

---

## 데드락 수정 — 피드와 동일한 패턴

클럽에서도 피드 댓글과 동일한 데드락 패턴이 나타났다.

```
joinClub():
  1. INSERT user_club  → FK로 club 행 S lock
  2. UPDATE club SET member_count+1  → 같은 club 행 X lock 필요

leaveClub() (동시 실행):
  1. DELETE user_club  → club 행 S lock
  2. UPDATE club SET member_count-1  → 같은 club 행 X lock 필요

→ 둘 다 S lock 보유 + X lock 대기 → 데드락
```

[피드 고부하 편](/feed-highload-force-index-caching-evolution/)에서 적용한 것과 동일한 패턴으로 해결했다. 클래스 레벨 `@Transactional`을 제거하고, `memberCount` 증감을 별도 트랜잭션으로 분리했다.

```java
public void joinClub(Long clubId) {
    // 1) 검증 + INSERT user_club (한 트랜잭션)
    Long userId = transactionTemplate.execute(status -> {
        Club club = findClubOrThrow(clubId);
        // validation...
        userClubRepository.save(userClub);
        userClubRepository.flush();
        return user.getUserId();
    });

    // 2) member_count 증가 (별도 트랜잭션 — club 행 lock 최소화)
    try {
        transactionTemplate.executeWithoutResult(status ->
                clubRepository.incrementMemberCount(clubId));
    } catch (Exception e) {
        log.warn("멤버 카운트 증가 실패 (가입은 정상): clubId={}", clubId);
    }

    evictAccessibleClubIds(userId);
}
```

count UPDATE 실패 시 가입 자체는 보존된다. count 불일치는 스케줄드 배치에서 `COUNT(*)` 실측과 `memberCount` 컬럼을 비교해 주기적으로 보정한다.

---

## Pinning 제거 전후 비교

Virtual Thread(pinning 상태)에서는 VU 100/150/200, sleep 1.0초, 총 요청 101K, 처리량 14.6 req/s에 서버 크래시가 빈번했고 에러율 0.59%였다. Platform Thread 전환 후에는 VU 200/300/400, sleep 0.2초, 총 요청 327K(**3.2배**), 처리량 698 req/s(**48배**), 서버 크래시 0건, 에러율 0.04%로 개선됐다.

Virtual Thread pinning 제거로 처리량 48배 증가, 에러율 15배 감소, 크래시 0건.

> 참고: 48배 차이는 pinning 제거뿐 아니라 VU 증가(200→400)와 sleep 감소(1.0s→0.2s)를 포함한 수치다. 동일 조건(200VU, 1.0s sleep)에서의 순수 pinning 제거 효과는 별도로 측정하지 않았다. 핵심은 pinning 상태에서는 100VU에서도 JVM이 크래시했고, 제거 후에는 400VU에서도 안정적이었다는 점이다.

---

## 최종 결과

최종 결과에서 클럽 상세 조회는 p50 33ms, p95 112ms로 500ms 임계값 이내(PASS). 클럽 가입은 p50 26ms, p95 126ms로 PASS. 클럽 탈퇴는 p50 19ms, p95 134ms로 PASS. 에러율은 0.00%로 10% 임계값 이내(PASS).

- 총 요청: 445,216건 (973 req/s)
- 5xx 에러: **0건**
- memberCount 경합: **0건** (0.00%)
- 테스트 시간: 7분 37초

Hot Club 10개에 200VU가 집중 공격해도 memberCount 경합 0건. 트랜잭션 분리 패턴이 유효하게 동작했다.

---

## 정리하며

클럽 도메인 자체의 쿼리 복잡도는 낮았다. API 5개, 최대 6쿼리. 피드 도메인처럼 N+1이나 IN절 폭발이 없었다. 문제는 인프라 레이어에 있었다.

**Virtual Thread + JDBC synchronized 조합은 잠재적 장애 요인이다.** 읽기 위주 워크로드에서는 pinning 시간이 짧아 문제가 발생하지 않는다. 쓰기 위주 워크로드에서 synchronized 블록 체류 시간이 길어지면 carrier thread가 고갈되고 JVM이 크래시한다. 같은 코드베이스에서 피드 테스트(읽기)는 통과하고 클럽 테스트(쓰기)에서 크래시가 발생한 원인이 이것이다.

Virtual Thread를 도입하려면 JDBC 드라이버가 `java.util.concurrent.locks.ReentrantLock`을 사용하는지 확인해야 한다. HikariCP 5.1.0+는 Virtual Thread 친화적이지만, MySQL Connector/J의 내부 synchronized 블록은 여전히 pinning을 유발한다. 쓰기 비중이 높은 도메인에서는 Platform Thread가 더 안전하다.

[정산 도메인](/finance-settlement-batch-kafka-tuning/)에서는 배치 UPDATE로 fallback 빈도 자체를 줄이는 방식으로 성능을 개선했다.

> **부하 테스트는 도메인별로 따로 돌려야 한다. 읽기 위주 도메인에서 통과했다고 쓰기 위주 도메인에서 통과하는 것이 아니다.**

---

## 수정 내역 요약

수정 1: Tomcat Virtual Thread executor 제거 — JVM 크래시 해결, 처리량 48배 증가. 수정 2: `ClubEventListener`에 `@Async` 추가 — 커넥션 이중 점유 제거. 수정 3: `joinClub`/`leaveClub` 트랜잭션 분리 — 데드락 해결, memberCount 경합 0%. 수정 4: 클래스 레벨 `@Transactional`을 메서드 레벨로 전환 — lock 보유 시간 최소화. 수정 5: k6 상태 추적 보정(400 응답 처리) — 불필요한 4xx 제거, p95 60% 개선.

---

## 시리즈 탐색

**◀ 이전 글**
[피드 도메인 고부하 테스트 — FORCE INDEX 실패부터 인메모리 캐싱까지](/feed-highload-force-index-caching-evolution/)

**▶ 다음 글**
[검색 도메인 부하 테스트 — teammates_clubs 쿼리 62배 개선 기록](/search-load-test-not-exists-optimization/)
