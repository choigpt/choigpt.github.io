---
title: 유저 도메인 — 인증 이후, 사용자 생명주기에서 무너진 것들
date: 2026-02-24
tags: [Spring Security, JWT, 카카오로그인, 트러블슈팅, Java, 사용자관리]
permalink: /user-lifecycle-bugs/
---

# 유저 도메인 — 인증 이후, 사용자 생명주기에서 무너진 것들

## 개요

[Spring Security·JWT 편](/spring-security-jwt-structure-bugs/)과 [SSE 인증 실패 편](/jwt-sse-auth-filter-mismatch/)에서 인증 인프라의 버그들을 정리했다. SSE 필터가 subject를 kakaoId로 착각해 SSE 연결이 전면 불통이었던 것, `JwtException` 패키지 불일치로 만료 토큰에서 500이 터졌던 것, 로그아웃이 실제로 카카오 연결을 해제하고 있었던 것. [필터 단일화 편](/sse-auth-filter-removal/)에서는 SSE 인증을 별도 필터(`SseAuthenticationFilter`)로 분리해서 처리하던 구조 자체를 걷어냈다. `/sse/**` 경로의 쿠키 토큰 읽기 로직을 `JwtAuthenticationFilter` 하나로 통합해, SSE 연결과 일반 API 요청이 동일한 인증 경로를 거치게 됐다. 두 필터가 서로 경로를 나눠 갖고 `shouldNotFilter`로 회피하던 복잡성이 사라졌다.

그 버그들은 인증 파이프라인 자체의 문제였다. 유저 도메인을 다시 살펴보면서 발견한 것들은 성격이 달랐다. 인증 파이프라인이 아닌, **인증 이후 사용자 생명주기**의 문제였다. 사용자가 처음 로그인해서 GUEST가 되고, 회원가입으로 ACTIVE가 되고, 어느 시점에 탈퇴해 INACTIVE가 되는 흐름. 그 각 단계마다 구멍이 있었다.

`getCurrentUserId()`가 kakaoId를 반환한다는 사실은 [SSE 인증 실패 편](/jwt-sse-auth-filter-mismatch/)에서 `SearchService` 예시로, [모임 도메인 편](/club-concurrency-ghost-members/)에서 모임 검색의 `isJoined=false` 문제로 각각 부분적으로 언급됐다. 두 글에서는 해당 서비스의 증상만 고쳤고, 근본 원인인 메서드 자체는 이번에 수정한다. 인증 이후 모든 비즈니스 로직의 출발점이 되는 메서드가 잘못된 값을 흘려보내고 있었다.

---

## 시스템 구조

### 사용자 생명주기

```
카카오 로그인 → GUEST 상태로 JWT 발급
    ↓
signup() 호출 → ACTIVE
    ↓
서비스 이용 (모임, 스케줄, 정산, 결제...)
    ↓
withdrawUser() → INACTIVE
```

인증은 필터에서 처리되지만, 생명주기의 각 전환점은 `user-service` 모듈의 `UserService`에서 처리된다. 필터가 올바르게 동작해도 각 전환점에 검증이 없으면 비정상 상태로 서비스에 진입할 수 있었다.

### 모듈 구조와 영향 범위

이 프로젝트는 멀티 모듈 구조다. `getCurrentUserId()`는 `common` 모듈의 `AuthService`에 있고, 이를 호출하는 14개 서비스는 `feed-service`, `club-service`, `schedule-service`, `notification-service`, `chat-service`, `payment-service` 등 각기 다른 모듈에 분산되어 있다. `common` 모듈 하나를 잘못 수정하면 전 모듈에 영향이 가고, 반대로 `common` 모듈을 올바르게 수정하면 모든 모듈의 증상이 한 번에 해소된다. 이번 수정이 그 케이스였다.

---

## 버그 상세

### `getCurrentUserId()` — kakaoId가 14개 서비스로 흘러들어가다

```java
// AuthService.java (common 모듈) — 수정 전
public Long getCurrentUserId() {
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
    Long userId = 0L;
    userId = Long.valueOf(authentication.getName());  // principal.name = kakaoId
    return userId;  // 이름은 userId인데 실제로는 kakaoId 반환
}
```

