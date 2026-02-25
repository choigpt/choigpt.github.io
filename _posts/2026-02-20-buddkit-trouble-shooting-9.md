---
title: 유저 도메인 — 인증 이후, 사용자 생명주기에서 무너진 것들
date: 2026-02-20
tags: [Spring Security, JWT, 카카오로그인, 트러블슈팅, Java, 사용자관리]
---
	
# 유저 도메인 — 인증 이후, 사용자 생명주기에서 무너진 것들

## 개요

1화와 2화에서 인증 인프라의 버그들을 정리했다. SSE 필터가 subject를 kakaoId로 착각해 SSE 연결이 전면 불통이었던 것, `JwtException` 패키지 불일치로 만료 토큰에서 500이 터졌던 것, 로그아웃이 실제로 카카오 연결을 해제하고 있었던 것.

유저 도메인을 다시 살펴보면서 발견한 것들은 성격이 달랐다. 인증 파이프라인이 아닌, **인증 이후 사용자 생명주기**의 문제였다. 사용자가 처음 로그인해서 GUEST가 되고, 회원가입으로 ACTIVE가 되고, 어느 시점에 탈퇴해 INACTIVE가 되는 흐름. 그 각 단계마다 구멍이 있었다.

---

## 버그 상세

### `getCurrentUserId()` — kakaoId가 14개 서비스로 흘러들어가다

```java
// UserService.java — 수정 전
public Long getCurrentUserId() {
    Long userId = Long.valueOf(authentication.getName());  // principal.name = kakaoId
    return userId;  // 이름은 userId인데 실제로는 kakaoId 반환
}
```

`authentication.getName()`은 kakaoId를 반환한다. `Long` 타입이라 컴파일러는 아무것도 잡지 않는다. 이 메서드를 호출하는 14개 서비스 모두 kakaoId를 userId로 착각하고 `WHERE user_id = :userId` 쿼리에 넘겼다.

```java
// AuthService.java — 수정 후
public Long getCurrentUserId() {
    UserPrincipal principal = (UserPrincipal) SecurityContextHolder
            .getContext().getAuthentication().getPrincipal();
    return principal.getUserId();  // SecurityContext에서 직접 추출
}
```

### INACTIVE 체크 — `String.equals(Enum)`은 항상 false

```java
// JwtAuthenticationFilter.java — 수정 전
if (Status.INACTIVE.name().equals(user.getStatus())) {  // String vs Enum
    throw new CustomException(ErrorCode.UNAUTHORIZED);
}
```

`user.getStatus()`는 `Status.INACTIVE` 열거형 객체를 반환한다. `Status.INACTIVE.name()`은 문자열 `"INACTIVE"`를 반환한다. `String.equals(Enum)`은 항상 false다. 탈퇴 처리된 사용자도 토큰이 만료될 때까지 모든 API를 사용할 수 있었다.

```java
// UserPrincipal.java — 수정 후
public boolean isEnabled() {
    return this.status != Status.INACTIVE;  // Enum 비교
}
```

### GUEST 사용자 — 회원가입 전에 모든 API에 접근 가능

카카오 로그인 직후 사용자는 GUEST 상태다. GUEST 사용자는 nickname이 `"guest"`, birth가 현재 시각, 지갑이 없다. 이 상태에서 모임을 만들면 nickname="guest"인 LEADER가 생긴다. 결제를 시도하면 지갑 null 참조로 NPE가 발생한다.

```java
// JwtAuthenticationFilter.java — 수정 후
if (principal.isGuest() && !isAllowedForGuest(request.getRequestURI())) {
    JwtTokenParser.writeErrorResponse(response, ErrorCode.NO_PERMISSION);
    return;
}

private boolean isAllowedForGuest(String uri) {
    return uri.startsWith("/api/v1/auth/signup")
        || uri.startsWith("/api/v1/auth/logout")
        || uri.startsWith("/api/v1/auth/withdraw");
}
```

### `signup()` — 이미 ACTIVE인 사용자도 재호출 가능

이미 ACTIVE인 사용자가 signup을 다시 호출하면 관심사는 기존 것을 삭제하지 않고 추가되어 중복 누적됐다. 지갑 생성은 unique 제약 위반으로 500을 반환했다.

```java
// UserService.java — 수정 후
if (user.getStatus() == Status.ACTIVE) {
    throw new CustomException(ErrorCode.ALREADY_SIGNED_UP);  // 409
}
userInterestRepository.deleteByUserId(user.getUserId());  // 기존 관심사 먼저 삭제
```

### `withdrawUser()` — 상태만 바꾸고 관련 리소스는 그대로

탈퇴 처리는 status를 INACTIVE로 바꾸는 것뿐이었다. `UserInterest` 기록이 남았다. 진행 중인 정산이 남아서 예약금이 영구히 묶였다. SSE Emitter가 `ConcurrentHashMap`에 남아서 이후 좀비 연결에 계속 알림이 전송됐다.

### Wallet `pendingOut` null 초기화 — 산술 연산 실패

```java
Wallet wallet = Wallet.builder()
        .user(user)
        .postedBalance(100000L)
        .build();   // pendingOut 미설정 → null
```

MySQL에서 `null + amount = null`, `100000 - null = null`이다. 신규 사용자가 첫 스케줄에 참여할 때 hold 연산이 항상 실패했다. `@Builder.Default 0L`로 초기화 적용.

### `KakaoService` — `new RestTemplate()` 직접 생성

커넥션 풀도 없고 타임아웃 설정도 없다. 카카오 API 장애 시 스레드가 모두 대기 상태로 빠지면서 서버 전체가 응답 불능에 빠질 수 있었다. `KakaoConfig`에서 타임아웃이 설정된 빈을 정의하고 생성자 주입으로 교체.

### `kakaoAccessToken` 평문 DB 저장

카카오 액세스 토큰이 DB에 암호화 없이 저장된다. `StringEncryptConverter`를 만들어 AES-256-GCM으로 JPA 컨버터를 적용했다.

---

## 현재 상태

| 문제 | 상태 |
|------|------|
| `getCurrentUserId()` kakaoId 반환 → 14개 서비스 오염 | ✅ 해결 |
| INACTIVE 체크 항상 false → 탈퇴자 차단 불가 | ✅ 해결 |
| GUEST 사용자 전체 API 접근 | ✅ 해결 |
| signup() 중복 호출 → 관심사 누적, 지갑 500 | ✅ 해결 |
| withdrawUser() 리소스 미정리 | ✅ 부분 해결 |
| Wallet pendingOut null → hold 연산 실패 | ✅ 해결 |
| KakaoService new RestTemplate() | ✅ 해결 |
| kakaoAccessToken 평문 저장 | ✅ 해결 |
| withdrawUser() 클럽/정산/SSE 연결 미정리 | ⏸️ 수용 — 배치 정리 예정 |

---

## 교훈

`Long userId`와 `Long kakaoId`는 컴파일러가 구분하지 못한다. 같은 타입이기 때문이다. 변수명, 메서드명, 실제 담긴 값이 일치하는지는 개발자가 확인해야 한다. 14개 서비스에 스며든 `getCurrentUserId()`의 잘못된 반환값은 아무도 몰랐다.

> **생명주기의 각 전환점에서 전제를 확인하자.**
> 인증이 끝났다고 비즈니스 로직이 유효한 상태를 받는다는 보장은 없다. GUEST인지, ACTIVE인지, 이미 처리된 요청인지 — 각 서비스가 받는 입력의 전제 조건을 명시하고 검증해야 한다.
