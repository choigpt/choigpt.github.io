---
title: Spring Security + JWT 구조 전반에 걸친 버그 및 설계 문제
date: 2026-02-17
tags:
  - JWT
  - OAuth2
  - 트러블슈팅
  - Java
permalink: /spring-security-jwt-structure-bugs/
excerpt: "컴파일과 테스트를 통과하지만 런타임에서 만료 토큰이 500을 반환하는 버그. import 패키지 불일치가 원인이었다."
---

> 이 글부터 2/24까지 8편에 걸쳐 약 60건의 버그를 다룬다. 부트캠프 5인 팀 프로젝트에서 나는 알림(SSE) 도메인을 맡았다. 프로젝트 종료 후, 내가 다루지 못한 Redis·Kafka·Elasticsearch 등을 공부하려고 다른 팀원들의 코드를 읽기 시작했고, 그 과정에서 발견한 버그들을 정리한 기록이다. 회고는 [유저 도메인 편](/user-lifecycle-bugs/)의 마지막 섹션에서.

---

## 한눈에 보기

```
이 글에서 다루는 문제 (6건):

1. JwtException 패키지 불일치 → 만료 토큰에서 500 에러  ← 핵심 버그
2. jjwt 이중 선언 + API 스타일 혼재
3. SecretKey charset 불일치 (UTF-8 vs 시스템 기본)
4. 로그아웃에서 unlink() 호출 (로그아웃이 아니라 탈퇴 동작)
5. SecurityConfig에 @Profile("!test") → 테스트에서 보안 미적용
6. 기타: withdraw 이중 호출, Refresh Token 미구현, kakaoId 노출
```

---

## 버그 1: `JwtException` 패키지 불일치 — 500 에러의 원인

### 증상

만료되거나 변조된 JWT 토큰으로 요청하면 401이 아니라 **500**이 반환된다.
컴파일 에러 없음. 테스트도 통과.

### 원인

```java
// JwtAuthenticationFilter.java
import org.springframework.security.oauth2.jwt.JwtException;  // ← Spring OAuth2의 클래스
import io.jsonwebtoken.Jwts;                                    // ← jjwt 라이브러리

try {
    Claims claims = Jwts.parser()...parseClaimsJws(token)...;
    //                   ↑ jjwt가 던지는 예외: io.jsonwebtoken.JwtException
} catch (JwtException | IllegalArgumentException e) {
    //     ↑ 잡으려는 예외: org.springframework.security.oauth2.jwt.JwtException
    //     → 완전히 다른 클래스! 잡히지 않는다!
    throw new CustomException(ErrorCode.UNAUTHORIZED);
}
```

```
예외 흐름:

jjwt가 던짐: io.jsonwebtoken.ExpiredJwtException
                  ↓
catch가 기대함: org.springframework.security.oauth2.jwt.JwtException
                  ↓
매칭 안 됨 → catch 블록을 건너뜀
                  ↓
예외가 상위로 전파 → 500 Internal Server Error
```

### 왜 발생했나

```
근본 원인: build.gradle에 spring-boot-starter-oauth2-client가 있었다.

이 의존성이 클래스패스에 올려놓는 것:
  org.springframework.security.oauth2.jwt.JwtException

IDE 자동완성:
  "JwtException" 입력 → 두 개가 뜸:
    1. io.jsonwebtoken.JwtException (jjwt)
    2. org.springframework.security.oauth2.jwt.JwtException (Spring)
  → 잘못된 쪽을 선택

한편 SseAuthenticationFilter는 올바르게 io.jsonwebtoken.JwtException을 쓰고 있었다.
→ 같은 프로젝트 안에서 두 필터의 import가 달랐다.
```

### 해결

```java
// 수정 전
import org.springframework.security.oauth2.jwt.JwtException;

// 수정 후
import io.jsonwebtoken.JwtException;
```

그리고 `spring-boot-starter-oauth2-client` 의존성 자체를 제거 (실제로 안 쓰고 있었음). 혼동의 근원을 없앤다.

---

## 버그 2: jjwt 이중 선언 + API 스타일 혼재

### 문제

```groovy
// build.gradle — 구버전과 신버전이 동시에 선언
implementation 'io.jsonwebtoken:jjwt-api:0.11.5'   // 구버전
implementation 'io.jsonwebtoken:jjwt-api:0.12.4'   // 신버전
```

