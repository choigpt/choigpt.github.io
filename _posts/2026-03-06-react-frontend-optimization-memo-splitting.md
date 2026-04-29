---
title: React 프론트엔드 리팩토링 — memo 0건, 코드 스플리팅 0건에서 시작한 최적화
date: 2026-03-06
tags: [React, TypeScript, Vite, 성능최적화, 코드스플리팅, 리팩토링, SSE, k6]
permalink: /react-frontend-optimization-memo-splitting/
excerpt: "백엔드 최적화 시리즈를 마치고 같은 프로젝트의 React 프론트엔드를 점검했다. React.memo 0건, useMemo 0건, 코드 스플리팅 0건, FeedList 1,037줄. 종합 5.6/10에서 시작해 4단계 1차 최적화와 4단계 2차 최적화를 거치고, SSE delivery 부하 테스트까지 통과시킨 기록."
---

## 개요

[백엔드 성능 최적화 시리즈](/spring-security-jwt-structure-bugs/)를 마치고, 같은 프로젝트의 React 프론트엔드를 점검했다. React 19, Vite 7, Tailwind v4, Zod 등 최신 스택을 사용하고 있었지만, 성능 최적화 관행이 거의 적용되지 않은 상태였다.

LLM에 코드 리뷰를 맡긴 결과 종합 **5.6 / 10**. `React.memo()` 사용 0건, `useMemo()` 사용 0건, 코드 스플리팅 없음, FeedList.tsx 1,037줄. 기능적으로 동작하지만 최적화가 많이 필요한 상태에서, 8단계에 걸친 1·2차 최적화를 진행했다.

---

## 초기 코드 리뷰 — 5.6/10

### 잘한 점

- **프로젝트 구조**: `api/`, `components/`, `contexts/`, `hooks/`, `pages/`, `services/` 분리
- **API 레이어**: axios 싱글턴 + 인터셉터, 토큰 자동 첨부, 401 시 refresh, 일관된 에러 매핑
- **SSE 서비스**: 지수 백오프 재연결, visibility change 핸들링, Last-Event-ID 추적
- **WebSocket (채팅)**: STOMP/SockJS 기반, 재연결 + jitter, 메시지 버퍼링, 언마운트 시 정리
- **기술 스택**: React 19, Vite 7, Tailwind v4, Zod

### 주요 문제점

**1. 성능 — 주요 문제**

- `React.memo()` 사용 0건 — 모든 컴포넌트가 부모 렌더 시 무조건 리렌더
- `useMemo()` 사용 0건 — 비싼 계산도 매 렌더마다 재실행
- 코드 스플리팅 없음 — `React.lazy()` 미사용, 모든 페이지가 초기 번들에 포함
- 가상 스크롤 없음 — 피드 리스트가 50+ 항목을 한번에 렌더링

**2. 컴포넌트 비대화**

- **FeedList.tsx** — 1,037줄 (CRITICAL)
- **ScheduleList.tsx** — 533줄 (HIGH)
- **MeetingForm.tsx** — 393줄 (HIGH)
- **ChatRoom.tsx** — 390줄 (HIGH)

**3. TypeScript 설정**

```json
{
  "strict": false,
  "noImplicitAny": false
}
```

`tsconfig.json`에서 엄격 모드가 꺼져 있었다. `tsconfig.app.json`에는 `strict: true`가 있지만, 상위 설정이 덮어쓰고 있었다.

**4. 기타**

- `console.log` 디버깅 코드가 프로덕션에 남아 있음
- Error Boundary 없음 — 하나의 컴포넌트 에러가 전체 앱 크래시
- 중복 의존성: `stompjs` + `@stomp/stompjs`, `eventsource` + `event-source-polyfill`

---

## 1차 최적화 — 리스크 낮은 순서부터

### Phase 1: Cleanup

- `stompjs`, `eventsource` 중복 의존성 제거 (사용처 없음, 각각 `@stomp/stompjs`와 `event-source-polyfill`만 사용)
- `console.log` **31건** 제거 (18개 파일). `console.error`/`console.warn`은 유지
- `tsconfig.json`의 `compilerOptions` 블록 제거 → `tsconfig.app.json`의 `strict: true`가 실제로 적용됨
- `Router.tsx`에서 중복 `chat/:chatRoomId/messages` 라우트 제거