`JwtAuthenticationFilter`는 principal에 kakaoId를 담는다. `authentication.getName()`은 kakaoId를 반환한다. 메서드명은 `getCurrentUserId()`이고 변수명은 `userId`지만, 실제로는 kakaoId가 반환된다. `Long` 타입이라 컴파일러는 아무것도 잡지 않는다.

[Spring Security·JWT 편](/spring-security-jwt-structure-bugs/)에서 `SearchService`의 `findByClubIdsByUserId(userId)`에 kakaoId가 넘어가는 예시를 다뤘고, [모임 도메인 편](/club-concurrency-ghost-members/)에서는 모임 검색의 `isJoined=false` 문제로 다시 언급됐다. 두 글에서는 해당 서비스의 증상만 고쳤고, 근본 원인인 `getCurrentUserId()` 자체는 이번에 수정한다. 이 메서드를 호출하는 곳은 14개였다.

```
[notification-service] SseStreamController, NotificationController
[payment-service]      WalletService, PaymentService, SettlementService
[schedule-service]     ScheduleService
[club-service]         ClubService
[feed-service]         FeedService, FeedMainService
[chat-service]         ChatRoomService, MessageRestController
[search-service]       SearchService
[user-service]         AuthController
```

각각의 서비스는 `getCurrentUserId()`가 DB PK를 반환한다고 믿고 `WHERE user_id = :userId` 형태의 쿼리에 넘겼다. kakaoId와 userId가 우연히 비슷한 범위에 있는 로컬 환경에서는 문제가 눈에 띄지 않았다.

```java
// AuthService.java (common 모듈) — 수정 후
public Long getCurrentUserId() {
    UserPrincipal principal = (UserPrincipal) SecurityContextHolder
            .getContext().getAuthentication().getPrincipal();
    return principal.getUserId();  // SecurityContext에서 직접 추출
}
```

`SecurityContext`에 `UserPrincipal`을 직접 담도록 바꾸고, `getUserId()`로 userId(DB PK)를 바로 꺼낸다. `common` 모듈 하나 수정으로 14개 서비스가 모두 올바른 값을 받게 됐다.

---

### INACTIVE 체크 — `String.equals(Enum)`은 항상 false

[SSE 인증 실패 편](/jwt-sse-auth-filter-mismatch/)에서 `@Profile("!test")`를 제거해 필터가 테스트 환경에서도 실행되게 됐다. 그런데 필터를 실제로 돌려보니 코드 자체가 잘못돼 있었다.

```java
// JwtAuthenticationFilter.java — 수정 전
if (Status.INACTIVE.name().equals(user.getStatus())) {  // String vs Enum
    throw new CustomException(ErrorCode.UNAUTHORIZED);
}
```

`user.getStatus()`는 `Status.INACTIVE` 열거형 객체를 반환한다. `Status.INACTIVE.name()`은 문자열 `"INACTIVE"`를 반환한다. `String.equals(Enum)`은 항상 false다.

탈퇴 처리된 사용자도 토큰이 만료될 때까지 모든 API를 사용할 수 있었다. 토큰 만료 시간이 길게 설정되어 있다면, 탈퇴 후에도 한참 동안 사용자 데이터에 접근하고 결제를 시도하고 모임에 참여할 수 있는 상태였다.

```java
// UserPrincipal.java — 수정 후
public boolean isEnabled() {
    return this.status != Status.INACTIVE;  // Enum 비교
}
```

`UserPrincipal`에 `isEnabled()`를 추가하고 필터에서 이를 호출한다. 이제 탈퇴 사용자는 요청 시점에 정확하게 차단된다.

---

### GUEST 사용자 — 회원가입 전에 모든 API에 접근 가능

카카오 로그인 직후 사용자는 GUEST 상태다. `signup()`을 완료해야 ACTIVE가 된다. 그런데 GUEST 상태에서도 JWT가 발급되고, 필터는 INACTIVE만 차단하므로 GUEST 사용자도 인증이 필요한 모든 API에 접근할 수 있었다.

