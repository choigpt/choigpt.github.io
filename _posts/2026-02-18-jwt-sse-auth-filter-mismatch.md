---
layout: post
title: JWT 인증 흐름 불일치로 인한 SSE 인증 실패 버그
date: 2026-02-18
tags:
  - JWT
  - SSE
  - 트러블슈팅
  - Java
permalink: /jwt-sse-auth-filter-mismatch/
excerpt: "토큰을 생성하는 쪽과 파싱하는 쪽이 서로 다른 JWT 필드를 참조해 SSE 연결이 전면 불통이 된 버그. 단위·통합·부하 테스트 어디서도 잡히지 않았다."
---

## 개요

Spring Security + JWT 기반 인증 시스템에서 **토큰을 생성하는 쪽과 파싱하는 쪽이 서로 다른 JWT 필드를 참조**하여 SSE 연결 시 인증이 실패하는 버그가 존재했다.

이 버그는 단위 테스트, 통합 테스트, k6 부하 테스트 **모두에서 검출되지 않았다**.

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

실제 데이터에서 userId(DB PK)는 1이고, kakaoId는 3847561234이다. 이 두 값은 완전히 다르다.

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

`JwtAuthenticationFilter`가 설정한 principal은 `kakaoId`인데, `getCurrentUserId()`라는 메서드명 때문에 호출부에서 DB PK로 착각하고 `WHERE uc.user.userId = :userId` 쿼리에 넘기고 있다.

---

## 왜 테스트에서 검출되지 않았는가

**단위/통합 테스트:** `@Profile("!test")`로 SecurityConfig가 미로드되어 필터 체인 자체가 실행되지 않았다. 인증 로직이 통째로 빠진 상태에서 테스트한 셈이다.

**테스트 토큰 구조:** 테스트용 토큰에서 `subject`와 `kakaoId` claim에 동일한 값을 넣었거나 우연히 일치하는 값을 사용해서 버그가 숨겨졌다.

**k6 부하 테스트:** 이 버그는 기능 정확성 문제이지 성능 문제가 아니다. k6는 응답 시간과 처리량을 측정하는 도구이므로, "401이 돌아오는데 빠르게 돌아온다"면 k6 관점에서는 이상 없다. 상태 코드 검증 assertion을 넣지 않는 한 이런 류의 버그는 부하 테스트에서 보이지 않는다.

다만 이 버그에는 기능 문제와 별개로 **설계 문제**도 있었다. Spring Security + JWT 구조에서 인증 필터가 기대하는 흐름은 "토큰 파싱 → claims에서 사용자 정보 추출 → `SecurityContext`에 `Authentication` 설정"이다. JWT에 필요한 정보가 이미 들어있으므로 DB를 칠 이유가 없다. 그런데 `SseAuthenticationFilter`는 매 요청마다 `userRepository.findByKakaoId()`를 호출하고 있었다. 인증은 모든 요청이 통과하는 경로이므로, 동시 접속이 늘어나면 DB 커넥션 풀이 인증 조회만으로 소진되어 비즈니스 쿼리가 커넥션을 잡지 못하게 된다. 이 경우에는 부하 테스트에서 시스템이 멈추는 것으로 드러난다. 테스트 환경의 동시 접속 수가 적었기 때문에 표면화되지 않았을 뿐이다.

정리하면, **부하 테스트는 기능 테스트를 대체하지 않는다.** 하지만 인증 필터에서 매 요청마다 DB를 조회하는 것 자체가 설계 오류이고, 이런 류의 설계 오류는 동시 접속이 늘어나면 커넥션 풀 고갈로 부하 테스트에서 잡힌다.

---

## 해결

`SseAuthenticationFilter`가 직접 claims를 파싱하지 않고 공통 `JwtTokenParser.parseToken()`에 위임하는 구조로 변경했다. 파싱 로직이 한 곳에만 존재하게 되어 불일치 재발 가능성을 줄였다.

`JwtAuthenticationFilter`에는 `/sse/**` 경로에 대해 `shouldNotFilter`를 추가해 두 필터가 같은 요청을 이중 처리하는 상황을 방지했다. 두 필터 구조 자체의 복잡성 문제는 [필터 단일화 편](/sse-auth-filter-removal/)에서 `SseAuthenticationFilter`를 제거하면서 함께 해소된다.

---

## 정리하며

> **Mock으로 우회한 부분은 테스트되지 않은 부분이다.**
> 인증처럼 시스템 전반에 영향을 미치는 횡단 관심사는 **실제와 동일한 흐름을 한 번이라도 통과시키는 통합 테스트**가 반드시 필요하다. 성능 수치 측정과 기능 정확성 검증은 서로를 대체하지 않는다.

---

## 시리즈 탐색

**◀ 이전 글**
[Spring Security + JWT 구조 전반에 걸친 버그 및 설계 문제](/spring-security-jwt-structure-bugs/)

**▶ 다음 글**
[SseAuthenticationFilter 제거 — SSE 인증을 필터 하나로 단일화](/sse-auth-filter-removal/)