### Phase 2: Error Boundary

`src/components/common/ErrorBoundary.tsx`를 class 컴포넌트로 생성했다. `getDerivedStateFromError` + `componentDidCatch`로 에러를 catch하고, 리셋 버튼으로 복구할 수 있다. `Router.tsx`에서 `DefaultLayout`, `SearchLayout`, `TitleLayout` 각각을 `<ErrorBoundary>`로 감쌌다.

### Phase 3: Code Splitting — 초기 번들 분리

30개 이상의 페이지 import를 `React.lazy()`로 전환했다. named export는 `.then()` 래핑이 필요하고, default export는 단순 `lazy()`를 사용한다.

3개 레이아웃 컴포넌트 내 `<Outlet />`을 `<Suspense>`로 감싸고, 레이아웃 없는 라우트(Login, Signup, KakaoCallback)는 `Router.tsx`에서 직접 `<Suspense>` 래핑했다.

정적 import로 유지한 것: `ProtectedRoute`, 3개 Layout, `ErrorBoundary` — 동기 로드가 필수인 컴포넌트.

### Phase 4: React.memo

- 채팅 메시지 컴포넌트 2개(`MyChatMessage`, `OtherChatMessage`) → `React.memo` 래핑
- `MeetingCard` → `React.memo` 래핑
- `FeedList.tsx`의 핸들러 6개를 `useCallback`으로 안정화 + `FeedItem`에 `React.memo` 적용

### 1차 검증

- `npx tsc --noEmit` — clean, no errors
- `npm run build` — 성공, 각 lazy-loaded 페이지가 별도 chunk로 분리 확인
- `npm test` — 12 suites, 80 tests 전부 통과

---

## 2차 최적화 — 성능 테스트 전 추가 개선

### Phase 1: 미사용 의존성 + 빌드 최적화

`date-fns`와 `react-toastify`를 제거했다. `src/` 전체에서 import 0건이었고, 각각 native Date와 커스텀 Toast 시스템이 이미 사용 중이었다.

`vite.config.ts`에 `manualChunks`를 추가해 vendor 코드를 분리했다.

