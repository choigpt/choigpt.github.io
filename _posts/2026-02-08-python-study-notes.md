---
layout: post
title: 파이썬 학습 노트
date: 2026-02-08 11:45:00
tags: [python, programming, study]
---

파이썬 학습 과정에서 습득한 핵심 개념들을 카테고리별로 정리한 노트입니다.

## 1. 함수 인자 (Arguments)

### 가변 인자
- `*args`: 입력 개수를 가변적으로 받음, 대입할 땐 리스트로 모음
- `**kwargs`: 키워드 인자를 딕셔너리로 받음
- `*`: 리스트 괄호를 벗겨줌 (언패킹)
- `/`: 매개변수에 `/` 있으면 그 앞에 있는 건 키워드 입력 못 쓰는 제약
- `,*,`: 사이에 있으면 키워드 강제 아규먼트

### partial
- 함수 입력 중 일부만 받는 부분, lambda 써도 됨
- partial vs lambda 차이: 부분입력 변수로 할 거면

## 2. 스코프 (Scope)

### 변수 스코프
- `global`: 제일 바깥, 지금보다 바깥에 있는 걸 바꾸겠다
- `nonlocal`: 지금보다 바깥
- nonlocal로 지금보다 바깥인 게 만약에 제일 밖이다? 그럼 nonlocal 안됨, 그건 global local

### 네임 맹글링
- `__name`: name mangling (private 변수)
- `__all__`: 의 기능 (모듈에서 export할 항목 지정)

## 3. 제너레이터 (Generator)

### yield 작동
- `yield`: return과 비슷하면서 다름, 코드 상태를 기억
- `def` 아래 `yield` 있으면 함수 아니라 generator로 인식
- yield 자체가 마지막을 기억
- 두 번째 하면 안됨
- `yield as` 변수처럼 사용 가능
- lazy evaluation

### generator expression
- 메모리 효율적인 반복

## 4. 컨텍스트 매니저 (Context Manager)

### with 문
- with 감싸라, contextmanager로 쉽게 사용해라
- yield 아래 에러나면 실행이 안됨
- 그것도 무조건 실행시키고 싶으면 try finally
- with 잘 쓰면 코드가 매우 깔끔해진다
- with class로 설계

### 구조
- `__enter__`: with 문 진입 시
- `__exit__`: with 문 종료 시
- `@contextmanager`: with문 원툴
  ```
  위
  yield
  아래
  ```

## 5. 데코레이터 (Decorator)

### 기본 데코레이터
- 코드 추상화
- `@wraps`: 코드 동작 안 바꿈

### 클래스 관련 데코레이터
- `@property`: private 같은, 계산 필요 읽기 전용
- `@staticmethod`: 정적 메서드
- `@classmethod`: 클래스 메서드
- `@abstractmethod`: 추상 메서드, 상속해라, 구현해라, 인스턴스화 하지마라

### 함수 관련 데코레이터
- `lru_cache`: 예전에 이미 받은 입력이 있으면 그대로 리턴, memoization
  - `dict()`로 직접 구현 가능
  - register 위 ram 아래

## 6. 내장 함수 (Built-in Functions)

### 반복/변환
- `map`: 함수 적용
- `imap`, `imap_unordered`: 멀티프로세싱
- `reduce`: 반복? 함수형 프로그래밍, 함수, 반복 대상
- `filter`: 필터링
- `zip`: 대응되는 정보들 직관적 간결
- `enumerate`: 시작을 세다

### 비교/검사
- `any`: or 연산
- `all`: and 연산
- `isinstance`: 타입 검사
- `is`: 메모리 기준
- `==`: 내용 기준
- `in`: 비교 연산자
- 비교연산자 두 개 쓰는 것: `in`은 사실 비교 연산자, 사잇값을 보는 거랑 동일

