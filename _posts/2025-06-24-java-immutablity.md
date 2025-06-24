---
layout: post
title: "자바에서 불변(immutability)의 개념과 설계 방식"
date: 2025-06-24 20:00:00 +0900
---

자바에서 "불변 객체(Immutable Object)"란, 생성 이후 객체의 모든 필드 값이 외부로부터 변경될 수 없는 객체를 말한다. 코드의 예측 가능성과 안정성을 확보하기 위해 사용하는 일반적인 설계 방식이다.

---

## 1. 불변 객체란 무엇인가

불변 객체는 모든 필드를 `final`로 선언해 재할당이 불가능하고, 상태 변경 메서드(setter 등)를 제공하지 않으며, 참조하는 객체가 있다면 그것도 불변이거나 복사해서 보관하는 특징이 있다. 대표적으로 `String`, `Integer`, `LocalDate` 등이 있다.

---

## 2. final은 내부 상태까지 막아주지 않는다

```java
public class Profile {
    private final Name name;

    public Profile(Name name) {
        this.name = name;
    }
}

Name original = new Name("John");
Profile profile = new Profile(original);
original.setValue("Jane"); // Name 클래스는 setValue(String)을 통해 내부 상태를 변경할 수 있다고 가정
```

위 코드에서 `name`은 `final`로 참조 자체는 바뀌지 않지만, `Name` 객체는 mutable이기 때문에 외부에서 내부 상태를 바꾸는 것이 가능하다. 불변을 보장하려면 참조 대상도 불변이어야 한다.

---

## 3. 자바는 call by value이며, 참조형은 주소를 복사한다

자바는 기본적으로 모든 값을 call by value 방식으로 복사해서 전달하기 때문에, 값 자체를 바꾸더라도 호출자 쪽에는 영향을 주지 않는다. 하지만 이 원칙은 기본형에만 완전히 안전하게 적용된다. 참조형의 경우에도 참조값(즉, 주소값)이 복사되어 전달되므로, 결국 같은 힙 객체를 바라보게 되고, 내부 상태를 변경할 수 있어 예상치 못한 결과가 발생할 수 있다.

```java
public class Person {
    private String name;

    public Person(String name) {
        this.name = name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}

public static void changeName(Person person) {
    person.setName("변경됨");
}

Person original = new Person("초기값");
changeName(original);
System.out.println(original.getName()); // "변경됨" 출력됨
```

---

## 4. 힙과 스택의 차이와 상태 공유 문제

자바에서 스택(Stack)은 지역 변수, 기본형, 참조값을 저장하고, 힙(Heap)은 new 키워드로 생성된 객체를 저장한다. 다음 예시에서 보듯 `name`과 `profile.name`은 같은 힙 객체를 공유하며, 한쪽의 상태 변경은 다른 쪽에도 영향을 준다. 이런 공유는 side effect의 주요 원인이다.

```java
Name name = new Name("Alex");
Profile profile = new Profile(name);
```

---

## 5. 불변 객체를 만드는 방법

### (1) 객체 자체를 불변으로 만들기

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

모든 필드를 final로 선언하고 setter를 만들지 않으면 객체 상태는 생성자에서 한 번만 설정되고 이후 변경될 수 없다.

### (2) 방어적 복사 사용

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

외부에서 전달받은 객체를 그대로 저장하지 않고 필요한 값만 복사해서 새로운 객체로 감싼다. 이 방식은 외부 변경이 내부에 영향을 주지 않도록 방지한다.

### (3) 단순 캡슐화(setter 없는 private 필드)만으로는 불변이 아니다

많은 경우, 필드를 `private`으로 감추고 `setter` 없이 `getter`만 제공하면 불변처럼 보이지만, 이는 원시값(int, long 등)에만 비교적 안전하다. 참조값(List, 객체 등)을 필드로 가질 경우 내부 상태를 외부에서 바꿀 수 있다.

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

List<String> external = new ArrayList<>();
external.add("java");
Article article = new Article(external);
external.add("immutability");  // article.tags 에도 "immutability" 추가됨
```

불변 객체로 만들려면 필드 타입에 따라 참조값에 대한 방어적 복사까지 고려해야 한다.

---

## 6. 실무에서 불변이 중요한 이유

불변 객체는 DB 저장 직전 상태를 보장할 수 있고, 여러 서비스나 스레드에서 객체를 공유할 때 안정성을 확보할 수 있으며, 디버깅과 테스트 시 상태 추적이 단순해지고, 의도하지 않은 변경으로 인한 버그를 방지할 수 있다. 특히 API 응답 객체, DTO, 도메인 모델 등에서 예기치 않은 상태 변화를 막기 위해 불변 설계를 기본값으로 두는 것이 좋다. 이런 특성 덕분에 불변 구조는 코드의 신뢰성과 유지보수성을 크게 높여준다.

---

## 7. 요약

final은 참조 자체만 고정하고, 참조 대상의 내부 상태까지 막아주지 않는다. 참조형 인자는 call by value로 전달되지만, 같은 객체를 공유하므로 side effect 발생 가능성이 있다. 힙과 스택 구조상 객체 상태는 공유되며, 불변 객체는 이러한 side effect를 방지하고 예측 가능한 코드를 만든다. 실무에서는 특별한 이유가 없는 한 불변 설계를 기본으로 고려하는 것이 좋다.