GUEST 사용자는 nickname이 `"guest"`, birth가 현재 시각, 지갑이 없다. 이 상태에서 모임을 만들면 nickname="guest"인 LEADER가 생긴다. 결제를 시도하면 지갑 null 참조로 NPE가 발생한다. 스케줄에 참여하면 wallet hold 로직이 null 지갑에 접근한다. 각 서비스는 자신이 받는 사용자가 회원가입을 완료한 상태라고 가정하고 작성되어 있었다.

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

GUEST에게는 signup, logout, withdraw 세 경로만 허용하고 나머지는 403으로 막는다.

---

### `signup()` — 이미 ACTIVE인 사용자도 재호출 가능

```java
// UserService.java — 수정 전
public void signup(SignupRequestDto signupRequest) {
    User user = getCurrentUser();
    // ACTIVE 여부 확인 없음
    user.update(city, district, profileImage, nickname, gender, birth);
    user.completeSignup();       // GUEST → ACTIVE

    for (String categoryName : categories) {
        userInterestRepository.save(userInterest);  // 기존 관심사 삭제 없이 추가만
    }

    Wallet wallet = Wallet.builder().user(user).postedBalance(100000L).build();
    walletRepository.save(wallet);   // user unique 제약 위반 → DataIntegrityViolationException → 500
}
```

이미 ACTIVE인 사용자가 signup을 다시 호출하면 세 가지 문제가 동시에 발생했다. 관심사는 기존 것을 삭제하지 않고 추가되어 중복 누적됐다. 지갑 생성은 `user` 컬럼의 unique 제약 위반으로 `DataIntegrityViolationException`을 던졌고, 이 예외가 처리되지 않아 500으로 응답됐다. 프로필 정보는 그대로 덮어씌워졌다.

```java
// UserService.java — 수정 후
public void signup(SignupRequestDto signupRequest) {
    User user = getCurrentUser();
    if (user.getStatus() == Status.ACTIVE) {
        throw new CustomException(ErrorCode.ALREADY_SIGNED_UP);  // 409
    }
    userInterestRepository.deleteByUserId(user.getUserId());  // 기존 관심사 먼저 삭제
    // ...
}
```

ACTIVE 사용자의 재호출은 409로 거부한다. 관심사 처리는 기존 것을 먼저 삭제한 뒤 추가하도록 수정했다.

---

### `withdrawUser()` — 상태만 바꾸고 관련 리소스는 그대로

```java
// UserService.java — 수정 전
public void withdrawUser() {
    User user = getCurrentUser();
    user.withdraw();         // status = INACTIVE, kakaoAccessToken = null
    userRepository.save(user);
    // 그 외 아무것도 정리하지 않음
}
```

탈퇴 처리는 status를 INACTIVE로 바꾸는 것뿐이었다. `UserInterest` 기록이 남았다. Club 멤버십(`UserClub`)이 남았다. 진행 중인 정산(`UserSettlement REQUESTED`)이 남아서 예약금이 영구히 묶였다. SSE Emitter가 `ConcurrentHashMap`에 남아서 이후 이 userId로 보내는 알림이 좀비 연결에 계속 시도됐다.

탈퇴한 사용자의 userId로 알림 전송이 시도되고, 스케줄 서비스가 해당 사용자를 참여자로 포함해 로직을 실행하고, 정산 서비스가 INACTIVE 사용자의 지갑에 접근하는 상황이 조용히 지속될 수 있었다. 멀티 모듈 구조에서 각 모듈이 독립적으로 user 상태를 참조하기 때문에, 한 곳에서 INACTIVE 처리가 되어도 다른 모듈이 해당 사용자를 여전히 유효한 것으로 보고 동작할 수 있다.

`withdrawUser()` 내에 `userInterestRepository.deleteByUserId()` 호출을 추가했다. 진행 중인 정산 취소, 클럽 멤버십 해제, SSE 연결 정리는 설계 결정으로 soft delete 방식을 유지하고 배치 정리를 예정으로 남겼다.

---

### Wallet `pendingOut` null 초기화 — 산술 연산 실패

```java
// UserService.java — 수정 전
Wallet wallet = Wallet.builder()
        .user(user)
        .postedBalance(100000L)
        .build();   // pendingOut 미설정 → null
```

