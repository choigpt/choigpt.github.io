---
title: 토비의 스프링 AOP 하편 — 트랜잭션 속성, @AspectJ, 실무 함정
date: 2026-03-13
tags:
  - Spring
  - AOP
  - Transactional
  - AspectJ
  - 토비의스프링
permalink: /spring-aop-proxy-evolution-2/
excerpt: 상편에서 프록시 진화 과정(데코레이터 → 다이나믹 프록시 → 자동 프록시 생성기)을 다뤘다. 하편에서는 트랜잭션 속성, @Transactional 동작 원리, @AspectJ, Spring AOP vs AspectJ 비교, 그리고 실무에서 자주 만나는 함정을 정리한다.
---

## 들어가며

**◀ 상편:** [수동 프록시에서 자동 프록시 생성기까지](/spring-aop-proxy-evolution/)

상편에서 `UserServiceTx` → 다이나믹 프록시 → `ProxyFactoryBean` → `DefaultAdvisorAutoProxyCreator`까지, 프록시가 어떻게 진화해왔는지를 따라갔다. 자동 프록시가 완성됐으니, 이제 트랜잭션의 세부 속성과 실무에서 만나는 함정을 정리한다.

---

## 5. 트랜잭션 속성

자동 프록시가 완성됐으니, 다음은 트랜잭션의 세부 속성이다. `DefaultTransactionDefinition`에 하드코딩되어 있던 것들이다.

### TransactionDefinition의 네 가지 속성

- **전파(propagation)**: 이미 진행 중인 트랜잭션이 있을 때 어떻게 할 것인가
- **격리 수준(isolation)**: 동시에 실행되는 트랜잭션 간 데이터 접근 수준
- **타임아웃(timeout)**: 트랜잭션 제한 시간
- **읽기 전용(readOnly)**: 트랜잭션 내에서 데이터 변경 허용 여부

### 전파 속성

1권 6장과 2권 2장에서 공통으로 강조하는 부분이다.

- **`REQUIRED`** (기본): 기존 트랜잭션 있으면 참여, 없으면 새로 생성
- **`REQUIRES_NEW`**: 항상 새 트랜잭션. 기존 트랜잭션은 일시 중단
- **`NESTED`**: 기존 트랜잭션 안에서 세이브포인트 생성. 내부 롤백해도 외부는 유지
- **`SUPPORTS`**: 기존 트랜잭션 있으면 참여, 없으면 트랜잭션 없이 실행
- **`NOT_SUPPORTED`**: 트랜잭션 없이 실행. 기존 트랜잭션 일시 중단
- **`MANDATORY`**: 기존 트랜잭션 필수. 없으면 예외
- **`NEVER`**: 트랜잭션 존재하면 예외

주의: `REQUIRES_NEW`는 물리적으로 **별도 DB 커넥션**을 사용한다. 남용하면 커넥션 풀이 고갈된다.

### TransactionInterceptor

스프링이 제공하는 트랜잭션 Advice다. 직접 만든 `TransactionAdvice`를 대체한다. 두 가지 프로퍼티를 받는다:

- **`transactionManager`**: 트랜잭션 기술 추상화
- **`transactionAttributes`**: 메서드 이름 패턴별 트랜잭션 속성 정의

```java
// 1권 6.6 — TransactionInterceptor의 속성 정의
Properties txAttributes = new Properties();
txAttributes.setProperty("get*", "PROPAGATION_REQUIRED,readOnly");
txAttributes.setProperty("upgrade*", "PROPAGATION_REQUIRED");
txAttributes.setProperty("*", "PROPAGATION_REQUIRED");
```

### 롤백 규칙

책에서 명확히 짚는 부분: `TransactionInterceptor`는 기본적으로 **RuntimeException과 Error에서 롤백**한다. Checked Exception은 **커밋**이다. 이유는 스프링이 비즈니스적 의미가 있는 예외(checked)와 시스템 예외(unchecked)를 구분하기 때문이다.

```java
// checked exception에서도 롤백하려면 명시적으로 설정
@Transactional(rollbackFor = Exception.class)
```

---

## 6. 애노테이션 트랜잭션 속성과 포인트컷

메서드 이름 패턴(`get*`, `upgrade*`)으로 트랜잭션 속성을 지정하는 방식에는 한계가 있다 — 메서드 이름만으로는 세밀한 제어가 어렵다. 대안이 **`@Transactional`** 어노테이션이다.

### @Transactional의 동작 원리

```
@EnableTransactionManagement
    → AutoProxyRegistrar (자동 프록시 생성기 등록)
    → ProxyTransactionManagementConfiguration
        → BeanFactoryTransactionAttributeSourceAdvisor (Advisor)
            → TransactionAttributeSourcePointcut (Pointcut)
            → TransactionInterceptor (Advice)
```

