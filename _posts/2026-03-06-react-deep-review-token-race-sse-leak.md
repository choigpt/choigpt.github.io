---
title: React 심화 리뷰 — 토큰 레이스 컨디션, SSE 메모리 누수, isProtectedRoute 보안 버그
date: 2026-03-06
tags: [React, TypeScript, SSE, WebSocket, STOMP, 보안, 트러블슈팅, 테스트]
permalink: /react-deep-review-token-race-sse-leak/
excerpt: "1·2차 최적화 후 심화 리뷰에서 종합 6.8/10. 동시 401 시 refresh 중복 호출, SSE 이벤트 리스너 메모리 누수, Auth 초기화 레이스, 채팅 구독 누수 등 Critical 4건을 포함해 12건을 수정하고 테스트를 80 → 149건으로 확대했다. isProtectedRoute에서 '/'의 startsWith가 모든 라우트를 public으로 만드는 보안 버그도 발견했다."
---

## 개요

[이전 글](/react-frontend-optimization-memo-splitting/)에서 코드 스플리팅, ErrorBoundary, React.memo 등 성능 최적화를 마쳤다. 이번에는 LLM에 기능 정합성과 안정성 심화 리뷰를 요청했다. 종합 6.8/10.

코드 스플리팅과 lazy loading은 잘 적용됐지만, 동시성 문제와 메모리 누수가 남아 있었다. 토큰 갱신 레이스 컨디션, SSE 이벤트 리스너 메모리 누수, Auth 초기화 레이스, 채팅방 구독 누수 — 4가지 Critical 이슈를 포함해 12건을 수정했다.

---

## 심화 리뷰 — 6.8/10

| 영역 | 점수 | 상태 |
|------|------|------|
| API 레이어 | 6.5/10 | 토큰 갱신 동시성 문제 |
| 라우터 | 8.5/10 | lazy loading 잘 구성됨 |
| Auth Context | 6/10 | 레이스 컨디션 |
| 컴포넌트 구조 | 7/10 | 일부 과도한 복잡성 |
| 타입 안전성 | 6.5/10 | ~40% any 사용 |
| SSE 서비스 | 6/10 | 메모리 누수 위험 |
| 채팅 소켓 | 7.5/10 | 에러 핸들링 양호 |
| 테스트 | 4/10 | 커버리지 ~6% |
| 빌드 설정 | 8.5/10 | 코드 스플리팅 우수 |

---

## Critical 수정 — 4건

### 1. API 토큰 갱신 레이스 컨디션

동시에 여러 요청이 401을 받으면 refresh 호출이 중복 발생했다.

```
요청 A → 401 → refresh 시작
요청 B → 401 → refresh 또 시작  ← 충돌
```

하나의 shared promise로 대기시키는 Mutex 패턴을 적용했다. 첫 번째 401이 refresh를 시작하면, 나머지 요청은 같은 promise를 await한다.

### 2. SSE 이벤트 리스너 메모리 누수

`addEventListener`로 등록된 콜백이 인라인 arrow function이면 `removeEventListener`로 제거할 수 없다. 컴포넌트 언마운트 후에도 콜백이 계속 호출됐다. 콜백 참조를 변수에 저장하고 cleanup에서 같은 참조로 제거하도록 수정했다.

### 3. Auth 초기화 레이스 컨디션

`setIsAuthenticated(true)` → `getCurrentUser()` 사이에 컴포넌트가 언마운트되면 state 업데이트가 시도됐다. `AbortController`로 요청을 취소하고, 상태를 일괄 업데이트하도록 변경했다.

### 4. 채팅방 구독 누수

방을 변경할 때 이전 구독을 해제하지 않고 새 구독을 생성하고 있었다. 이전 방의 메시지가 새 방에 표시될 수 있었다. 방 변경 시 기존 STOMP subscription을 `unsubscribe()` 한 뒤 새 구독을 생성하도록 수정했다.

---

## Moderate 수정 — 4건

### 5. API 에러 핸들러 미작동

`setErrorHandler()`로 등록은 되지만 실제로 호출되는 곳이 없었다. 글로벌 에러 처리 패턴이 동작하지 않는 상태였다.

### 6. CommentSection 양방향 데이터 흐름

`onCommentsUpdate` 콜백으로 부모↔자식 간 댓글 상태를 동기화하고 있어 stale data 문제가 발생했다. `CommentSection`이 자체 state로 댓글을 관리하고, 부모에는 `onCommentCountChange?(count)` 단방향 콜백만 전달하도록 변경했다.

### 7. SSE JSON 파싱 미검증

서버에서 malformed JSON이 오면 핸들러 전체가 크래시했다. `try-catch` + 스키마 검증을 추가했다.

### 8. 토큰 갱신 실패 시 무시

`refreshRes.success === false`인 경우 아무 처리 없이 원래 에러를 throw하고 있었다. 명시적 로그아웃 처리를 추가했다.

---

## Minor 수정

