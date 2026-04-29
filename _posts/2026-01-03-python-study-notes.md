---
layout: post
title: 파이썬 학습 노트
date: 2026-02-08 11:45:00
tags: [python, programming, study]
---

파이썬 학습 과정에서 습득한 핵심 개념들을 카테고리별로 정리한 노트.

## 1. 함수 인자 (Arguments)

### 가변 인자
- `*args`: 위치 인자를 개수 제한 없이 받아서 **튜플**로 묶어줌
- `**kwargs`: 키워드 인자를 **딕셔너리**로 받음
- `*`: 리스트/튜플 앞에 붙이면 괄호를 벗겨줌 (언패킹). `print(*[1,2,3])` → `1 2 3`
- `/`: 매개변수 정의에서 `/` 앞에 있는 인자는 **위치 전용** (키워드로 넘길 수 없음)
  ```python
  def f(a, b, /, c):  # a, b는 위치 전용, c는 둘 다 가능
      pass
  f(1, 2, c=3)  # OK
  f(a=1, b=2, c=3)  # TypeError
  ```
- `*,`: `*` 뒤에 오는 인자는 **키워드 전용** (반드시 이름을 명시해야 함)
  ```python
  def f(a, *, b):  # b는 키워드 전용
      pass
  f(1, b=2)  # OK
  f(1, 2)    # TypeError
  ```

### partial
- `functools.partial`: 함수의 인자 일부를 미리 고정한 새 함수를 만듦
- lambda로도 비슷하게 가능하지만 차이가 있음:
  - `partial`은 고정 시점의 값을 **바인딩** (나중에 변수가 바뀌어도 영향 없음)
  - `lambda`는 **호출 시점**에 변수를 평가 (클로저이므로 변수가 바뀌면 결과도 바뀜)
  ```python
  from functools import partial

  x = 10
  f1 = partial(pow, x)    # x=10이 바인딩됨
  f2 = lambda y: pow(x, y) # x를 호출 시점에 평가

  x = 20
  f1(2)  # 100 (10² — partial은 바인딩 시점의 10)
  f2(2)  # 400 (20² — lambda는 호출 시점의 20)
  ```

## 2. 스코프 (Scope)

### 변수 스코프 — LEGB 규칙
Python은 변수를 찾을 때 **Local → Enclosing → Global → Built-in** 순서로 탐색.

- `global`: 함수 안에서 **모듈 최상위(전역)** 변수를 수정하겠다고 선언
- `nonlocal`: 함수 안에서 **한 단계 바깥 함수**의 변수를 수정하겠다고 선언
- `nonlocal`은 모듈 최상위까지 올라갈 수 없음 — 제일 바깥이면 `global`을 써야 함
  ```python
  x = 0  # global

  def outer():
      x = 1  # enclosing
      def inner():
          nonlocal x  # outer의 x를 수정
          x = 2
      inner()
      print(x)  # 2

  outer()
  print(x)  # 0 (global x는 안 바뀜)
  ```

### 네임 맹글링
- `__name` (언더스코어 두 개 시작): 클래스 안에서 `_ClassName__name`으로 변환됨 — 외부 접근을 어렵게 만드는 관례적 private
- `__all__`: 모듈에서 `from module import *` 할 때 **내보낼 이름 목록**을 지정
  ```python
  __all__ = ['public_func', 'PublicClass']  # 이것만 export
  ```

## 3. 제너레이터 (Generator)

### yield 작동
- `yield`: 값을 반환하되 함수 상태를 **보존**함. 다음 호출 시 yield 다음 줄부터 재개
- `def` 안에 `yield`가 하나라도 있으면 일반 함수가 아니라 **generator 함수**로 인식
- generator는 **한 번만 순회 가능** — 소진(exhausted)되면 다시 쓸 수 없음
  ```python
  def gen():
      yield 1
      yield 2

  g = gen()
  list(g)  # [1, 2]
  list(g)  # [] — 이미 소진됨. 다시 쓰려면 gen()을 새로 호출해야 함
  ```