`Wallet.pendingOut`에 `@Builder.Default`도 없고 초기값도 없다. signup 시 생성되는 지갑의 `pendingOut`은 null이다.

스케줄 참여 시 wallet hold 로직이 네이티브 쿼리를 실행한다.

```sql
SET pending_out = pending_out + :amount
WHERE posted_balance - pending_out >= :amount
```

MySQL에서 `null + amount = null`, `100000 - null = null`이다. 신규 사용자가 첫 스케줄에 참여할 때 hold 연산이 항상 실패했다. [지갑·결제 편](/wallet-payment-toss-db-failure/)에서 지갑·결제 도메인 작업 중 함께 발견해 `@Builder.Default 0L`로 초기화를 적용했다.

---

### `KakaoService` — `new RestTemplate()` 직접 생성

```java
// KakaoService.java — 수정 전
private final RestTemplate restTemplate = new RestTemplate();
private final ObjectMapper objectMapper = new ObjectMapper();
```

Spring 빈이 아닌 `new RestTemplate()`은 커넥션 풀이 없고 타임아웃 설정도 없다. 카카오 API가 3초만 응답하지 않아도 스레드가 그 시간 동안 블로킹된다. 카카오 API 장애 시 스레드가 모두 대기 상태로 빠지면서 서버 전체가 응답 불능에 빠질 수 있었다.

`KakaoConfig`에서 커넥션 타임아웃과 read 타임아웃이 설정된 `kakaoRestTemplate` 빈을 정의하고 생성자 주입으로 교체했다.

---

### `User.update()` — null 파라미터가 기존 값을 덮어씀

```java
// User.java — 수정 전
public void update(String city, String district, String profileImage,
                   String nickname, Gender gender, LocalDate birth) {
    this.city = city;       // null이면 기존 값이 null로 변경
    this.district = district;
    this.profileImage = profileImage;
    // ...
}
```

프로필 수정 시 일부 필드만 보내면 나머지가 null로 덮어씌워졌다. `UserController`에 `@Valid`가 누락되어 있었고, DTO의 `@NotBlank`, `@NotNull` 검증이 동작하지 않았다. null이 서비스까지 도달해 `User.update()`에 그대로 전달됐다.

`ProfileUpdateCommand` record 패턴으로 교체하고, `UserController`에 `@Valid`를 추가했다. DTO 검증이 동작해야 null 필드가 서비스에 도달하지 않는다.

---

### `kakaoAccessToken` 평문 DB 저장

```java
// User.java
@Column(name = "kakao_access_token")
private String kakaoAccessToken;
```

카카오 액세스 토큰이 DB에 암호화 없이 저장된다. DB가 유출되면 저장된 토큰으로 사용자의 카카오 프로필 조회, 카카오 API를 통한 정보 접근이 가능하다.

`StringEncryptConverter`를 만들어 AES-256-GCM으로 JPA 컨버터를 적용했다. `@Convert(converter = StringEncryptConverter.class)` 어노테이션 하나로 저장 시 암호화, 로딩 시 복호화가 자동으로 적용된다. `ENC:` 접두사로 기존 평문 데이터와 호환성을 유지했다.

---

### 그 외 정비

Refresh Token은 [SSE 인증 실패 편](/jwt-sse-auth-filter-mismatch/)에서 사실상 동작하지 않는 상태로 언급됐다. 생성은 되지만 재발급 엔드포인트가 없어서다. `POST /api/v1/auth/refresh` 엔드포인트를 구현하고 화이트리스트에 등록했다.

`SignupRequestDto`의 categories 필드는 `@NotNull`만 있어 빈 리스트 `[]`나 100개짜리 리스트도 통과했다. 메시지에 "최소 1개 이상 최대 5개 이하"라고 쓰여 있는데 검증이 없었다. `@Size(min=1, max=5)`를 추가했다.

`LoginResponse`의 `boolean isNewUser` 필드는 Java record로 전환하면서 `isNewUser()` accessor가 생성되고, Jackson이 `isNewUser`로 직렬화해 정상 동작함을 확인했다.

---

## 현재 상태