- 빈 파일 4개 삭제: `format.ts`, `validation.ts`, `axios.ts`, `meeting.api.ts`
- 오타 수정: `MeetingScheduleCrate` → `MeetingScheduleCreate`
- `MeetingDetail.tsx`, `MeetingFeedDetail.tsx`에 AbortController 추가
- `FeedData` 재귀 `any` 타입 → `FeedBase` / `NestedFeed` 타입 계층 정의

---

## TS 빌드 에러 수정 — 41개 → 0개

타입 정의 3건을 수정했다.

| 파일 | 변경 |
|------|------|
| user.api.ts | `userId: string` → `number` |
| chat.ts | `PagedResponse<ChatRoomSummary[]>` → `ChatRoomSummary[]` (이중 배열 수정) |
| sse.ts | EventSource addEventListener 타입 캐스팅 수정 |

14개 파일에 API 호출 제네릭 타입을 추가하고, 코드 레벨 수정 6건을 처리했다.

| 파일 | 변경 |
|------|------|
| Notice.tsx | `handleClick` 구현 — 알림 타입별 페이지 라우팅 |
| Search.tsx | 미사용 `useNavigate` 제거 |
| ChatRoom.tsx | `currentUserId!` → `currentUserId ?? 0` (non-null assertion 제거) |
| ParticipationStatus.tsx | `as any` → 타입 안전한 캐스팅 |
| ScheduleList.tsx | 일정 삭제 기능 구현 (API 호출 + 모달 확인 + 목록 새로고침) |

---

## 추가 테스트 — 80 → 149

| # | 테스트 파일 | 테스트 수 | 검증 내용 |
|---|-----------|-----------|----------|
| 1 | client.test.ts | 13 | 요청/응답 인터셉터, 토큰 리프레시 mutex, HTTP 메소드 5종 |
| 2 | upload.test.ts | 12 | 파일 타입 검증, presigned URL, S3 업로드 |
| 3 | useDynamicTitle.test.ts | 8 | 채팅방/모임 제목 로딩, API 실패 폴백, abort cleanup |
| 4 | Toast.test.tsx | 8 | ToastContext, showToast, 자동닫기, 타입별 스타일 |
| 5 | ErrorBoundary.test.tsx | 6 | 에러 catch, 기본/커스텀 fallback, 로깅 |
| 6 | KakaoCallback.test.tsx | 7 | OAuth 콜백 전체 플로우 (성공/실패/신규사용자/403) |
| 7 | kakaoAuth.test.ts | 5 | Kakao SDK 초기화, 로그인 리다이렉트 |
| 8 | getNotificationMessage.test.ts | 5 | 알림 메시지 생성 (settlement, comment, like, chat) |

12 → 20 suites, 80 → **149 tests** (+69개).

---

## 보안 버그 — isProtectedRoute

3차 평가 과정에서 보안 결함을 확인했다.

```typescript
// src/utils/auth.ts
const publicRoutes = ['/login', '/signup', '/auth/kakao/callback', '/'];
return !publicRoutes.some(route => pathname.startsWith(route));
```

`'/'`가 `publicRoutes`에 포함되어 있고 `startsWith('/')`는 **모든 pathname에 매칭**된다. 결과적으로 모든 라우트가 public으로 판정되어 보호 라우트가 작동하지 않았다.

```typescript
// 수정
export const isProtectedRoute = (pathname: string): boolean => {
  const publicRoutes = ['/login', '/signup', '/auth/kakao/callback'];
  if (pathname === '/') return false;  // 정확한 매칭
  return !publicRoutes.some(route => pathname.startsWith(route));
};
```

---

## 정리하며

성능 최적화(코드 스플리팅, memo)만으로는 부족하다. 동시성 버그와 메모리 누수는 부하 상태에서 발생한다. 토큰 갱신 레이스는 동시 401이 발생할 때 재현되고, SSE 메모리 누수는 페이지를 여러 번 이동할 때 누적된다. 채팅 구독 누수는 방을 빠르게 전환할 때 발생한다.

`isProtectedRoute`의 `startsWith('/')` 보안 버그는 [백엔드 시리즈](/spring-security-jwt-structure-bugs/)에서 다뤘던 `JwtException` 패키지 불일치와 같은 패턴이다. 컴파일도 되고 테스트도 통과하는데, 런타임에서 전혀 다른 동작을 한다.

> **`startsWith`로 경로를 비교할 때 `'/'`를 포함하면 안 된다.** 모든 경로가 `/`로 시작하므로 모든 것이 매칭된다.

---

## 시리즈 탐색

**◀ 이전 글**
[React 프론트엔드 리팩토링 — memo 0건, 코드 스플리팅 0건에서 시작한 최적화](/react-frontend-optimization-memo-splitting/)

**▶ 다음 글**
[React 코드 품질 — C(68점)에서 B(82점)으로, React Query 도입과 컴포넌트 분리](/react-code-quality-c-to-b-react-query/)
