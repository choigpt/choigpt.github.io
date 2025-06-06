---
layout: post
title: "DBMS의 본질과 파일 입출력: MySQL 공부 전에 알아두기"
date: 2025-04-18 11:30:00 -0700
---

MySQL을 공부하다 보니 데이터베이스의 본질적인 부분에 관심이 생겼다. DBMS가 실제로는 어떻게 작동하는지, 그리고 일반 파일 시스템과는 어떤 차이가 있는지 궁금해졌다. 이 글에서는 데이터베이스 공부를 하면서 알게 된 DBMS의 핵심 개념들을 정리해봤다.

## DBMS란 무엇인가

DBMS(Database Management System)는 데이터베이스를 관리하는 소프트웨어 시스템이다. 단순히 데이터를 저장하는 것뿐만 아니라 DB 자체와 운영에 필요한 모든 기능을 포함한다.

```
DBMS = Database + Management System
```

관계형 DBMS(RDBMS)에서는 데이터 간의 관계가 핵심이다. 객체지향 프로그래밍에서는 클래스로 관계를 표현하지만, DB에서는 테이블로 이런 관계를 나타낸다. MySQL, Oracle, PostgreSQL 같은 것들이 RDBMS의 대표적인 예시다.

## 파일 저장과 DBMS 비교

처음에는 "왜 그냥 파일에 저장하지 않고 DBMS를 쓰는 걸까?"라는 의문이 들었다. 데이터가 적다면 그냥 파일에 저장해도 될 것 같은데.

알아보니 파일 저장 방식에는 몇 가지 문제점이 있다. 파일 형식 호환성이 떨어지고, 관리 시스템을 직접 만들어야 하며, 여러 사용자가 동시에 접근할 때 문제가 생긴다. 특히 동시성 문제는 생각보다 복잡하다.

## DBMS와 파일 I/O의 관계

알고 보니 DBMS의 본질은 결국 파일 입출력(I/O)이었다. DB에 저장되는 자료는 2차 메모리(하드디스크나 SSD)에 저장되는데, 이게 결국 파일 I/O 작업이다.

```
SQL 쿼리 작성 → DBMS 처리 → 파일 입출력 발생 → 디스크 저장
```

2차 메모리는 RAM보다 훨씬 느리다. 그래서 DBMS는 이 차이를 극복하기 위해 여러 최적화 기법을 사용한다. 자주 쓰는 데이터는 메모리에 캐싱하고, 인덱싱으로 검색 속도를 높이며, 쿼리 최적화와 병렬 처리로 효율성을 향상시킨다.

## DBMS의 주요 특성

DBMS의 중요한 특성으로는 동시성 제어, 데드락 방지, I/O 최적화가 있다. 동시성 제어는 여러 사용자가 같은 데이터에 접근할 때 일관성을 유지하는 것이고, 데드락 방지는 트랜잭션이 서로 리소스를 기다리며 무한 대기하는 상황을 막는 것이다. I/O 최적화는 디스크 접근을 최소화하는 기술이다.

성능 향상을 위해서는 DBMS의 내부 구조를 이해하는 것이 중요하다. 쿼리 최적화만 잘해도 성능이 크게 달라진다.

## DBMS와 파일 입출력 관계의 의미

DBMS를 공부하면서 알게 된 건, DBMS는 단순히 데이터를 저장하고 검색하는 게 아니라 파일 시스템 위에 구축된 고급 추상화 계층이라는 것이다. 이 계층은 데이터의 무결성, 보안, 동시성, 복구 기능을 제공하고, 파일 I/O 성능의 한계를 극복하려는 다양한 기법들의 모음이라고 볼 수 있다.

이제 이런 개념을 알고 MySQL의 구체적인 기능과 문법을 공부하면 더 잘 이해할 수 있을 것 같다.
