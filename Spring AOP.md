지금까지 질문한 내용을 **개념 흐름 기준으로 정리**하겠습니다. 핵심은 세 가지입니다.

1. 일반 프록시의 문제
    
2. 다이나믹 프록시가 해결하는 방식
    
3. 스프링 AOP에서 ProxyFactory의 역할
    

---

# 1. 일반(수동) 프록시

### 구조

예:

interface OrderService {  
    void save();  
    void find();  
}

프록시:

class OrderServiceProxy implements OrderService {  
  
    private OrderService target;  
  
    public void save() {  
        log();  
        target.save();  
    }  
  
    public void find() {  
        log();  
        target.find();  
    }  
}

### 특징

- 프록시 클래스를 **개발자가 직접 작성**
    
- 인터페이스 메서드를 **전부 구현해야 함**
    
- 대부분 코드가 **위임 코드**
    

### 문제

메서드가 많아질수록

before  
target 호출  
after

같은 코드가 계속 반복됨.

즉 문제의 본질은

**프록시 클래스 + 위임 메서드 반복 생성**

입니다.

---

# 2. 다이나믹 프록시

### 해결 아이디어

프록시 메서드를 사람이 만들지 말고  
**JVM이 런타임에 자동 생성**

---

### 구조

JDK 동적 프록시는 모든 메서드 호출을 **하나의 핸들러로 모읍니다**

invoke(proxy, method, args)

즉 실제 구조는 이런 느낌입니다.

save()  → invoke()  
find()  → invoke()

---

### 중요한 포인트

많이 헷갈리는 부분입니다.

**메서드를 일부만 구현하는 것이 아닙니다.**

프록시는 내부적으로

save()  
find()

모두 생성합니다.

차이는 여기입니다.

부가기능을 적용할지 여부를 **invoke 내부에서 결정**

예:

if(method.getName().equals("save")) {  
    log();  
}

즉

|항목|수동 프록시|동적 프록시|
|---|---|---|
|메서드 구현|개발자가 작성|JVM 자동 생성|
|부가기능 적용|메서드별 코드 작성|invoke에서 조건 처리|

정확한 이해는

**"save만 구현한다"가 아니라  
"save에만 부가기능을 적용한다"**

입니다.

---

# 3. AOP가 등장하는 이유

다이나믹 프록시만 사용하면 이런 코드가 생깁니다.

if(method.getName().equals("save"))  
if(method.getName().startsWith("find"))  
if(annotation 존재)

즉 적용 조건이 코드에 섞이기 시작합니다.

그래서 AOP는 이것을 분리합니다.

### AOP 구성요소

|구성|의미|
|---|---|
|Advice|실행할 공통 로직|
|Pointcut|적용 위치|
|Advisor|Advice + Pointcut|

즉

무엇을 할지  
+  
어디에 할지

를 분리합니다.

---

# 4. ProxyFactory의 역할

프록시를 만들려면 필요한 정보가 많습니다.

- target 객체
    
- advice
    
- pointcut
    
- JDK proxy / CGLIB 선택
    

이걸 매번 직접 조립하기 어렵기 때문에  
**ProxyFactory가 프록시 생성 과정을 담당**합니다.

---

### 사용 예

ProxyFactory factory = new ProxyFactory(target);  
  
factory.addAdvice(new LogAdvice());  
  
OrderService proxy = (OrderService) factory.getProxy();

---

### ProxyFactory 내부 역할

대략 이런 순서입니다.

1. target 저장
    
2. advice / advisor 저장
    
3. 프록시 방식 결정
    
4. 프록시 객체 생성
    

---

### 프록시 생성 방식 결정

인터페이스 있음 → JDK Dynamic Proxy  
인터페이스 없음 → CGLIB

---

# 5. 최종 호출 구조

AOP가 적용되면 실제 구조는 다음입니다.

client  
  ↓  
proxy  
  ↓  
advice chain  
  ↓  
target

즉

Controller  
   ↓  
Service Proxy  
   ↓  
ServiceImpl

---

# 6. "프록시를 주입한다"는 의미

스프링이 빈을 만들 때 과정은 다음입니다.

1. 원본 객체 생성
    
2. AOP 적용 대상인지 검사
    
3. ProxyFactory로 프록시 생성
    
4. **프록시 객체를 빈으로 등록**
    

그래서 실제 주입되는 것은

ServiceImpl

이 아니라

ServiceProxy

입니다.

---

# 7. 전체 개념 흐름

정리하면 이렇게 이어집니다.

수동 프록시  
   ↓  
반복 코드 문제  
   ↓  
동적 프록시  
   ↓  
invoke로 모든 호출 처리  
   ↓  
조건 로직 복잡  
   ↓  
AOP 등장  
   ↓  
Advice / Pointcut 분리  
   ↓  
ProxyFactory가 프록시 생성  
   ↓  
스프링이 프록시를 빈으로 주입

---

# 마지막 핵심 요약

**수동 프록시**

- 프록시 클래스를 직접 구현
    
- 모든 메서드 위임 코드 작성 필요
    

**동적 프록시**

- 프록시 메서드를 JVM이 자동 생성
    
- 모든 호출이 `invoke()`로 모임
    
- 특정 메서드만 부가기능 적용 가능
    

**AOP**

- 공통 기능(Advice)과 적용 위치(Pointcut)를 분리
    

**ProxyFactory**

- target + advice를 받아
    
- JDK proxy 또는 CGLIB을 선택하고
    
- **AOP가 적용된 프록시 객체를 생성**
    

---

마지막으로 이해를 확실히 점검할 질문 하나만 던지겠습니다.