- **lazy evaluation**: 값을 미리 다 만들지 않고 요청할 때마다 하나씩 생성 → 메모리 절약
- `send()`: generator에 **값을 보내면서** 다음 yield로 진행. `value = yield`로 받음
  ```python
  def accumulator():
      total = 0
      while True:
          value = yield total  # 바깥에서 send()로 받은 값이 value에 들어감
          total += value

  acc = accumulator()
  next(acc)        # 0 (첫 yield까지 진행, 아직 send 못 함)
  acc.send(10)     # 10
  acc.send(20)     # 30
  ```
- `yield from`: 다른 iterable/generator에 **위임**. 중첩 generator를 평탄하게 연결
  ```python
  def chain(*iterables):
      for it in iterables:
          yield from it  # it의 모든 값을 하나씩 yield

  list(chain([1,2], [3,4]))  # [1, 2, 3, 4]
  ```

### generator expression
- list comprehension과 동일한 문법인데 `[]` 대신 `()` 사용
- 메모리를 거의 안 씀 (lazy)
  ```python
  squares = (x**2 for x in range(1000000))  # 메모리에 100만 개를 올리지 않음
  ```

## 4. 컨텍스트 매니저 (Context Manager)

### with 문
- 리소스(파일, 연결, 락 등)의 **획득과 해제를 보장**하는 패턴
- `with` 블록을 빠져나가면 에러가 나든 안 나든 정리 코드가 실행됨

### 클래스로 구현
```python
class Timer:
    def __enter__(self):
        self.start = time.time()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        # 에러 여부와 관계없이 항상 실행됨
        print(f"경과: {time.time() - self.start:.2f}초")
        return False  # 예외를 삼키지 않음

with Timer():
    do_something()
```

### @contextmanager로 간단하게
```python
from contextlib import contextmanager

@contextmanager
def timer():
    start = time.time()    # __enter__에 해당
    try:
        yield              # with 블록 실행
    finally:
        # __exit__에 해당 — try/finally로 감싸야 에러 시에도 실행됨
        print(f"경과: {time.time() - start:.2f}초")
```
- `yield` 위 = `__enter__`, `yield` 아래 = `__exit__`
- yield 아래를 `try/finally`로 감싸지 않으면 **with 블록에서 에러 시 정리 코드가 실행 안 됨**

## 5. 데코레이터 (Decorator)

### 기본 데코레이터
- 함수를 감싸서 **앞뒤에 동작을 추가**하는 패턴. 로깅, 타이밍, 인증 체크 등
- `@wraps(func)`: 데코레이터가 원본 함수의 `__name__`, `__doc__` 등을 **보존**하도록 함
  ```python
  from functools import wraps

  def log(func):
      @wraps(func)  # 이걸 안 쓰면 help(my_func)에 wrapper 정보가 나옴
      def wrapper(*args, **kwargs):
          print(f"호출: {func.__name__}")
          return func(*args, **kwargs)
      return wrapper
  ```

### 클래스 관련 데코레이터
- `@property`: getter를 메서드가 아닌 **속성처럼** 접근하게 함. 계산이 필요한 읽기 전용 값에 적합
  ```python
  class Circle:
      def __init__(self, r):
          self._r = r

      @property
      def area(self):
          return 3.14 * self._r ** 2

  c = Circle(5)
  c.area   # 78.5 — 메서드 호출이 아닌 속성 접근처럼 사용
  ```
- `@staticmethod`: `self` 없이 클래스에 묶인 일반 함수. 인스턴스/클래스 상태에 접근 안 함
- `@classmethod`: 첫 인자가 `cls`. 팩토리 메서드(대안 생성자)에 자주 사용
- `@abstractmethod`: ABC(Abstract Base Class)에서 사용. **반드시 하위 클래스에서 구현**해야 함. 직접 인스턴스화 불가

