---
title: "Ch. 03 Spring DI와 AOP"
source: "Notion local cache"
notion_url: "https://app.notion.com/p/e27779d1d7da83a1b63d01b101d51759"
imported_at: "2026-07-02"
tags:
  - notion-import
  - backend-developer
---

# Ch. 03 Spring DI와 AOP

## 목차

- 01~03. Spring DI 흉내내기
  - 1. 변경에 유리한 코드
  - 2. 객체 컨테이너ApplicationContext 만들기
  - 3. 자동 객체 등록하기 - Component Scanning
  - 4. 객체 찾기 - by Name, by Type
  - 5. 객체를 자동 연결하기
  - @Autowired
  - @Resource
  - 5. ApplicationContext와 XML 설정
- 04~08. DI 활용하기
  - 실습1
  - 실습2 -

[스프링의 정석 : 남궁성과 끝까지 간다 | 패스트캠퍼스](https://fastcampus.co.kr/dev_academy_nks)
국비지원 조기 마감 신화, 베스트셀러 'JAVA의 정석'의 저자 남궁성의 Spring 강의입니다! 오픈톡방과 카페에서 평생 AS를 제공하며 완강과 취업까지 도와드립니다. 지금 할인가로 확인하세요!

## 01~03. Spring DI 흉내내기

이 문서는 강의 내용을 따라가며 DI가 필요한 이유를 코드 예제로 정리한 학습 노트입니다. 핵심은 객체 생성 책임을 분리하고, 변경에 강한 구조로 옮겨가는 과정입니다.

### 1. 변경에 유리한 코드

#### 1) 다형성, factory method

어떻게 하면 변경에 유리한 코드를 작성할 수 있을까?

변경에 유리한 코드는 더 적은 노력으로 수정할 수 있고, 실수할 확률도 줄여준다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a9df32d8-dbe9-4149-bd03-d91b87704d4c/Untitled.png)

#### 2) Map과 외부 파일

코드를 작성하고, 개선해가면서 실력이 느는 것이지 이론서 하나 봤다고 실력이 늘지 않는다.

OOP 설계가 잘 안 되는 이유는 실습 부족인 경우가 많다. 이론을 외우는 것보다, 코드를 직접 바꿔 보면서 “어떻게 하면 변경에 유리한 구조를 만들 수 있을까?”를 고민해야 한다. 변경에 유리한 코드의 핵심은 **분리**다.

**<분리>**

1. 변하는 것, 변하지 않는 것
1. 관심사
1. 중복 코드
→ 이런 걸 잘 해야 객체지향적으로 설계를 잘 하는 것이다.

**<스프링 핵심>**

1. DI
1. AOP → 단순히 중복코드를 분리하는 것

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5ad1770f-ef72-464b-8a45-1d156a712a2b/Untitled.png)

#### <실습>

SportsCar를 Truck으로 바꾸고 싶다면 아래 자바 코드는 수정하지 않아도 된다. config 파일만 바꾸면 된다.

Spring이 이런 방식으로 동작하는 이유도 변경에 유리한 구조를 만들기 위해서다.

```
public static void main(String[] args) throws Exception {
//        Car car = new SportsCar();
    Car car = getCar();
    System.out.println("car = " + car);
}

static Car getCar() throws Exception {
    // config.txt를 읽어서 Properties에 저장
    Properties p = new Properties();
    p.load(new FileReader("config.txt"));

    // 클래스 객체(설계도)를 얻어서
    Class clazz = Class.forName(p.getProperty("car"));
    return (Car) clazz.newInstance(); // 객체를 생성해서 반환
}
```

```
// car=com.fastcampus.ch3.diCopy1.SportsCar
car=com.fastcampus.ch3.diCopy1.Truck
```

위 코드를 좀 더 유용하게 바꿔보자. `getObject()`를 만들고 반환 타입도 `Object`로 둔다.

프로그램 코드는 고치지 않아도 된다. `config.xml`만 수정하면 된다.

```
public static void main(String[] args) throws Exception {
//        Car car = new SportsCar();
//        Car car = getCar();
    Car car = (Car) getObject("car");
    Engine engine = (Engine) getObject("engine");
    System.out.println("car = " + car);
    System.out.println("engine = " + engine);
}

static Object getObject(String key) throws Exception {
    // config.txt를 읽어서 Properties에 저장
    Properties p = new Properties();
    p.load(new FileReader("config.txt"));

    // 클래스 객체(설계도)를 얻어서
    Class clazz = Class.forName(p.getProperty(key));
    return clazz.newInstance(); // 객체를 생성해서 반환
}

static Car getCar() throws Exception {
    // config.txt를 읽어서 Properties에 저장
    Properties p = new Properties();
    p.load(new FileReader("config.txt"));

    // 클래스 객체(설계도)를 얻어서
    Class clazz = Class.forName(p.getProperty("car"));
    return (Car) clazz.newInstance(); // 객체를 생성해서 반환
}
```

```
car=com.fastcampus.ch3.diCopy1.SportsCar
engine=com.fastcampus.ch3.diCopy1.Engine
```

### 2. 객체 컨테이너(ApplicationContext) 만들기

<위의 예제와 달라진 것>

1. getObject → getBean
1. Properties 대신에 Map을 사용

#### ❓getBean에서 Bean이란?

Bean이라는 것은 자바빈에서 온 것이다. 빈은 커피콩인데 그 콩이 객체임.

간단하게 `자바빈 == 객체`라고 생각하면 된다. 스프링에서는 빈이라는 이름을 사용하는 것뿐임

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/612df01c-0b6c-4e83-b811-a3a98e526613/Untitled.png)

#### 실습1

```
class Car {}
class SportsCar extends Car {}
class Truck extends Car {}
class Engine {}

class AppContext {
    Map map; // 객체 저장소

    AppContext() {
        map = new HashMap();
        map.put("car", new SportsCar());
        map.put("engine", new Engine());
    }

    Object getBean(String key) {
        return map.get(key);
    }
}

public class Main2 {
    public static void main(String[] args) throws Exception {
        AppContext ac  = new AppContext();

//        Car car = new SportsCar();
//        Car car = getCar();
        Car car = (Car) ac.getBean("car");
        Engine engine = (Engine) ac.getBean("engine");
        System.out.println("car = " + car);
        System.out.println("engine = " + engine);
    }
}
```

#### 실습2

실습1에서 map에 객체를 직접 생성해서 넣어준 부분(하드코딩)이 없어지고 Properties를 이용해서 config.txt의 내용을 읽어와 객체를 만들고 map에 저장한다.

```
class Car {}
class SportsCar extends Car {}
class Truck extends Car {}
class Engine {}

class AppContext {
    Map map; // 객체 저장소

    AppContext() {
        map = new HashMap();

        try {
            Properties p = new Properties();
            p.load(new FileReader("config.txt"));

            // Properties에 저장된 내용을 Map에 저장
            map = new HashMap(p); // 지정된 Map의 모든 요소를 포함하는 HashMap을 생성.

            // 반복문으로 클래스 이름을 얻어서 객체를 생성해서 다시 map에 저장
            for (Object key : map.keySet()) {
                // Map에 저장된 key를 이용해서 class 정보를 가져와야 한다.
                Class clazz = Class.forName((String) map.get(key));

                // 객체를 만들어서 Map에 저장
                map.put(key, clazz.newInstance());
            }

            Class clazz = Class.forName(p.getProperty("car"));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    Object getBean(String key) {
        return map.get(key);
    }
}

public class Main2 {
    public static void main(String[] args) throws Exception {
        AppContext ac  = new AppContext();

//        Car car = new SportsCar();
//        Car car = getCar();
        Car car = (Car) ac.getBean("car");
        Engine engine = (Engine) ac.getBean("engine");
        System.out.println("car = " + car);
        System.out.println("engine = " + engine);
    }
}
```

### 3. 자동 객체 등록하기 - Component Scanning

> 💡 <Component Scanning> = 자동 객체 등록
`@Component` 애너테이션을 이용해서 Application Context(객체 저장소)에 저장할 Bean들을 지정해주는 것

기존(위의 실습2 참고)에는 config.txt에 사용할 자바 빈(객체)을 등록해두고 AppContext에서 읽어다가 Map에 등록하는 방식으로 사용했었다.

이 방법 말고도 `Component Scanning` 으로도 가능하다.

Component Scanning은 `@Component` 애너테이션이 붙은 클래스들을 자동으로 Bean으로 등록해준다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f894bd85-cc64-466a-bf05-dd7cde0a7e7f/Untitled.png)

#### 실습3 - Component Scanning

코드 자세한 건 별로 안 중요함. 핵심은

1. 패키지의 모든 Class를 읽어서
1. 그 중에 @Component 애너테이션 붙은 게 있으면
1. 객체 생성해서 Map에 저장

여기 왼쪽에 콩이 자바 Bean 표시다. 자바 Bean으로 등록된다는 뜻

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/923d3b50-349d-4265-a2d8-24134462374c/Untitled.png)

```
@Component
class Car {}
@Component
class SportsCar extends Car {}
@Component
class Truck extends Car {}
//@Component
class Engine {}

class AppContext {
    Map map; // 객체 저장소

    AppContext() {
        map = new HashMap();

        doComponentScan();
    }

    private void doComponentScan() {
        try {
            // 1. 패키지내의 클래스 목록을 가져온다.
            ClassLoader classLoader = AppContext.class.getClassLoader();
            ClassPath classPath = ClassPath.from(classLoader);

            Set<ClassPath.ClassInfo> set = classPath.getTopLevelClasses("com.fastcampus.ch3.diCopy3");

            // 2. 반복문으로 클래스를 하나씩 읽어와서 @Component 애너테이션이 붙어있는지 확인
            for (ClassPath.ClassInfo classInfo : set) {
                Class clazz = classInfo.load();
                Component component = (Component) clazz.getAnnotation(Component.class);
                // 3. @Componet 애너테이션이 붙어있으면 객체를 생성해서 Map에 저장
                if (component != null) {
                    String id = StringUtils.uncapitalize(classInfo.getSimpleName());
                    map.put(id, clazz.newInstance());
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    Object getBean(String key) {
        return map.get(key);
    }
}

public class Main3 {
    public static void main(String[] args) throws Exception {
        AppContext ac  = new AppContext();

//        Car car = new SportsCar();
//        Car car = getCar();
        Car car = (Car) ac.getBean("car");
        Engine engine = (Engine) ac.getBean("engine");
        System.out.println("car = " + car);
        System.out.println("engine = " + engine);
    }
}
```

**[ 실행결과 ]**

engine은 @Component 애너테이션이 붙어있지 않아서 객체가 생성되지 않은 것을 확인할 수 있다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/64366abf-8a4a-468a-8f0a-11102f4ce8a1/Untitled.png)

### 4. 객체 찾기 - by Name, by Type

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/6acf616d-2363-4cb5-92fe-63a7ba1dca84/Untitled.png)

#### 실습4

