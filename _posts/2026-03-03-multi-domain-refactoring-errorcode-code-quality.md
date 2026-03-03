---
title: 멀티 도메인 리팩토링 — ErrorCode 분리부터 29파일 코드 품질 정리까지
date: 2026-03-03
tags: [Java, 리팩토링, Spring, 코드품질, 멀티모듈, ErrorCode, 클린코드]
permalink: /multi-domain-refactoring-errorcode-code-quality/
excerpt: "150줄짜리 단일 ErrorCode enum을 인터페이스 + 10개 도메인별 enum으로 분리했다. 이어서 성능 테스트 과정에서 쌓인 기술 부채를 정리했다. 5개 도메인, 29개 파일. 용어 불일치, 사일런트 예외, 매직넘버, 레거시 주석 70줄, 중복 record — 두 라운드에 걸쳐 정리한 기록."
---

## 개요

부하 테스트를 도메인별로 돌리면서 성능 문제만 쫓다 보니 코드 품질이 밀려 있었다. 도메인별 부하 테스트와 심화 테스트가 끝난 뒤, Storage 추상화 작업에 들어가기 전에 두 라운드에 걸쳐 정리했다.

- **1라운드**: ErrorCode enum 분리 — 모든 도메인 에러가 한 파일에 150줄로 몰려 있던 구조를 인터페이스 + 도메인별 enum으로 분리
- **2라운드**: 5개 도메인 코드 품질 일괄 정리 — 22개 이슈, 29개 파일 수정

---

## 1라운드 — ErrorCode enum → 인터페이스 분리

### 문제

`ErrorCode`가 단일 enum 하나에 150줄로 모든 도메인 에러가 한곳에 있었다. 채팅 에러 코드를 추가하려면 금융 에러 코드 사이를 뒤져야 하고, 모든 도메인 모듈이 이 파일에 의존했다.

### 전략

1. `ErrorCode` enum → `ErrorCode` 인터페이스로 변환 (common 모듈)
2. 공통 에러는 `GlobalErrorCode` enum으로 남김 (SSE, DB, 글로벌 에러)
3. 각 도메인 모듈에 `XxxErrorCode` enum 생성 (인터페이스 구현)
4. `CustomException`은 인터페이스 타입을 받도록 변경
5. `GlobalExceptionHandler`도 인터페이스 타입으로 처리

### 분리 결과

| 모듈 | ErrorCode enum | 상수 수 |
|------|---------------|---------|
| common | GlobalErrorCode | 15 (SSE, DB, 글로벌) |
| domain-user | UserErrorCode | 7 |
| domain-interest | InterestErrorCode | 2 |
| domain-club | ClubErrorCode | 9 |
| domain-chat | ChatErrorCode | 15 |
| domain-feed | FeedErrorCode | 6 |
| domain-finance | FinanceErrorCode | 22 |
| domain-schedule | ScheduleErrorCode | 14 |
| domain-notification | NotificationErrorCode | 6 |
| domain-image | ImageErrorCode | 5 |
| domain-search | SearchErrorCode | 8 |

### 순환 의존성 해결

멀티 모듈에서 ErrorCode를 분리하면 모듈 간 의존 관계가 드러난다.

- **finance ↔ schedule**: `SCHEDULE_NOT_FOUND`, `ALREADY_SETTLING_SCHEDULE`를 `FinanceErrorCode`에 중복 정의 (같은 HTTP 상태 코드 유지)
- **chat → image**: `INVALID_IMAGE_CONTENT_TYPE`를 `ChatErrorCode`에 중복 정의

동일한 에러 코드 값을 유지하되 각 모듈이 독립적으로 선언하는 방식으로 순환 참조를 끊었다.

### 테스트 수정

- `ImageControllerTest`, `SearchControllerTest`: `@Import(GlobalExceptionHandler.class)` 제거 (api 모듈로 이동되어 접근 불가)
- `UserServiceTest`: `INTEREST_NOT_FOUND` 테스트의 사용자 상태를 `GUEST`로 수정 (기존 버그 수정)

---

## 2라운드 — 5개 도메인 코드 품질 일괄 정리

성능 테스트를 진행하면서 알림, 채팅, 피드, 금융, 검색 5개 도메인에 걸쳐 쌓인 22개 이슈를 분류하고 일괄 수정했다.

### 공통 이슈

| 패턴 | 해당 도메인 | 심각도 |
|------|-----------|--------|
| 사일런트 예외 무시 (로깅 없이 삼킴) | 알림, 채팅, 피드 | High |
| `catch (Exception e)` 포괄적 예외 처리 | 알림, 피드, 금융 | Medium |
| `var` 키워드 사용 불일치 | 알림, 금융 | Low |
| 섹션 구분 주석 스타일 불일치 | 알림, 채팅, 피드 | Low |