### 캐싱 데코레이터
- `@lru_cache`: 같은 입력이 들어오면 이전 결과를 **그대로 반환** (memoization)
  - 피보나치 같은 재귀에서 $$ O(2^n) $$ → $$ O(n) $$으로 만들어줌
  - dict로 직접 구현할 수도 있지만 `lru_cache`가 스레드 세이프하고 maxsize 관리도 해줌
  ```python
  from functools import lru_cache

  @lru_cache(maxsize=128)
  def fib(n):
      if n < 2: return n
      return fib(n-1) + fib(n-2)
  ```

## 6. 클래스 / OOP

### 기본
```python
class Animal:
    species_count = 0  # 클래스 변수 (모든 인스턴스가 공유)

    def __init__(self, name):
        self.name = name  # 인스턴스 변수
        Animal.species_count += 1

    def speak(self):
        raise NotImplementedError  # 하위 클래스에서 구현 강제
```

### super()와 MRO
- `super()`: 부모 클래스의 메서드를 호출. 다중 상속에서 **MRO(Method Resolution Order)** 순서를 따름
- MRO 확인: `ClassName.__mro__` 또는 `ClassName.mro()`
  ```python
  class A:
      def greet(self): print("A")

  class B(A):
      def greet(self):
          super().greet()  # A.greet() 호출
          print("B")
  ```
- 다중 상속 시 MRO는 **C3 선형화** 알고리즘으로 결정됨. 다이아몬드 상속 문제를 해결

### 매직 메서드
- `__repr__`: `repr(obj)` — 개발자용 표현. **복붙하면 그대로 재현 가능**하게 작성
- `__str__`: `str(obj)` — 사용자용 표현. 읽기 좋게
- `__len__`, `__getitem__`, `__iter__`: 컨테이너 프로토콜
- `__eq__`, `__lt__` 등: 비교 연산자 오버로딩

### dataclass
- 보일러플레이트(`__init__`, `__repr__`, `__eq__`)를 자동 생성
  ```python
  from dataclasses import dataclass

  @dataclass
  class Point:
      x: float
      y: float
      label: str = ""  # 기본값

  p = Point(1.0, 2.0)
  print(p)  # Point(x=1.0, y=2.0, label='')
  ```
- `frozen=True`: 불변 객체로 만듦 (해시 가능해짐 → set/dict 키로 사용 가능)

## 7. 예외 처리 (Exception Handling)

```python
try:
    result = do_something()
except ValueError as e:
    print(f"값 에러: {e}")    # 특정 예외만 잡기
except (TypeError, KeyError):
    print("타입 또는 키 에러")  # 여러 예외 한꺼번에
except Exception as e:
    print(f"기타: {e}")        # 나머지 모든 예외 (보통 최후의 수단)
else:
    print(f"성공: {result}")    # 예외가 안 났을 때만 실행
finally:
    cleanup()                  # 예외 여부와 관계없이 항상 실행
```

- `else`는 `try` 블록이 **예외 없이 성공**했을 때만 실행됨. try 안에 성공 로직을 넣는 것보다 의도가 명확
- 커스텀 예외: `Exception`을 상속
  ```python
  class InsufficientBalance(Exception):
      def __init__(self, balance, amount):
          super().__init__(f"잔액 {balance}원, 요청 {amount}원")
  ```

## 8. 내장 함수 (Built-in Functions)

### 반복/변환
- `map(func, iter)`: 각 요소에 함수 적용. lazy (iterator 반환)
- `filter(func, iter)`: func이 True인 요소만. lazy
- `reduce(func, iter)`: 누적 연산. `from functools import reduce`
  ```python
  reduce(lambda acc, x: acc + x, [1,2,3,4])  # 10 (((1+2)+3)+4)
  ```
- `zip(*iters)`: 같은 위치의 요소끼리 튜플로 묶음. 가장 짧은 iterable 기준으로 끊김
- `enumerate(iter, start=0)`: 인덱스와 값을 함께 반환
- `sorted(iter, key=, reverse=)`: 새 리스트 반환. `key`에 정렬 기준 함수