```
@Component
class Car {}
@Component
class SportsCar extends Car {}
@Component
class Truck extends Car {}
@Component
class Engine {}

class AppContext {
    Map map; // 객체 저장소

    AppContext() {
        map = new HashMap();

        doComponentScan();
    }

    private void doComponentScan() {
        try {
            // 1. 패키지내의 클래스 목록을 가져온다.
            ClassLoader classLoader = AppContext.class.getClassLoader();
            ClassPath classPath = ClassPath.from(classLoader);

            Set<ClassPath.ClassInfo> set = classPath.getTopLevelClasses("com.fastcampus.ch3.diCopy3");

            // 2. 반복문으로 클래스를 하나씩 읽어와서 @Component 애너테이션이 붙어있는지 확인
            for (ClassPath.ClassInfo classInfo : set) {
                Class clazz = classInfo.load();
                Component component = (Component) clazz.getAnnotation(Component.class);
                // 3. @Componet 애너테이션이 붙어있으면 객체를 생성해서 Map에 저장
                if (component != null) {
                    String id = StringUtils.uncapitalize(classInfo.getSimpleName());
                    map.put(id, clazz.newInstance());
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // by Name
    Object getBean(String key) {
        return map.get(key);
    }

    // by Type 타입을 주면 그 타입과 일치하는 객체 반환
    Object getBean(Class clazz) {
         // object가 clazz의 객체거나 자손이면 true
        for (Object obj : map.values()) {
            if (clazz.isInstance(obj))  // obj instanceof clazz
                return obj;
        }
        return null; // 못 찾으면 null 반환
    }
}

public class Main3 {
    public static void main(String[] args) throws Exception {
        AppContext ac  = new AppContext();

        Car car = (Car) ac.getBean("car"); // by Name으로 객체를 검색
        Car car2 = (Car) ac.getBean(Car.class); // by Type으로 객체를 검색

        Engine engine = (Engine) ac.getBean("engine");
        Engine engine2 = (Engine) ac.getBean(Engine.class);

        System.out.println("car = " + car);
        System.out.println("engine = " + engine);

        System.out.println("car2 = " + car2);
        System.out.println("engine2 = " + engine2);
    }
}
```

### 5. 객체를 자동 연결하기

> 💡 **@Autowired** : by Type
**@Resource**  : by Name

### @Autowired

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d6002e02-9b61-487c-a3f8-92a551d2cb47/Untitled.png)

### @Resource

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/047396bc-bf54-4789-84d4-3cc25ed12e86/Untitled.png)

#### 실습5 - @Autowired

Car 클래스의 멤버에 Engine과 Door를 추가.

코드가 이해 안 되면 아래 2개만 이해하고 넘어가도 된다.

1. Map에 저장된 객체의 iv중 @Autowired가 붙어있으면
1. Map에서 iv와 타입이 맞는 객체를 찾아서 연결(객체의 주소를 iv에 저장)
```
@Component
class Car {
    @Autowired Engine engine;
    @Autowired Door door;

    @Override
    public String toString() {
        return "Car{" +
                "engine=" + engine +
                ", door=" + door +
                '}';
    }
}
@Component
class SportsCar extends Car {}
@Component
class Truck extends Car {}
@Component
class Engine {}
@Component
class Door {}

class AppContext {
    Map map; // 객체 저장소

    AppContext() {
        map = new HashMap();
        doComponentScan();
        doAutowired();
    }

    // Map의 객체 연결
    private void doAutowired() {
        try {
            // Map에 저장된 객체의 iv중 @Autowired가 붙어있으면
            for (Object bean : map.values()) {
                for (Field fld : bean.getClass().getDeclaredFields()) {
                    if (fld.getAnnotation(Autowired.class) != null) // byType
                        // Map에서 iv와 타입이 맞는 객체를 찾아서 연결(객체의 주소를 iv에 저장)
                        fld.set(bean, getBean(fld.getType())); // car.engine = obj;
                }
            }
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }

    private void doComponentScan() {
        try {
            // 1. 패키지내의 클래스 목록을 가져온다.
            ClassLoader classLoader = AppContext.class.getClassLoader();
            ClassPath classPath = ClassPath.from(classLoader);

            Set<ClassPath.ClassInfo> set = classPath.getTopLevelClasses("com.fastcampus.ch3.diCopy4");

            // 2. 반복문으로 클래스를 하나씩 읽어와서 @Component 애너테이션이 붙어있는지 확인
            for (ClassPath.ClassInfo classInfo : set) {
                Class clazz = classInfo.load();
                Component component = (Component) clazz.getAnnotation(Component.class);
                // 3. @Componet 애너테이션이 붙어있으면 객체를 생성해서 Map에 저장
                if (component != null) {
                    String id = StringUtils.uncapitalize(classInfo.getSimpleName());
                    map.put(id, clazz.newInstance());
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // by Name
    Object getBean(String key) {
        return map.get(key);
    }

    // by Type 타입을 주면 그 타입과 일치하는 객체 반환
    Object getBean(Class clazz) {
         // object가 clazz의 객체거나 자손이면 true
        for (Object obj : map.values()) {
            if (clazz.isInstance(obj))  // obj instanceof clazz
                return obj;
        }
        return null; // 못 찾으면 null 반환
    }
}

public class Main4 {
    public static void main(String[] args) throws Exception {
        AppContext ac  = new AppContext();

        Car car = (Car) ac.getBean("car"); // by Name으로 객체를 검색
        Engine engine = (Engine) ac.getBean("engine");
        Door door = (Door) ac.getBean(Door.class);

        // 수동으로 객체를 연결, 우리가 이런 코드를 작성하지 않아도 되도록 스프링이 알아서 해주는 것이다. 우리가 관리할 코드가 적어짐. 변경사항이 생겼을 때 고쳐야 할 부분이 적어진다. 테스트할 부분도 적어짐
//        car.door = door;
//        car.engine = engine;

        System.out.println("car = " + car);
        System.out.println("engine = " + engine);
        System.out.println("door = " + door);
    }
}
```

**<실행결과>**

수동으로 객체를 넣어줬을 때와 같은 결과가 나왔다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/44bc3e52-e98c-46d5-ba6a-cb810e07ed4f/Untitled.png)

#### 실습6 - @Resource

```
@Component
class Car {
    @Resource Engine engine;
//    @Resource
    Door door;

    @Override
    public String toString() {
        return "Car{" +
                "engine=" + engine +
                ", door=" + door +
                '}';
    }
}
@Component
class SportsCar extends Car {}
@Component
class Truck extends Car {}
@Component
class Engine {}
@Component
class Door {}

class AppContext {
    Map map; // 객체 저장소

    AppContext() {
        map = new HashMap();
        doComponentScan();
        doAutowired();
        doResource();
    }

    private void doResource() {
        try {
            // Map에 저장된 객체의 iv중 @Resource 붙어있으면
            for (Object bean : map.values()) {
                for (Field fld : bean.getClass().getDeclaredFields()) {
                    if (fld.getAnnotation(Resource.class) != null) // by Name
                        // Map에서 iv와 이름이 맞는 객체를 찾아서 연결(객체의 주소를 iv에 저장)
                        fld.set(bean, getBean(fld.getName())); // car.engine = obj;
                }
            }
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }

    // Map의 객체 연결
    private void doAutowired() {
        try {
            // Map에 저장된 객체의 iv중 @Autowired가 붙어있으면
            for (Object bean : map.values()) {
                for (Field fld : bean.getClass().getDeclaredFields()) {
                    if (fld.getAnnotation(Autowired.class) != null) // byType
                        // Map에서 iv와 타입이 맞는 객체를 찾아서 연결(객체의 주소를 iv에 저장)
                        fld.set(bean, getBean(fld.getType())); // car.engine = obj;
                }
            }
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }

    private void doComponentScan() {
        try {
            // 1. 패키지내의 클래스 목록을 가져온다.
            ClassLoader classLoader = AppContext.class.getClassLoader();
            ClassPath classPath = ClassPath.from(classLoader);

            Set<ClassPath.ClassInfo> set = classPath.getTopLevelClasses("com.fastcampus.ch3.diCopy4");

            // 2. 반복문으로 클래스를 하나씩 읽어와서 @Component 애너테이션이 붙어있는지 확인
            for (ClassPath.ClassInfo classInfo : set) {
                Class clazz = classInfo.load();
                Component component = (Component) clazz.getAnnotation(Component.class);
                // 3. @Componet 애너테이션이 붙어있으면 객체를 생성해서 Map에 저장
                if (component != null) {
                    String id = StringUtils.uncapitalize(classInfo.getSimpleName());
                    map.put(id, clazz.newInstance());
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // by Name
    Object getBean(String key) {
        return map.get(key);
    }

    // by Type 타입을 주면 그 타입과 일치하는 객체 반환
    Object getBean(Class clazz) {
         // object가 clazz의 객체거나 자손이면 true
        for (Object obj : map.values()) {
            if (clazz.isInstance(obj))  // obj instanceof clazz
                return obj;
        }
        return null; // 못 찾으면 null 반환
    }
}

public class Main4 {
    public static void main(String[] args) throws Exception {
        AppContext ac  = new AppContext();

        Car car = (Car) ac.getBean("car"); // by Name으로 객체를 검색
        Engine engine = (Engine) ac.getBean("engine");
        Door door = (Door) ac.getBean(Door.class);

        // 수동으로 객체를 연결
//        car.door = door;
//        car.engine = engine;

        System.out.println("car = " + car);
        System.out.println("engine = " + engine);
        System.out.println("door = " + door);
    }
}
```

### 5. ApplicationContext와 XML 설정

```
**AppContext ac = new AppContext();**
Car car = (Car) ac.getBean("car");
Engine engine = (Engine) ac.getBean("engine");
```

**→**

```
**ApplicationContext ac = new GenericXmlApplicationContext("file:src/main/webapp/WEB-INF/spring/**/root-context.xml");**
Car car = (Car) ac.getBean("car");
Engine engine = (Engine) ac.getBean("engine");
```

[config.txt]

```
car=com.fastcampus.ch3.SportsCar
engine=com.fastcampus.ch3.Engine
```

**→**

`[config.xml]`

```
<bean id="car" class="com.fastcampus.ch3.SportsCar" />
<bean id="engine" class="com.fastcampus.ch3.SportsCar" />
```

## 04~08. DI 활용하기

### 실습1

config.txt → config.xml로 변경

이전에 만들었던 config.txt처럼 car가 key값. 뒤가 클래스 정보. txt → xml이라는 것만 다르지 내용은 똑같다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1bbdd399-d85f-4293-8081-ac90c04e65ce/Untitled.png)

getBean 시 같은 객체를 반환한다. bean은 기본적으로 싱글톤이다.(대부분의 서버 프로그램은 싱글톤으로 되어있음)

getBean 할 때마다 새로운 객체가 만들어지기를 원하면, prototype을 사용하면 된다.

```
<bean id="car" class="com.fastcampus.ch3.Car" scope="prototype" />
<!-- <bean id="car" class="com.fastcampus.ch3.Car" scope="singleton" /> // singleton이 기본값 -->
```

💡싱글톤 : 클래스의 객체를 하나만 생성하는 것

`SpringDITest.java`

```
class Car {}
class Engine {}
class Door {}

public class SpringDITest {
    public static void main(String[] args) {
        // ac 저장소 안에 Map 형태로 객체가 만들어지고
        ApplicationContext ac = new GenericXmlApplicationContext("config.xml");

        // getBean으로 그걸 가져다쓰는 것
//        Car car = (Car) ac.getBean("car"); // by Name. 아래와 같은 문장
        Car car = ac.getBean("car", Car.class); // by Name
        Car car2 = ac.getBean(Car.class); // by Type

        Engine engine = (Engine) ac.getBean("engine");
        Door door = (Door) ac.getBean("door");

        System.out.println("car = " + car);
        System.out.println("car2 = " + car2);
        //
        System.out.println(car == car2);
        System.out.println("engine = " + engine);
        System.out.println("door = " + door);
    }
}
```

### 실습2 -

