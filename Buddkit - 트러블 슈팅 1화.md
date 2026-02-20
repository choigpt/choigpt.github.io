---
title: JWT 인증 흐름 불일치로 인한 SSE 인증 실패 버그
date: 2026-02-19
tags: [Spring Security, JWT, SSE, 트러블슈팅, Java]
---

# JWT 인증 흐름 불일치로 인한 SSE 인증 실패 버그

## 개요

Spring Security + JWT 기반 인증 시스템을 구현하고 코드를 다시 살펴보던 중, **토큰을 생성하는 쪽과 파싱하는 쪽이 서로 다른 JWT 필드를 참조**하여 SSE 연결 시 인증이 실패하는 버그를 발견했다.

가장 골치 아팠던 점은 단위 테스트, 통합 테스트, k6 부하 테스트 **모두에서 검출되지 않았다**는 것이다. 실제 카카오 로그인 토큰으로만 재현되는 구조적 문제였다.

---

## 시스템 구조

### 인증 흐름

```
카카오 로그인 → UserService.generateAccessToken() → JWT 발급
    ↓
요청 시 JWT 전달
    ↓
SseAuthenticationFilter (/sse/** 전용, 쿠키+헤더 지원)
    ↓
JwtAuthenticationFilter (일반 요청)
    ↓
SecurityContextHolder에 principal 설정
    ↓
각 서비스에서 userService.getCurrentUser()로 현재 유저 조회
```

### 필터 체인 순서 (SecurityConfig)

```java
.addFilterBefore(sseAuthenticationFilter, JwtAuthenticationFilter.class)
.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
```

`/sse/**` 요청은 `SseAuthenticationFilter → JwtAuthenticationFilter → Controller` 순서로 처리된다.

---

## 버그 상세

### 토큰 생성 (`UserService.generateAccessToken`)

```java
return Jwts.builder()
        .subject(user.getUserId().toString())      // subject = userId (DB PK)
        .claim("kakaoId", user.getKakaoId())       // claim = kakaoId (카카오 ID)
        .claim("nickname", user.getNickname())
        .claim("type", "access")
        .signWith(key, Jwts.SIG.HS512)
        .compact();
```

`subject`에는 **userId(DB PK)**, `"kakaoId"` claim에는 **카카오 ID**가 들어간다. 이 두 값은 다르다.

### 토큰 파싱 — `JwtAuthenticationFilter` (정상)

```java
String kakaoIdString = claims.get("kakaoId").toString();   // "kakaoId" claim에서 추출
// → principal = kakaoId ✓
```

### 토큰 파싱 — `SseAuthenticationFilter` (버그)

```java
String kakaoIdString = claims.getSubject();                // subject에서 추출 → 실제로는 userId!
Long kakaoId = Long.valueOf(kakaoIdString);
Optional<User> userOpt = userRepository.findByKakaoId(kakaoId);  // userId를 kakaoId로 조회
```

변수명은 `kakaoIdString`이지만, `claims.getSubject()`가 반환하는 값은 `userId`다. 이 값으로 `findByKakaoId()`를 호출하므로, `userId ≠ kakaoId`인 사용자는 조회 실패 → **401 Unauthorized**.

### 실제 데이터 예시

| 필드 | 값 |
|------|-----|
| userId (DB PK) | 1 |
| kakaoId | 3847561234 |

```
SseAuthenticationFilter:
  claims.getSubject()     → "1" (userId)
  findByKakaoId(1L)       → Empty (kakaoId가 1인 유저 없음)
  → 401 Unauthorized, return
  → JwtAuthenticationFilter 도달 불가
  → SSE 연결 실패
```

`SseAuthenticationFilter`가 실패 시 `filterChain.doFilter()`를 호출하지 않고 직접 401 응답을 반환하므로, 후속 `JwtAuthenticationFilter`가 보정해줄 기회도 없다.

---

## 연관 이슈: `SearchService`의 `getCurrentUserId()` 오용

같은 인증 체계에서 파생된 별도 문제도 발견되었다.

```java
// UserService
public Long getCurrentUserId() {
    Long userId = Long.valueOf(authentication.getName());  // 실제로는 kakaoId 반환
    return userId;
}

// SearchService
Long userId = userService.getCurrentUserId();                        // kakaoId를 받음
List<Long> joinedClubIds = userClubRepository.findByClubIdsByUserId(userId);  // userId 기대하는 쿼리
```

`JwtAuthenticationFilter`가 설정한 principal은 `kakaoId`인데, `getCurrentUserId()`라는 메서드명 때문에 호출부에서 DB PK로 착각하고 `WHERE uc.user.userId = :userId` 쿼리에 넘기고 있다. `kakaoId ≠ userId`인 사용자는 가입한 모임이 없는 것처럼 보이는 문제가 생긴다.

---

## 왜 테스트에서 검출되지 않았는가

### 1. 단위/통합 테스트 — 필터 체인 자체가 실행되지 않음

```java
@Profile("!test")      // test 프로필에서 SecurityConfig 미로드
public class SecurityConfig { ... }

@ActiveProfiles("test") // 모든 테스트가 test 프로필
class SseStreamControllerTest {
    @MockBean
    private UserService userService;

    given(userService.getCurrentUser()).willReturn(testUser);  // 인증 흐름 전체 우회
}
```