### 비교/검사
- `any(iter)`: 하나라도 True면 True (OR)
- `all(iter)`: 전부 True여야 True (AND)
- `isinstance(obj, type)`: 타입 검사 (상속 포함)
- `is` vs `==`: `is`는 **메모리 주소**(identity) 비교, `==`는 **값**(equality) 비교
  ```python
  a = [1, 2]; b = [1, 2]
  a == b  # True (내용 같음)
  a is b  # False (다른 객체)
  ```
- `in`: 멤버십 검사. `1 < x < 10` 같은 **체이닝 비교**도 가능 — `(1 < x) and (x < 10)`과 동일

### 속성 관련
- `getattr(obj, 'name', default)`: 속성을 **문자열 이름으로** 가져옴. 런타임에 동적 접근
- `setattr(obj, 'name', value)`: 속성을 문자열 이름으로 설정
- `dir(obj)`: 모든 attribute 목록 (상속 포함)
- `obj.__dict__`: **인스턴스 변수만** 딕셔너리로 반환 (클래스 변수, 메서드 제외)
- `locals()`: 현재 스코프의 지역 변수를 딕셔너리로 반환. 디버깅 시 현재 상태 확인에 유용
  ```python
  def debug():
      x, y = 10, 20
      print(locals())  # {'x': 10, 'y': 20}
  ```

### 문자열 관련
- `repr(obj)`: 코드에 그대로 복붙하면 재현되는 문자열 (`'hello'` → `"'hello'"`)
- `str(obj)`: 사람이 읽기 좋은 문자열
- `ord('A')`: 문자 → 유니코드 숫자 (65). 역방향은 `chr(65)` → `'A'`
- f-string 포맷: `f"{value:>10.2f}"` — 우측 정렬 10칸, 소수점 2자리

### 기타
- `assert condition, message`: 조건이 False면 `AssertionError` 발생. 디버깅/테스트용. **운영 코드에서 입력 검증에 쓰면 안 됨** (`-O` 옵션으로 비활성화되므로)

## 9. 자료구조 (Data Structures)

### 리스트
- list comprehension: `[x**2 for x in range(10) if x % 2 == 0]`
- `.sort()`: **제자리 정렬** (in-place). 반환값 None. 원본이 바뀜
- `sorted()`: **새 리스트** 반환. 원본 안 바뀜
- `.sort()`의 함정: `result = my_list.sort()` 하면 `result`는 `None`

### 딕셔너리
- 존재하지 않는 키로 접근하면 `KeyError` → `.get(key, default)`로 안전하게 접근
- `defaultdict(int)`: 없는 키 접근 시 기본값(int → 0) 자동 생성
- `**d`: 딕셔너리 언패킹. 함수에 넘기면 키가 매개변수 이름이 됨
  ```python
  config = {"host": "localhost", "port": 8080}
  connect(**config)  # connect(host="localhost", port=8080)
  ```
- `Counter`: 요소 개수 세기. `Counter("hello")` → `{'l': 2, 'h': 1, 'e': 1, 'o': 1}`
- `deque`: 양쪽 끝 삽입/삭제가 $$ O(1) $$. 큐, 슬라이딩 윈도우에 적합
- `namedtuple`: 이름으로 접근 가능한 튜플. 가벼운 불변 데이터 구조
  ```python
  from collections import namedtuple
  Point = namedtuple('Point', ['x', 'y'])
  p = Point(1, 2)
  p.x  # 1
  ```

### 배열 (numpy)
- `np.mean()` vs `np.average()`: average는 **가중 평균** 가능 (`weights` 인자)
- `axis`: 연산 방향. `axis=0`은 행 방향(세로), `axis=1`은 열 방향(가로)
- `np.std(ddof=1)`: `ddof=1`이면 표본 표준편차 (n-1로 나눔)

