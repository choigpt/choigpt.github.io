---
title: "Spring DI 스터디"
source: "Notion local cache"
notion_url: "https://app.notion.com/p/389779d1d7da81de823ddcf6626e4332"
imported_at: "2026-07-02"
tags:
  - notion-import
  - backend-developer
---

# Spring DI 스터디

## 핵심 관점

DI는 객체 생성과 의존성 연결 책임을 개발자가 직접 처리하지 않고 컨테이너에게 맡기는 구조다.

Spring DI의 흐름은 다음처럼 이해할 수 있다.

```
직접 생성
-> Factory Method
-> config.txt
-> Reflection
-> AppContext
-> Bean
-> XML
-> Component Scan
-> @Autowired
```

## 1. 직접 객체 생성

```
UserRepository repository = new MemoryUserRepository();
UserService service = new UserService(repository);
```

단순하지만 구현체가 바뀌면 코드를 수정해야 한다.

## 2. 설정 파일과 Reflection

객체 생성 정보를 코드 밖으로 분리한다.

```
repository=com.example.MemoryUserRepository
```

```
Properties p = new Properties();
p.load(new FileReader("config.txt"));
Class clazz = Class.forName(p.getProperty("repository"));
Object obj = clazz.newInstance();
```

핵심은 코드가 구체 클래스를 직접 알지 않게 만드는 것이다.

## 3. AppContext와 Bean

객체를 매번 새로 만들지 않고 저장소에 보관한다.

```
class AppContext {
    Map<String, Object> map = new HashMap<>();

    Object getBean(String key) {
        return map.get(key);
    }
}
```

Spring에서 컨테이너가 관리하는 객체를 Bean이라고 부른다.

## 4. XML 기반 DI

초기 Spring은 XML로 Bean을 등록했다.

```
<bean id="userRepository" class="com.example.MemoryUserRepository"/>
<bean id="userService" class="com.example.UserService"/>
```

의존성 연결은 property 또는 constructor-arg로 처리했다.

```
<bean id="userService" class="com.example.UserService">
    <property name="repository" ref="userRepository"/>
</bean>
```

```
<bean id="userService" class="com.example.UserService">
    <constructor-arg ref="userRepository"/>
</bean>
```

## 5. Component Scan

XML이 커지자 Spring은 클래스를 자동으로 탐색하기 시작했다.

```
@Repository
public class MemoryUserRepository {
}

@Service
public class UserService {
}
```

```
패키지 탐색
-> @Component 계열 발견
-> Bean 생성
-> ApplicationContext 저장
```

## 6. 자동 주입

XML의 property ref는 @Autowired로 대체된다.

```
@Autowired
private UserRepository repository;
```

@Autowired는 Type 기준으로 Bean을 찾는다.

@Resource는 Name 기준으로 Bean을 찾는다.

```
@Resource
private UserRepository userRepository;
```

@Value는 값 주입에 사용한다.

```
@Value("${page.size}")
private int pageSize;
```

동일 타입 Bean이 여러 개 있으면 @Qualifier로 선택한다.

```
@Autowired
@Qualifier("jdbcRepo")
private UserRepository repository;
```

## 정리

DI는 단순히 @Autowired를 쓰는 기능이 아니다.

객체 생성 책임과 의존성 연결 책임을 컨테이너로 넘겨서 변경에 유리한 구조를 만드는 방식이다.
