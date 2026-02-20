---
title: Spring Security + JWT 구조 전반에 걸친 버그 및 설계 문제
date: 2026-02-19
tags: [Spring Security, JWT, OAuth2, 트러블슈팅, Java]
---

# Spring Security + JWT 구조 전반에 걸친 버그 및 설계 문제

## 개요

Security 관련 코드를 다시 살펴보다가 버그와 설계 문제가 한 곳이 아니라 여러 곳에 동시에 퍼져 있다는 걸 발견했다. 의존성, 필터, OAuth 흐름, 비즈니스 로직까지 전반에 걸쳐 있었다.

그중에서 가장 심각한 건 컴파일도 되고 테스트도 통과하는데 런타임에서 **만료된 토큰이 401이 아닌 500을 반환하는 버그**였다. 겉으로는 멀쩡해 보이는데 실제로는 터지는 구조, `import` 한 줄 차이였다.

---

## 버그 상세

### `JwtException` 패키지 불일치 — 만료/변조 토큰에서 500 에러

`JwtAuthenticationFilter`에서 잘못된 패키지의 `JwtException`을 import하고 있었다.

```java
// JwtAuthenticationFilter
import org.springframework.security.oauth2.jwt.JwtException;  // Spring OAuth2의 JwtException ← 잘못됨
import io.jsonwebtoken.Jwts;                                    // jjwt 파서

try {
    Claims claims = Jwts.parser()...parseClaimsJws(token)...;  // jjwt가 예외를 던짐
} catch (JwtException | IllegalArgumentException e) {          // 다른 클래스라서 잡히지 않음
    throw new CustomException(ErrorCode.UNAUTHORIZED);
}
```

jjwt가 던지는 예외는 `io.jsonwebtoken.JwtException`이고, catch하는 건 `org.springframework.security.oauth2.jwt.JwtException`이다. 두 클래스는 완전히 별개다. 토큰이 만료되거나 변조되면 예외가 catch되지 않고 상위로 전파되어 의도한 401 대신 **500** 에러가 발생한다.

반면 `SseAuthenticationFilter`는 올바르게 `io.jsonwebtoken.JwtException`을 사용하고 있어서, 두 필터 사이에 불일치가 있었다.

이 혼란의 근원은 `build.gradle`에 있었다. `spring-boot-starter-oauth2-client` 의존성이 포함되어 있었고, 이 때문에 `org.springframework.security.oauth2.jwt` 패키지가 클래스패스에 올라왔다. 필터를 작성할 때 IDE의 자동완성이 같은 이름의 두 클래스 중 Spring OAuth2 패키지 쪽을 먼저 제안했고, 그대로 쓴 것으로 보인다.

---

### 의존성 문제 — jjwt 이중 선언, 미사용 라이브러리

`build.gradle`에 jjwt 구버전과 신버전이 함께 선언되어 있었다.

```groovy
implementation 'io.jsonwebtoken:jjwt-api:0.11.5'   // 구버전
runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.11.5'
runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.11.5'

implementation 'io.jsonwebtoken:jjwt-api:0.12.4'   // 신버전
runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.4'
runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.4'
```

Gradle이 최신 버전(0.12.4)을 선택하므로 0.11.5는 실제로 사용되지 않는다. 그런데 두 버전의 API 스타일이 달라서 코드 내에 두 스타일이 혼재하고 있었다.

```java
// JwtAuthenticationFilter — 0.11.x 스타일 (deprecated)
Jwts.parser()
    .setSigningKey(key)       // deprecated
    .parseClaimsJws(token)    // deprecated
    .getBody();               // deprecated

// SseAuthenticationFilter — 0.12.x 스타일 (현재)
Jwts.parser()
    .verifyWith(key)
    .parseSignedClaims(token)
    .getPayload();
```

여기에 더해 `com.auth0:java-jwt:3.14.0`은 프로젝트 전체에 `import com.auth0`이 한 곳도 없는 미사용 의존성이었고, `spring-boot-starter-oauth2-client`도 실제로는 `RestTemplate`으로 카카오 OAuth를 직접 구현하고 있어서 불필요했다.

---

### `SecretKey` 생성 시 charset 불일치

세 곳에서 같은 secret으로 `SecretKey`를 생성하는데, charset 지정이 제각각이었다.

```java
// UserService, JwtAuthenticationFilter
SecretKey key = Keys.hmacShaKeyFor(jwtSecret.getBytes());  // 시스템 기본 charset

// SseAuthenticationFilter만
SecretKey key = Keys.hmacShaKeyFor(jwtSecret.getBytes(StandardCharsets.UTF_8));
```

Windows처럼 기본 charset이 UTF-8이 아닌 환경에서는 서로 다른 키가 생성되어 토큰 검증이 실패할 수 있다. 또한 매 요청마다 `Keys.hmacShaKeyFor()`를 반복 호출하고 있었다.

---

### 로그아웃에서 `unlink()` 호출 — 카카오 연결 완전 해제

```java
// AuthController.logout()
kakaoService.unlink(currentUser.getKakaoAccessToken());  // 연결 끊기 → logout이어야 함
```

`logout()`에서 `kakaoService.unlink()`를 호출하고 있었다. `unlink`는 카카오 계정과의 연결을 완전히 해제하는 것이다. 다음 로그인 시 동의 화면이 다시 뜨고 카카오 ID가 바뀔 수도 있다. `KakaoService.logout()`이 이미 구현되어 있는데 사용하지 않고 있었다.

---

### 기타 설계 문제

`SecurityConfig`에 `@Profile("!test")`가 붙어 있어 테스트 프로필에서 `SecurityConfig` 자체가 로드되지 않았다. 1화의 SSE 인증 실패 버그가 테스트에서 잡히지 않은 근본 원인 중 하나였다.