### 복사
- 파이썬 변수는 **객체의 참조**(메모리 주소)를 저장. `b = a`는 같은 객체를 가리킴
- `.copy()` / `copy.copy()`: **얕은 복사** — 최상위 컨테이너는 새로 만들지만, 내부 객체는 여전히 공유
- `copy.deepcopy()`: **깊은 복사** — 내부 중첩 객체까지 전부 새로 생성
- 함수 기본값에 **mutable 객체 금지**: `def f(lst=[])` → 호출마다 같은 리스트를 공유. `def f(lst=None)` 패턴 사용
  ```python
  def f(lst=None):
      if lst is None:
          lst = []
      lst.append(1)
      return lst
  ```

## 10. itertools & 함수형 프로그래밍

### itertools
```python
from itertools import chain, product, combinations, permutations, islice, groupby

list(chain([1,2], [3,4]))          # [1, 2, 3, 4] — 여러 iterable 연결
list(product('AB', '12'))          # [('A','1'), ('A','2'), ('B','1'), ('B','2')] — 곱집합
list(combinations('ABC', 2))       # [('A','B'), ('A','C'), ('B','C')] — 조합
list(permutations('ABC', 2))       # [('A','B'), ('A','C'), ('B','A'), ...] — 순열
list(islice(range(1000), 5))       # [0, 1, 2, 3, 4] — lazy 슬라이싱
```

### 함수형 패턴
- `lambda`: 이름 없는 한 줄 함수. 복잡한 로직이면 `def`를 써라
- 삼항 연산자: `value_if_true if condition else value_if_false` — 하나의 **표현식**(값)
- **단축 평가 (short-circuit)**: `a or b`에서 a가 truthy면 b를 평가하지 않음
  ```python
  result = cache.get(key) or compute(key)  # 캐시에 있으면 compute 안 함
  ```

## 11. 파일/경로 처리

### os.path (레거시)
- `os.path.join('dir', 'file.txt')`: OS별 경로 구분자 자동 처리
- `os.path.basename('/a/b/c.txt')` → `'c.txt'`
- `os.path.dirname('/a/b/c.txt')` → `'/a/b'`
- `os.path.exists()`, `os.path.isfile()`, `os.path.isdir()`
- `os.makedirs('a/b/c', exist_ok=True)`: 중간 폴더까지 한꺼번에 생성, 이미 있어도 에러 안 남

### pathlib (현대적 방식, 권장)
```python
from pathlib import Path

p = Path('data') / 'raw' / 'file.csv'   # / 연산자로 경로 조합
p.exists()          # 존재 여부
p.is_file()         # 파일인지
p.stem              # 'file' (확장자 제외)
p.suffix            # '.csv'
p.parent            # Path('data/raw')
p.read_text()       # 파일 내용 읽기
p.mkdir(parents=True, exist_ok=True)  # makedirs와 동일
```
- `os.path`보다 **직관적이고 객체지향적**. 새 코드에서는 pathlib 사용 권장

## 12. 임포트 (Import)

### 기본
- `from . import module`: **상대 임포트** (같은 패키지 내). 패키지 외부에서 실행하면 에러
- `__init__.py`: 폴더를 패키지로 인식시킴. 비어있어도 됨
- `python -m module`: 모듈을 **스크립트로 실행**. 패키지 구조에서 상대 임포트가 작동하려면 이 방식 필요

### 동적 임포트
- `importlib.import_module('module_name')`: **문자열로** 모듈을 임포트. 플러그인 시스템 구현에 유용
- `importlib.reload(module)`: 이미 import된 모듈을 **다시 로드**. REPL/디버깅에서 코드 수정 후 반영할 때

### 기타
- `pip install -e .`: editable 설치. 소스 코드 수정이 **즉시 반영**됨 (개발 중 패키지)
- `__pycache__`: `.pyc` 파일 캐시 폴더. import 시 바이트코드 컴파일 결과를 저장해서 **다음 import가 빨라짐**. 실행 속도와 무관

## 13. 타입 힌팅 (Type Hints)