- **vendor-react** (react, react-dom, react-router-dom) — 89KB
- **vendor-stomp** (@stomp/stompjs, sockjs-client) — 69KB
- **vendor-payment** (@tosspayments/*) — 3.5KB
- **vendor-forms** (react-hook-form, @hookform/resolvers, zod) — 0.04KB

### Phase 2: React 렌더링 최적화

**FeedList 콜백 의존성 수정**: `handleLikeClick`과 `handleCommentClick`의 `useCallback` 의존성에 `feeds`가 있어, 피드 상태 변경마다 콜백이 재생성되고 memo된 `FeedItem` 20개가 전부 리렌더됐다. `feedsRef` 패턴으로 해결했다.

```javascript
const feedsRef = useRef(feeds);
feedsRef.current = feeds;

// 의존성 배열에서 feeds 제거 → []
const handleLikeClick = useCallback(() => {
  feedsRef.current.find(...)  // feeds 대신 ref 사용
}, []);
```

**정적 상수 모듈 스코프 추출**: `Search.tsx`의 interests 배열, `Notice.tsx`의 `getNotificationIcon`/`getNotificationTypeText`/`getNotificationTypeColor` 함수를 컴포넌트 바깥으로 이동했다. state/props를 사용하지 않는 함수는 렌더 사이클 밖에 있어야 한다.

**스크롤 핸들러 쓰로틀**: `ChatRoom.tsx`와 `Search.tsx`의 스크롤 이벤트에 `requestAnimationFrame` 쓰로틀 + `{ passive: true }` 옵션을 적용했다.

### Phase 3: TitleLayout 리팩토링

`src/components/layout/title/Layout.tsx`에 240줄짜리 `switch(true)` 블록이 있었다. 경로별 헤더 설정을 전부 if/else로 분기하고 있었다.

`routeHeaderConfig.ts`로 분리했다. 정적 라우트는 `Record<string, HeaderConfig>`로 O(1) 조회, 동적 라우트는 `RegExp` 매칭으로 처리한다. `resolveHeaderProps()` 한 줄로 240줄 switch를 대체했다.

헤더의 동적 제목(채팅방 이름, 모임 이름)은 `useDynamicTitle` 커스텀 훅으로 추출했다. `AbortController`도 함께 적용해 컴포넌트 언마운트 시 fetch가 취소된다.

### Phase 4: AbortController

`ScheduleList`, `ParticipationStatus`, `Notice`, `ChatRoom`의 초기 데이터 로드에 `AbortController`를 적용했다. 컴포넌트 언마운트 시 진행 중인 API 요청이 취소되고, abort된 경우 state 업데이트를 건너뛴다.

### 2차 검증

- `npx tsc --noEmit` — clean
- `npm run build` — 성공 (2.32s)
- `npm test` — 12 suites, 80 tests 전부 통과

---

## SSE Delivery 부하 테스트

프론트엔드 최적화 후 SSE 실시간 전송을 200 receivers / 100 senders / 90초로 부하 테스트했다.

- **연결 성공률**: 100% (600/600), threshold >95% — PASS
- **전송 성공률**: 100% (41,180 msgs), threshold >90% — PASS
- **API p95**: 18.8ms, threshold <1000ms — PASS
- **Latency p50**: 243ms, threshold <500ms — PASS
- **Latency p95**: 478ms, threshold <2000ms — PASS
- **Latency max**: 512ms

6/6 threshold 전부 PASS. k6 Docker(UTC)와 서버(KST+9)의 타임존 차이로 latency가 0ms로 잡히는 문제가 있었는데, `createdAt`을 epoch ms로 전환해 해결했다.

---

## 수정 내역 요약

1. **중복 의존성 제거** (stompjs, eventsource, date-fns, react-toastify) — 번들 경량화
2. **console.log 31건 제거** (18파일) — 프로덕션 정리
3. **tsconfig strict 활성화** — 타입 안전성 확보
4. **ErrorBoundary 생성 + 레이아웃별 적용** — 앱 안정성
5. **30+ 페이지 React.lazy + Suspense** — 초기 번들 분리
6. **React.memo + useCallback 적용** — 불필요 리렌더 방지
7. **manualChunks vendor 분리** — 캐시 효율
8. **feedsRef 패턴으로 콜백 안정화** — FeedItem 20개 리렌더 방지
9. **TitleLayout 240줄 switch → config + hook** — 유지보수성
10. **AbortController 4개 페이지 적용** — 언마운트 시 요청 취소

---

## 정리하며

`React.memo`, `React.lazy`, 코드 스플리팅 — React 성능 최적화의 기본이지만 하나도 적용되지 않은 상태에서 시작했다. 1,037줄짜리 `FeedList.tsx`는 이미지 캐러셀, 좋아요, 삭제, 댓글을 전부 한 컴포넌트에서 처리하고 있었다.

1차 최적화에서 가장 높은 개선 폭을 보인 항목은 코드 스플리팅이었다. 30개 이상의 페이지가 초기 번들에 포함되어 있던 것을 `React.lazy`로 전환하니 각 페이지가 별도 chunk로 분리됐다. 2차 최적화에서는 `feedsRef` 패턴이 주요 변경점이었다. `useCallback` 의존성에 `feeds` 배열이 있으면 피드 상태가 바뀔 때마다 콜백이 재생성되고, memo된 자식 20개가 전부 리렌더된다.

> **성능 최적화는 빌드 설정(코드 스플리팅)과 렌더링 패턴(memo + useCallback) 두 축으로 나뉜다.** 전자는 초기 로드를, 후자는 인터랙션 시 성능을 개선한다. 두 축 모두 적용되지 않았다면, 코드 스플리팅부터 시작하는 게 ROI가 가장 높다.

---

## 시리즈 탐색

**◀ 이전 글**
[검색 엔진 추상화 — ES vs MySQL FULLTEXT, 236배 차이의 기록](/search-engine-abstraction-es-vs-fulltext/)

**▶ 다음 글**
[React 심화 리뷰 — 토큰 레이스 컨디션, SSE 메모리 누수, isProtectedRoute 보안 버그](/react-deep-review-token-race-sse-leak/)
