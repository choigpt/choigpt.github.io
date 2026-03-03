---
title: 클럽 도메인 부하 테스트 — Virtual Thread Pinning이 JVM을 죽인 날
date: 2026-03-03
tags: [Java, VirtualThread, k6, 부하테스트, MySQL, Spring, HikariCP, 데드락, 성능최적화, JVM]
permalink: /club-load-test-virtual-thread-pinning/
excerpt: "클럽 도메인 부하 테스트에서 100VU만으로 JVM이 크래시했다. 원인은 Virtual Thread + JDBC synchronized 조합의 carrier thread pinning. 피드 테스트(읽기 위주)에서는 안 터졌지만, 클럽 테스트(100% 쓰기)에서 터졌다. Pinning 제거 후 처리량 48배 증가, 데드락 수정, 이벤트 리스너 비동기화를 거쳐 400VU에서 전 구간 통과까지의 기록."
---

## 개요

피드 도메인 부하 테스트를 마치고 클럽 도메인으로 넘어왔다. 피드가 클럽에 종속되어 있고, 피드 최적화에서 `UserClubRepository` 캐싱을 건드렸으니 클럽 테스트로 영향 확인이 필요했다.

클럽 도메인은 API 5개로 단순하지만, 가입/탈퇴가 `memberCount` 갱신 + 이벤트 발행 + 캐시 eviction을 동반하는 쓰기 중심 도메인이다. 100VU 테스트에서 JVM이 크래시하면서 Virtual Thread의 구조적 한계를 마주했다.

---

## 도메인 구조 분석

| HTTP | 엔드포인트 | 쿼리 수 | 주요 위험 |
|------|-----------|---------|----------|
| POST | /clubs | 5 | 이벤트 발행 오버헤드 |
| PATCH | /clubs/{id} | 5 | 권한 체크 쿼리 다수 |
| GET | /clubs/{id} | 4 | `countByClub_ClubId()` — memberCount 필드가 이미 있는데 COUNT 실행 |
| POST | /clubs/{id}/join | 6 | memberCount 경합 + 비관적 락 미사용 |
| DELETE | /clubs/{id}/leave | 6 | memberCount 경합 |

핵심 병목 후보 3개:
1. `getClubDetail()` — 비정규화 필드가 있는데 매번 COUNT 쿼리 실행
2. `joinClub()` — 동시 가입 시 memberCount 경합 + userLimit 초과 가능
3. `accessibleClubIds` 캐시 — TTL 미설정 (기본 10초 fallback)

---

## 테스트 설계

3-Phase 병목 특화 시나리오를 설계했다.

| Phase | 목적 | VU | 공격 지점 |
|-------|------|-----|----------|
| P1 | Hot Club (10개 집중) | 200 | 동일 클럽에 동시 join/leave → memberCount 경합 |
| P2 | Wide Club (10,000개 분산) | 300 | 범용 부하 |
| P3 | 고부하 혼합 | 400 | 최대 부하 |

테스트 스크립트에는 상태 추적 로직을 넣었다:
- VU별 고유 유저 할당 (동일 유저 동시 접근 방지)
- `detectMembership()` — detail API로 현재 멤버 여부 1회 감지
- `toggleMembership()` — 멤버면 leave, 비멤버면 join (4xx 최소화)
- LEADER 보호 — LEADER 역할이면 leave 스킵

---

## JVM 크래시 — 100VU에서 서버가 죽다

첫 테스트에서 100VU만으로 JVM이 크래시했다. 피드 테스트에서는 400VU까지 버텼는데 클럽에서 죽은 것이다.

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

`@Async`와 `REQUIRES_NEW` 조합이 핵심이다. `@TransactionalEventListener(AFTER_COMMIT)`은 Spring 6.2+에서 `REQUIRES_NEW` 또는 `NOT_SUPPORTED`만 허용하고, `@Async`가 있으면 별도 스레드라 커넥션 1개만 사용한다. AsyncConfig에 bounded executor(max 80, queue 500)가 설정되어 있어 스레드 폭발도 없다.

### 2차 분석: Virtual Thread Pinning

`@Async`로 커넥션 이중 점유를 해결했는데도 여전히 크래시가 발생했다. 설정을 뒤져보니 핵심 원인이 나왔다.

```java
// SseConfig.java:32
protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
```

Tomcat의 모든 요청이 Virtual Thread로 처리되고 있었다. Virtual Thread + JDBC(synchronized) 조합에서 carrier thread pinning이 발생한 것이다.