- **Pointcut**: `@Transactional` 어노테이션이 있는 클래스/메서드를 찾는다
- **Advice**: `TransactionInterceptor`가 어노테이션의 속성(`propagation`, `isolation`, `rollbackFor` 등)을 읽어 트랜잭션을 관리한다

### 대체 정책 (fallback policy)

책에서 강조하는 `@Transactional`의 적용 우선순위:

```
1. 타깃 메서드의 @Transactional           ← 최우선
2. 타깃 클래스의 @Transactional
3. 인터페이스 메서드의 @Transactional
4. 인터페이스 클래스의 @Transactional       ← 최후순위
```

권장 패턴: 클래스 레벨에 공통 속성을 두고, 예외적인 메서드에만 개별 지정한다.

```java
@Transactional                    // 클래스 레벨 — 기본 REQUIRED
public class UserServiceImpl implements UserService {

    public void upgradeLevels() { ... }  // REQUIRED 적용

    @Transactional(readOnly = true)      // 메서드 레벨 — 읽기 전용으로 오버라이드
    public User get(String id) { ... }
}
```

---

## 7. 트랜잭션 지원 테스트

AOP로 분리된 트랜잭션의 테스트 방법도 다룬다.

### 트랜잭션 동기화와 테스트

`@Transactional`을 테스트 클래스에 붙이면, 테스트 메서드 실행 전체가 하나의 트랜잭션으로 묶인다. 테스트가 끝나면 **자동 롤백**되므로 DB를 오염시키지 않는다.

```java
@Transactional
@SpringBootTest
class UserServiceTest {
    @Test
    void upgradeLevels() {
        userService.upgradeLevels();
        // 검증 후 자동 롤백
    }
}
```

주의: 테스트의 `@Transactional`은 편리하지만, 서비스 메서드 내부에서 `REQUIRES_NEW` 전파를 사용하면 **테스트 트랜잭션과 별도로 커밋**된다. 전파 속성을 제대로 이해하지 않으면 테스트 결과가 예상과 다를 수 있다.

---

## 8. @AspectJ 스타일 AOP (2권)

6장 본문은 저수준 API(`MethodInterceptor`, `ProxyFactoryBean`)에 집중하지만, 실무에서는 `@AspectJ` 스타일로 AOP를 선언한다.

```java
@Aspect
@Component
public class LogAspect {

    @Around("execution(* com.example.service..*Service.*(..))")
    public Object log(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("호출: {}", joinPoint.getSignature());
        Object result = joinPoint.proceed();
        log.info("완료: {}", joinPoint.getSignature());
        return result;
    }
}
```

내부적으로 Spring은 `@Aspect` 클래스를 파싱해 Advisor(Advice + Pointcut)로 변환하고, `AnnotationAwareAspectJAutoProxyCreator`가 자동으로 프록시를 생성한다. 상편 4절의 `DefaultAdvisorAutoProxyCreator`를 확장해 `@Aspect`까지 인식하는 버전이다.

- **`@Before`**: 타깃 메서드 실행 전
- **`@AfterReturning`**: 정상 리턴 후
- **`@AfterThrowing`**: 예외 발생 후
- **`@After`**: 정상/예외 무관
- **`@Around`**: 전체 제어 (가장 강력)

### Pointcut 표현식

- **`execution`**: 메서드 실행을 매칭한다. 예시: `execution(* com.example..*Service.*(..))`
- **`within`**: 클래스/패키지 범위를 매칭한다. 예시: `within(com.example.service..*)`
- **`@annotation`**: 어노테이션이 붙은 메서드를 매칭한다. 예시: `@annotation(com.example.Log)`
- **`@within`**: 어노테이션이 붙은 클래스의 모든 메서드를 매칭한다. 예시: `@within(Service)`
- **`bean`**: 빈 이름 기준으로 매칭한다 (Spring 전용). 예시: `bean(*Service)`

---

## 9. Spring AOP vs AspectJ (2권)

2권에서는 프록시 기반 Spring AOP의 근본적 한계를 인정하고, AspectJ와 비교한다.

Spring AOP는 런타임에 프록시 기반으로 위빙하며, 스프링 빈의 메서드 실행만 가로챌 수 있고, self-invocation에는 적용되지 않는다. 프록시 오버헤드가 있지만 설정이 간단하다. 반면 AspectJ는 컴파일 타임 또는 로드 타임에 위빙하며, 모든 Java 객체의 생성자, 필드 접근, 메서드 실행 등을 가로챌 수 있고, self-invocation에도 적용된다. 바이트코드를 직접 수정하므로 오버헤드가 최소이지만, 별도 컴파일러나 에이전트가 필요해 설정 복잡도가 높다.