```
class Car {
    String color;
    int oil;
    Engine engine;
    Door[] doors;

    public String getColor() {
        return color;
    }

    public void setColor(String color) {
        this.color = color;
    }

    public int getOil() {
        return oil;
    }

    public void setOil(int oil) {
        this.oil = oil;
    }

    public Engine getEngine() {
        return engine;
    }

    public void setEngine(Engine engine) {
        this.engine = engine;
    }

    public Door[] getDoors() {
        return doors;
    }

    public void setDoors(Door[] doors) {
        this.doors = doors;
    }

    @Override
    public String toString() {
        return "Car{" +
                "color='" + color + '\'' +
                ", oil=" + oil +
                ", engine=" + engine +
                ", doors=" + Arrays.toString(doors) +
                '}';
    }
}
class Engine {}
class Door {}

public class SpringDITest {
    public static void main(String[] args) {
        // ac 저장소 안에 Map 형태로 객체가 만들어지고
        ApplicationContext ac = new GenericXmlApplicationContext("config.xml");

        // getBean으로 그걸 가져다쓰는 것
//        Car car = (Car) ac.getBean("car"); // by Name. 아래와 같은 문장
        Car car = ac.getBean("car", Car.class); // by Name
        Engine engine = (Engine) ac.getBean("engine");
        Door door = (Door) ac.getBean("door");

        car.setColor("red");
        car.setOil(100);
        car.setEngine(engine);
        car.setDoors(new Door[]{ ac.getBean("door", Door.class), (Door) ac.getBean("door")});

        System.out.println("car = " + car);
    }
}
```

Car의 멤버에 door가 배열로 들어가있기 때문에 prototype으로 선언.

```
<bean id="door" class="com.fastcampus.ch3.Door" scope="prototype" />
```

Door를 prototype으로 선언해 각각 다른 객체가 doors에 들어가있는 걸 확인할 수 있다.

![Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/092eab6c-34f7-4660-867d-3a95002a41c2/Untitled.png)

### 실습3 - setter대신 property 태그, 생성자로 xml에서 iv 초기화

실습2에서처럼 setter를 호출하는 대신 property 설정을 해줘서 4개의 iv를 property 태그를 이용해 초기화해보자.

```
ApplicationContext ac = new GenericXmlApplicationContext("config.xml");
Car car = ac.getBean("car", Car.class); // by Name
Engine engine = (Engine) ac.getBean("engine");
Door door = (Door) ac.getBean("door");
**
car.setColor("red");
car.setOil(100);
car.setEngine(engine);
car.setDoors(new Door[]{ ac.getBean("door", Door.class), (Door) ac.getBean("door")});**
```

#### property 태그

이렇게 setter를 호출하지 않고도 property 태그로 처리가 가능하다.

property 태그를 사용하려면 `setter()`가 꼭 있어야 한다.

```
<bean id="car" class="com.fastcampus.ch3.Car">
    <property name="color" value="red" />
    <property name="oil" value="100" />
    <property name="engine" ref="engine" /> **<!-- 참조라 value가 아닌 ref를 써야 한다. 아래에 있는 engine bean을 가져다 쓴다는 뜻 -->**
    <property name="doors">
        <array value-type="com.fastcampus.ch3.Door">
            <ref bean="door" />
            <ref bean="door" />
        </array>
    </property>
</bean>
<bean id="engine" class="com.fastcampus.ch3.Engine" />
<bean id="door" class="com.fastcampus.ch3.Door" scope="prototype" />
```

#### 생성자

setter 대신에 생성자로도 가능하다.(기본으로는 프로퍼티 태그 씀)

```
<bean id="car" class="com.fastcampus.ch3.Car">
    <constructor-arg name="color" value="red" />
    <constructor-arg name="oil" value="100" />
    <constructor-arg name="engine" ref="engine" /> <!-- 참조라 value가 아닌 ref를 써야 한다. 아래에 있는 engine bean을 가져다 쓴다는 뜻 -->
    <constructor-arg name="doors">
        <array value-type="com.fastcampus.ch3.Door">
            <ref bean="door" />
            <ref bean="door" />
        </array>
    </constructor-arg>
</bean>
<bean id="engine" class="com.fastcampus.ch3.Engine" />
<bean id="door" class="com.fastcampus.ch3.Door" scope="prototype" />
```

```
public Car(String color, int oil, Engine engine, Door[] doors) {
    this.color = color;
    this.oil = oil;
    this.engine = engine;
    this.doors = doors;
}
```

### 실습4 - 컴포넌트 스캐닝

```
@Component
class Car { .. // 생략 }
@Component class Engine {}
@Component class Door {}
```

config.xml에 `component-scan` 태그를 넣어주면 이 태그가 지정한 패키지를 뒤져서 @Component 애너테이션이 붙어있는 클래스들을 Bean으로 자동 등록해준다.

```
<context:component-scan />
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c1a41791-53ba-463f-b653-836400f76135/Untitled.png)

### 실습5 - `@Autowired`

`@Autowired`를 이용해서 setter를 대신해 주입해보도록하자. `@Autowired`는 Bean들간의 관계를 맺어준다.

```
@Component
class Car {
    String color;
    int oil;

    @Autowired
    Engine engine;

    @Autowired
    Door[] doors;
}
```

실행 시 결과가 잘 나오는 것을 확인할 수 있다.

그런데 color와 oil은 값이 없다.

→ 이건 `@Value` 애너테이션 쓰면 해결됨

`@Autowired`는 타입에 맞는 거 하나만 넣어주기 때문에 doors는 하나만 주입됨.

![Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a8e626c6-10eb-4d43-8e94-bc918f488aea/Untitled.png)

+) 원래는 `@Autowired` 를 쓰려면 annotation-config가 있어야 한다.

component-scan 태그가 annotation-config에서 사용하는 빈들을 다 등록해주기 때문에 없어도 되었었던 건데, 원래는 @Autowired 애너테이션을 쓰려면 아래 태그가 있어야 한다.

```
<context:annotation-config />
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e5292613-a774-492e-94f0-a9cd12743646/Untitled.png)

### 실습6 - `@Value`

`String color = "red";` 이렇게 해도 되는데 굳이 애너테이션 쓰는 이유?

Value 애너테이션에 다른 기능이 있다. 파일에서 읽어온다던가.. 이런 장점이 있음

```
@Component
class Car {
    @Value("red")
    String color;

    @Value("100")
    int oil;

    @Autowired Engine engine;

    @Autowired Door[] doors;
}
```

### 실습7 - By Type 주의해야 할 점

아래 코드 실행 시 에러가 발생한다.

```
@Component
class Car {
    @Value("red")
    String color;
    @Value("100")
    int oil;
    @Autowired Engine engine;
    @Autowired Door[] doors;

		// getter, setter, 생성자, toString 생략
}

@Component class Engine {} // id 생략. <bean id="engine" class="com.fastcampus.ch3.Engine"/>
@Component class SuperEngine extends Engine {}
@Component class TurboEngine extends Engine {}
@Component class Door {}

public class SpringDITest {
    public static void main(String[] args) {
        ApplicationContext ac = new GenericXmlApplicationContext("config.xml");

        Car car = ac.getBean("car", Car.class); // by Type
        Engine engine = (Engine) ac.getBean(Engine.class); // by Type
        Door door = (Door) ac.getBean("door");

        System.out.println("car = " + car);
    }
}
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/6dceafa0-ec71-471b-89e2-612b8478686f/Untitled.png)

```
No qualifying bean of type 'com.fastcampus.ch2.DI.Engine' available: expected single matching bean but found 3: engine,superEngine,turboEngine

사용 가능한 'com.fastcampus.ch2.DI.Engine' 유형의 적합한 Bean 없음: 일치하는 단일 Bean이 예상되지만 3개 발견: engine,superEngine,turboEngine
```

`ac.getBean(Engine.class)` 에서, 매칭되는 Bean이 1개를 예상했는데, 3개가 발견됨.

Type으로 찾는 건 일치하는 게 하나만 있어야 하는데 3개나 있어서 에러가 발생하였다.

이렇게 Type이 여러개일 때는 Type으로 찾으면 안 되고 이름으로 찾아야 한다.

```
// Engine engine = (Engine) ac.getBean(Engine.class); // by Type 같은 타입이 3개라서 에러
	 Engine engine = (Engine) ac.getBean("superEngine"); // 위와 같은 경우에는 이름으로 찾아야 한다.
```

### 실습8 - @Autowired(by type)의 특징

#### ❓위의 실습 7에서와 똑같은 getBean인데 위에서는 에러가 발생하고 아래 코드는 정상적으로 실행되는 이유?

→ @Autowired의 특징 때문이다. @Autowired는 type으로 먼저 검색하고 여러 개가 발견될 경우에 **이름**으로 찾는다.

// by Type - 타입으로 먼저 검색, 여러 개면 이름으로 검색 - engine, superEngine, turboEngine

```
@Component
class Car {
    @Value("red")
    String color;
    @Value("100")
    int oil;
		@Autowired Engine engine; // by Type - 타입으로 먼저 검색, 여러 개면 이름으로 검색 - engine, superEngine, turboEngine
    @Autowired Door[] doors;
}
@Component("engine") class Engine {} // id 생략. <bean id="engine" class="com.fastcampus.ch3.Engine"/>
@Component class SuperEngine extends Engine {}
@Component class TurboEngine extends Engine {}
@Component class Door {}

public class SpringDITest {
    public static void main(String[] args) {
        ApplicationContext ac = new GenericXmlApplicationContext("config.xml");
        Car car = ac.getBean("car", Car.class); // by Type

        System.out.println("car = " + car);
    }
}
```

**[실행결과]**

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ece4112b-9a05-4cf5-b710-54a93c5d399c/Untitled.png)

---

### 실습9 - @Qualifier

실습8 코드에서 Engine 클래스에 @Component 애너테이션 지울 경우 빈이 1개만 검색되어야 하는데 2개가 발견되었다는 에러가 발생한다.

```
@Component
class Car {
    @Value("red")
    String color;
    @Value("100")
    int oil;
    @Autowired Engine engine; // by Type - 타입으로 먼저 검색, 여러 개면 이름으로 검색 - engine, superEngine, turboEngine
    @Autowired Door[] doors;
}
//@Component("engine")
class Engine {} // id 생략. <bean id="engine" class="com.fastcampus.ch3.Engine"/>
@Component class SuperEngine extends Engine {}
@Component class TurboEngine extends Engine {}
@Component class Door {}

public class SpringDITest {
    public static void main(String[] args) {
        ApplicationContext ac = new GenericXmlApplicationContext("config.xml");
        Car car = ac.getBean("car", Car.class); // by Type

        System.out.println("car = " + car);
    }
}
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/10916620-bae1-4288-ad96-3f36b7db7cd2/Untitled.png)

```
No qualifying bean of type 'com.fastcampus.ch3.Engine' available: expected single matching bean but found 2: superEngine,turboEngine
```

이럴 경우에 `@Qualifier`로 어떤 걸 쓸 건지 결정해주어야 한다.

```
@Autowired // 타입으로 검색하는데 결과 중에서
@Qualifier("superEngine") // 이름으로 고르는 거
// @Resource(name = "superEngine") // 이름으로만 찾는다. 위 두 줄과는 엄연히 다른 것. 지금 상황에서는 동일한 뜻이긴 함
Engine engine; // by Type - 타입으로 먼저 검색, 여러 개면 이름으로 검색 - engine, superEngine, turboEngine
```

에러 없이 잘 나온다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a13382bf-aca0-4160-af19-b1ce7509f0c6/Untitled.png)

#### **<결론>**

