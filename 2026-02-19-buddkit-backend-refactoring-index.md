---
layout: post
title: "Buddkit 백엔드 리팩토링 여정: 9개 모듈, 30개 PR, 15,000 LOC"
date: 2026-02-19 15:00:00 +0900
categories: [Backend, Refactoring]
tags: [Spring Boot, Elasticsearch, Performance, SOLID, CQRS]
---

# Buddkit Backend Refactoring Index

> **Repository**: [GitHub - OnlyOne-Back-Original](https://github.com/choigpt/OnlyOne-Back-Original)

전체 백엔드 모듈의 리팩토링 및 성능 최적화 여정을 기록한 종합 인덱스입니다.

---

## 📖 프로젝트 배경

### Buddkit이란?

**Buddkit**(구 OnlyOne)은 관심사 기반 소셜 모임 플랫폼입니다. 사용자들이 관심사를 공유하고, 클럽을 만들고, 일정을 계획하며, 실시간으로 소통할 수 있는 서비스입니다.

### 핵심 기능

- 🏘️ **Club**: 관심사 기반 커뮤니티 생성 및 관리
- 📅 **Schedule**: 모임 일정 생성, 참가, 정산
- 💬 **Chat**: STOMP WebSocket 기반 실시간 채팅
- 📱 **Feed**: 소셜 미디어 피드 (게시글, 댓글, 좋아요, 리피드)
- 💳 **Finance**: 결제, 지갑, 정산 시스템
- 🔍 **Search**: Elasticsearch 기반 모임 검색/추천
- 🔐 **User**: 카카오 OAuth 로그인, 프로필 관리

### 리팩토링 동기

#### 초기 상태 (부스트캠프 프로젝트)

```
❌ 문제점
- 코드 읽기 어려움 (파악 불가)
- 유지보수 어려움
- 성능 병목 존재
- 순환 의존성 (모듈 간 결합)
- 테스트 부재 또는 미흡
```

#### 목표

```
✅ 개선 목표
1. 멀티 모듈 전환 → 도메인 경계 명확화
2. 성능 테스트 → 병목점 파악
3. 1차 성능 개선 → 즉각적 효과
4. 리팩토링 → 유지보수성/확장성 확보
```

### 프로젝트 타임라인

```
Phase 1 (Week 1-2): 멀티 모듈 전환
  ├─ 순환 의존성 해소
  ├─ 도메인 경계 명확화
  └─ 이벤트 기반 통신 도입

Phase 2 (Week 3-5): 성능 최적화
  ├─ k6 부하 테스트 (병목 식별)
  ├─ N+1 쿼리 제거
  ├─ 인덱스/캐싱 적용
  └─ 동시성 제어 개선

Phase 3 (Week 6-10): SOLID 리팩토링
  ├─ God Service 분해
  ├─ Command/Query 분리
  ├─ DIP 적용
  └─ 테스트 커버리지 확대

Result: 30 PRs, 9 Modules, 15,000+ LOC Refactored
```

---

## 📋 모듈별 상세 문서

### Core Domain Modules

- [User Module Refactoring](/buddkit-refactoring/user-module-refactoring) - 인증/프로필 관리
- [Chat Module Refactoring](/buddkit-refactoring/chat-module-refactoring) - 실시간 채팅 (WebSocket/Redis)
- [Schedule Module Refactoring](/buddkit-refactoring/schedule-module-refactoring) - 일정 관리 + 성능 최적화
- [Feed and Club Module Refactoring](/buddkit-refactoring/feed-and-club-module-refactoring) - 소셜 미디어 + 커뮤니티
- [Finance Module Refactoring](/buddkit-refactoring/finance-module-refactoring) - 결제/지갑/정산
- [Search Module Refactoring](/buddkit-refactoring/search-module-refactoring) - Elasticsearch 검색/추천

### Supporting Modules

- [Image Module Refactoring](/buddkit-refactoring/image-module-refactoring) - S3 이미지 처리
- [Interest Module Refactoring](/buddkit-refactoring/interest-module-refactoring) - 카테고리/관심사
- [Infra Module Refactoring](/buddkit-refactoring/infra-module-refactoring) - Redis/Kafka/Elasticsearch 설정

---

## 🎯 전체 리팩토링 성과

| Module | Status | PRs | Key Achievement |
|--------|--------|-----|----------------|
| **User** | ✅ Complete | 3 | Service 분해, DIP 적용, Tests +30% |
| **Chat** | ✅ Complete | 2 | Command/Query 분리, Event 통합 -50% |
| **Schedule** | ✅ Complete | 5 | 상태 머신 캡슐화, 성능 4.6배 향상 |
| **Feed** | ✅ Complete | 5 | N+1 완전 제거, Timeout 해소 |
| **Club** | ✅ Complete | 5 | Deadlock 99.6% 감소, 타입 안전 |
| **Finance** | ✅ Complete | 4 | CAS 선점, Wallet 분리 |
| **Search** | ✅ Complete | 2 | ES 최적화, teammates 99.9% 개선 |
| **Image** | ✅ Complete | 2 | 타입 안전성, Tests +183% |
| **Interest** | ✅ Complete | 1 | Category enum 강화 |
| **Infra** | ✅ Complete | 1 | 설정 표준화, 에러 핸들링 |

**Total: 30 PRs Completed** 🚀

---

## 📈 Performance Improvements

### Feed Module
- **Before**: 99.54% error (timeout)
- **After**: 0.61% error
- **Improvement**: popular_feed p95 44% 감소, N+1 완전 제거

### Club Module
- **Before**: 234,330 deadlock errors
- **After**: 858 errors
- **Improvement**: 99.6% 감소

### Schedule Module
- **Before**: 255 req/s, schedule_list p95 1,080ms
- **After**: 1,173 req/s (4.6배), schedule_list p95 387ms
- **Improvement**: 처리량 4.6배, 응답시간 64% 개선

### Finance Module
- **Before**: 결제 경합 발생
- **After**: CAS 기반 선점, 100% 성공률
- **Improvement**: 동시성 제어 완벽

### Search Module
- **Before**: 에러율 56%, HTTP p95 1,980ms
- **After**: 에러율 0.04%, HTTP p95 79ms
- **Improvement**: 에러율 -99.9%, 응답시간 -96%, 처리량 +110%

---

## 🔑 Key Patterns Applied

### 1. Command/Query Separation (CQRS)

**Before**: 단일 Service (300+ LOC, 8+ deps)  
**After**: CommandService + QueryService (각 100-150 LOC, 4-5 deps)

**Benefits**:
- ✅ @Transactional(readOnly=true) 명시적 적용
- ✅ 의존성 분산 (테스트 Mock 감소)
- ✅ 책임 명확화 (SRP)

### 2. Event-Driven Architecture

**Pattern**: 
```
Schedule Created Event
  → ChatRoomLifecycleHandler (채팅방 생성)
  → SettlementEventHandler (정산 초기화)
```

**Benefits**:
- ✅ 모듈 간 느슨한 결합
- ✅ @TransactionalEventListener(AFTER_COMMIT)로 데이터 정합성 보장
- ✅ 확장 용이 (새 리스너 추가만)

### 3. Performance Patterns

**Techniques**:
- Two-Pass Query (ID 조회 → JOIN FETCH)
- 커버링 인덱스 (7개 컬럼)
- 비정규화 컬럼 (commentCount)
- Redis Lua Script (좋아요 토글)
- CAS (결제 선점)
- ES filter 컨텍스트 (스코어링 스킵)

---

## 🎓 Lessons Learned

### 1. God Service는 반드시 분해
- 300+ LOC, 8+ deps = 변경 사유가 너무 많음
- Command/Query 분리로 책임 명확화
- 테스트 Mock 50% 이상 감소

### 2. 성능 최적화는 측정 기반
- k6 부하 테스트로 병목 식별
- 쿼리 수 측정 (N+1 감지)
- Before/After 메트릭 비교

### 3. ES filter vs must 구분
- exact-match 조건은 filter 컨텍스트
- 키워드 검색은 should (스코어링)
- filter는 캐싱되므로 반복 쿼리 향상

### 4. 상관 서브쿼리의 위험성
- EXISTS는 외부 행마다 실행
- 비상관 서브쿼리(IN)로 전환
- teammates 쿼리 99.9% 개선

---

## 🛠️ Tools & Technologies

### Development
- Java 17
- Spring Boot 3.x
- JPA/Hibernate
- QueryDSL

### Infrastructure
- Redis (캐싱, Pub/Sub, Lua)
- Kafka (이벤트 스트리밍)
- Elasticsearch (검색, nori 분석기)
- S3 (이미지 저장)

### Testing
- JUnit 5
- Mockito/BDDMockito
- k6 (부하 테스트)
- Docker (테스트 환경)

---

## 📚 Documentation Structure

모든 모듈 문서는 일관된 5단계 구조:

1. **개요** — 모듈 특성, 리팩토링 배경
2. **구조적 문제 분석** — 문제 목록, 테스트 스멜
3. **리팩토링 설계** — 목표 구조, 핵심 변경
4. **실행 결과** — PR별 요약, 메트릭 비교
5. **결론** — 성과, 교훈, 특이사항

---

*Last Updated: 2026-02-19*  
*Total Documentation: 9 modules, 30 PRs, ~15,000 LOC refactored*