### AspectJ가 필요한 케이스

- **도메인 객체에 AOP 적용**: `new User()`로 생성한 객체는 스프링 빈이 아니므로 Spring AOP 불가. AspectJ의 `@Configurable` + LTW(Load-Time Weaving)로 가능
- **필드 접근 가로채기**: Spring AOP는 메서드 호출만 가로챈다
- **self-invocation이 반드시 필요한 구조**: 클래스 분리가 설계상 불가능할 때

2권의 결론: **대부분의 경우 Spring AOP로 충분하다.** AspectJ는 정말 필요할 때만 도입하라.

### Spring Boot의 CGLIB 기본 선택

Spring Boot 2.0부터 기본값이 **CGLIB**이다 (`spring.aop.proxy-target-class=true`). 인터페이스가 있어도 CGLIB을 쓴다. JDK 프록시는 인터페이스 타입으로만 캐스팅 가능해서, 구체 클래스 타입으로 주입받으면 `ClassCastException`이 터지는 문제를 방지하기 위해서다.

---

## 전체 진화 흐름 (상편 + 하편)

```
UserService에 트랜잭션 코드가 섞임
    ↓
1. UserServiceTx로 분리 — 데코레이터 패턴
    ↓  문제: 위임 코드 반복 + 클래스 폭발
2. JDK 다이나믹 프록시 — invoke()로 호출 통합
    → 팩토리 빈 — FactoryBean으로 프록시를 빈으로 등록
    ↓  문제: 타깃마다 팩토리 빈 설정 필요 + 부가 기능 재사용 어려움
3. ProxyFactoryBean — Advice/Pointcut/Advisor 분리
    ↓  문제: 빈마다 ProxyFactoryBean 설정 필요
4. DefaultAdvisorAutoProxyCreator — BeanPostProcessor로 자동 프록시
    ↓                                                    ← 상편 여기까지
5. TransactionInterceptor — 트랜잭션 속성 세분화
    ↓
6. @Transactional — 어노테이션 기반 선언적 트랜잭션
    ↓
7. 테스트에서의 트랜잭션 활용
    ↓
8. @AspectJ 스타일 AOP (2권)
    ↓
9. Spring AOP 한계 인식 → AspectJ 선택적 도입 (2권)
```

---

## 실무에서 자주 만나는 함정

### 1. self-invocation

같은 클래스 내부에서 `this.method()`를 호출하면 프록시를 거치지 않는다.

```java
public class UserServiceImpl implements UserService {
    public void process() {
        this.upgradeLevels();  // 프록시 아닌 this — @Transactional 무시
    }

    @Transactional
    public void upgradeLevels() { ... }
}
```

프록시는 외부에서 들어오는 호출만 가로챈다. `this`는 프록시가 아닌 타깃 오브젝트를 가리킨다.

해결:
- 메서드를 별도 클래스로 분리 (책에서 권장하는 방식 — 관심사 분리)
- `AopContext.currentProxy()` (`@EnableAspectJAutoProxy(exposeProxy = true)` 필요)
- AspectJ 위빙 도입

### 2. final 클래스/메서드

CGLIB은 상속 기반이므로 `final`에는 프록시를 적용할 수 없다. **final 클래스**는 빈 생성 시점에 예외가 발생한다(서브클래스를 만들 수 없으므로). **final 메서드**는 프록시 객체는 생성되지만 해당 메서드의 AOP가 조용히 무시된다 — 컴파일 에러도 런타임 에러도 없이 advice가 적용되지 않는다. 더 위험한 쪽은 final 메서드다.

### 3. @Transactional과 checked exception

기본 설정에서 checked exception은 롤백되지 않는다. `IOException`을 던졌는데 커밋되는 버그가 실무에서 자주 발생한다. (5절 참고)

### 4. 프록시 순서

여러 Aspect가 적용되면 `@Order`로 순서를 제어한다. 같은 `@Aspect` 안에서 여러 Advice의 순서는 보장되지 않으므로, 순서가 중요하면 Aspect 클래스를 분리해야 한다.

### 5. private 메서드

Spring의 `@Transactional`은 Spring 6 이전에는 public 메서드에만 적용된다. `TransactionAttributeSourcePointcut`이 non-public 메서드를 필터링하기 때문이다. 단, Spring AOP 자체는 CGLIB 프록시 사용 시 protected/package-private 메서드도 가로챌 수 있다 — 커스텀 `@Aspect`의 pointcut이 매칭되면 동작한다. `private` 메서드는 상속으로 오버라이드할 수 없으므로 CGLIB에서도 불가능하다. Spring 6부터 `@Transactional`이 protected 메서드에도 공식 지원된다.