@Autowired는 Type으로 검색해서 주입을 하는데, 만약 같은 Type이 여러 개인 경우 그때 @Qualifier 애너테이션이 동작해서 후보 중에 id로 가진 Bean이 선택된다.

→ @Resource 애너테이션으로 대체 가능

#### 💡@Resource보다 @Autowired를 많이 쓰는 이유?

이름은 바뀔 가능성이 높은데 타입은 잘 안 바뀌기 때문이다.

## 13. Spring으로 DB연결하기

### 실습1 : Java로 DB에 접근해서 데이터를 가져오는 예제

`JDBC API`를 이용해서 DB에 접근한다.

❓JDBC? - Java Database Connectivity의 약자로 자바로 DB에 연결하기 위한 API.

```
package com.fastcampus.ch3;

import java.sql.*;

public class DBConnectionTest {
    public static void main(String[] args) throws Exception {
        // 1. DB에 접속하기 위한 계정과 URL
        // 스키마의 이름(springbasic)이 다른 경우 알맞게 변경해야 함
        String DB_URL = "jdbc:mysql://localhost:3306/springbasic?useUnicode=true&characterEncoding=utf8";

        // DB의 userid와 pwd를 알맞게 변경해야 함
        String DB_USER = "coals0115";
        String DB_PASSWORD = "1234";

        // 2. DriverManager를 이용해서 연결 정보를 주고, 커넥션을 얻어온다.
        Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD); // 데이터베이스의 연결을 얻는다.
        // 3. 연결된 DB에 sql 명령을 내리려면 Statement가 필요
        Statement stmt  = conn.createStatement(); // Statement를 생성한다.

        String query = "SELECT now()"; // 시스템의 현재 날짜시간을 출력하는 쿼리(query)
        // 위의 쿼리를 실행한 결과를 ResultSet에 담는다.
        ResultSet rs = stmt.executeQuery(query); // query를 실행한 결과를 rs에 담는다.

        // 실행결과가 담긴 rs에서 한 줄씩 읽어서 출력
        while (rs.next()) {
            String curDate = rs.getString(1);  // 읽어온 행의 첫번째 컬럼의 값을 String으로 읽어서 curDate에 저장
            System.out.println(curDate);       // 2022-01-11 13:53:00.0
        }
    } // main()
}

/* [실행결과]
2022-01-11 13:53:00.0
*/
```

#### Java에서 MySQL 연결하기

Java 프로그램에서 MySQL에 연결하려면 JDBC 드라이버가 필요하다.

MySQL Connector/J : Java로 MySQL과 연결할 수 있는 드라이버

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c3891516-7cf0-4b62-ae8a-04de21813ce9/Untitled.png)

### 실습2 : 실습1예제를 Spring JDBC 사용하도록 변경

maven repository에서 Spring JDBC를 가져온다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7cbf38ce-3e2c-44eb-a907-0ed39f02eaf4/Untitled.png)

실습1 예제에서는 JDBC가 제공하는 driverManager를 썼는데, 이번에는 Spring JDBC가 제공하는 `DriverManagerDataSource` 를 사용할 것이다.

```
public class DBConnectionTest2 {
    public static void main(String[] args) throws Exception {
        // 스키마의 이름(springbasic)이 다른 경우 알맞게 변경
        String DB_URL = "jdbc:mysql://localhost:3306/springbasic?useUnicode=true&characterEncoding=utf8";

        // DB의 userid와 pwd를 알맞게 변경
        String DB_USER = "coals0115";
        String DB_PASSWORD = "1234";
        String DB_DRIVER = "com.mysql.jdbc.Driver";

        DriverManagerDataSource ds = new DriverManagerDataSource();
        ds.setDriverClassName(DB_DRIVER);
        ds.setUrl(DB_URL);
        ds.setUsername(DB_USER);
        ds.setPassword(DB_PASSWORD);

//        ApplicationContext ac = new GenericXmlApplicationContext("file:src/main/webapp/WEB-INF/spring/**/root-context.xml");
//        DataSource ds = ac.getBean(DataSource.class);

        Connection conn = ds.getConnection(); // 데이터베이스의 연결을 얻는다.

        System.out.println("conn = " + conn);
//        assertTrue(conn != null);
    }
}
```

### 실습3 : `DriverManagerDataSource`객체를 직접 생성하지 않고 AC에서 가져오는 예제(실습2 변경)

```
ApplicationContext ac = new GenericXmlApplicationContext("file:src/main/webapp/WEB-INF/spring/**/root-context.xml");
DataSource ds = ac.getBean(DataSource.class);

Connection conn = ds.getConnection(); // 데이터베이스의 연결을 얻는다.

System.out.println("conn = " + conn);
```

```
<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
	<property name="driverClassName" value="com.mysql.jdbc.Driver"></property>
	<property name="url" value="jdbc:mysql://localhost:3306/springbasic?useUnicode=true&amp;characterEncoding=utf8"></property>
	<property name="username" value="coals0115"></property>
	<property name="password" value="1234"></property>
</bean>
```

### TDD

실습3에서는 우리가 직접 눈으로 결과가 잘 나왔는지 확인해야 한다.

Junit이라는 테스트 프레임워크를 이용하면 테스트를 자동화할 수 있다.

⇒ TDD(Test Driven Development : 테스트 주도 개발)

실행 결과를 직접 눈으로 확인하는 게 아니라 Junit이라는 테스트 프레임워크를 이용해서 테스트를 자동화하면, 테스트할 코드가 아무리 많아도 일괄적으로 한번에 돌려서 어떤 테스트가 성공, 실패했는지 여부만 보면 된다.

테스트할 메서드에는 `@Test` 애너테이션을 붙이면 된다. 반환 타입은 void로 하면 된다.

```
public class DBConnectionTest2Test {
    @Test
    public void main() throws Exception {
        ApplicationContext ac = new GenericXmlApplicationContext("file:src/main/webapp/WEB-INF/spring/**/root-context.xml");
        DataSource ds = ac.getBean(DataSource.class);

        Connection conn = ds.getConnection(); // 데이터베이스의 연결을 얻는다.

        assertTrue(conn != null); // 괄호 안의 조건식이 true면, 테스트 성공, 아니면 실패
    }
}
```

#### 실습4 : DB 연결 검증 테스트 코드

ApplicationContext도 주입받아서 사용 가능하다.

@Autowired

ApplicationContext ac

```
// 하나의 ac를 만들어두고 재사용하기 때문에 성능에 이점이 있음
@RunWith(SpringJUnit4ClassRunner.class) // ac를 자동으로 만들어줌
@ContextConfiguration(locations = {"file:src/main/webapp/WEB-INF/spring/**/root-context.xml"}) // xml 설정 파일 위치를 지정해줌. root-context.xml에 존재하는 bean으로 테스트함
public class DBConnectionTest2Test {
    @Autowired
    DataSource ds; // 컨테이너로부터 자동 주입받는다.

    @Test
    public void main() throws Exception {
//        ApplicationContext ac = new GenericXmlApplicationContext("file:src/main/webapp/WEB-INF/spring/**/root-context.xml");
//        DataSource ds = ac.getBean(DataSource.class);

        Connection conn = ds.getConnection(); // 데이터베이스의 연결을 얻는다.

        assertTrue(conn != null); // 괄호 안의 조건식이 true면, 테스트 성공, 아니면 실패
    }
}
```

### 인텔리제이 내장 DB Tool

워크벤치 말고 인텔리제이가 제공하는 DB 기능을 사용해보자. 워크벤치와 사용 방법이 비슷하다.

트랜잭션 격리 = Transaction Isolation

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ab91d0b3-7987-4be8-9d2d-ce1848ea091d/Untitled.png)

## 14. Spring으로 DB다루기 - TDD

### User 클래스 생성

1. user_info 테이블에 있는 컬럼 iv로 선언
1. getter, setter 생성
1. 생성자 생성
1. equals, hashCode

### 실습1 : DBConnectionTest2Test

```
// 하나의 ac를 만들어두고 재사용하기 때문에 성능에 이점이 있음
@RunWith(SpringJUnit4ClassRunner.class) // 얘가 ac를 자동으로 만들어줌
@ContextConfiguration(locations = {"file:src/main/webapp/WEB-INF/spring/**/root-context.xml"})
// xml 설정 파일 위치를 지정해줌. root-context.xml에 존재하는 bean으로 테스트함
public class DBConnectionTest2Test {
    @Autowired
    DataSource ds; // 컨테이너로부터 자동 주입받는다.

    @Test
    public void insertUserTest() throws Exception {
        User user = new User("asdf3", "1234", "abc", "aaa@aaa.com", new java.util.Date(), "fb", new java.util.Date());
//        deleteAll();
        int rowCnt = insertUser(user);

        assertTrue(rowCnt == 1);
    }

    @Test
    public void selectUserTest() throws Exception {
        deleteAll();
        User user = new User("asdf3", "1234", "abc", "aaa@aaa.com", new java.util.Date(), "fb", new java.util.Date());
        int rowCont = insertUser(user);
        User user2 = selectUser("asdf3");

        assertTrue(user.getId().equals("asdf3"));
    }

    @Test
    public void updateUserTest() throws Exception {
        User user = new User("asdf3", "h", "h", "h", new java.util.Date(), "h", new java.util.Date());
        updateUser(user);
        User user2 = selectUser(user.getId());

        System.out.println("user.toString().equals(user2.toString()) = " + user.toString().equals(user2.toString()));

        System.out.println("user.toString() = " + user.toString());
        System.out.println("user2.toString() = " + user2.toString());

        assertTrue(user.toString().equals(user2.toString()));
    }

    // 매개변수로 받은 사용자 정보로 user_info 테이블을 update하는 메서드
    public int updateUser(User user) throws Exception {
        // 1. DB 커넥션을 얻어온다.
        Connection conn = ds.getConnection();

        // 2. sql문 작성
//        String sql = "update user_info" +
//                        "set pwd = ?" +
//                          ", name = ?" +
//                          ", email = ?" +
//                          ", birth = ?" +
//                          ", sns = ?" +
//                          ", reg_date = ?" +
//                        "where id = ?";

        String sql = "update user_info set pwd = ?, name = ?, email = ?, birth = ?, sns = ? where id = ?";
        PreparedStatement pstms = conn.prepareStatement(sql);
        pstms.setString(1, user.getPwd());
        pstms.setString(2, user.getName());
        pstms.setString(3, user.getEmail());
        pstms.setDate(4, new java.sql.Date(user.getBirth().getTime()));
        pstms.setString(5, user.getSns());
        pstms.setString(6, user.getId());

        // 3. sql문 실행
        return pstms.executeUpdate(); // insert, update, delete
    }

    @Test
    public void deleteUserTest() throws Exception {
//        deleteAll();
        int rowCnt = deleteUser("asdf");

        assertTrue(rowCnt == 0);

        User user = new User("asdf3", "1234", "abc", "aaa@aaa.com", new java.util.Date(), "fb", new java.util.Date());
        rowCnt = insertUser(user);

        assertTrue(rowCnt == 1); // 위에서 다 지웠기 때문에 1이 맞음

        rowCnt = deleteUser(user.getId());
        assertTrue(rowCnt == 1);

        assertTrue(selectUser(user.getId()) == null);
    }

    // id에 해당하는 user 삭제
    public int deleteUser(String id) throws Exception {
        Connection conn = ds.getConnection();
        String sql = "delete from user_info where id = ?";

        PreparedStatement  pstmt = conn.prepareStatement(sql);
        pstmt.setString(1, id);

        return pstmt.executeUpdate(); // insert, update, delete
    }

    public User selectUser(String id) throws Exception {
        Connection conn = ds.getConnection();

        String sql = "select * from user_info where id = ?";
        PreparedStatement pstmt = conn.prepareStatement(sql); // SQL Injection 공격, 성능 향상(sql 재사용 가능하기 때문에 실행시간 빨라짐)
        pstmt.setString(1, id);

        ResultSet rs = pstmt.executeQuery(); // select

        // 쿼리 결과가 있으면 값을 채워서 반환한다.
        if (rs.next()) {
            User user = new User();
            user.setId(rs.getString(1));
            user.setPwd(rs.getString(2));
            user.setName(rs.getString(3));
            user.setEmail(rs.getString(4));
            user.setBirth(new Date(rs.getDate(5).getTime()));
            user.setSns(rs.getString(6));
            user.setReg_date(new Date(rs.getTimestamp(7).getTime()));

            return user;
        }

        return null;
    }

    private void deleteAll() throws Exception {
        // 1. DataSource로부터 DB 연결을 가져온다.
        Connection conn = ds.getConnection();

        // 2. sql문 작성
        String sql = "delete from user_info where id = 'asdf3'";
        PreparedStatement pstmt = conn.prepareStatement(sql); // SQL Injection 공격, 성능 향상(sql 재사용 가능하기 때문에 실행시간 빨라짐)

        // 3. sql문 실행
        pstmt.executeUpdate(); // insert, delete, update / 몇 개의 행이 영향을 받았는지 반환된다. DML에 사용 가능(select 제외)
    }

    // 사용자 정보를 user_info 테이블에 저장하는 메서드
    public int insertUser(User user) throws Exception {
        // 1. DataSource로부터 DB 연결을 가져온다.
        Connection conn = ds.getConnection();

//        insert into user_info (id, pwd, name, email, birth, sns, reg_date)
//        values ('asdf', '1234', '방채민', 'coals0115@naver.com', '2000-01-01', '', now());

        // 2. sql문 작성
        String sql = "insert into user_info values (?, ?, ?, ?, ?, ?, now())";
        PreparedStatement pstmt = conn.prepareStatement(sql); // SQL Injection 공격, 성능 향상(sql 재사용 가능하기 때문에 실행시간 빨라짐)

        // ?에 해당하는 값들 채우기
        pstmt.setString(1, user.getId());
        pstmt.setString(2, user.getPwd());
        pstmt.setString(3, user.getName());
        pstmt.setString(4, user.getEmail());
        pstmt.setDate(5, new java.sql.Date(user.getBirth().getTime()));
        pstmt.setString(6, user.getSns());

        // 3. sql문 실행
        int rowCnt = pstmt.executeUpdate(); //

        return rowCnt;
    }

    @Test
    public void main() throws Exception {
//        ApplicationContext ac = new GenericXmlApplicationContext("file:src/main/webapp/WEB-INF/spring/**/root-context.xml");
//        DataSource ds = ac.getBean(DataSource.class);

        Connection conn = ds.getConnection(); // 데이터베이스의 연결을 얻는다.

        assertTrue(conn != null); // 괄호 안의 조건식이 true면, 테스트 성공, 아니면 실패
    }
}
```

