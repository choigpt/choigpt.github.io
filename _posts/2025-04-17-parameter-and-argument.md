---
layout: post
title: "매개변수와 인수, 그 미묘한 차이"
date: 2025-04-17 12:00:00
---

개발을 처음 배우던 시절, 매개변수와 인수라는 단어는 정말 혼란스러웠다. 겉으로 보기에는 같은 말 아니냐고 생각했지만, 실제로는 미묘하지만 중요한 차이가 있다.

프로그래밍에서 매개변수는 함수가 기대하는 입력의 역할을 정의한다. 함수가 필요로 하는 입력의 **청사진**이라고 볼 수 있다. 반면 인수는 실제로 그 자리에 들어가는 **구체적인 값**이다.

## 매개변수와 인수의 차이

```java
public class ParameterArgumentDemo {
    // 매개변수로 역할 정의
    public int calculate(String operation, int a, int b) {
        switch(operation) {
            case "add": return a + b;
            case "subtract": return a - b;
            default: return 0;
        }
    }

    public void demonstrateParameterArgument() {
        // 실제 인수 전달
        int result = calculate("add", 10, 20);
        System.out.println(result); // 30 출력
    }
}
```

여기서 `operation`, `a`, `b`는 매개변수로 함수의 역할을 정의한다. `"add"`, `10`, `20`은 실제로 전달되는 인수다. 매개변수는 함수가 기대하는 입력의 **틀**이고, 인수는 그 **틀**에 들어가는 구체적인 **값**이다.

## 값의 복사와 전달

```java
public class ValuePassingDemo {
    // 기본형 타입 값 복사
    public void modifyPrimitive(int number) {
        number = 100; // 원본 값에 영향 없음
    }

    // 참조형 타입 참조 복사
    public void modifyReference(int[] array) {
        array[0] = 100; // 원본 배열 직접 수정
    }

    public void demonstrateValuePassing() {
        // 기본형 타입
        int x = 10;
        modifyPrimitive(x);
        System.out.println(x); // 10 (변경 없음)

        // 참조형 타입
        int[] arr = {1, 2, 3};
        modifyReference(arr);
        System.out.println(arr[0]); // 100 (변경됨)
    }
}
```

기본형 타입은 값 자체가 복사되어 원본에 영향을 주지 않는다. 반면 참조형 타입은 참조값이 복사되어 같은 객체를 공유하게 된다. 마치 건축의 청사진처럼, 매개변수는 함수의 구조와 기대되는 입력을 미리 정의하고, 인수는 그 청사진에 실제로 들어갈 구체적인 재료와 같다.