Gradle이 최신 버전(0.12.4)을 선택하므로 0.11.5는 무시된다. 하지만 코드에 두 스타일이 혼재:

```java
// JwtAuthenticationFilter — 0.11.x 스타일 (deprecated)
Jwts.parser()
    .setSigningKey(key)       // deprecated
    .parseClaimsJws(token)    // deprecated
    .getBody();               // deprecated

// SseAuthenticationFilter — 0.12.x 스타일
Jwts.parser()
    .verifyWith(key)
    .parseSignedClaims(token)
    .getPayload();
```

### 해결

- 0.11.5 선언 제거, 0.12.4 단일 버전
- 모든 코드를 0.12.x 스타일로 통일
- 미사용 `com.auth0:java-jwt:3.14.0`도 제거 (프로젝트 전체에 `import com.auth0` 없음)

---

## 버그 3: `SecretKey` charset 불일치

### 문제

```java
// UserService, JwtAuthenticationFilter
SecretKey key = Keys.hmacShaKeyFor(jwtSecret.getBytes());
//                                              ↑ 시스템 기본 charset

// SseAuthenticationFilter만
SecretKey key = Keys.hmacShaKeyFor(jwtSecret.getBytes(StandardCharsets.UTF_8));
//                                                    ↑ UTF-8 명시
```

Windows(기본 charset = MS949)에서는 두 코드가 **다른 키**를 생성한다. 토큰 발급(UserService)과 검증(SseAuthenticationFilter)에서 키가 달라지면 검증 실패.

> 단, JWT secret이 ASCII 문자로만 구성되어 있다면 charset 차이가 결과에 영향을 주지 않는다. 이 버그가 실제로 발생하려면 secret에 한글 등 non-ASCII 문자가 포함되어야 한다. 그럼에도 charset을 명시하는 것이 방어적 코딩의 기본이다.

추가로 매 요청마다 `Keys.hmacShaKeyFor()`를 반복 호출하고 있었다.

### 해결

- 전부 `StandardCharsets.UTF_8` 통일
- `SecretKey`를 빈으로 등록하여 싱글턴으로 한 번만 생성

---

## 버그 4: 로그아웃에서 `unlink()` 호출

### 문제

```java
// 수정 전: logout인데 unlink를 호출
public void logoutUser(...) {
    kakaoService.unlink(currentUser.getKakaoAccessToken());
    //          ↑ 카카오 연결 해제 (탈퇴 수준의 동작!)
}
```

```
logout()이 해야 하는 것: 카카오 세션만 종료. 다음 로그인 시 동의 화면 안 뜸.
unlink()가 실제로 하는 것: 카카오 계정과의 연결을 완전 해제.
                          다음 로그인 시 동의 화면 다시 뜸.
                          카카오 ID가 바뀔 수도 있음.
```

`KakaoService.logout()`이 이미 구현되어 있었는데 사용하지 않고 있었다.

### 해결

```java
// 수정 후
logoutUser()   → kakaoService.logout()   // 세션만 종료
withdrawUser() → kakaoService.unlink()   // 연결 해제 (탈퇴 시에만)
```

---

## 버그 5: `SecurityConfig`에 `@Profile("!test")`

### 문제

```java
@Configuration
@Profile("!test")  // ← 테스트 프로필에서 이 설정이 아예 로드되지 않음!
public class SecurityConfig { ... }
```

테스트 환경에서 보안 필터가 전혀 적용되지 않았다. [SSE 인증 실패 버그](/jwt-sse-auth-filter-mismatch/)가 테스트에서 잡히지 않은 근본 원인.

### 해결

```java
// @Profile 제거 → SecurityConfig가 항상 로드됨

// 테스트 환경에서는 별도 설정으로 오버라이드:
@TestConfiguration
@Import(SecurityConfig.class)
public class IntegrationTestConfig {
    @Bean
    public SecurityFilterChain testFilterChain(HttpSecurity http) throws Exception {
        http.securityMatcher("/**")
            .authorizeHttpRequests(auth -> auth.anyRequest().permitAll());
        return http.build();
    }
}
```

> 이 테스트 설정은 테스트 클래스에서 `@Import(IntegrationTestConfig.class)`로 명시적으로 가져와야 한다. `securityMatcher("/**")`로 경로를 한정하면 이 필터 체인이 모든 요청에 우선 매칭되어 SecurityConfig의 필터 체인보다 먼저 적용된다.