`withdraw` 엔드포인트는 `try` 블록과 `catch` 블록 양쪽에서 `withdrawUser()`를 호출하고 있어 특정 예외 상황에서 이중 호출이 가능한 구조였다. Refresh Token은 생성해서 클라이언트에 반환하지만 재발급 엔드포인트가 없어 사실상 동작하지 않는 상태였고, `/auth/me` 응답에 내부 식별자인 `kakaoId`가 그대로 노출되고 있었다.

`JwtAuthenticationFilter`의 INACTIVE 체크 코드(`Status.INACTIVE.name().equals(user.getStatus())`)는 `String.equals(Enum)` 구조라 항상 false를 반환하는 문제도 있었다. 이 수정은 `UserPrincipal` 구조 개편과 묶어 유저 도메인(10화)에서 함께 처리했다.

---

## 해결

### 미사용 의존성 제거

`com.auth0:java-jwt:3.14.0`과 `spring-boot-starter-oauth2-client`를 모든 모듈에서 제거했다. 실제 `import com.auth0` 코드가 없고, OAuth2는 직접 구현하고 있음을 확인한 뒤 제거했다. JWT 처리는 jjwt 0.12.4 단일 라이브러리로 통일했다. `oauth2-client`를 제거하면서 `JwtException` 패키지 혼동의 근원도 함께 없어졌다.

---

### `SecurityConfig` 구조 수정

`@Profile("!test")`를 제거해 `SecurityConfig`가 항상 로드되도록 변경했다. 테스트 환경에서는 `IntegrationTestConfig`의 `@Order(0)` `SecurityFilterChain`이 우선 적용되어 전체 `permitAll`을 유지한다.

```java
// IntegrationTestConfig.java
@Order(0)
@Bean
public SecurityFilterChain testFilterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth.anyRequest().permitAll());
    return http.build();
}
```

인증 경로도 `/api/v1/auth/**` 전체 허용에서 `/api/v1/auth/kakao/callback`만 허용으로 좁혔다. `logout`, `me`, `withdraw`는 인증이 필요한 엔드포인트이므로 `permitAll`에서 제외했다.

`@Component` 필터가 서블릿 컨테이너에 이중 등록되는 것을 방지하기 위해 `FilterRegistrationBean`도 추가했다. `PasswordEncoder`, `AuthenticationManager`, `@EnableMethodSecurity` 등 프로젝트 내 사용처가 없는 설정은 제거했다.

---

### 필터 에러 응답 통일

두 필터가 에러 응답을 각자 다른 방식으로 처리하던 것을 `JwtTokenParser`의 공통 메서드로 통일했다.

```java
// JwtTokenParser.java
public static void writeErrorResponse(HttpServletResponse response, ErrorCode errorCode) {
    response.setStatus(errorCode.getStatus().value());
    response.setContentType("application/json;charset=UTF-8");
    response.getWriter().write(
        "{\"success\":false,\"data\":{\"code\":\"" + errorCode.getCode()
        + "\",\"message\":\"" + errorCode.getMessage() + "\"}}"
    );
}
```

필터는 `GlobalExceptionHandler`를 거치지 않으므로 동일한 JSON 형식으로 직접 응답하도록 했다. `JwtAuthenticationFilter`는 기존의 잘못된 패키지 import를 수정하고, 예외를 던지는 대신 직접 응답 후 `return`하도록 변경했다.

```java
// 수정 전
} catch (JwtException | IllegalArgumentException e) {   // 잘못된 패키지
    throw new CustomException(ErrorCode.UNAUTHORIZED);

// 수정 후
} catch (io.jsonwebtoken.JwtException | IllegalArgumentException e) {
    JwtTokenParser.writeErrorResponse(response, ErrorCode.UNAUTHORIZED);
    return;
}
```

`/sse/**` 경로는 `SseAuthenticationFilter`가 전담하므로 `JwtAuthenticationFilter`에 `shouldNotFilter`를 추가해 이중 실행도 방지했다.

---

### 로그아웃 / 탈퇴 분리

```java
// 수정 전
logoutUser()   → kakaoService.unlink()  // 연결 해제 (잘못됨)

// 수정 후
logoutUser()   → kakaoService.logout()  // 세션만 종료
withdrawUser() → kakaoService.unlink()  // 연결 해제 (기존 유지)
```

`UserInfoResponse`에서 `kakaoId`도 제거해 내부 식별자가 API 응답에 노출되지 않도록 했다.

---

### 빌드 검증 결과

- 전체 모듈 컴파일 성공
- `JwtAuthenticationFilterTest` 7/7 통과
- `UserServiceTest` 전체 통과
- 실패 19건: Testcontainers/Docker 관련 통합 테스트 (Docker 미설치 환경의 기존 실패, 이번 변경과 무관)

---

## 교훈

보안 관련 코드는 컴파일이 되고 테스트가 통과해도 런타임에서 전혀 다른 동작을 할 수 있다. `JwtException` 패키지 불일치가 대표적인 예로, 아무런 컴파일 에러 없이 만료 토큰에서 500이 터지는 버그가 숨어 있었다.

불필요한 의존성이 패키지 충돌을 유발하고, 그게 import 혼동으로 이어지고, 그게 런타임 버그로 이어졌다. 처음부터 필요한 것만 추가하고, 쓰지 않는 의존성은 정기적으로 정리해야 한다.

> **`import` 한 줄이 보안 허점이 된다.**
> 같은 이름의 클래스가 여러 패키지에 존재할 때 IDE 자동완성을 그냥 믿으면 안 된다. 특히 예외 처리처럼 흐름을 제어하는 코드에서는 import 경로까지 확인하는 습관이 필요하다.
