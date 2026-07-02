---
layout: post
title: "Spring 학습 지도: IoC, DI, AOP, Transaction을 어떤 순서로 볼 것인가"
date: 2026-07-02
tags: [Spring, IoC, DI, AOP, Transaction, Backend]
category: Backend
permalink: /spring-study-map/
excerpt: "Spring을 애너테이션 사용법으로 외우기보다 객체 생성 책임 분리, IoC Container, Bean, DI, 계층 분리, Transaction, AOP 흐름으로 정리한다."
---

Spring 프레임워크와 Spring Boot 관련 개념을 정리하는 카테고리입니다.

## 학습 흐름

Spring은 단순히 애너테이션 사용법으로 외우기보다 아래 흐름으로 정리합니다.

```
객체 생성 책임 분리
-> IoC Container
-> Bean
-> DI
-> 계층 분리
-> Transaction
-> AOP
-> Spring Boot 자동화
```

## 핵심 주제

### 1. IoC / DI / Bean

- [Spring DI 공부](/spring-di-study/)

- 객체 생성 책임을 개발자 코드에서 컨테이너로 이동
- Bean 등록과 조회
- XML 기반 설정
- Component Scan
- @Autowired, @Resource, @Value, @Qualifier

### 2. AOP / Proxy / Transaction

- 공통 관심사 분리
- Advice, Pointcut, Aspect
- Proxy 기반 동작
- @Transactional
- Commit / Rollback
- Spring Bean Lifecycle
- [Spring AOP 스터디](https://app.notion.com/p/389779d1d7da819f86fbedb08cf8dd3a)
- @Autowired vs 생성자 주입

### 3. 전체 개요

DI와 AOP를 포함한 Spring 학습 흐름을 빠르게 확인하는 개요 문서입니다.

- [Spring 스터디 개요](https://app.notion.com/p/389779d1d7da81a59e76cf05ca8d94ce)

### 4. 원본 / 참고 자료

원문 복사본 또는 참고용 자료입니다. 새 포스팅에는 원문을 그대로 쓰지 않고, 학습 노트 형태로 재구성합니다.

- [Spring DI와 AOP 학습 노트](/spring-di-aop-study-notes/)
- [Spring DI와 AOP 핵심 정리](https://app.notion.com/p/389779d1d7da81f4b0f2fe126a40f76c)
- JDK Dynamic Proxy vs CGLIB
- @Transactional 동작 원리
- Transaction Propagation
- Spring MVC
- Spring Security
- Spring Boot AutoConfiguration

## 정리 원칙

- 원문은 참고 자료로만 둔다.
- 새 문서는 내 언어로 재구성한다.
- 예제 코드는 원문 그대로 복사하지 않고 도메인을 바꿔서 사용한다.
- 개념 문서는 Spring 하위에 둔다.
- 프로젝트 적용 사례는 Project 문서와 연결한다.
