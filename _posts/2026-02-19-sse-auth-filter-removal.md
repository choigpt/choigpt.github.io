---
title: SseAuthenticationFilter 제거 — SSE 인증을 필터 하나로 단일화
date: 2026-02-19
tags: [Spring Security, JWT, SSE, 트러블슈팅, Java]
permalink: /sse-auth-filter-removal/
---

# SseAuthenticationFilter 제거 — SSE 인증을 필터 하나로 단일화

## 개요

[SSE 인증 실패 편](/jwt-sse-auth-filter-mismatch/)에서 `SseAuthenticationFilter`의 `subject`/`kakaoId` 불일치 버그를 `JwtTokenParser.parseToken()`으로 파싱 위임하는 방식으로 수정하고, `JwtAuthenticationFilter`에 `shouldNotFilter`를 추가해 `/sse/**` 경로의 이중 처리를 방지했다.

패치는 됐지만 코드를 다시 보니 구조 자체가 불필요하게 복잡했다. SSE 전용 필터와 일반 요청 필터, 두 개의 JWT 필터가 서로 경로를 나눠 갖고 있었다. `SseAuthenticationFilter`가 존재해야 하는 이유는 브라우저의 `EventSource` API가 커스텀 헤더를 지원하지 않아 SSE 연결 시 쿠키로 토큰을 전달해야 한다는 것이었다. 하지만 이 쿠키 읽기 로직은 `SseAuthenticationFilter`만의 것이 아니라 `JwtAuthenticationFilter`에도 들어갈 수 있었다.

`SseAuthenticationFilter`를 완전히 제거하고, SSE 경로의 쿠키 인증 로직을 `JwtAuthenticationFilter`에 통합했다.

---

## 기존 구조의 문제

### 두 필터가 같은 일을 나눠서 함

```
/sse/**   → SseAuthenticationFilter → 쿠키 or 헤더에서 JWT 추출 → 인증 처리
           → JwtAuthenticationFilter (shouldNotFilter로 스킵)

/api/**   → SseAuthenticationFilter (shouldNotFilter로 스킵)
           → JwtAuthenticationFilter → Authorization 헤더에서 JWT 추출 → 인증 처리
```

두 필터가 각각 "어떤 경로는 내가 처리, 어떤 경로는 건너뜀" 로직으로 서로를 회피하고 있었다. 파싱 로직은 `JwtTokenParser`로 통일됐지만 토큰 추출과 인증 처리 코드가 두 곳에 흩어져 있었다.

### `shouldNotFilter` 조건 관리 부담

```java
// JwtAuthenticationFilter.java
@Override
protected boolean shouldNotFilter(HttpServletRequest request) {
    return request.getRequestURI().startsWith("/sse/");
}

// SseAuthenticationFilter.java
@Override
protected boolean shouldNotFilter(HttpServletRequest request) {
    return !request.getRequestURI().startsWith("/sse/");
}
```

경로 패턴이 변경될 때마다 두 필터를 동시에 수정해야 했다. [SSE 인증 실패 편](/jwt-sse-auth-filter-mismatch/)에서 이 조합이 실제로 한 번 어긋나 이중 실행이 발생했었다.

### `@Component` 필터 이중 등록 위험 상존

[SSE 인증 실패 편](/jwt-sse-auth-filter-mismatch/)에서 `FilterRegistrationBean`으로 서블릿 컨테이너 이중 등록을 방지했지만, `SseAuthenticationFilter`가 `@Component`로 빈에 등록되어 있는 한 설정 실수로 다시 이중 등록될 위험은 남아 있었다.

---

## 변경 사항

### `JwtAuthenticationFilter`에 쿠키 읽기 통합

```java
// JwtAuthenticationFilter.java
@Override
protected void doFilterInternal(HttpServletRequest request,
                                HttpServletResponse response,
                                FilterChain filterChain) throws ServletException, IOException {

    String token = extractToken(request);  // 헤더 or 쿠키

    if (token == null) {
        filterChain.doFilter(request, response);
        return;
    }

    // 이후 파싱 및 인증 처리 (기존 로직)
}

private String extractToken(HttpServletRequest request) {
    // 1. Authorization 헤더
    String header = request.getHeader("Authorization");
    if (header != null && header.startsWith("Bearer ")) {
        return header.substring(7);
    }
    // 2. 쿠키 (SSE EventSource 등 헤더 설정 불가 클라이언트)
    if (request.getCookies() != null) {
        for (Cookie cookie : request.getCookies()) {
            if ("accessToken".equals(cookie.getName())) {
                return cookie.getValue();
            }
        }
    }
    return null;
}
```

토큰 추출 우선순위: Authorization 헤더 → 쿠키. SSE 연결이든 일반 API 요청이든 동일한 필터를 거친다.

### `shouldNotFilter` 제거

경로 기반으로 필터를 건너뛰는 조건이 더 이상 필요 없다. `JwtAuthenticationFilter`가 모든 요청의 토큰 추출을 담당한다. 토큰이 없으면 인증 없이 통과시키고, `SecurityConfig`의 `authorizeHttpRequests`가 이후 접근 제어를 처리한다.

### `SseAuthenticationFilter` 삭제

파일 자체를 삭제했다. `SecurityConfig`의 필터 체인 등록 코드, `FilterRegistrationBean`, `shouldNotFilter` 조건도 모두 제거했다.

```java
// SecurityConfig.java — 수정 전
.addFilterBefore(sseAuthenticationFilter, JwtAuthenticationFilter.class)
.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)

// SecurityConfig.java — 수정 후
.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
```

---

## 결과

필터 체인이 단순해졌다. SSE 연결과 일반 API 요청 모두 `JwtAuthenticationFilter` 하나를 거친다. 경로별로 필터를 나눠야 하는 이유가 없었고, 그 분리가 오히려 [Spring Security·JWT 편](/spring-security-jwt-structure-bugs/)의 버그 원인 중 하나였다.

`SseAuthenticationFilter`가 제거되면서 [SSE 인증 실패 편](/jwt-sse-auth-filter-mismatch/)에서 언급한 `FilterRegistrationBean`도 함께 정리됐다. `@Component`를 붙인 필터가 두 곳에 자동 등록되는 문제 자체가 사라졌다.

---

## 교훈

> **분리가 복잡성을 줄이는지, 오히려 늘리는지 확인하라.**
> SSE 전용 필터를 별도로 두는 게 관심사 분리처럼 보였지만, 실제로는 같은 인증 로직이 두 곳에 흩어지고 경로 조건을 두 필터가 동시에 관리해야 하는 부담이 생겼다. 하나의 필터가 쿠키와 헤더를 모두 처리하는 쪽이 훨씬 단순했다.

---

## 시리즈 탐색

**◀ 이전 글**  
[JWT 인증 흐름 불일치로 인한 SSE 인증 실패 버그](/jwt-sse-auth-filter-mismatch/)

**다음 글 ▶**  
[알림 SSE 구현 중 발견한 연쇄 버그와 구조 개선](/sse-notification-chained-bugs/)