---

### Notification — 용어 정규화 + 예외 처리

**용어 불일치 (High)**: Port 인터페이스는 `markDeliveredByIds` / `findUndeliveredByUserId`인데, Repository 계층이 `markSseSentByIds` / `findUnsentNotificationsByUserId`로 SSE 특화 이름을 사용하고 있었다.

| 파일 | 변경 |
|------|------|
| NotificationRepositoryCustom.java | `markSseSentByIds` → `markDeliveredByIds` |
| NotificationRepositoryImpl.java | 구현 메서드 동일 변경 |
| MysqlNotificationStorageAdapter.java | 호출부 동일 변경 |

**사일런트 예외 (High)**: `FcmNotificationDeliveryAdapter`에서 `catch (Exception ignored) {}`로 예외를 삼키고 있었다. `log.debug("FCM content 추출 실패", e)`로 변경했다.

**var → 명시적 타입**: `var node` → `JsonNode node`, `var entry` → `Map.Entry<Long, BlockingQueue<...>> entry`.

---

### Chat — 예외 타입 통일 + 네이밍

**IllegalStateException → CustomException (High)**: `ChatScheduleEventListener`에서 채팅방을 못 찾으면 `IllegalStateException`을 던지고 있었다. 같은 모듈의 `ChatRoomCommandService`는 `CustomException(ChatErrorCode.CHAT_ROOM_NOT_FOUND)`를 사용한다. 통일했다.

| 파일 | 변경 |
|------|------|
| ChatScheduleEventListener.java | `IllegalStateException` → `CustomException(ChatErrorCode.CHAT_ROOM_NOT_FOUND)` |
| ChatPublisher.java | silent return → `log.debug` 추가 |
| MysqlChatMessageStorageAdapter.java | `fromProjection` → `toDto` (기존 매핑 메서드와 일관성) |
| MongoChatMessageStorageAdapter.java | `// ========== x ==========` → `// ── x ──` |

---

### Feed — 중복 record 제거 + 캐시 키 상수화

**FeedIdWithCounts 중복 정의 (High)**: `FeedRepositoryCustom`과 `FeedStoragePort` 두 곳에 동일한 record가 정의되어 있었다. `FeedStoragePort`쪽을 삭제하고 `FeedRepositoryCustom.FeedIdWithCounts`로 통합했다. `MysqlFeedStorageAdapter`의 불필요한 `.map()` 변환도 함께 제거했다.

**캐시 키 매직 스트링**: `"pf:"`, `"ppf:"`를 상수로 추출했다.

```java
// 수정 전
String cacheKey = "pf:" + userId + ":" + page + ":" + size;

// 수정 후
private static final String PERSONAL_FEED_KEY_PREFIX = "pf:";  // pf:{userId}:{page}:{size}
private static final String POPULAR_FEED_KEY_PREFIX = "ppf:";
```

**캐시 실패 사일런트 반환**: `catch (Exception e) { return null; }`에 `log.debug("pass1 캐시 조회 실패: {}", e.getMessage())`를 추가했다. Redis 장애 시 `null` 반환(캐시 miss 처리)은 유지하되, miss인지 error인지 구분할 수 있게 됐다.

**댓글 수 업데이트 실패**: `updateCountSafely`에 best-effort 의도 주석과 스택트레이스를 추가했다.

---

### Finance — 레거시 코드 70줄 삭제 + 매직넘버 상수화

금융 도메인이 가장 많은 수정이 필요했다. 11개 파일.

**TODO + 주석처리된 코드 삭제**

| 파일 | 삭제 내용 |
|------|----------|
| Settlement.java | TODO + 주석 import + 주석 코드 |
| SettlementRepository.java | TODO + 주석 import |
| UserSettlementRepository.java | **~60줄 레거시 주석 코드 전체** |
| Transfer.java | TODO + 주석 import + 주석 코드 |

`UserSettlementRepository`에 주석처리된 쿼리가 60줄 가까이 남아 있었다. 순환 의존성 방지를 위해 임시로 주석처리한 것인데, ErrorCode 분리로 순환이 해소된 뒤에도 그대로 남아 있었다.

**매직넘버 상수화**