추가로 인증 경로도 `/api/v1/auth/**` 전체 허용 → `/api/v1/auth/kakao/callback`만 허용으로 축소. `logout`, `me`, `withdraw`는 인증 필요.

---

## 기타 설계 문제

- **`withdraw` 이중 호출**: try/catch 양쪽에서 `withdrawUser()` 호출 → 중복 실행 가능. [유저 도메인 편](/user-lifecycle-bugs/)에서 단일 호출로 정리.
- **Refresh Token 미구현**: 생성만 하고 재발급 엔드포인트 없음. [유저 도메인 편](/user-lifecycle-bugs/)에서 구현.
- **`kakaoId` 노출**: `/auth/me` 응답에 내부 식별자 포함. `UserInfoResponse`에서 제거.
- **INACTIVE 체크 실패**: `String.equals(Enum)` → 항상 false. [유저 도메인 편](/user-lifecycle-bugs/)에서 Enum 비교로 변경.
- **필터 이중 등록**: `@Component` 필터가 서블릿에도 등록. `FilterRegistrationBean`으로 방지.

---

## 필터 에러 응답 통일

두 필터가 에러 응답을 각자 다른 방식으로 처리하던 것을 공통 메서드로 통일.

```java
// JwtTokenParser.java — 공통 에러 응답
public static void writeErrorResponse(HttpServletResponse response, ErrorCode errorCode) {
    response.setStatus(errorCode.getStatus().value());
    response.setContentType("application/json;charset=UTF-8");
    response.getWriter().write(
        "{\"success\":false,\"data\":{\"code\":\"" + errorCode.getCode()
        + "\",\"message\":\"" + errorCode.getMessage() + "\"}}"
    );
}
```

```java
// 수정 전: 예외를 던짐 → 글로벌 핸들러까지 가지 않으므로 500
throw new CustomException(ErrorCode.UNAUTHORIZED);

// 수정 후: 직접 응답 후 return
JwtTokenParser.writeErrorResponse(response, ErrorCode.UNAUTHORIZED);
return;
```

필터는 `@ControllerAdvice`의 `GlobalExceptionHandler`를 거치지 않는다. 직접 응답을 작성해야 한다.

> 참고: 이 시점에서는 `SseAuthenticationFilter`와 `JwtAuthenticationFilter` 두 필터 구조가 유지된다. 두 필터 구조의 복잡성 문제는 [필터 단일화 편](/sse-auth-filter-removal/)에서 해소된다.

---

## 핵심 교훈

```
1. import 한 줄이 보안 허점이 된다.
   같은 이름의 클래스가 여러 패키지에 존재할 때
   IDE 자동완성에 의존하지 말고 패키지 경로까지 확인.

2. 안 쓰는 의존성이 클래스패스를 오염시킨다.
   oauth2-client → JwtException 패키지 혼동 → import 실수 → 500 에러.
   의존성은 쓸 것만 추가하고, 안 쓰면 바로 제거.

3. @Profile로 보안 설정을 끄면 테스트가 버그를 못 잡는다.
   테스트 환경에서도 보안 필터는 로드하되,
   별도 설정으로 오버라이드하는 것이 올바른 방법.
```

---

## 현재 상태

- **`JwtException` 패키지 불일치 → 500**: 해결 — `io.jsonwebtoken.JwtException`으로 수정
- **jjwt 이중 선언 + 스타일 혼재**: 해결 — 0.12.4 단일, 0.12.x 스타일 통일
- **미사용 의존성 (`auth0`, `oauth2-client`)**: 해결 — 제거
- **`SecretKey` charset 불일치 + 매 요청 재생성**: 해결 — UTF_8 통일, 빈으로 싱글턴화
- **로그아웃에서 `unlink()` 호출**: 해결 — `logout()`으로 교체
- **`SecurityConfig` 테스트에서 미로드**: 해결 — `@Profile` 제거, `IntegrationTestConfig` 추가

---

## 시리즈 탐색

**시리즈 첫 글** — [전체 목차](/onlyone-project/)

**다음 글:** [JWT 인증 흐름 불일치로 인한 SSE 인증 실패 버그](/jwt-sse-auth-filter-mismatch/)