```
Virtual Thread → JDBC 호출 → synchronized 블록 진입 → carrier thread에 pin
→ carrier thread가 해제될 때까지 다른 virtual thread 실행 불가
→ carrier thread 고갈 → 네이티브 스레드 무한 생성 → 메모리 초과 → JVM 크래시
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

| 구분 | 변경 전 | 변경 후 |
|------|---------|---------|
| Tomcat 요청 처리 | Virtual Thread (pinning 위험) | Platform Thread (NIO, 200 threads) |
| SSE 이벤트 전송 | Virtual Thread | Virtual Thread (유지) |
| JDBC 쓰기 부하 | Carrier thread 고갈 → JVM 크래시 | 일반 thread pool → 안정 |

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

피드 댓글에서 적용한 것과 동일한 패턴으로 해결했다. 클래스 레벨 `@Transactional`을 제거하고, `memberCount` 증감을 별도 트랜잭션으로 분리했다.

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

count UPDATE 실패 시 가입 자체는 보존된다. count 불일치는 별도 동기화 프로시저로 보정 가능하다.

---

## Pinning 제거 전후 비교

| 항목 | Virtual Thread (pinning) | Platform Thread |
|------|-------------------------|-----------------|
| VU | 100/150/200 | 200/300/400 |
| Sleep | 1.0s | 0.2s |
| 총 요청 | 101K | 327K **(3.2배)** |
| 처리량 | 14.6 req/s | 698 req/s **(48배)** |
| 서버 크래시 | 빈번 | 없음 |
| Error rate | 0.59% | 0.04% |

Virtual Thread pinning 제거로 처리량 48배 증가, 에러율 15배 감소, 크래시 0건.

---

## 최종 결과

| 메트릭 | p50 | p95 | 임계값 | 상태 |
|--------|-----|-----|--------|------|
| 클럽 상세 조회 | 33ms | 112ms | < 500ms | PASS |
| 클럽 가입 | 26ms | 126ms | < 500ms | PASS |
| 클럽 탈퇴 | 19ms | 134ms | < 500ms | PASS |
| 에러율 | — | 0.00% | < 10% | PASS |

- 총 요청: 445,216건 (973 req/s)
- 5xx 에러: **0건**
- memberCount 경합: **0건** (0.00%)
- 테스트 시간: 7분 37초

Hot Club 10개에 200VU가 집중 공격해도 memberCount 경합 0건. 트랜잭션 분리 패턴이 효과적이었다.

---

## 정리하며

클럽 도메인 자체의 쿼리 복잡도는 낮았다. API 5개, 최대 6쿼리. 피드 도메인처럼 N+1이나 IN절 폭발이 없었다. 문제는 인프라 레이어에 있었다.

**Virtual Thread + JDBC synchronized = 시한폭탄이다.** 읽기 위주 워크로드에서는 pinning이 짧아서 터지지 않는다. 쓰기 위주 워크로드에서 synchronized 블록 체류 시간이 길어지면 carrier thread가 고갈되고 JVM이 죽는다. 같은 코드베이스에서 피드 테스트(읽기)는 통과하고 클럽 테스트(쓰기)에서 죽은 이유가 이것이다.

Virtual Thread를 도입하려면 JDBC 드라이버가 `java.util.concurrent.locks.ReentrantLock`을 사용하는지 확인해야 한다. HikariCP 5.1.0+는 Virtual Thread 친화적이지만, MySQL Connector/J의 내부 synchronized 블록은 여전히 pinning을 유발한다. 쓰기 비중이 높은 도메인에서는 Platform Thread가 더 안전하다.

[정산 도메인](/finance-settlement-batch-virtual-thread/)에서는 Virtual Thread + JDBC를 fallback 병렬 처리에 사용하고 있다. 이 경우 동시 실행이 최대 10건 수준이라 carrier thread 고갈이 발생하지 않는다. Tomcat 레벨(수백 VU 동시 접근)과 애플리케이션 레벨(소수 병렬 태스크)은 pinning 위험도가 다르다.

> **부하 테스트는 도메인별로 따로 돌려야 한다. 읽기 위주 도메인에서 통과했다고 쓰기 위주 도메인에서 통과하는 것이 아니다.**

---

## 수정 내역 요약

| # | 변경 내용 | 효과 |
|---|----------|------|
| 1 | Tomcat Virtual Thread executor 제거 | JVM 크래시 해결, 처리량 48배 |
| 2 | `ClubEventListener`에 `@Async` 추가 | 커넥션 이중 점유 제거 |
| 3 | `joinClub`/`leaveClub` 트랜잭션 분리 | 데드락 해결, memberCount 경합 0% |
| 4 | 클래스 레벨 `@Transactional` → 메서드 레벨 | lock 보유 시간 최소화 |
| 5 | k6 상태 추적 보정 (400 응답 처리) | 불필요한 4xx 제거, p95 60% 개선 |

---

## 시리즈 탐색

**◀ 이전 글**
[피드 도메인 고부하 테스트 — FORCE INDEX 실패부터 인메모리 캐싱까지](/feed-highload-force-index-caching-evolution/)

**▶ 다음 글**
[검색 도메인 부하 테스트 — NOT IN 한 줄이 만든 62배 차이](/search-load-test-not-exists-optimization/)