## 15~16. DAO의 작성과 적용

### 1. DAO(Data Access Object)란?

> 💡 **DAO** : 데이터에 접근하기 위한 객체(**D**ata **A**ccess **O**bject)
> - DB에 저장된 데이터를 읽기, 쓰기, 삭제, 변경(CRUD)을 수행
> - DB 테이블당 하나의 DAO를 작성

DAO를 통해서 DB에 있는 USER_INFO 테이블로부터 CRUD를 한다. (DAO : TABLE = 1 : 1)

CRUD 전담 → DAO(관심사의 분리)

위의 실습1 : DBConnectionTest2Test에서 메서드를 만들고 TDD로 테스트 했었음. 그 메서드들을 모아서 만든 게 DAO다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/12cdbe69-6d94-4547-abca-c411b58d7c40/Untitled.png)

### 2. 계층(layer)의 분리

Controller가 직접 DB에 접근하게되면 중복이 발생할 수밖에 없다.(회원을 조회해오는 메서드가 중복으로 정의되어있다.)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/08c7e749-74da-4472-8a07-5b65c5d3954d/Untitled.png)

UserDao에 user_info 테이블을 다루는 작업들을 만들어두고 Controller가 UserDao를 통해서 간접적으로 DB에 접근하도록 변경하였다.

→ 중복 코드 제거, 계층 분리됨

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e2a41350-2f6d-46f1-80d7-f291d5974338/Untitled.png)

#### ❓이렇게 분리하는 이유?

1. **관심사의 분리** : Data 보여주기(Controller), DB에 접근(UserDao)하는 역할이 다름. 서로 다른 관심사가 하나에 공존하고 있다.
1. **변하는 것, 변하지 않는 것의 분리**
1. **중복코드 분리, 제거**
⇒ 장점 : 변경에 유리

#### 실습1 - UserDao

🚨 생성 시에는 conn → pstmt → rs 순서였는데, close 할 때는 rs → pstmt → conn 순서로 반대로 닫아야 한다.

💡 자원반환(close)은 try-with-resources 구문으로 더 간단하게 할 수 있다.

```
public class UserDao {
    @Autowired
    DataSource ds;
    final int FAIL = 0;

    public int deleteUser(String id) {
        int rowCnt = FAIL; //  insert, delete, update

        Connection conn = null;
        PreparedStatement pstmt = null;

        String sql = "delete from user_info where id= ? ";

        try {
            conn = ds.getConnection();
            pstmt = conn.prepareStatement(sql);
            pstmt.setString(1, id);
//        int rowCnt = pstmt.executeUpdate(); //  insert, delete, update
//        return rowCnt;
            return pstmt.executeUpdate(); //  insert, delete, update
        } catch (SQLException e) {
            e.printStackTrace();
            return FAIL;
        } finally {
            // close()를 호출하다가 예외가 발생할 수 있으므로, try-catch로 감싸야함.
//            try { if(pstmt!=null) pstmt.close(); } catch (SQLException e) { e.printStackTrace();}
//            try { if(conn!=null)  conn.close();  } catch (SQLException e) { e.printStackTrace();}
            close(pstmt, conn); //     private void close(AutoCloseable... acs) {
        }
    }

    public User selectUser(String id) throws Exception {
        User user = null;

        Connection conn = null;
        PreparedStatement pstmt = null;
        ResultSet rs = null;

        String sql = "select * from user_info where id= ? ";

        try {
            conn = ds.getConnection();
            pstmt = conn.prepareStatement(sql); // SQL Injection공격, 성능향상
            pstmt.setString(1, id);

            rs = pstmt.executeQuery(); //  select

            if (rs.next()) {
                user = new User();
                user.setId(rs.getString(1));
                user.setPwd(rs.getString(2));
                user.setName(rs.getString(3));
                user.setEmail(rs.getString(4));
                user.setBirth(new Date(rs.getDate(5).getTime()));
                user.setSns(rs.getString(6));
                user.setReg_date(new Date(rs.getTimestamp(7).getTime()));
            }
        } catch (SQLException e) {
            return null;
        } finally {
            // close()를 호출하다가 예외가 발생할 수 있으므로, try-catch로 감싸야함.
            // close()의 호출순서는 생성된 순서의 역순
//            try { if(rs!=null)    rs.close();    } catch (SQLException e) { e.printStackTrace();}
//            try { if(pstmt!=null) pstmt.close(); } catch (SQLException e) { e.printStackTrace();}
//            try { if(conn!=null)  conn.close();  } catch (SQLException e) { e.printStackTrace();}
            close(rs, pstmt, conn);  //     private void close(AutoCloseable... acs) {
        }

        return user;
    }

    // 사용자 정보를 user_info테이블에 저장하는 메서드
    public int insertUser(User user) {
        int rowCnt = FAIL;

        Connection conn = null;
        PreparedStatement pstmt = null;

//        insert into user_info (id, pwd, name, email, birth, sns, reg_date)
//        values ('asdf22', '1234', 'smith', 'aaa@aaa.com', '2022-01-01', 'facebook', now());
        String sql = "insert into user_info values (?, ?, ?, ?,?,?, now()) ";

        try {
            conn = ds.getConnection();
            pstmt = conn.prepareStatement(sql); // SQL Injection공격, 성능향상
            pstmt.setString(1, user.getId());
            pstmt.setString(2, user.getPwd());
            pstmt.setString(3, user.getName());
            pstmt.setString(4, user.getEmail());
            pstmt.setDate(5, new java.sql.Date(user.getBirth().getTime()));
            pstmt.setString(6, user.getSns());

            return pstmt.executeUpdate(); //  insert, delete, update;
        } catch (SQLException e) {
            e.printStackTrace();
            return FAIL;
        } finally {
            close(pstmt, conn);  //     private void close(AutoCloseable... acs) {
        }
    }

    // 매개변수로 받은 사용자 정보로 user_info테이블을 update하는 메서드
    public int updateUser(User user) {
        int rowCnt = FAIL; //  insert, delete, update

//        Connection conn = null;
//        PreparedStatement pstmt = null;

        String sql = "update user_info " +
                "set pwd = ?, name=?, email=?, birth =?, sns=?, reg_date=? " +
                "where id = ? ";

        // try-with-resources - since jdk7
        try (
                Connection conn = ds.getConnection();
                PreparedStatement pstmt = conn.prepareStatement(sql); // SQL Injection공격, 성능향상
        ){
            pstmt.setString(1, user.getPwd());
            pstmt.setString(2, user.getName());
            pstmt.setString(3, user.getEmail());
            pstmt.setDate(4, new java.sql.Date(user.getBirth().getTime()));
            pstmt.setString(5, user.getSns());
            pstmt.setTimestamp(6, new java.sql.Timestamp(user.getReg_date().getTime()));
            pstmt.setString(7, user.getId());

            rowCnt = pstmt.executeUpdate();
        } catch (SQLException e) {
            e.printStackTrace();
            return FAIL;
        }

        return rowCnt;
    }

    public void deleteAll() throws Exception {
        Connection conn = ds.getConnection();

        String sql = "delete from user_info ";

        PreparedStatement pstmt = conn.prepareStatement(sql); // SQL Injection공격, 성능향상
        pstmt.executeUpdate(); //  insert, delete, update
    }

    private void close(AutoCloseable... acs) { // 가변인자
        for(AutoCloseable ac :acs)
            try { if(ac!=null) ac.close(); } catch(Exception e) { e.printStackTrace(); }
    }
}
```

**<구현체에서 인터페이스로 추출하기>**

클래스 이름에서 우클릭 > 리팩터링 > 인터페이스 추출

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4053a905-fbe6-4397-8d00-6ad289bc9b68/Untitled.png)

```
public interface UserDao {
    int deleteUser(String id);

    User selectUser(String id) throws Exception;

    // 사용자 정보를 user_info테이블에 저장하는 메서드
    int insertUser(User user);

    // 매개변수로 받은 사용자 정보로 user_info테이블을 update하는 메서드
    int updateUser(User user);
}
```

UserDaoImpl은 UserDao 인터페이스를 구현했다는 뜻이다.

UserDao가 인터페이스가 아니라 클래스라면, 만약 다른 DB를 사용하게 되었을 때 UserDao를 변경해야 한다.

이런 상황이 발생했을 때 UserDao를 인터페이스로 정의해두고 그대로 사용하면 나는 구현체만 변경하면 된다.