| 문제 | 상태 |
|------|------|
| `getCurrentUserId()` kakaoId 반환 → 14개 서비스 오염 | ✅ 해결 — SecurityContext → UserPrincipal.getUserId() |
| INACTIVE 체크 항상 false → 탈퇴자 차단 불가 | ✅ 해결 — UserPrincipal.isEnabled() Enum 비교 |
| GUEST 사용자 전체 API 접근 | ✅ 해결 — isGuest() 체크, 3개 경로만 허용 |
| signup() 중복 호출 → 관심사 누적, 지갑 500 | ✅ 해결 — ACTIVE 가드 추가, 관심사 삭제 후 추가 |
| withdrawUser() 리소스 미정리 | ✅ 부분 해결 — UserInterest 삭제, 나머지 배치 예정 |
| Wallet pendingOut null → hold 연산 실패 | ✅ 해결 — @Builder.Default 0L ([지갑·결제 편](/wallet-payment-toss-db-failure/)에서 적용) |
| KakaoService new RestTemplate() → 타임아웃 없음 | ✅ 해결 — 타임아웃 설정 Bean 주입 |
| User.update() null 덮어쓰기 + @Valid 누락 | ✅ 해결 — ProfileUpdateCommand + @Valid 추가 |
| kakaoAccessToken 평문 저장 | ✅ 해결 — StringEncryptConverter AES-256-GCM |
| Refresh Token 재발급 엔드포인트 없음 | ✅ 해결 — POST /api/v1/auth/refresh 구현 |
| categories @Size 누락 | ✅ 해결 — @Size(min=1, max=5) 추가 |
| LoginResponse isNewUser 직렬화 | ✅ 해결 — Java record 전환으로 정상 동작 |
| withdrawUser() 클럽/정산/SSE 연결 미정리 | ⏸️ 수용 — soft delete 방식, 배치 정리 예정 |

---

## 교훈

인증 파이프라인의 버그들([Spring Security·JWT 편](/spring-security-jwt-structure-bugs/)부터 [필터 단일화 편](/sse-auth-filter-removal/)까지)은 눈에 띄었다. SSE가 전면 불통이었고, 만료 토큰이 500을 반환했다. 분명한 증상이 있었다.

이번 버그들은 증상이 달랐다. `getCurrentUserId()`가 잘못된 값을 반환해도 로컬 환경에서는 userId와 kakaoId가 우연히 비슷해서 괜찮아 보였다. INACTIVE 체크가 항상 false여도 탈퇴한 사용자가 테스트 계정이 아닌 한 로그에 남지 않았다. GUEST 상태에서 signup 없이 API를 호출하는 건 정상 플로우에서 일어나지 않았다.

멀티 모듈 구조에서 이런 버그는 더 조용하다. `common` 모듈의 `getCurrentUserId()`가 잘못된 값을 반환해도, 각 모듈에서 발생하는 증상은 서로 다른 방식으로 나타난다. `club-service`에서는 `isJoined=false`, `schedule-service`에서는 스케줄 참여 실패, `notification-service`에서는 알림 미수신. 각각을 별개의 버그로 보면 원인을 찾기 어렵다. `common` 모듈이 모든 증상의 공통 원인이라는 걸 알아야 한 번에 해결된다.

`Long userId`와 `Long kakaoId`는 컴파일러가 구분하지 못한다. 같은 타입이기 때문이다. 변수명, 메서드명, 실제 담긴 값이 일치하는지는 개발자가 확인해야 한다. 14개 서비스에 스며든 `getCurrentUserId()`의 잘못된 반환값은 파악하기 어려웠다.

> **생명주기의 각 전환점에서 전제를 확인하자.**
> 인증이 끝났다고 비즈니스 로직이 유효한 상태를 받는다는 보장은 없다. GUEST인지, ACTIVE인지, 이미 처리된 요청인지 — 각 서비스가 받는 입력의 전제 조건을 명시하고 검증해야 한다.

---

## 시리즈 탐색

**◀ 이전 글**  
[지갑·결제 도메인 — 토스 승인 후 DB가 죽으면 자금이 유실될 수 있다](/wallet-payment-toss-db-failure/)

**다음 글 ▶**  
[채팅 도메인 — 상관 서브쿼리·Redis 리스너 미등록·N+1이 만든 성능 저하](/chat-subquery-redis-listener/)