```python
from typing import Optional, Union, Protocol

def greet(name: str, age: int = 0) -> str:
    return f"{name} ({age})"

# Optional: None일 수 있는 타입
def find(key: str) -> Optional[str]:
    ...  # str 또는 None 반환

# Union: 여러 타입 중 하나 (3.10+에서는 str | int)
def process(data: Union[str, int]) -> None:
    ...

# Protocol: 덕 타이핑을 정적으로 검증 (구조적 서브타이핑)
class Printable(Protocol):
    def __str__(self) -> str: ...
```
- 타입 힌트는 **런타임에 영향 없음** — 문서화 + IDE 지원 + mypy 정적 검사용
- Python 3.10+: `str | None` 으로 `Optional[str]` 대체 가능

## 14. 비동기 (async/await)

```python
import asyncio

async def fetch(url):
    # await: 비동기 작업이 끝날 때까지 양보 (다른 코루틴이 실행됨)
    response = await aiohttp.get(url)
    return await response.text()

async def main():
    # gather: 여러 코루틴을 동시에 실행
    results = await asyncio.gather(
        fetch("https://a.com"),
        fetch("https://b.com"),
        fetch("https://c.com"),
    )

asyncio.run(main())
```
- `async def`: 코루틴 함수 정의
- `await`: 코루틴 결과를 기다림. **I/O 대기 중에 다른 코루틴이 실행**됨
- `asyncio.run()`: 이벤트 루프 생성 + 코루틴 실행 + 루프 정리
- GIL 때문에 CPU 연산은 병렬화 안 됨 → I/O 바운드 작업(네트워크, 파일)에만 효과적
- CPU 바운드는 `multiprocessing` 사용

## 15. 멀티프로세싱

```python
from multiprocessing import Pool, cpu_count

def heavy(x):
    return x ** 2

with Pool(cpu_count()) as pool:
    pool.map(heavy, range(100))         # 순서 보장, 전체 완료 후 반환
    pool.imap(heavy, range(100))        # 순서 보장, iterator (lazy)
    pool.imap_unordered(heavy, range(100))  # 순서 비보장, 빨리 끝나는 순
```
- `multiprocessing`: **프로세스 단위** 병렬 처리. GIL 우회. CPU 바운드에 적합
- `threading`: **스레드 단위**. GIL 때문에 CPU 병렬화 안 됨. I/O 바운드에 적합
- `concurrent.futures.ProcessPoolExecutor`: multiprocessing의 고수준 인터페이스
- 언제 뭘 쓸까:
  - **I/O 바운드** (HTTP 요청, 파일 읽기) → `async/await` 또는 `threading`
  - **CPU 바운드** (행렬 연산, 이미지 처리) → `multiprocessing`

## 16. 디버깅 (Debugging)

### pdb
- `breakpoint()`: Python 3.7+ 디버거 진입점. 이전에는 `import pdb; pdb.set_trace()`
- 디버거 명령어:
  - `n` (next): 다음 줄 실행 (함수 안으로 들어가지 않음)
  - `s` (step): 다음 줄 실행 (함수 안으로 들어감)
  - `c` (continue): 다음 breakpoint까지 실행
  - `l` (list): 현재 위치 주변 코드 보기
  - `p expr`: 표현식 평가/출력
  - `pp expr`: pretty print
  - `w` (where): 콜 스택 보기

### NaN 주의
- `float("nan")`: NaN은 자기 자신과도 같지 않음. **모든 비교가 False**
  ```python
  x = float("nan")
  x == x   # False (!)
  x != x   # True  (NaN 감지 방법)
  ```
- `math.isnan(x)`: NaN 검사는 이걸로

## 17. 기타

### walrus operator
- `:=` (Python 3.8+): 표현식 안에서 **대입과 사용을 동시에**
  ```python
  # 이전
  line = input()
  while line != "quit":
      process(line)
      line = input()

  # walrus
  while (line := input()) != "quit":
      process(line)
  ```

### argparse
```python
import argparse

parser = argparse.ArgumentParser(description="설명")
parser.add_argument("input", help="입력 파일")
parser.add_argument("-o", "--output", default="out.txt")
parser.add_argument("-v", "--verbose", action="store_true")
args = parser.parse_args()
# args.input, args.output, args.verbose 로 접근
```