DataSource도 인터페이스로 정의되어있다. 이렇게 인터페이스로 정의해놓고 실제 DataSource는 우리가 필요한 걸 bean으로 등록하게 해두면, 얼마든지 다른 구현체로 변경이 가능하다. 스프링이 다 이런 방식으로 되어있다.

⇒ 인터페이스로 이용해 변경에 유리한 코드가됨

#### 실습2 - 실습1에서 작성한 UserDaoImpl을 테스트

<details>
<summary>테스트코드 쉽게 작성하는 방법?</summary>

클래스에서 우클릭 > 이동 > 테스트를 누르면

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e088dbaa-4f4e-4b6c-ad56-7f06606f3eb0/Untitled.png)

이렇게 테스트 메서드가 만들어진다.

```
package com.fastcampus.ch3;

import org.junit.Test;

import static org.junit.Assert.*;

public class UserDaoImplTest {

    @Test
    public void deleteUser() {
    }

    @Test
    public void selectUser() {
    }

    @Test
    public void insertUser() {
    }

    @Test
    public void updateUser() {
    }
}
```

</details>

```
@RunWith(SpringJUnit4ClassRunner.class) // ac를 자동으로 만들어줌
@ContextConfiguration(locations = {"file:src/main/webapp/WEB-INF/spring/**/root-context.xml"})
public class UserDaoImplTest {
    @Autowired
    UserDao userDao; // userDao를 구현한 클래스가 빈으로 등록되려면 Repository 붙여야 함

    @Test
    public void deleteUser() {
    }

    @Test
    public void selectUser() {
    }

    @Test
    public void insertUser() {
    }

    @Test
    public void updateUser() {
        Calendar cal = Calendar.getInstance();
        cal.clear();
        cal.set(2023, 4, 11);

        userDao.deleteUser("asdf");

        User user = new User("asdf", "1234", "abc", "aaa@aaa.com", new Date(cal.getTimeInMillis()), "fb", new Date());
        int rowCnt = userDao.insertUser(user);
        assertTrue(rowCnt == 1);

        user.setPwd("4321");
        user.setEmail("bbb@bbb.com");
        rowCnt = userDao.updateUser(user);
        assertTrue(rowCnt == 1);

        User user2 = userDao.selectUser(user.getId());

        System.out.println("user = " + user);
        System.out.println("user2 = " + user2);

        assertTrue(user.equals(user2));

    }
}
```

`UserDaoImplTest.java` 코드 실행 시 에러가 발생하는데, `root-context.xml`에 컴포넌트 스캔 태그가 없어서 그렇다. 추가해주자.

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:mvc="http://www.springframework.org/schema/mvc"
			 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
			 xmlns="http://www.springframework.org/schema/beans"
			 xmlns:context="http://www.springframework.org/schema/context"
			 xsi:schemaLocation="http://www.springframework.org/schema/mvc https://www.springframework.org/schema/mvc/spring-mvc.xsd
		http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">

	<!-- Root Context: defines shared resources visible to all other web components -->
	<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
		<property name="driverClassName" value="com.mysql.jdbc.Driver"></property>
		<property name="url" value="jdbc:mysql://localhost:3306/springbasic?useUnicode=true&amp;characterEncoding=utf8"></property>
		<property name="username" value="coals0115"></property>
		<property name="password" value="1234"></property>
	</bean>

**	<context:component-scan base-package="com.fastcampus.ch3">
		<context:exclude-filter type="regex" expression="com.fastcampus.ch3.diCopy*.*"/>
	</context:component-scan>**
</beans>
```

원래는 더 철저하게 검증을 해야 한다. 단순히 기본적인 것만 테스트하는 것이 아니라 여러가지 경우를 만들어서 테스트를 통과하도록 꼼꼼히 작성해야 한다.

#### ❓@Component를 쓰면 되는데 @Controller, @Service, @Repository가 존재하는 이유? 차이?

```
@Repository
public class UserDaoImpl implements UserDao { ... }
```

`@Controller`, `@Service`, `@Repository` 애너테이션이 `@Component` 애너테이션을 포함하고 있기 때문에, Component Scan에 의해서 AC에 Bean으로 자동 등록이 가능한 애너테이션이다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8acf2903-cf92-41d6-95d1-f30554b48c15/Untitled.png)

똑같은 컴포넌트라도 종류별로 애너테이션을 다르게 지정해놓은 것이다.(계층, Layer 별로 명시적으로 눈에 들어오게 하기 위해서 나누었다고 생각하면됨)

#### 실습3 - 작성한 UserDao를 Controller에 적용하기

LoginController와 RegisterController를 추가한다. ch2에서 만든 것과 동일하다.

1. LoginController에 UserDao를 주입
```
@Controller
@RequestMapping("/login")
public class LoginController {
    @Autowired
    UserDao userDao;
```

1. loginCheck 추가
```
private boolean loginCheck(String id, String pwd) {
    User user = userDao.selectUser(id);

    if (user == null) return false;

    return user.getPwd().equals(pwd);
//        return "asdf".equals(id) && "1234".equals(pwd);
}
```

## 17. Transaction, Commit, Rollback

트랜잭션의 정의와 속성, isolation level

### 1. Transaction이란?

- 더이상 나눌 수 없는 작업의 단위
- 계좌 이체의 경우, 출금과 입금이 하나의 Tx로 묶여야 됨
- ‘모’아니면 ‘도’. 출금과 입금이 모두 성공하지 않으면 실패
![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1e96107f-0dd2-4e45-b603-f9027c094075/Untitled.png)

insert, update, select와 같은 명령 하나하나가 트랜잭션이다.

둘 이상의 작업으로 이루어져있는 경우, 하나의 트랜잭션으로 묶어야 한다. 하나의 작업이 실패하면 모두 실패되어야 함

### 2. Transaction의 속성 - ACID

> 💡 원자성(**A**tomicity) - 나눌 수 없는 하나의 작업으로 다뤄져야 한다.
일관성(**C**onsistency) - Tx 수행 전과 후가 일관된 상태를 유지해야 한다.
고립성, 격리성(**I**solation) - 각 Tx는 독립적으로 수행되어야 한다.
영속성(**D**urability) - 성공한 Tx의 결과는 유지되어야 한다.

#### 원자성

계좌이체 = 출금 + 입금

계좌이체는 2개의 작업으로 나누어져 있는데 이걸 나눌 수 있으면 안 된다. 반드시 하나의 작업으로 묶어야 한다.

#### 일관성

계좌이체를 하기 전과 후의 금액이 정상적이어야 한다는 얘기

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/6042639f-4890-4a7c-a805-cf0ee7fd188c/Untitled.png)

#### 고립성, 격리성

트랜잭션 Tx1과 Tx2가 있다.

Tx2의 작업이 Tx1에 영향을 미쳐서 Tx1의 작업 결과가 달라지면 안 된다는 뜻이다.

아예 영향을 안 줘야 될 때도 있고, 어느정도 허용하는 것도 있는데 그걸 Isolation Level이라고 한다.

Isolation Level이 너무 높으면 한 사람이 작업하는데 많은

높다고 무조건 좋은 게 아님

#### 영속성

계좌이체가 되었으면 그 결과가 계속 유지되어야 한다는 뜻이다.

ex) 계좌이제를 했는데 다음날 롤백되어있으면 안 된다는 얘기

### 3. 커밋(commit)과 롤백(rollback)

> 💡 커밋(commit) : 작업 내용을 DB에 영구적으로 저장
롤백(rollback) : 최근 변경사항을 취소(마지막 커밋으로 복귀)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f06ddc4a-9f91-45d3-a4ea-e0e8c8f48f5d/Untitled.png)

### 4. 자동 커밋과 수동 커밋

#### 자동 커밋 : 명령 실행 후, 자동으로 커밋이 수행(rollback 불가)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9f0f1bae-19a1-4fcf-875a-3e47e97140b3/Untitled.png)

#### 수동 커밋 : 명령 실행 후, 명시적으로 commit 또는 rollback을 입력

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7b9e48e8-1c54-4476-98d2-3a629ebbd892/Untitled.png)

계좌이체 같은 2개 이상의 명령으로 이루어져있는 트랜잭션은 수동 커밋으로 해놓고 명령을 내려야 한다.

### 5. Tx의 isolation level

isolation level은 각 Tx를 고립시키는 정도를 설정한다. 아래로 내려갈수록 고립도가 높아진다.

> 💡 **READ UNCOMMITED** : 커밋되지 않은 데이터도 읽기 가능
**READ COMMITED** : 커밋된 데이터만 읽기 가능
**REPEATABLE READ** : Tx이 시작된 이후 변경은 무시됨
**SERIALIZABLE** : 한번에 하나의 Tx만 독립적으로 수행

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/10cd5b07-c120-43cf-bcbf-c0b31eba71f0/Untitled.png)

#### READ UNCOMMITED - 커밋되지 않은 데이터도 읽기 가능

un commited ← 커밋되지 않은 read ← 읽다.

말 그대로 커밋되지 않은 데이터도 읽기 가능

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/637aa6c6-9764-4f6b-9ad7-5b5643c4eccb/Untitled.png)

#### READ COMMITED - 커밋된 데이터만 읽기 가능

처음엔 없었는데 갑자기 나타나서 phantom read라고도 한다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0a93e213-1528-443e-ad81-79b6cd99a7d4/Untitled.png)

#### REPEATABLE READ - Tx이 시작된 이후 변경은 무시됨

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ccde5a78-7834-4bee-86f9-c68f162db8e2/Untitled.png)

#### SERIALIZABLE - 한번에 하나의 Tx만 독립적으로 수행

**품질↑** **성능↓ **⇒ 매우 중요한 Tx인 경우에 사용한다.

insert로 시작했으면 select도 불가. 다른 Tx가 간섭하지 못한다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a74e44af-5495-433a-8feb-a4b30382a07a/Untitled.png)

#### 실습1 : 트랜잭션 구현하기

사용자 정보를 user_info 테이블에 저장하는 메서드

```
@Test
public void transactionTest() throws Exception {
    Connection conn = null;
    try {
        deleteAll();
        conn = ds.getConnection();
        conn.setAutoCommit(false); // ㄱㅣ본이 true다. 여러 명령으로된 트랜잭션의 경우 제대로된 처리 불간으.

        String sql = "insert into user_info values (?, ?, ?, ?, ?, ?, now())";
        PreparedStatement pstmt = conn.prepareStatement(sql); // SQL Injection 공격, 성능 향상(sql 재사용 가능하기 때문에 실행시간 빨라짐)

        pstmt.setString(1, "asdf");
        pstmt.setString(2, "1234");
        pstmt.setString(3, "abc");
        pstmt.setString(4, "aaa@aaa.com");
        pstmt.setDate(5, new java.sql.Date(new Date().getTime()));
        pstmt.setString(6, "fb");

        int rowCnt = pstmt.executeUpdate(); // insert, delete, update

        pstmt.setString(1, "asdf");
        rowCnt = pstmt.executeUpdate();

        conn.commit();
    } catch (Exception e) {
        conn.rollback();
    }
}
```

## 18. AOP의 개념과 용어

### 1. 공통 코드의 분리

여러 메서드에 공통 코드를 추가해야 한다면?

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1720985e-4b4c-4843-9998-da190d47b961/Untitled.png)

직접 코드를 추가하지 않아도 코드를 추가한 것처럼 실행되는 것

추가할 코드를 따로 분리해서 작성해놓고 그 코드가 마치 추가된 것처럼 하려고 하는 게 AOP다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d6abaa68-dad8-4de0-a10c-f7d200b0deae/Untitled.png)

#### 실습1 : AOP

```
public class AopMain {
    public static void main(String[] args) throws Exception {
        MyAdvice myAdvice = new MyAdvice();

        Class myClass = Class.forName("com.fastcampus.ch3.aop.MyClass");
        Object obj = myClass.newInstance();

        for (Method m : myClass.getDeclaredMethods()) { // myClass에 정의되어있는 메서드들을 가져옴
            myAdvice.invoke(m, obj, null);
        }
    }
}

class MyAdvice {
    void invoke(Method m, Object obj, Object... args) throws Exception {
        System.out.println("[before] {");
        m.invoke(obj, args); // aaa(), aaa2(), bbb() 호출 가능
        System.out.println("}[after]");
    }
}

class MyClass {
    void aaa() {
        System.out.println("aaa() is called.");
    }
    void aaa2() {
        System.out.println("aaa2() is called.");
    }
    void bbb() {
        System.out.println("bbb() is called.");
    }
}
```

#### 실습2 : 패턴을 지정해 특정 메서드에만 before, after를 추가하고 싶을 때

어떤 클래스의 메서드들에 before, after 코드를 추가하려고 한다. 그런데 직접 추가하려니 코드 중복이 발생해서 MyAdvice 클래스에 별도의 메서드를 만들어두고 invoke 메서드에 before, after 코드를 추가한 뒤에 리플렉션 api로 MyClass에 있는 메서드들을 호출하도록 코드를 작성하였다.

```
public class AopMain {
    public static void main(String[] args) throws Exception {
        MyAdvice myAdvice = new MyAdvice();

        Class myClass = Class.forName("com.fastcampus.ch3.aop.MyClass");
        Object obj = myClass.newInstance();

        for (Method m : myClass.getDeclaredMethods()) { // myClass에 정의되어있는 메서드들을 가져옴
            myAdvice.invoke(m, obj, null);
        }
    }
}

class MyAdvice {
**    Pattern p = Pattern.compile("a.*"); // a로 시작하는 단어