`SecurityConfig`가 로드되지 않으므로 두 필터 모두 체인에 등록되지 않는다. 요청이 필터를 거치지 않고 바로 컨트롤러에 도달하며, `getCurrentUser()`는 Mock이 응답한다.

테스트 코드 주석에서도 이미 인지하고 있었다:

```java
// 실제로는 SseAuthenticationFilter에서 401이 반환되어야 하지만
// 테스트 환경에서는 인증 필터가 우회되므로 현재 상태 확인
.andExpect(status().isOk()) // 실제: 401, 테스트: 200 (Mock 설정으로 인해)
```

### 2. 테스트 토큰 구조가 실제와 다름

```java
// 테스트의 generateTestToken()
.subject(kakaoId.toString())          // subject = kakaoId
.claim("kakaoId", kakaoId)            // claim = kakaoId  (subject == claim ← 버그 숨김)

// 실제 generateAccessToken()
.subject(user.getUserId().toString()) // subject = userId
.claim("kakaoId", user.getKakaoId()) // claim = kakaoId  (subject ≠ claim ← 버그 발현)
```

테스트 토큰에서는 `subject`와 `kakaoId` claim이 **동일한 값**이다. 어느 쪽을 읽든 같은 결과이므로 버그가 드러나지 않는다.

### 3. k6 부하 테스트 — 토큰 수동 생성

k6에서 JWT secret 키와 DB 데이터로 토큰을 직접 생성할 때 `subject`에 `kakaoId`를 넣었거나, 테스트 유저의 `userId`와 `kakaoId`가 우연히 일치했을 가능성이 높다. 결과적으로 401이 발생하지 않았다.

### 검출 실패 원인 요약

```
[실제 카카오 로그인] 토큰 생성(subject=userId) → getSubject → findByKakaoId(userId) → 실패
[단위 테스트]        필터 미실행 → Mock getCurrentUser → 성공
[k6 테스트]         토큰 수동 생성(subject=kakaoId 추정) → getSubject → findByKakaoId(kakaoId) → 성공
```

**실제 카카오 로그인에서만 발생하는 `subject ≠ kakaoId` 조건을 어떤 테스트도 재현하지 못했다.**

---

## 해결 — 리팩토링을 통한 파싱 로직 공통화

단순히 1줄을 고치는 것에서 나아가, 리팩토링 과정에서 **파싱 로직 자체를 공통화**하는 방향으로 해결했다.

### 토큰 생성 (`JwtTokenProvider:44`)

```java
return Jwts.builder()
        .subject(user.getUserId().toString())      // subject = userId (DB PK)
        .claim("kakaoId", user.getKakaoId())       // kakaoId는 별도 claim으로 저장
        ...
```

### 토큰 파싱 (`JwtTokenParser:59-68`) — 공통 파서 도입

```java
String userIdString  = claims.getSubject();              // subject → userId (올바름)
String kakaoIdString = claims.get("kakaoId").toString(); // claim   → kakaoId (올바름)
```

`subject`와 `kakaoId`의 역할이 명확하게 분리되어 있다.

### `SseAuthenticationFilter` — 직접 파싱 제거

```java
// 수정 전: 필터가 직접 claims를 파싱 → subject를 kakaoId로 오인
String kakaoIdString = claims.getSubject();

// 수정 후: 공통 파서에 위임 (line 51)
UserPrincipal principal = jwtTokenParser.parseToken(token);
```

`SseAuthenticationFilter`가 더 이상 claims를 직접 다루지 않고 `JwtTokenParser.parseToken()`을 호출해 `UserPrincipal`을 받는 구조로 변경되었다. 파싱 로직이 한 곳에만 존재하므로, 앞으로 같은 종류의 불일치 버그가 재발할 수 없다.

---

## 교훈

인증은 필터 → SecurityContext → 서비스로 이어지는 여러 컴포넌트의 협력으로 동작한다. 각 컴포넌트를 격리해서 테스트하면 개별적으로는 정상이지만, **연결 지점의 불일치**(`subject`에 `userId`를 넣고 `kakaoId`로 읽는 것)는 발견할 수 없다.

특히 테스트용 토큰이 실제 토큰과 구조가 다르면, 필터가 동작하더라도 버그가 숨겨진다.

돌아보면 시간이 빠듯한 상황에서 k6로 TPS, 응답시간 같은 수치를 뽑아내는 데 집중하다 보니 "테스트를 잘 하고 있다"는 착각에 빠졌다. 숫자는 나오는데 정작 인증 흐름 자체가 올바른지는 보지 못한 것이다. **성능 수치에 눈이 팔려 기능 정확성을 검증하지 못한 셈이다.**

> **Mock으로 우회한 부분은 테스트되지 않은 부분이다.**
> 인증처럼 시스템 전반에 영향을 미치는 횡단 관심사는 **실제와 동일한 흐름을 한 번이라도 통과시키는 통합 테스트**가 반드시 필요하다. 성능 테스트는 기능이 올바르게 동작한다는 확신이 생긴 이후에 하자.