### 속성 관련
- `getattr`: 속성 가져오기 (runtime에서 문자열로 변수 접근)
- `setattr`: 속성 설정 (runtime에서 문자열로 변수 바꿔줄 수 있음)
- `dir`: 어떤 attribute 파악
- `__dict__`: 요게 더 유용, 인스턴스 변수만 가져옴
- `locals`: 활용?

### 문자열 관련
- `repr`: 이걸 그대로 복붙할 때 코드 그대로 나오게
- `str`: 읽기 좋게
- `ord`: 문자를 숫자로
- 문자열 포맷: `{:{}}`

### 기타
- `assert`: 디버깅용
- `NotImplementedError`: 구현 강제

## 7. 자료구조 (Data Structures)

### 리스트
- list comprehension: 리스트 한줄 for문
- `.sort()`: 정렬 시키기만 하고 리턴값은 없다, in place sorting, 좀 조심
- `sorted()`: 입력을 받아서 정렬한 결과를 리턴

### 딕셔너리
- 딕셔너리 최초에는 처음 넣는 자료가 키가 없으면 에러
- `defaultdict`: 기본값 설정
- `**`: 딕셔너리 언패킹, 함수 인자 이름이 되고

### 배열
- array는 list랑 비슷
- `mean`, `std`
- `axis`: 차원
- `average`: weight를 줄 수 있음, mean과 다름
- `std ddof`: n-1

### 복사
- 파이썬은 메모리 위치 정보만 가지고 있음
- `.copy()`: 메모리 위치 정보 복사
- `deepcopy`: 깊은 복사
- 기본값에 리스트 금지

## 8. 파일/경로 처리 (File/Path)

### os.path
- `join`: OS가 달라져도 알아서 디렉토리 자동으로 맞춰줌
- `basename`: 파일 이름만 추출
- `dirname`: 폴더명까지만
- `split`: 으로 둘다 뽑을 수도 있음
- `exists`: 존재 여부
- `isfile`: 파일인지
- `isdir()`: 디렉토리인지

### 폴더 생성
- `makedir`: 미리 폴더 존재하면 에러, 두 단계 이상 폴더 못 만듦
- `makedirs`: `exist_ok` 쓰면 에러 안남

## 9. 임포트 (Import)

### 기본 임포트
- `import util`: 디버깅
- `import inspect`: 검사
- `from import` 점 붙이는 relative import: 상대 경로
- `__init__.py`: 폴더를 임포트할 때 쓰임
- `-m` 옵션: 모듈로 실행

### 동적 임포트
- `importlib`: import module 쓰는 이유는 문자열로 임포트하고 싶을 때 runtime에서 쓰면 됨
- `reload`: 프로세스 중에 디버깅 중에 새로 import 하고 있을 때

### 설치
- `pip install -e .`: 수정 반영 가능하게
- `site packages` 폴더

### 기타
- `__pycache__`: 코드 자체가 길면 import하는 거 bytecode로 바꿔놓는 거 미리 해놓음, import 속도만 빨라짐

## 10. 디버깅 (Debugging)

### 디버거
- `pdb`: 파이썬 디버거
- `breakpoint()`: 3.7 이상은 브레이크포인트 가능
- `.l`: 코드 보기
- `n`, `s`: 디버깅 명령어

### 기타
- `verbose debug`: 상세 디버그 모드
- `float("nan")`: 이면 좀 이상해짐

## 11. 함수형 프로그래밍

### 람다
- `lambda`: 익명 함수

### 한줄 표현식
- 한줄 if: 3항 연산자 같음, 하나의 값으로 취급
- 한줄 for: list comprehension

### 평가
- short circuit evaluation: 단축 평가

## 12. 멀티프로세싱

- 멀티프로세싱 pool cpu count
- `map`
- `imap`
- `imap_unordered`

## 13. 기타

### 연산자
- `:=`: walrus operator (대입 표현식)

### 전처리
- preprocessing

### 잡기술
- jpg 벡터로 읽고 무채색으로 만드는 법: 픽셀 평균?
- `argparse`: 명령줄 인자 파싱