    boolean matches(Method m) {
        Matcher matcher = p.matcher(m.getName());
        return matcher.matches();
    }**

    void invoke(Method m, Object obj, Object... args) throws Exception {
        **if (matches(m))
            System.out.println("[before] {");**

        m.invoke(obj, args); // aaa(), aaa2(), bbb() 호출 가능

        **if (matches(m))
            System.out.println("}[after]");**
    }
}

class MyClass {
    void aaa() {
        System.out.println("aaa() is called.");
    }
    void aaa2() {
        System.out.println("aaa2() is called.");
    }
    void bbb() {
        System.out.println("bbb() is called.");
    }
}
```

#### 실습3 : @Transactional 애너테이션이 붙은 경우에만 before, after를 추가하고 싶을 때

```
void invoke(Method m, Object obj, Object... args) throws Exception {
    if (**m.getAnnotation(Transactional.class) != null**)
        System.out.println("[before] {");

    m.invoke(obj, args); // aaa(), aaa2(), bbb() 호출 가능

    if (**m.getAnnotation(Transactional.class) != null**)
        System.out.println("}[after]");
}

**@Transactional**
void aaa() {
    System.out.println("aaa() is called.");
}
```

### 2. 코드를 자동으로 추가한다면, 어디에?

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/25b5da6e-8b11-4222-9d3c-1e367f8666b9/Untitled.png)

### 3. AOP(Aspect Oriented Promgramming)

- 관점 지향 프로그래밍? 횡단 관심사? cross-cutting-concern?
서로 다른 모듈, 계층에서 공통적으로 쓰이는 부분을 횡단 관심사라고 한다.

각 모듈을 세로로 그리고 그 모듈은 관통(공통으로 쓰인다.)

- 부가 기능(advice)을 `동적`으로 추가해주는 기술
동적 → 우리가 직접 넣는 게 아니라 코드가 실행되는 과정에서 자동 추가된다는 뜻

부가 기능 ↔ 핵심 기능

- 메서드(target, 핵심)의 시작 또는 끝에 자동으로 코드(advice, 부가)를 추가
![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7eceb8ab-ddf2-419f-a5a2-74f9eb8e0538/Untitled.png)

이 횡단 관심사를 모듈별로 따로 추가해주기보다는 advice(부가기능)를 따로 별도로 떼어내서 동적으로 추가해주는 기술이 AOP다.

<결론>

메서드의 시작 또는 끝에 자동으로 코드를 추가하는 기술이 AOP

위의 실습2의 MyAdvice가 횡단 관심사, 공통적으로 쓰이는 부분이다.

### 4. AOP 관련 용어

**target + advice = proxy**

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/557684fe-0c49-4c27-b800-8a113ab75ef1/Untitled.png)

**weaving**

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/493511b3-bd26-497c-be80-abab198c8987/Untitled.png)

AOP는 SQL의 Join과 똑같다.

목적은 중복제거로, 테이블을 쪼갠다. AOP랑 똑같이 Join을 하면 실행 시에 합쳐짐.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/06dcfb6f-ab8c-4cfe-86ef-5029071ffbc3/Untitled.png)

### 5. Advice의 종류

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b6dc70f2-47ed-44b8-957b-e401d2c14ec0/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c72a2414-add4-4ad4-ab59-e3a81f886785/Untitled.png)

### 6. pointcut expression

- advice가 추가될 메서드를 지정하기 위한 패턴
- **execution(**[접근제어자 생략 가능]**반환타입**** ****패키지명****.****클래스명****.****메서드명****(****매개변수 목록****)) **[예외는 생략]
![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f0579202-269f-49fd-b621-a659ab239069/Untitled.png)

advice가 여러 개일 경우 `@Order(1)` 애너테이션을 붙일 수 있다.

+) JSP의 filter 호출과 똑같다고 생각하면 이해 쉬움

AOP와 비슷한 개념

1. SQL Join
1. JSP filter
전부 목적이 중복제거이다.

#### 실습 - AOP

`pom.xml`

```
<!-- https://mvnrepository.com/artifact/org.aspectj/aspectjweaver -->
<dependency>
	<groupId>org.aspectj</groupId>
	<artifactId>aspectjweaver</artifactId>
	<version>1.9.19</version>
	<scope>runtime</scope>
</dependency>

<!-- https://mvnrepository.com/artifact/org.springframework/spring-aop -->
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-aop</artifactId>
	<version>${org.springframework-version}</version>
</dependency>
```

`root-context_aop.xml`

```
<!-- aop 기능을 쓰려면 이게 있어야 한다. -->
	<aop:aspectj-autoproxy />
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9b1f02fd-d346-40e7-8dd3-a3e9c2ad6d37/Untitled.png)

`LoggingAdvice.java`

```
@Component // advice도 빈으로 등록되어야 한다.
@Aspect // 얘도 붙여줘야함
public class LoggingAdvice {
    @Around("execution(* com.fastcampus.ch3.aop.MyMath.add(..))") // pointcut - 부가 기능이 적용될 대상(적용될 메서드의 패턴)
    public Object methodCalling(ProceedingJoinPoint pjp) throws Throwable {
        // before advice
        long start = System.currentTimeMillis();
        System.out.println("<< [start] " + pjp.getSignature().getName() + Arrays.toString(pjp.getArgs()));

        Object result = pjp.proceed(); // target의 메서드를 호출

        // after advice
        System.out.println("result = " + result);
        System.out.println("[end] >>" + (System.currentTimeMillis() - start) + "ms");

        return result;
    }
}
```

`AopMain2.java`

```
public class AopMain2 {
    public static void main(String[] args) {
        ApplicationContext ac = new GenericXmlApplicationContext("file:src/main/webapp/WEB-INF/spring/**/root-context_aop.xml");
        MyMath mm = (MyMath) ac.getBean("myMath");
//        System.out.println("mm.add(3, 5) = " + mm.add(3, 5));
//        System.out.println("mm.multiply(3, 5) = " + mm.multiply(3, 5));

        mm.add(3, 5);
        mm.add(1, 2, 3);
        mm.multiply(3, 5);
    }

}
```

## 19~21. 서비스 계층의 분리와 @Transactional

### 1. 서비스 계층(Layer)의 분리 - 비지니스 로직의 분리

Before : @Controller에서 직접 DB접근

After : UserDao로 계층을 따로 분리해서(@Repository) 중복 제거 & 관심사의 분리(성격이 다른 코드, 변경되는 이유가 다른 코드)

@Controller : Presentation Layer. 스프링 MVC에서 사용자의 요청을 받아 흐름을 제어하는 역할

@Repository : Persistance Layer. 영속 계층(Data Acess Layer)

DB에 접근해서 데이터를 다루는 일은 영속 계층에 두고 Controller는 본인에 역할에 전념하도록 코드를 분리한 것

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2f3df16e-e665-45f5-8895-5a19556aa445/Untitled.png)

위의 두 계층 외에 또 다른 계층으로 분리해보려 한다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/65ea71b9-53f1-42fb-b7f6-b61816558737/Untitled.png)

회원가입을 담당하고 있는 RegisterController가 있다. 여기서 사용자 이력을 다룰 일이 생겨 UserHistoryDao가 추가되었고 그럼 RegisterController에 DAO를 주입해야 한다.

UserHistoryDao가 추가되었는데 Controller가 변경되었다.

그런데 회원가입 시 사용자 이력이 추가되는 것은 DB와는 관련이 없는 비니지스 로직이다. DAO는 단순히 DB에 있는 테이블하고 1:1 관계로 데이터를 다루는 곳이다. 그렇기 때문에 비지니스 로직을 DAO에 두면 안 된다.

DAO는 CRUD 기능만 가지고 있기 때문에 비니지스 로직이 들어가기에 적합하지 않다. 그래서 컨트롤러에 DAO를 추가하면 비지니스 로직이 변경된 건데 컨트롤러가 변경되었다.

컨트롤러의 역할은 비지니스 로직이 아니기 때문에 비지니스 로직을 담당하는 계층을 하나 만드려고 하는 것이다.

---

UserService라는 새로운 클래스를 만들고, 2개의 DAO를 주입받아서 처리한다.(UserDao, UserHistoryDao)

그럼 Controller는 UserService만 주입받아서 사용하면 된다.

#### ❓영속 계층

DB가 영속성(데이터가 유지되는)을 가지기 때문에 영속 계층이라고 표현한다.

Service 계층을 보면 메서드 이름이 다르다. DAO에서는 select, insert, delete, update 이렇게 SQL적인 메서드 이름을 가지고 있다면, Biz Logic의 메서드들은 업무 용어스럽다.(registerUser)

만약 UserService의 registerUser 메서드가 없었다면 Controller에서 아래와 같은 코드들이 들어가야 한다.

```
userDao.insertUser();
userHistoryDao.insertUserHisotry();
```

위의 코드를 서비스 계층에 옮기고, registerUser라는 메서드를 만들어서 두 줄 코드를 집어 넣으면 Controller에서는 registerUser()만 호출해서 사용하면 된다.

그럼 회원가입 정책이 바뀌어도 Presentation Layer인 Controller는 전혀 안 바뀌어도 된다. 이렇게 3개 계층으로 나눔으로써 관심사가 제대로 분리된 것이다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e677c2a1-3a69-459b-bbf0-5280f24b5dc0/Untitled.png)

+) @Controller, @Service, @Repository 애너테이션이 @Component 애너테이션을 다 포함하고 있기 때문에 <component-scan>으로 자동스캔이 다 된다.

---

그리고 또 하나의 장점은 Tx를 적용하기에는 서비스 계층이 적합하다.