| 파일 | 매직넘버 | 상수명 |
|------|---------|--------|
| SettlementKafkaEventListener.java | `10000` | `OUTBOX_AGGREGATE_MULTIPLIER` |
| SettlementKafkaEventListener.java | `10` (seconds) | `FUTURE_TIMEOUT_SECONDS` |
| RedisLuaService.java | `5` | `MAX_RETRY_ATTEMPTS` |
| RedisLuaService.java | `5`, `20` | `RETRY_MIN_DELAY_MS`, `RETRY_MAX_DELAY_MS` |
| UserSettlementService.java | `10` | `WALLET_GATE_TTL_SECONDS` |

`settlementId * 10000 + participantId` 같은 식이 상수 없이 쓰이면 의도를 파악하기 어렵고, 오버플로우 위험도 보이지 않는다.

**@Slf4j 누락**: `OutboxAppender`, `FailedEventAppender`에 `@Slf4j`가 없어서 로깅이 불가능했다.

**var → 명시적 타입**: `KafkaProducerConfig`, `OutboxRelayService`에서 `var` → 구체 타입으로 변경.

---

### Search — 필드명 통일 + 중복 JOIN 제거

**DTO 필드명 불일치 (High)**: `ClubResponseDto`는 `interest`, `image` 필드를 쓰는데 `ClubSearchResult`는 `interestKoreanName`, `clubImage`였다. 통일했다.

| 파일 | 변경 |
|------|------|
| ClubSearchResult.java | `interestKoreanName` → `interest`, `clubImage` → `image` |
| ElasticsearchSearchAdapter.java | 매핑 필드명 갱신 |
| MysqlFulltextSearchAdapter.java | 매핑 필드명 갱신 |
| SearchService.java | 변환부 필드명 갱신 |

**하드코딩 switch → enum 활용**: `mapCategoryToKorean()`이 카테고리별 한국어 이름을 switch문으로 하드코딩하고 있었다. `Category.valueOf(category).getKoreanName()`으로 교체했다.

**중복 JOIN 제거**: `MysqlFulltextSearchAdapter`에서 `JOIN interest i ON ...`과 `LEFT JOIN interest cat ON ...`이 동시에 있었다. 전자는 사용처가 없어서 삭제했다.

**size 파라미터 의미 불명확**: `SearchService`에서 `size` 파라미터를 받지만 항상 `DEFAULT_PAGE_SIZE`를 사용하고 있었다. 파라미터명을 `sampleSize`로 변경하고 의도를 문서화했다.

---

## 수정 총괄

| 도메인 | 파일 수 | 주요 변경 |
|--------|---------|----------|
| Notification | 5 | 용어 정규화, 예외 로깅, var→타입 |
| Chat | 4 | 예외 타입 통일, 네이밍 일관성, 구분자 통일 |
| Feed | 6 | 중복 record 삭제, 캐시 키 상수화, 예외 로깅 |
| Finance | 11 | 레거시 70줄 삭제, 매직넘버 상수화, @Slf4j 추가 |
| Search | 4 | 필드명 통일, enum 활용, 중복 JOIN 제거 |
| **합계** | **30** | |

ErrorCode 분리 포함 전체 검증:
- `./gradlew compileJava` — 컴파일 성공
- `./gradlew test` — 전체 테스트 통과
- 인프라 의존 통합 테스트(Redis/Kafka/ES)만 기존과 동일하게 스킵

---

## 정리하며

성능 테스트를 돌리면서 코드를 급하게 고치면 기술 부채가 쌓인다. 트랜잭션 분리, 캐싱 추가, 인덱스 변경 — 각각은 올바른 수정이지만, 그 과정에서 매직넘버가 늘고, 사일런트 예외가 생기고, 주석처리된 코드가 남는다.

두 가지를 느꼈다.

**ErrorCode 같은 공유 자원은 초기에 분리해야 한다.** 150줄짜리 단일 enum이 10개 모듈의 의존 관계를 만들고 있었다. 인터페이스로 분리하니 각 모듈이 자기 에러 코드만 관리하게 됐다. 순환 의존성도 에러 코드 중복 정의로 끊을 수 있었다.

**코드 품질 정리는 성능 테스트 직후가 적기다.** 성능 테스트에서 코드를 많이 건드렸으니 어디가 지저분한지 기억이 선명하다. 시간이 지나면 `UserSettlementRepository`의 주석 60줄이 왜 있는지, `settlementId * 10000`이 무슨 의미인지 잊어버린다.

---

## 시리즈 탐색

**◀ 이전 글**
[채팅 도메인 — Storage 추상화, WHERE IN 풀스캔, WebSocket 1,000VU 극한 테스트](/chat-storage-websocket-extreme-test/)

**▶ 다음 글**
[알림 도메인 — Port/Adapter Storage 추상화, MySQL의 벽, MongoDB 1,061배 차이](/notification-storage-abstraction-mysql-mongodb/)
