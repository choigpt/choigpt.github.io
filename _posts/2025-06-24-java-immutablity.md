---
layout: post
title: "자바에서 불변(immutability)의 개념과 설계 방식"
date: 2025-06-24 20:00:00 +0900
---

처음에는 '왜 이런 걸 써야 하지?' 싶었던 개념이 있다. 바로 불변 객체다. 그런데 개발을 오래 하다 보면, 바뀌지 않아야 할 값이 바뀌어서 생기는 문제를 한두 번 겪고 나면 이 개념이 얼마나 중요한지 체감하게 된다. 

자바에서 불변 객체(Immutable Object)란, 객체가 한 번 생성된 후 그 상태가 절대 바뀌지 않는 걸 말한다. 말은 쉬운데, 제대로 만들려면 지켜야 할 규칙들이 있다. 이 글에서는 그 개념부터 설계 방식, 실제 어디다 쓰면 좋은지까지 정리해본다.

---

## 불변 객체란 뭘까?

불변 객체는 모든 필드가 `final`이고, 상태를 바꿀 수 있는 setter 같은 메서드를 제공하지 않는다. 그리고 만약 필드로 다른 객체를 들고 있다면, 그 객체도 불변이거나 복사해서 보관해야 진짜 불변이 된다.

대표적인 예로는 `String`, `Integer`, `LocalDate` 같은 클래스들이 있다.

---

## final이면 안전한 거 아니야?

```java
public class Profile {
    private final Name name;

    public Profile(Name name) {
        this.name = name;
    }
}

Name original = new Name("John");
Profile profile = new Profile(original);
original.setValue("Jane");
```

위 예제에서 `name`은 `final`로 선언돼 있으니 바꿀 수 없을 것 같지만, `Name` 객체가 변경 가능(mutable)하다면 결국 내부 상태는 바뀐다. `final`은 참조 자체만 막아줄 뿐이지, 내부 필드까지 막아주지는 않는다.

---

## 자바는 call by value지만, 참조형은 주소를 넘긴다

```java
public class Person {
    private String name;
    public Person(String name) { this.name = name; }
    public void setName(String name) { this.name = name; }
    public String getName() { return name; }
}

public static void changeName(Person person) {
    person.setName("변경됨");
}

Person original = new Person("초기값");
changeName(original);
System.out.println(original.getName());
```

자바는 값을 복사해서 넘기지만, 참조형은 그 '주소'를 복사해서 넘긴다. 결국 같은 객체를 바라보는 것이고, 한 쪽에서 바꾸면 다른 쪽에도 영향을 준다.

---

## 힙 vs 스택: 어디서 공유되는 걸까?

```java
Name name = new Name("Alex");
Profile profile = new Profile(name);
```

여기서 `name`과 `profile.name`은 같은 힙 객체를 참조하고 있다. 힙은 객체가 저장되는 공간이고, 변수는 그 주소를 들고 있다. 그래서 name을 바꾸면 profile 안의 name도 바뀌는 것처럼 보인다.

---

## 어떻게 만들면 불변 객체가 될까?

### 클래스 자체를 불변으로 만들기

```java
public final class Name {
    private final String value;
    public Name(String value) {
        this.value = value;
    }
    public String getValue() {
        return value;
    }
}
```

클래스를 `final`로 선언하고, 필드를 `private final`로 만들며, setter 없이 생성자에서 한 번만 값을 주입한다.

### 방어적 복사 사용하기

```java
public class Profile {
    private final Name name;

    public Profile(Name name) {
        this.name = new Name(name.getValue());
    }

    public Name getName() {
        return new Name(name.getValue());
    }
}
```

외부에서 받은 객체를 그대로 저장하거나 반환하지 않고 복사해서 보관한다. 그래야 외부에서 바뀌더라도 내부는 영향을 안 받는다.

### 단순 캡슐화만으로는 부족하다

```java
public class Article {
    private List<String> tags;

    public Article(List<String> tags) {
        this.tags = tags;
    }

    public List<String> getTags() {
        return tags;
    }
}

List<String> list = new ArrayList<>();
list.add("java");
Article article = new Article(list);
list.add("immutability");
```

캡슐화만 해두고 방어적 복사를 하지 않으면 외부에서 내부 리스트를 바꿔버릴 수 있다.

---

## 불변 객체, 언제 써야 할까?

- API 응답 객체나 DTO: 외부에 노출된 값을 실수로 바꾸는 걸 방지  
- 설정 값이나 사용자 정보처럼 자주 안 바뀌는 데이터  
- 여러 레이어에서 공유되는 값 객체 (Value Object)  
- 동시성 문제가 생기기 쉬운 멀티스레드 환경  

딱 한 마디로 말하면, '바뀌면 안 되는 데이터'에 쓰면 된다.

---

## 편하게 만드는 방법들: record와 Lombok

### Java record (Java 14+)

```java
public record Name(String value) {}
```

record는 클래스 선언부터 불변을 기본으로 만들어준다. 필드도 `final`, getter도 자동 생성, `equals`, `hashCode`, `toString`까지 다 만들어준다.

### Lombok의 @Value

```java
import lombok.Value;

@Value
public class Name {
    String value;
}
```

Lombok의 `@Value`를 쓰면 클래스 전체를 불변처럼 만들어준다. 단점은 의존성이 생기고, IDE 설정에 따라 다르게 보일 수 있다는 점.

---

## 마무리하며

- `final`은 참조만 막지 내부는 못 막는다  
- 참조형은 call by value여도 결국 같은 객체를 공유하게 된다  
- 방어적 복사, 클래스 설계를 통해 진짜 불변을 만들 수 있다  
- record나 Lombok을 쓰면 간편하게 구현 가능하다  

나중에 "이 객체 왜 바뀐 거야?"라는 디버깅을 피하고 싶다면, 바뀌면 안 되는 건 그냥 처음부터 불변으로 만들어두자. 그게 마음 편하다.