```
public void registerUser(User user) {
		userDao.insertUser(user);
		userHistoryDao.insertUserHistory(user);
}
```

registerUser 메서드에 User 정보를 주면, userDao와 userHistoryDao에 insert한다. 둘 다 DB를 다루는 작업이라 1, 2번 작업 중에 둘 중 하나라도 실패하면 회원가입이 실패해야 한다. 이게 바로 트랜잭션이다.

트랜잭션 처리를 Controller에서 해도 상관은 없는데 하게 되면 불필요한 Tx 코드가 들어가기 때문에 Controller가 너무 비대하고 복잡해진다. 이보다는 서비스 계층에서 트랜잭션 처리를 하는 것이 맞다.

### 2. TransactionManager란?

- **DAO의 각 메서드는 개별 Connection을 사용**
트랜잭션은 1개의 Connection에서 이루어저야 한다. 트랜잭션을 관리해주는 게 필요함.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8cb09984-e78e-4bdf-9d8a-f18f4252f452/Untitled.png)

- **TransactionManager**
같은 트랜잭션 내에서 같은 Conecction을 사용할 수 있게 관리해준다.

→ 같은 트랜잭션에 있는 명령어들은 같은 커넥션을 쓰게 해주는 게 TransactionManager

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7a5f47f2-1b0a-4e98-9cac-697598be8317/Untitled.png)

- **DAO에서 Conecction을 얻거나 반환할 때 **`**DataSourceUtils**`**를 사용해야 한다.**
![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/58b10c50-b193-42c4-a8cb-1ae4ec1f7407/Untitled.png)

### 3. TransactionManager로 Transaction 적용하기

```
public void insertWithTx() {
    PlatformTransactionManager tm = new DataSourceTransactionManager(ds); // TxManager를 생성
    TransactionStatus status = tm.getTransaction(new DefaultTransactionDefinition());

    // Tx 시작
    try {
        a1Dao.insert(1, 100);
        a1Dao.insert(1, 200);

        tm.commit(status); // Tx 끝 - 성공(커밋)
    } catch (Exception e) {
        tm.rollback(status); // Tx 끝 - 실패(롤백)
    }
}
```

`root-context.xml`

```
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource"/> <!-- dataSource를 setter로 지정 -->
</bean>
<tx:annotation-driven/> <!-- transactional 애너테이션을 사용하려면 이게 필요하다. -->
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/32b88c7f-bd56-4030-9445-1099d752a5f5/Untitled.png)

### 4. @Transactional로 Transaction 적용하기

- AOP를 이용한 **핵심 기능**과 **부가 기능**의 **분리**
AOP = 자동 코드 추가. Around = Before + After

- @Transactional은 클래스나 인터페이스에도 붙일 수 있음

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/75cd911c-2885-4558-a2d0-46c9de629723/Untitled.png)

#### 실습1 - TxManager 직접 생성

`A1Dao.java`

```
@Repository
public class A1Dao {
    @Autowired
    DataSource ds;

    public int insert(int key, int value) throws Exception {
        Connection conn = null;
        PreparedStatement pstmt = null;

        try {
//            conn = ds.getConnection();
            conn = DataSourceUtils.getConnection(ds);
            System.out.println("conn = " + conn);

            pstmt = conn.prepareStatement("insert into a1 values(?, ?)");
            pstmt.setInt(1, key);
            pstmt.setInt(2, value);

            return pstmt.executeUpdate();
        } catch (SQLException e) {
            e.printStackTrace();
            throw e; // 예외를 처리하고 호출한 곳에 또 던져야 거기서 예외를 받아서 tx rollback을 수행할 수 있다.
        } finally {
//            close(conn, pstmt);
            // connection을 닫으면 tx가 종료되기 때문에 pstmt만 닫는다.
            close(pstmt);
            DataSourceUtils.releaseConnection(conn, ds); // 무조건 conn을 close하지 않고 TxManager가 판단을 한다.
        }
    }

    private void close(AutoCloseable... acs) { // 가변인자
        for(AutoCloseable ac :acs)
            try { if(ac!=null) ac.close(); } catch(Exception e) { e.printStackTrace(); }
    }

    public void deleteAll() throws SQLException {
        Connection conn = ds.getConnection(); // deleteAll은 Tx와 별개로 동작해야 하기 때문에 DataSourceUtils.getConnection(ds)로 하지 않는다.
        String sql = "delete from a1";

        PreparedStatement pstmt = conn.prepareStatement(sql);
        pstmt.executeUpdate();

        close(pstmt, conn);

    }
}
```

`A1DaoTest.java`

```
@RunWith(SpringJUnit4ClassRunner.class) // ac를 자동으로 만들어줌
@ContextConfiguration(locations = {"file:src/main/webapp/WEB-INF/spring/**/root-context.xml"})
public class A1DaoTest {
    @Autowired
    A1Dao a1Dao;

    @Autowired
    DataSource ds;

    @Test
    public void insertTest() throws Exception {
        // TxManager를 생성
        PlatformTransactionManager tm = new DataSourceTransactionManager(ds);
        // tx 시작
        TransactionStatus status = tm.getTransaction(new DefaultTransactionDefinition());

        // +) Dao에서 Connection 가져오는 부분을 고쳐야 함

        try {
            a1Dao.deleteAll();
            a1Dao.insert(1, 100);
            a1Dao.insert(1, 200);

            tm.commit(status);
        } catch (Exception e) {
            e.printStackTrace();
            tm.rollback(status);
        } finally {

        }
    }
}
```

#### 실습2 - TxManager 주입 받아서 처리

이번에는 TxManager를 직접 생성하지 않고 주입받아서 처리해보자.

`root-context.xml`

transactionManager를 bean으로 등록.

```
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource"/> <!-- dataSource를 setter로 지정 -->
</bean>
```

```
PlatformTransactionManager tm = new DataSourceTransactionManager(ds);
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/77764d69-6e02-496d-b753-aee5ec4d23d7/Untitled.png)

`A1DaoTest.java`

```
@RunWith(SpringJUnit4ClassRunner.class) // ac를 자동으로 만들어줌
@ContextConfiguration(locations = {"file:src/main/webapp/WEB-INF/spring/**/root-context.xml"})
public class A1DaoTest {
    @Autowired
    A1Dao a1Dao;

    @Autowired
    DataSource ds;

    **@Autowired
    DataSourceTransactionManager tm;
**

    @Test
    public void insertTest() throws Exception {
        // TxManager를 생성
//        PlatformTransactionManager tm = new DataSourceTransactionManager(ds);
        // tx 시작
        TransactionStatus status = tm.getTransaction(new DefaultTransactionDefinition());

        // +) Dao에서 Connection 가져오는 부분을 고쳐야 함

        try {
            a1Dao.deleteAll();
            a1Dao.insert(1, 100);
            a1Dao.insert(2, 200);

            tm.commit(status);
        } catch (Exception e) {
            e.printStackTrace();
            tm.rollback(status);
        } finally {

        }
    }
}
```

#### 실습3

```
create table b1 select * from a1 where false; # 테이블만 생성
create table b1 select * from a1; # 테이블 생성 & 데이터도 복사
```

`TxService.java`

```
@Service
public class TxService {
    @Autowired
    A1Dao a1Dao;

    @Autowired
    B1Dao b1Dao;

    public void insertA1WithoutTx() throws Exception {
        a1Dao.insert(1, 100);
        a1Dao.insert(1, 100);
    }

    @Transactional // RuntimeException, Error만 rollback
//    @Transactional(rollbackFor = Exception.class) // Exception과 그 자손을 rollback
    public void insertA1WithTxFail() throws Exception {
        a1Dao.insert(1, 100); // 성공
//        throw new RuntimeException();
        throw new Exception();
//        a1Dao.insert(1, 100); // 실패
    }

    @Transactional
    public void insertA1WithTxSuccess() throws Exception {
        a1Dao.insert(1, 100);
        a1Dao.insert(2, 100);
    }
}
```

`TxServiceTest.java`

```
// 위의 두 줄이 왜 필요한지 이해가 안됨..
@RunWith(SpringJUnit4ClassRunner.class) // ac를 자동으로 만들어줌
@ContextConfiguration(locations = {"file:src/main/webapp/WEB-INF/spring/**/root-context.xml"})
public class TxServiceTest {
    @Autowired
    TxService txService;

    @Test
    public void insertA1WithoutTxTest() throws Exception {
//        txService.insertA1WithTxSuccess();
        txService.insertA1WithTxFail();
    }
}
```

### 5. @Transactional의 속성

### 6. propagation 속성값

트랜잭션의 경계가 어디냐에 따라 롤백 시 돌아가는 위치가 달라지기 때문에 굉장히 중요하다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/76fb8a35-6e1a-4728-83a9-f31c389be970/Untitled.png)

### 7. REQUIRED와 REQUIREDS_NEW

#### REQUIRED

Tx이 진행중이면 참여하고, 없으면 새로운 Tx 시작(디폴트)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5f4e4d0f-1d75-4b4e-bbf9-a9e58569d953/Untitled.png)

🚨 내부적 호출은 Tx 안 먹는다?

#### REQUIRED

Tx이 진행 중이건 아니건, 새로 Tx 시작(Tx안에 다른 Tx ← 서로 완전히 다른 Tx)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ff66cd74-8a59-4a35-8dc1-d2bb3829a05e/Untitled.png)

#### 실습1 - REQUIRES

![이미지 2023. 5. 13. 오후 3.19.jpg](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2b81de9d-2745-49ca-8a70-334edc477757/%E1%84%8B%E1%85%B5%E1%84%86%E1%85%B5%E1%84%8C%E1%85%B5_2023._5._13._%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_3.19.jpg)

`TxService.java`

```
@Service
public class TxService {
    @Autowired A1Dao a1Dao;
    @Autowired B1Dao b1Dao;

    @Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
    public void insertA1WithTx() throws Exception {
        a1Dao.insert(1, 100); // 성공
        insertB1WithTx();
        a1Dao.insert(2, 100); // 성공
    }

    @Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
    public void insertB1WithTx() throws Exception {
        b1Dao.insert(1, 100); // 성공
        b1Dao.insert(1, 200); // 실패
    }
}
```

`TxServiceTest.java`

```
// 위의 두 줄이 왜 필요한지 이해가 안됨..
@RunWith(SpringJUnit4ClassRunner.class) // ac를 자동으로 만들어줌
@ContextConfiguration(locations = {"file:src/main/webapp/WEB-INF/spring/**/root-context.xml"})
public class TxServiceTest {
    @Autowired
    TxService txService;

    @Test
    public void insertA1WithoutTxTest() throws Exception {
//        txService.insertA1WithTxSuccess();
        txService.insertA1WithTx();
    }
}
```

#### 실습2 - REQUIRES_NEW

![무제.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/671e68cf-0a48-402c-bbe5-f70d70b80598/%E1%84%86%E1%85%AE%E1%84%8C%E1%85%A6.png)

**[강의 내용 보완]**

@Transactional이 동작하지 않는 이유는 같은 클래스에 속한 메서드끼리의 호출(내부 호출)이기 때문.

프록시 방식(디폴트)의 AOP는 내부 호출인 경우, Advice가 적용되지 않음. 그래서 Tx가 적용되지 않는 것임.

두 메서드를 별도의 클래스로 분리하면 Tx가 적용됨.

근본적인 해결은 프록시 방식이 아닌 다른 방식을 사용해야함.(자세한 설명을 생략)

중요한 내용이니 반복해서 보아야 한다.
