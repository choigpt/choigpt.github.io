---
layout: post
title: "DML 튜닝: INSERT 채번부터 Lock, 트랜잭션 동시성 제어까지"
date: 2026-01-09
categories: [Database]
tags: [lock, transaction, concurrency, database, dml-tuning, partition, mvcc]
last_modified_at: 2026-03-15
---

이 글은 『친절한 SQL 튜닝』 6장을 기반으로 하되, DML → Lock → 트랜잭션 흐름에 집중하기 위해 구조를 재편했다.

**뺀 것:** 6.1.1(DML 성능에 영향을 미치는 요소), 6.1.2(데이터베이스 Call과 성능), 6.1.3(Array Processing), 6.1.4(인덱스 및 제약 해제를 통한 대량 DML 튜닝).

**옮긴 것:** 6.4.3(채번 방식에 따른 INSERT 성능 비교)을 Part 1로 옮겼다. 6.4(Lock과 트랜잭션 동시성 제어)는 로우 락, 테이블 락, MVCC, 데드락, 격리수준, 커밋 전략으로 나누어 Part 2~3으로 구성했다.

DML이 빠르면 Lock이 짧아지고, Lock이 짧으면 동시성이 좋아지고, 트랜잭션 설계가 이 전체를 감싼다. 세 파트는 이 흐름을 따른다.

**Part 1: DML 튜닝** — INSERT/UPDATE/DELETE를 어떻게 빠르게 할 것인가

- [1. 기본 DML 튜닝](#1-기본-dml-튜닝) — INSERT 채번 방식(시퀀스 vs MAX+1), UPDATE/DELETE 조인, MERGE문
- [2. Direct Path I/O](#2-direct-path-io) — 버퍼 캐시를 우회하는 대량 INSERT, NOLOGGING의 위험성
- [3. 파티션을 활용한 DML 튜닝](#3-파티션을-활용한-dml-튜닝) — Range/List/Hash 파티셔닝, 파티션 Exchange

**Part 2: Lock과 동시성** — 여러 사용자가 동시에 접근할 때 어떻게 보호할 것인가

- [4. DML 로우 락](#4-dml-로우-락-row-lock) — 로우 락의 동작, INSERT와 유니크 인덱스 충돌
- [5. 테이블 락](#5-테이블-락-table-lock) — RS/RX/S/SRX/X 모드, 호환성
- [6. MVCC와 읽기 일관성](#6-mvcc와-읽기-일관성) — SELECT가 Lock을 안 거는 이유, Undo 세그먼트
- [7. 블로킹과 데드락](#7-블로킹과-데드락) — 인덱스로 인한 데드락, 예방 전략
- [8. 락 성능 최적화](#8-락-성능-최적화) — 쿼리 튜닝, 인덱스 설계, 트랜잭션 분리

**Part 3: 트랜잭션 관리** — 트랜잭션을 어떻게 설계하고 커밋할 것인가

- [9. 트랜잭션 기본과 주의사항](#9-트랜잭션-기본과-주의사항) — ACID, 트랜잭션 길이 문제, Snapshot Too Old
- [10. 커밋과 LGWR](#10-커밋과-lgwr-log-writer) — 메모리 구조, 동기/비동기 커밋
- [11. 동시성 제어와 격리수준](#11-동시성-제어와-격리수준) — 비관적/낙관적 제어, Dirty Read부터 Phantom Read까지, Oracle vs MySQL
- [12. 핵심 원칙](#12-핵심-원칙) — 락과 성능, 커밋 타이밍, 상황별 전략

---

# Part 1: DML 튜닝

### 1. 기본 DML 튜닝

DML(INSERT, UPDATE, DELETE)은 각각 성능 특성이 다르다. 왜 느린지 이해해야 튜닝할 수 있다.

#### INSERT 튜닝

INSERT는 DML 중 가장 수행 빈도가 높고, **채번(PK 생성) 방식**에 따라 성능 차이가 크다.

**채번 방식 비교:**

```
방식 1: 시퀀스(Sequence)

INSERT INTO orders (id, ...) VALUES (order_seq.NEXTVAL, ...);

내부 동작:
  1. 시퀀스에서 다음 번호 획득 (메모리에서 처리, 매우 빠름)
  2. 행 삽입
  3. 인덱스 갱신

성능: ★★★★★
이유: 시퀀스는 DB 내부에서 원자적으로 처리.
      캐시 옵션(CACHE 20)을 쓰면 20번에 1번만 딕셔너리 갱신.

CACHE 크기에 따른 성능 차이:

CACHE 1:  매 INSERT마다 딕셔너리 테이블 UPDATE → 딕셔너리 락 경합
          동시 10세션 INSERT: 초당 약 500건 (딕셔너리 경합이 병목)
CACHE 100: 100번에 1번만 딕셔너리 UPDATE → 경합 거의 없음
          동시 10세션 INSERT: 초당 약 5,000건 (10배 차이)

시퀀스 CACHE가 작으면 매번 SYS.SEQ$ 딕셔너리 테이블에 UPDATE가 발생하고,
이 과정에서 Row Cache Lock(딕셔너리 락) 경합이 심해진다.
동시 INSERT가 많은 테이블이라면 CACHE 크기를 충분히(100~1000) 늘려야 한다.
```

```
방식 2: MAX + 1

SELECT MAX(id) + 1 INTO v_id FROM orders;   -- ← 여기가 문제
INSERT INTO orders (id, ...) VALUES (v_id, ...);

내부 동작:
  1. 전체 테이블/인덱스를 스캔하여 MAX 값 조회
  2. 동시 트랜잭션이 같은 MAX를 읽으면 → PK 중복 오류!
  3. 이를 막으려면 테이블 락 필요 → 동시성 붕괴

성능: ★☆☆☆☆
이유: MAX 조회가 O(n)이고, 동시성도 죽음.
```

```
방식 3: 채번 테이블

-- 별도 테이블에 현재 번호를 관리
UPDATE seq_table SET curr_val = curr_val + 1 WHERE seq_name = 'ORDER';
SELECT curr_val INTO v_id FROM seq_table WHERE seq_name = 'ORDER';
INSERT INTO orders (id, ...) VALUES (v_id, ...);

내부 동작:
  1. 채번 테이블의 해당 행에 로우 락 설정
  2. 번호 증가 후 커밋
  3. 다음 트랜잭션이 대기 후 진행

성능: ★★★☆☆
이유: 시퀀스보다 느리지만 MAX+1보다 안전.
      채번 행에 대한 직렬화가 발생하여 병목이 될 수 있음.
```

**INSERT 시 인덱스의 영향:**

테이블에 인덱스가 많을수록 INSERT가 느려진다. 행 하나를 넣을 때마다 모든 인덱스에도 엔트리를 추가해야 하기 때문이다.

```
인덱스 0개:  INSERT 1건 → 테이블에 1번 쓰기
인덱스 3개:  INSERT 1건 → 테이블 1번 + 인덱스 3번 = 4번 쓰기
인덱스 10개: INSERT 1건 → 테이블 1번 + 인덱스 10번 = 11번 쓰기

대량 INSERT 100만 건 기준:
  인덱스 0개: 약 10초
  인덱스 5개: 약 60초  ← 6배 느림
```

따라서 대량 INSERT 전에 인덱스를 비활성화(DROP 또는 UNUSABLE)하고, INSERT 후 재생성하는 것이 빠르다.

#### UPDATE/DELETE 튜닝

UPDATE와 DELETE는 **대상 행을 찾는 과정**이 성능의 핵심이다. 즉, WHERE 절의 조건이 인덱스를 잘 타는지가 관건이다.

```sql
-- 느린 UPDATE: 인덱스 없는 컬럼으로 검색
UPDATE orders SET status = 'CANCELLED'
WHERE customer_name = '홍길동';
-- 테이블 풀 스캔 → 100만 건 순회 → 해당 행 찾기 → 수정
-- 수정 대상은 3건인데 100만 건을 읽음

-- 빠른 UPDATE: 인덱스 있는 컬럼으로 검색
UPDATE orders SET status = 'CANCELLED'
WHERE customer_id = 12345;
-- 인덱스 범위 스캔 → 3건 직접 접근 → 수정
-- 3건만 읽고 3건 수정
```

**핵심:** DML의 성능은 결국 **인덱스 설계**로 결정된다. DML 자체를 튜닝하는 것이 아니라, DML이 실행하는 내부 SELECT(조건 검색)를 튜닝하는 것이다.

#### 대량 UPDATE 튜닝: 수정 가능 조인 뷰와 MERGE 문

대량 데이터를 다른 테이블의 값을 참조하여 UPDATE할 때, 서브쿼리 방식은 매우 느리다.

```sql
-- 느린 방법: 서브쿼리로 한 건씩 수정
UPDATE orders o
SET o.status = (SELECT s.new_status FROM status_mapping s WHERE s.old_status = o.status)
WHERE EXISTS (SELECT 1 FROM status_mapping s WHERE s.old_status = o.status);
-- 100만 건이면 서브쿼리가 100만 번 실행
```

이 경우 orders 테이블의 100만 건 각각에 대해 status_mapping 테이블을 조회하므로, 사실상 NL(Nested Loop) 조인과 같은 비효율이 발생한다.

**수정 가능 조인 뷰(Updatable Join View)**를 사용하면 조인 한 번으로 처리할 수 있다:

```sql
-- Oracle: 수정 가능 조인 뷰
UPDATE (
  SELECT o.status AS old_status, s.new_status
  FROM orders o, status_mapping s
  WHERE o.status = s.old_status
)
SET old_status = new_status;
-- 단, status_mapping의 old_status에 UNIQUE 인덱스가 있어야 함 (키 보존 테이블 조건)
```

**MERGE 문**은 더 범용적이다:

```sql
-- 빠른 방법: MERGE 문
MERGE INTO orders o
USING status_mapping s ON (o.status = s.old_status)
WHEN MATCHED THEN UPDATE SET o.status = s.new_status;
-- 조인 한 번으로 처리, 키 보존 테이블 제약도 없음
```

MERGE 문은 WHEN MATCHED / WHEN NOT MATCHED를 조합하여 UPDATE와 INSERT를 동시에 처리할 수도 있어, ETL이나 배치 작업에서 특히 유용하다.

### 2. Direct Path I/O

#### 일반 INSERT vs Direct Path Insert

일반 INSERT는 데이터를 버퍼 캐시를 거쳐 디스크에 쓴다:

```
일반 INSERT:
  데이터 → [버퍼 캐시] → [디스크]
            ↑ 여기서 Freelist 탐색 (빈 블록 찾기)
            ↑ Undo 생성
            ↑ Redo 생성

Direct Path Insert:
  데이터 → [디스크에 직접 쓰기]
            ↑ HWM(High Water Mark) 뒤에 순차적으로 기록
            ↑ Freelist 탐색 불필요
            ↑ Undo 불필요
            ↑ Redo 최소화 가능 (NOLOGGING)
```

Direct Path Insert는 버퍼 캐시를 거치지 않고 디스크에 직접 쓴다. 빈 블록을 찾지 않고 테이블 끝(HWM 뒤)에 순차적으로 붙이므로 훨씬 빠르다.

**HWM(High Water Mark)이란?**

테이블에 데이터가 들어간 적 있는 영역의 최고 지점을 의미한다.

```
테이블 생성 직후:
  [빈블록][빈블록][빈블록]...
  ↑ HWM

100만 건 INSERT 후:
  [데이터][데이터][데이터]...[데이터][빈블록][빈블록]
                                      ↑ HWM

50만 건 DELETE 후:
  [데이터][빈블록][데이터]...[빈블록][빈블록][빈블록]
                                      ↑ HWM는 내려가지 않음!

풀 스캔 시 HWM까지 읽으므로, DELETE를 해도 풀 스캔 성능은 개선되지 않는다.
TRUNCATE나 ALTER TABLE MOVE로 HWM를 리셋해야 한다.
```

Direct Path Insert는 이 HWM 뒤쪽 영역에 순차적으로 기록하므로, Freelist에서 빈 블록을 탐색하는 오버헤드가 없다.

#### 사용 방법

```sql
-- Oracle: APPEND 힌트
INSERT /*+ APPEND */ INTO target_table
SELECT * FROM source_table;

-- NOLOGGING과 함께 쓰면 더 빠름
ALTER TABLE target_table NOLOGGING;
INSERT /*+ APPEND */ INTO target_table
SELECT * FROM source_table;
ALTER TABLE target_table LOGGING;
```

#### NOLOGGING의 위험성

NOLOGGING 옵션을 사용하면 Redo 로그 생성을 최소화하여 속도가 크게 향상된다. 그러나 **Data Guard(Standby DB) 환경에서는 심각한 문제**를 일으킬 수 있다.

```
Primary DB                               Standby DB
──────────                               ──────────
NOLOGGING + Direct Path Insert           Redo 로그를 받아서 복제
  → Redo 로그를 생성하지 않음             → 전달받을 Redo가 없음
  → Primary에는 데이터 있음              → Standby에는 데이터 없음!

장애 발생으로 Standby로 전환하면?
  → 해당 데이터가 없는 상태로 서비스!
  → 해당 블록 접근 시 ORA-01578: ORACLE data block corrupted 에러 발생
```

따라서 Data Guard 환경에서는:
- NOLOGGING 후 반드시 `ALTER DATABASE FORCE LOGGING;` 상태를 확인하거나
- NOLOGGING INSERT 후 `RMAN BACKUP`으로 Standby에 데이터를 전파해야 한다
- 가장 안전한 방법은 DB 레벨에서 FORCE LOGGING을 켜두는 것이다

#### 주의사항

```
Direct Path Insert의 제약:

1. Exclusive 테이블 락이 걸린다
   → 다른 트랜잭션이 해당 테이블에 DML 수행 불가
   → 서비스 중에는 사용하면 안 됨!

2. HWM 뒤에만 쓰므로 기존 빈 공간을 재활용하지 않는다
   → 테이블 크기가 계속 커질 수 있음

3. 커밋 전에는 해당 테이블을 읽을 수도 없다
   → 자기 자신의 트랜잭션에서도!
```

**적합한 상황:** 야간 배치, 데이터 마이그레이션, DW 적재 등 단독으로 실행하는 대량 INSERT 작업.

#### 병렬 DML

```sql
-- 병렬 INSERT
ALTER SESSION ENABLE PARALLEL DML;
INSERT /*+ PARALLEL(target_table, 4) */ INTO target_table
SELECT /*+ PARALLEL(source_table, 4) */ * FROM source_table;
```

4개의 프로세스가 동시에 INSERT를 수행한다. 단, 이것도 Exclusive 테이블 락이 걸리므로 서비스 중에는 사용 불가.

### 3. 파티션을 활용한 DML 튜닝

#### 파티셔닝이란

테이블이나 인덱스를 **파티션 키** 값에 따라 별도의 세그먼트(물리적 저장 단위)로 분할하는 것이다.

```
파티셔닝 전:
  orders 테이블 (1억 건) → 하나의 거대한 세그먼트

파티셔닝 후 (월별 Range 파티션):
  orders_202601 (800만 건) → 세그먼트 1
  orders_202602 (900만 건) → 세그먼트 2
  orders_202603 (850만 건) → 세그먼트 3
  ...
```

WHERE 절에 파티션 키가 포함되면 해당 파티션만 읽는다(**파티션 Pruning**).

```sql
-- 3월 주문만 조회 → orders_202603 세그먼트만 스캔
SELECT * FROM orders WHERE order_date BETWEEN '2026-03-01' AND '2026-03-31';
-- 1억 건이 아니라 850만 건만 읽음!
```

#### 파티션 종류

```sql
-- 1. Range 파티션: 날짜, 숫자 범위로 분할 (가장 많이 사용)
CREATE TABLE orders (
    order_id NUMBER,
    order_date DATE,
    ...
) PARTITION BY RANGE (order_date) (
    PARTITION p_2026_01 VALUES LESS THAN (DATE '2026-02-01'),
    PARTITION p_2026_02 VALUES LESS THAN (DATE '2026-03-01'),
    PARTITION p_2026_03 VALUES LESS THAN (DATE '2026-04-01')
);

-- 2. Hash 파티션: 해시 함수로 균등 분배
CREATE TABLE customers (
    customer_id NUMBER,
    ...
) PARTITION BY HASH (customer_id) PARTITIONS 8;
-- 8개 파티션에 데이터가 고르게 분산

-- 3. List 파티션: 명시적 값 목록으로 분할
CREATE TABLE sales (
    region VARCHAR2(20),
    ...
) PARTITION BY LIST (region) (
    PARTITION p_seoul VALUES ('서울', '경기'),
    PARTITION p_busan VALUES ('부산', '울산', '경남'),
    PARTITION p_other VALUES (DEFAULT)
);
```

#### 인덱스 파티셔닝

```
로컬 파티션 인덱스:
  테이블 파티션과 1:1 대응.
  테이블 파티션 추가/삭제 시 인덱스 파티션도 자동으로 관리.

  orders_202601 ↔ idx_orders_202601
  orders_202602 ↔ idx_orders_202602
  orders_202603 ↔ idx_orders_202603

글로벌 파티션 인덱스:
  테이블 파티션과 독립적.
  테이블 파티션을 DROP/TRUNCATE하면 글로벌 인덱스가 UNUSABLE 상태!
  → 인덱스 재생성 필요 → 서비스 중단 위험
```

**실무 원칙:** 가능하면 로컬 파티션 인덱스를 사용한다. 글로벌 인덱스는 파티션 관리 작업 시 장애 원인이 된다.

#### 파티션 Exchange를 활용한 대량 DML

대량 데이터를 수정해야 할 때, 직접 UPDATE/DELETE하면 오래 걸리고 락도 오래 잡힌다. 파티션 Exchange는 이를 우회하는 기법이다.

```sql
-- 시나리오: 2026년 1월 데이터 800만 건의 가격을 10% 인상

-- 방법 1: 직접 UPDATE (느림)
UPDATE orders SET price = price * 1.1
WHERE order_date >= '2026-01-01' AND order_date < '2026-02-01';
-- 800만 건 UPDATE → 800만 개 Undo/Redo → 수 시간 소요

-- 방법 2: 파티션 Exchange (빠름)
-- 1단계: 임시 테이블에 수정된 데이터 생성
CREATE TABLE orders_temp AS
SELECT order_id, order_date, price * 1.1 AS price, ...
FROM orders
WHERE order_date >= '2026-01-01' AND order_date < '2026-02-01';

-- 2단계: 임시 테이블에 인덱스 생성
CREATE INDEX ... ON orders_temp (...);

-- 3단계: 파티션과 임시 테이블을 교환 (순간적)
ALTER TABLE orders
EXCHANGE PARTITION p_2026_01 WITH TABLE orders_temp;
-- 메타데이터만 교환하므로 데이터 이동 없음 → 0.1초!

-- 4단계: 임시 테이블 삭제
DROP TABLE orders_temp;
```

핵심은 **데이터를 물리적으로 이동시키지 않는다**는 것이다. 세그먼트의 소유권(메타데이터)만 바꾸므로 데이터 크기와 관계없이 순간적으로 완료된다.

---

# Part 2: Lock과 동시성

### 4. DML 로우 락 (Row Lock)

#### 로우 락이란?

로우 락은 **특정 행(row)에 대한 배타적 접근 권한**을 의미한다. 쉽게 말해, 한 사용자가 특정 행을 수정하고 있을 때 다른 사용자는 그 행을 수정할 수 없도록 막는 것이다.

**왜 필요한가?**

예를 들어, 은행 계좌에서 잔액이 1000원인 상태에서:
- 사용자 A가 500원을 출금하려고 함 (잔액: 1000 → 500)
- 동시에 사용자 B도 500원을 출금하려고 함 (잔액: 1000 → 500)

만약 락이 없다면 두 트랜잭션이 모두 "잔액 1000원"을 읽고 각각 500원을 빼서 저장할 수 있다. 결과적으로 잔액은 500원이 되지만, 실제로는 1000원이 출금되었으므로 데이터 불일치가 발생한다.

로우 락을 사용하면:
1. 사용자 A가 먼저 해당 행에 락을 획득하고 500원 출금 진행
2. 사용자 B는 A가 커밋할 때까지 대기
3. A가 커밋 후 잔액이 500원으로 업데이트됨
4. B가 락을 획득하고 500원을 읽어서... 잔액 부족으로 출금 실패

이렇게 데이터 일관성이 보장된다.

#### 로우 락의 특징

- **배타적 모드(Exclusive Mode)**: 한 번에 하나의 트랜잭션만 특정 행을 수정할 수 있음
- **UPDATE, DELETE가 진행 중인 로우를 다른 트랜잭션이 UPDATE, DELETE 할 수 없음**
- 락은 자동으로 설정되며, 개발자가 명시적으로 설정할 필요는 대부분 없음

#### INSERT와 유니크 인덱스

**일반적인 INSERT는 왜 락이 필요 없을까?**

INSERT는 **새로운 행을 추가**하는 작업이다. 테이블에 빈 공간이 있다면 각 트랜잭션은 서로 다른 물리적 위치에 데이터를 삽입하므로 충돌할 이유가 없다.

예시:
```
테이블 상태: [row1] [row2] [빈공간] [빈공간] [빈공간]

트랜잭션 A: INSERT INTO users VALUES ('Alice') → [빈공간3]에 삽입
트랜잭션 B: INSERT INTO users VALUES ('Bob')   → [빈공간4]에 삽입

서로 다른 공간에 데이터를 쓰므로 충돌 없음!
```

**유니크 인덱스가 있는 경우는 다르다**

유니크 인덱스는 **특정 컬럼의 값이 중복되지 않도록 보장**하는 제약조건이다. 이메일 주소, 주민등록번호처럼 고유해야 하는 값에 사용한다.

문제 상황:
```sql
-- 유니크 인덱스 생성
CREATE UNIQUE INDEX uk_email ON users(email);

-- 두 트랜잭션이 동시에 실행된다면?
트랜잭션 A: INSERT INTO users (email) VALUES ('test@example.com');
트랜잭션 B: INSERT INTO users (email) VALUES ('test@example.com');
```

만약 락이 없다면:
1. 두 트랜잭션이 동시에 "test@example.com이 존재하는지" 확인 → 둘 다 "없음" 확인
2. 두 트랜잭션이 모두 데이터 삽입 시도
3. 유니크 제약조건 위반!

따라서 유니크 인덱스가 있는 경우:
1. 먼저 도착한 트랜잭션 A가 인덱스 키 "test@example.com"에 락 설정
2. 트랜잭션 B는 A가 커밋하거나 롤백할 때까지 대기
3. A가 커밋되면 B는 중복 키 오류 발생
4. A가 롤백되면 B가 정상적으로 삽입 가능

**정리:**
- 일반 컬럼: 물리적으로 다른 위치에 데이터 삽입 → 락 불필요
- 유니크 인덱스: 논리적 중복 검사 필요 → 락 필요

### 5. 테이블 락 (Table Lock)

로우 락만 있으면 안 될까? 왜 테이블 락이 필요한가?

**테이블 락의 역할**

데이터베이스는 **테이블 구조의 일관성**을 보장해야 한다. 예를 들어:

```sql
-- 트랜잭션 A: 데이터 수정 중
UPDATE orders SET total = 100 WHERE id = 1;

-- 트랜잭션 B: 테이블 구조 변경 시도
ALTER TABLE orders DROP COLUMN total;  -- 이러면 안 됨!
```

만약 A가 수정 중인 컬럼을 B가 삭제한다면? 데이터베이스가 깨진다.

**테이블 락의 종류 (Oracle 기준)**

1. **Row Share (RS)**: SELECT ... FOR UPDATE
   - 다른 트랜잭션의 읽기 허용
   - 다른 트랜잭션의 DML 허용
   - 테이블 구조 변경은 차단

2. **Row Exclusive (RX)**: INSERT, UPDATE, DELETE
   - 다른 트랜잭션의 읽기 허용
   - 다른 트랜잭션의 DML 허용
   - 테이블 락(전체 락)은 차단

3. **Share (S)**: LOCK TABLE ... IN SHARE MODE
   - 다른 트랜잭션의 읽기 허용
   - DML 차단

4. **Exclusive (X)**: LOCK TABLE ... IN EXCLUSIVE MODE
   - 모든 작업 차단 (테이블 유지보수 시)

5. **Share Row Exclusive (SRX)**: LOCK TABLE ... IN SHARE ROW EXCLUSIVE MODE
   - 다른 트랜잭션의 읽기 허용
   - 다른 트랜잭션의 DML과 테이블 락 모두 차단

**테이블 락 호환성 매트릭스:**

```
              RS    RX    S    SRX    X
  RS           O     O    O     O    X
  RX           O     O    X     X    X
  S            O     X    O     X    X
  SRX          O     X    X     X    X
  X            X     X    X     X    X

O: 호환 (동시 수행 가능)
X: 비호환 (대기 필요)
```

**구체적 시나리오:**

```
세션 A: INSERT INTO orders VALUES (...);  → RX 테이블 락 획득
세션 B: INSERT INTO orders VALUES (...);  → RX 테이블 락 요청
       RX vs RX = 호환(O) → 동시 수행 가능!

세션 C: ALTER TABLE orders ADD COLUMN ...;  → X 테이블 락 요청
       X vs RX = 비호환(X) → A, B 모두 끝날 때까지 대기!
```

이 매트릭스가 실무에서 중요한 이유는, DML끼리는 테이블 락 수준에서 호환되므로 동시 수행이 가능하지만, DDL(ALTER TABLE 등)은 Exclusive 테이블 락을 요구하기 때문에 진행 중인 모든 DML이 끝날 때까지 기다려야 한다는 것이다. 운영 중 DDL을 실행하면 서비스 중단으로 이어질 수 있는 이유가 바로 여기에 있다.

**실무에서의 의미**

대부분의 경우 개발자가 테이블 락을 신경 쓸 필요는 없다. DML 실행 시 데이터베이스가 자동으로 적절한 테이블 락을 설정한다.

```sql
UPDATE orders SET status = 'SHIPPED' WHERE id = 100;
-- 자동으로:
-- 1. orders 테이블에 RX(Row Exclusive) 락 설정
-- 2. id=100인 행에 로우 락 설정
-- 3. 작업 수행
-- 4. COMMIT 시 모든 락 해제
```

**주의할 점:**
```sql
-- DDL(Data Definition Language)은 테이블 전체를 락!
ALTER TABLE orders ADD COLUMN new_col VARCHAR(50);
-- 실행 중에는 아무도 orders 테이블에 접근 불가
-- 대용량 테이블에서는 몇 분~몇 시간 소요 가능!
-- 운영 중 실행 시 서비스 장애 발생 가능
```

### 6. MVCC와 읽기 일관성

많은 현대 데이터베이스(Oracle, PostgreSQL, MySQL InnoDB 등)는 **MVCC(Multi-Version Concurrency Control)** 모델을 사용한다.

#### SELECT는 왜 다른 트랜잭션을 방해하지 않을까?

**전통적인 방식 (락 기반)**

```
상황: 사용자 A가 주문 정보를 수정 중, 사용자 B가 같은 주문 정보를 조회

트랜잭션 A: UPDATE orders SET status = 'SHIPPED' WHERE id = 100;
트랜잭션 B: SELECT * FROM orders WHERE id = 100;

락 기반 DB의 동작:
- A가 UPDATE 중이므로 해당 행에 배타 락 설정
- B의 SELECT는 A가 커밋할 때까지 대기 (읽기도 막힘!)
```

이 방식의 문제점:
- 단순히 데이터를 읽기만 하려는데도 기다려야 함
- 읽기 작업이 많은 시스템에서 성능 저하

**MVCC 방식**

MVCC는 **각 트랜잭션마다 데이터의 스냅샷을 제공**한다. 마치 사진을 찍어두는 것처럼, 쿼리 시작 시점의 데이터 상태를 별도로 보관한다.

```
시간 순서로 보는 MVCC 동작:

1. 초기 상태: orders 테이블에 id=100인 행의 status는 'PENDING'

2. 트랜잭션 A 시작
   UPDATE orders SET status = 'SHIPPED' WHERE id = 100;
   → 새 버전 생성: status = 'SHIPPED' (아직 커밋 전)
   → 기존 버전: status = 'PENDING' (Undo 블록에 보관)

3. 트랜잭션 B 시작 (A가 커밋하기 전)
   SELECT * FROM orders WHERE id = 100;
   → A가 수정한 'SHIPPED'를 보는 게 아니라
   → Undo 블록에 있는 'PENDING'을 읽음 (쿼리 시작 시점의 데이터)
   → 대기 없이 즉시 읽기 가능!

4. 트랜잭션 A 커밋
   → 'SHIPPED' 버전이 공식 버전이 됨

5. 새로운 트랜잭션 C 시작
   SELECT * FROM orders WHERE id = 100;
   → 이제 'SHIPPED'를 읽음 (최신 커밋된 데이터)
```

**Undo 블록이란?**
- 변경 전 데이터를 임시로 저장하는 공간
- 롤백이 필요할 때 원래 상태로 되돌리는 데 사용
- MVCC에서는 이전 버전의 데이터를 읽는 데도 사용

**MVCC의 장점:**
- SELECT는 다른 트랜잭션의 수정 작업을 기다리지 않음
- UPDATE/DELETE도 SELECT 때문에 기다리지 않음
- **읽기와 쓰기가 서로를 방해하지 않아 동시성이 크게 향상됨**

#### MVCC를 사용하지 않는 데이터베이스

일부 데이터베이스(오래된 버전의 SQL Server 등)는 MVCC 대신 **공유락(Shared Lock)**을 사용한다.

```
락 호환성 표:

         | 공유락(S) | 배타락(X)
---------|----------|----------
공유락(S) |    O     |    X
배타락(X) |    X     |    X

O: 호환 (동시 가능)
X: 비호환 (대기 필요)
```

**동작 방식:**

```sql
-- 트랜잭션 A (읽기)
SELECT * FROM orders WHERE id = 100;  -- 공유락 획득

-- 트랜잭션 B (읽기)
SELECT * FROM orders WHERE id = 100;  -- 공유락 획득 가능 (호환됨)

-- 트랜잭션 C (쓰기)
UPDATE orders SET status = 'SHIPPED' WHERE id = 100;
-- 배타락 필요하지만 공유락과 호환 안됨 → A, B가 끝날 때까지 대기!
```

**문제점:**
- 여러 사용자가 같은 데이터를 읽고 있으면 수정 작업이 계속 대기
- 읽기 작업이 많은 시스템에서 심각한 성능 저하

**왜 아직도 사용할까?**
- 구현이 상대적으로 간단함
- 일부 워크로드에서는 MVCC보다 효율적일 수 있음
- 최신 SQL Server는 MVCC 옵션 제공 (Read Committed Snapshot Isolation)

### 7. 블로킹과 데드락

#### 블로킹(Blocking)

Lock 경합이 발생해 특정 세션이 작업을 진행하지 못하고 멈춰 선 상태다. 이것 자체는 정상적인 동작이다. 한 트랜잭션이 끝나면 대기 중인 트랜잭션이 진행된다.

```
트랜잭션 A: UPDATE accounts SET balance = 500 WHERE id = 1;  -- 로우 락 획득
트랜잭션 B: UPDATE accounts SET balance = 300 WHERE id = 1;  -- 대기 (블로킹)
            ⏳ A가 커밋하면 진행됨
```

블로킹이 문제가 되는 것은 **A가 커밋/롤백을 안 할 때**다. 개발자가 트랜잭션을 열어놓고 커밋을 안 하거나, 애플리케이션이 커넥션을 반환하지 않으면 다른 모든 세션이 무한 대기에 빠진다.

#### 데드락(Deadlock)

두 세션이 서로가 잡고 있는 Lock을 기다리는 상태. **어느 쪽도 진행할 수 없다.**

```
시간 순서:

1. 트랜잭션 A: UPDATE accounts SET balance = 500 WHERE id = 1;
   → id=1에 로우 락 획득 ✓

2. 트랜잭션 B: UPDATE accounts SET balance = 300 WHERE id = 2;
   → id=2에 로우 락 획득 ✓

3. 트랜잭션 A: UPDATE accounts SET balance = 100 WHERE id = 2;
   → id=2는 B가 잡고 있음 → 대기 ⏳

4. 트랜잭션 B: UPDATE accounts SET balance = 200 WHERE id = 1;
   → id=1은 A가 잡고 있음 → 대기 ⏳

   A는 B를 기다리고, B는 A를 기다림 → 영원히 풀리지 않음!

        A ──(id=1 잡음)──→ id=2 기다림
        ↑                      ↓
        └──── id=1 기다림 ←── B ──(id=2 잡음)
```

#### DB의 데드락 해결

데이터베이스는 데드락을 자동으로 감지한다. 주기적으로(Oracle은 3초마다) 대기 그래프(wait-for graph)를 검사하여 순환이 발견되면, 한쪽 트랜잭션을 강제 롤백한다.

```
데드락 발생!
→ DB가 감지
→ 트랜잭션 B를 선택하여 강제 롤백 (ORA-00060: deadlock detected)
→ 트랜잭션 A는 정상 진행
→ B는 에러를 받고, 애플리케이션에서 재시도 결정
```

#### 인덱스로 인한 데드락 (실무에서 자주 발생)

데드락은 테이블의 서로 다른 행을 수정하는데도 발생할 수 있다. 인덱스의 리프 블록이 겹치는 경우다.

```
테이블: accounts (id PK, balance, name)
인덱스: idx_name (name)

세션 A: UPDATE accounts SET balance = 100, name = 'AAA' WHERE id = 1;
  → id=1 행에 로우 락 + idx_name 인덱스 엔트리 변경

세션 B: UPDATE accounts SET balance = 200, name = 'BBB' WHERE id = 2;
  → id=2 행에 로우 락 + idx_name 인덱스 엔트리 변경

만약 인덱스의 리프 블록이 같다면?
  A가 인덱스 블록 X에 락 → B가 인덱스 블록 Y에 락
  A가 블록 Y도 필요 → 대기
  B가 블록 X도 필요 → 대기 → 데드락!
```

이것은 테이블 행은 서로 다른데도 인덱스 때문에 데드락이 발생하는 케이스다. 특히 인덱스가 많은 테이블에서 동시 UPDATE가 빈번할 때 나타나며, 불필요한 인덱스를 제거하는 것이 근본적인 해결책이 될 수 있다. UPDATE 대상 컬럼에 인덱스가 걸려 있는지 확인하는 것이 데드락 분석의 첫 단계다.

#### 데드락 예방 전략

```sql
-- 1. 테이블 접근 순서를 통일
-- 나쁜 예: A는 accounts → orders, B는 orders → accounts
-- 좋은 예: 모두 accounts → orders 순서로 접근

-- 2. 트랜잭션을 짧게 유지
-- 락을 잡고 있는 시간이 짧으면 데드락 확률이 낮아진다

-- 3. 한 번에 필요한 모든 락을 획득
SELECT * FROM accounts WHERE id IN (1, 2) FOR UPDATE ORDER BY id;
-- id 순서로 정렬하여 락을 획득하면 순환이 발생하지 않음
```

### 8. 락 성능 최적화

락은 데이터 일관성을 보장하지만, 동시에 **성능 저하의 주범**이기도 하다.

**왜 성능이 저하되는가?**

```
시나리오: 인기 있는 상품의 재고 업데이트

트랜잭션 1: UPDATE products SET stock = stock - 1 WHERE id = 999;
             -- 복잡한 비즈니스 로직 처리 (10초 소요)
             COMMIT;

트랜잭션 2~100: 같은 상품 구매 시도
                → 모두 락 대기!
                → 트랜잭션 1이 끝날 때까지 10초간 대기

결과: 99명의 사용자가 10초씩 기다림 → 사용자 이탈, 서비스 품질 저하
```

**성능 최적화 전략:**

1. **필요 이상으로 락을 유지하지 않기**

나쁜 예:
```sql
BEGIN TRANSACTION;
  -- 1. 데이터 조회 및 락 획득
  SELECT * FROM orders WHERE id = 100 FOR UPDATE;

  -- 2. 외부 API 호출 (5초 소요) - 락을 잡은 채로!
  -- call_external_payment_api()

  -- 3. 복잡한 계산 (3초 소요) - 여전히 락 유지!
  -- calculate_shipping_cost()

  -- 4. 업데이트
  UPDATE orders SET status = 'PAID' WHERE id = 100;
COMMIT;
-- 총 8초 이상 락 유지!
```

좋은 예:
```sql
-- 1. 외부 API 호출 (락 없이)
-- payment_result = call_external_payment_api()

-- 2. 복잡한 계산 (락 없이)
-- shipping_cost = calculate_shipping_cost()

-- 3. 최소한의 시간만 락 유지
BEGIN TRANSACTION;
  UPDATE orders SET status = 'PAID' WHERE id = 100;
COMMIT;
-- 0.1초만 락 유지!
```

2. **SQL 튜닝으로 락 지속 시간 최소화**

느린 쿼리:
```sql
-- 인덱스 없는 컬럼으로 검색 → 전체 테이블 스캔
UPDATE users SET last_login = NOW()
WHERE email = 'user@example.com';  -- 1초 소요
-- 1초 동안 해당 행에 락!
```

빠른 쿼리:
```sql
-- 인덱스 생성
CREATE INDEX idx_email ON users(email);

-- 같은 쿼리가 이제는
UPDATE users SET last_login = NOW()
WHERE email = 'user@example.com';  -- 0.01초 소요
-- 0.01초만 락 유지!
```

**핵심 원칙:**
- 쿼리가 빠르면 락도 빨리 해제됨
- 쿼리가 느리면 락도 오래 유지됨
- 따라서 SQL 튜닝은 락 경합 감소에도 직접적인 영향

---

# Part 3: 트랜잭션 관리

### 9. 트랜잭션 기본과 주의사항

#### 트랜잭션이란?

트랜잭션은 **논리적으로 하나의 작업 단위**를 의미한다. "모두 성공" 또는 "모두 실패"만 존재하며, 중간 상태는 없다.

예: 계좌 이체
```sql
BEGIN TRANSACTION;
  -- 1. A 계좌에서 출금
  UPDATE accounts SET balance = balance - 100 WHERE id = 'A';

  -- 2. B 계좌에 입금
  UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
COMMIT;

만약 1번만 성공하고 2번 실패한다면? → 돈이 사라짐!
트랜잭션은 이런 일이 없도록 보장한다.
```

#### 트랜잭션 길이의 문제

**너무 긴 트랜잭션의 문제점:**

```sql
-- 안 좋은 예: 30분짜리 트랜잭션
BEGIN TRANSACTION;
  -- 100만 건의 데이터 수정
  FOR i IN 1..1000000 LOOP
    UPDATE products SET price = price * 1.1 WHERE id = i;
  END LOOP;

  -- 문제 발생! 에러가 났다면?
  ROLLBACK;  -- 30분 동안 한 작업을 다시 되돌림 → 또 30분 소요!
```

**문제점:**

1. **롤백 시간**: 작업 시간만큼 롤백에도 시간 소요
2. **Undo Segment 고갈**:
   - Undo Segment는 이전 버전 데이터를 저장하는 공간
   - 너무 많은 변경이 일어나면 공간 부족 발생
   - "ORA-30036: unable to extend segment" 에러
3. **락 경합**: 30분 동안 관련 데이터에 아무도 접근 못함

**해결책:**

```sql
-- 좋은 예: 작은 단위로 분할
FOR i IN 1..1000000 LOOP
  UPDATE products SET price = price * 1.1 WHERE id = i;

  -- 1000건마다 커밋
  IF MOD(i, 1000) = 0 THEN
    COMMIT;
  END IF;
END LOOP;
COMMIT;

-- 장점:
-- - 중간에 실패해도 이미 커밋된 부분은 유지
-- - 롤백 시간 최소화
-- - Undo Segment 공간 절약
-- - 락 경합 감소 (1000건씩만 락)
```

**그러나 주의!**

위 방법은 **원자성을 희생**한다. 중간에 실패하면 일부만 업데이트된 상태로 남는다. 따라서:
- 계좌 이체처럼 원자성이 중요한 작업: 절대 분할하면 안 됨
- 가격 일괄 업데이트처럼 독립적인 작업: 분할 가능

#### Snapshot Too Old (ORA-01555)

트랜잭션을 길게 유지하면 Undo Segment 고갈 외에도 **Snapshot too old** 에러가 발생할 수 있다. 이는 장시간 실행되는 SELECT 쿼리가 이전 버전 데이터를 읽으려 할 때, 해당 Undo 데이터가 이미 다른 트랜잭션에 의해 덮어쓰여진 경우 발생한다.

```
문제 상황:
  1. 트랜잭션 A가 100만 건을 SELECT 시작 (오래 걸림)
  2. 이 사이 다른 트랜잭션들이 같은 데이터를 UPDATE + COMMIT
  3. Undo 영역이 가득 차서 A가 읽어야 할 이전 버전 데이터를 덮어씀
  4. A가 해당 블록을 읽으려 할 때 → ORA-01555: snapshot too old

해결:
  - Undo tablespace 크기 증가
  - UNDO_RETENTION 파라미터 조정 (기본 900초 → 더 크게)
  - 장시간 쿼리를 피하거나 분할 처리
  - Undo tablespace에 AUTOEXTEND 설정
```

MVCC는 Undo 데이터를 기반으로 일관된 읽기를 제공하므로, Undo가 충분히 유지되지 않으면 읽기 일관성이 깨질 수 있다. 이것이 "트랜잭션을 짧게 유지하라"는 원칙의 또 다른 이유다.

### 10. 커밋과 LGWR (Log Writer)

커밋은 단순히 "저장"이 아니다. 데이터베이스 내부에서는 복잡한 과정이 진행된다.

#### 데이터베이스의 메모리 구조

```
[서버 프로세스] ─→ [메모리] ─→ [디스크]
                    ├ 데이터 버퍼 캐시
                    └ 로그 버퍼 ─→ LGWR ─→ Redo 로그 파일
```

**왜 이렇게 복잡한가?**

디스크는 메모리보다 수천 배 느리다:
- 메모리 읽기: 나노초 (0.000000001초)
- SSD 읽기: 마이크로초 (0.000001초)
- HDD 읽기: 밀리초 (0.001초)

따라서 모든 작업을 메모리에서 하고, 필요할 때만 디스크에 쓴다.

#### LGWR의 역할

LGWR(Log Writer)는 데이터베이스의 **백그라운드 프로세스**다. 사용자가 직접 실행하는 것이 아니라, 데이터베이스가 자동으로 실행한다.

**역할:**
- 메모리의 로그 버퍼에 있는 변경 이력(Redo 로그)을 디스크의 Redo 로그 파일에 기록

**Redo 로그란?**
- 모든 변경 사항의 이력을 기록
- "orders 테이블의 id=100 행의 status 컬럼을 'PENDING'에서 'SHIPPED'로 변경" 같은 내용
- 데이터베이스 장애 시 복구에 사용

**LGWR가 작동하는 시점:**

1. **사용자가 COMMIT 실행**
2. 로그 버퍼의 1/3이 차거나 1MB 이상 쌓임
3. 3초마다 주기적으로
4. DBWR(데이터를 디스크에 쓰는 프로세스)가 작동하기 전

#### 동기 커밋 (Synchronous Commit) - 기본 방식

**동작 과정을 시간 순으로 보면:**

```
시간 →

1. 사용자가 COMMIT 입력
   |
2. 서버 프로세스가 LGWR에게 신호: "로그 버퍼를 지금 디스크에 써!"
   |
3. 서버 프로세스는 LGWR의 작업이 끝날 때까지 대기 ⏳
   |
4. LGWR가 로그 버퍼를 Redo 로그 파일에 기록 (디스크 I/O)
   | (이 과정이 5ms~10ms 정도 소요)
   |
5. LGWR가 서버 프로세스에게 신호: "완료!"
   |
6. 서버 프로세스가 사용자에게 응답: "커밋 성공!"
```

**왜 기다려야 하는가?**

만약 디스크에 쓰기 전에 "커밋 성공"이라고 응답한다면:
```
1. 사용자: UPDATE ... COMMIT; → "성공!" 응답 받음
2. 하지만 아직 로그는 메모리에만 존재
3. 갑자기 정전!
4. 메모리 내용은 모두 사라짐
5. 재시작 후 → 사용자는 "커밋 성공"을 받았는데 데이터가 없음!
```

따라서 데이터 안정성을 위해 디스크에 쓰기가 완료될 때까지 기다린다.

**성능 영향:**

```sql
-- 10,000번 커밋하는 경우
FOR i IN 1..10000 LOOP
  INSERT INTO logs VALUES (i);
  COMMIT;  -- 매번 5ms 대기 = 총 50,000ms = 50초!
END LOOP;

-- 100번 커밋하는 경우
FOR i IN 1..10000 LOOP
  INSERT INTO logs VALUES (i);
  IF MOD(i, 100) = 0 THEN
    COMMIT;  -- 100번만 5ms 대기 = 총 500ms = 0.5초!
  END IF;
END LOOP;
```

**100배 차이!** 이것이 "너무 자주 커밋하면 느리다"는 말의 의미다.

#### 비동기 커밋 (Asynchronous Commit)

"데이터가 조금 유실되어도 괜찮으니, 빠른 게 중요해!"라는 경우에 사용한다.

**동작 과정:**

```
시간 →

1. 사용자가 COMMIT 입력
   |
2. 서버 프로세스가 LGWR에게 신호: "나중에 써"
   |
3. 서버 프로세스가 즉시 사용자에게 응답: "커밋 성공!"
   |
4. LGWR는 백그라운드에서 천천히 디스크에 기록
```

**사용 예시:**

```sql
-- Oracle
ALTER SESSION SET COMMIT_WRITE = 'BATCH,NOWAIT';

-- PostgreSQL
SET synchronous_commit = OFF;

-- 이후 모든 COMMIT은 비동기로 동작
INSERT INTO access_logs VALUES (...);
COMMIT;  -- 즉시 반환!
```

**동기 vs 비동기 비교:**

| 항목 | 동기 커밋 (Synchronous) | 비동기 커밋 (Asynchronous) |
|------|----------------------|-------------------------|
| **응답 시간** | 5~10ms (디스크 I/O 대기) | <1ms (즉시) |
| **데이터 안정성** | 매우 높음 (장애 시 복구 가능) | 낮음 (장애 시 유실 가능) |
| **처리량** | 낮음 (초당 수백~수천 TPS) | 높음 (초당 수만~수십만 TPS) |
| **사용 사례** | 금융, 주문, 인증 등 중요 데이터 | 로그, 통계, 임시 데이터 |
| **기본 설정** | O (대부분 DB의 기본값) | X (명시적으로 설정 필요) |

![PostgreSQL synchronous_commit 옵션과 WAL 흐름](/assets/img/posts/dml-tuning/sync_async_commit.png)
*동기 커밋은 WAL이 디스크에 flush될 때까지 대기하고, 비동기 커밋은 즉시 클라이언트에 응답한다 (출처: [Percona](https://www.percona.com/blog/postgresql-synchronous_commit-options-and-synchronous-standby-replication/))*

### 11. 동시성 제어와 격리수준

#### 비관적 동시성 제어 (Pessimistic Concurrency Control)

**개념:** "다른 사람이 수정할 거야. 미리 잠가놓자!"

```sql
-- 트랜잭션 A: 주문 조회 및 락 획득
SELECT * FROM orders WHERE order_id = 1001 FOR UPDATE;
-- 이 순간 다른 트랜잭션은 이 행을 수정할 수 없음

-- 비즈니스 로직 처리 ...

-- 수정
UPDATE orders SET status = 'SHIPPED' WHERE order_id = 1001;
COMMIT;  -- 락 해제
```

**NOWAIT 옵션:**

```sql
-- 즉시 락 획득 실패 시 에러 반환
SELECT * FROM orders WHERE order_id = 1001 FOR UPDATE NOWAIT;

-- Java 예시
try {
    Order order = orderRepository.findByIdForUpdate(1001);
    // 성공 → 처리 진행
} catch (LockTimeoutException e) {
    return "현재 이 주문을 다른 사용자가 처리하고 있습니다.";
}
```

**언제 사용?**
- 동시 수정이 빈번한 경우
- 데이터 충돌이 발생하면 안 되는 경우
- 예: 재고 관리, 좌석 예매

#### 낙관적 동시성 제어 (Optimistic Concurrency Control)

**개념:** "대부분은 충돌이 안 일어날 거야. 일단 진행하고, 마지막에 확인하자!"

**구현 방법 1: 버전 컬럼 사용**

```sql
-- 1단계: 데이터 조회 (락 없이)
SELECT order_id, status, version
FROM orders WHERE order_id = 1001;
-- 결과: order_id=1001, status='PENDING', version=5

-- 2단계: 비즈니스 로직 처리 (시간이 오래 걸릴 수 있음)

-- 3단계: 업데이트 시도
UPDATE orders
SET status = 'SHIPPED',
    version = version + 1  -- 버전 증가
WHERE order_id = 1001
  AND version = 5;  -- 읽었을 때의 버전과 같은지 확인!

-- 4단계: 결과 확인
IF SQL%ROWCOUNT = 0 THEN
    -- 업데이트된 행이 없음 = 다른 트랜잭션이 먼저 수정함!
    RAISE_APPLICATION_ERROR(-20001, '다른 사용자가 이미 수정했습니다.');
END IF;
```

**실패 시나리오:**

```
시간 →

트랜잭션 A:
  1. SELECT ... WHERE order_id=1001 → version=5 조회
  2. (비즈니스 로직 처리 중...)

트랜잭션 B:
  1. SELECT ... WHERE order_id=1001 → version=5 조회
  2. UPDATE ... WHERE version=5 → 성공! version=6으로 변경
  3. COMMIT

트랜잭션 A (계속):
  3. UPDATE ... WHERE version=5 → 실패! (현재 version=6)
  4. 에러 발생: "다른 사용자가 이미 수정했습니다"
```

**JPA에서의 낙관적 락:**

```java
@Entity
public class Order {
    @Id
    private Long orderId;

    private String status;

    @Version  // JPA가 자동으로 버전 관리
    private Long version;
}

// 사용
try {
    Order order = orderRepository.findById(1001L);
    order.setStatus("SHIPPED");
    orderRepository.save(order);  // version 자동 체크
} catch (OptimisticLockException e) {
    return "충돌 발생. 다시 시도해주세요.";
}
```

**비관적 vs 낙관적 선택 기준:**

- **동시 수정이 빈번한 경우** (재고, 좌석): **비관적** 방식을 권장한다. 충돌이 많아 재시도 비용이 크기 때문이다.
- **동시 수정이 드문 경우** (게시글): **낙관적** 방식을 권장한다. 락으로 인한 대기가 낭비이기 때문이다.
- **짧은 트랜잭션**: **비관적** 방식을 권장한다. 락 유지 시간이 짧아 영향이 적기 때문이다.
- **긴 트랜잭션**: **낙관적** 방식을 권장한다. 락을 오래 잡으면 다른 사용자가 대기해야 하기 때문이다.
- **실시간 시스템**: **낙관적** 방식을 권장한다. 대기 없이 빠른 응답이 가능하기 때문이다.

#### 트랜잭션 격리수준과 동시성 이상현상

여러 트랜잭션이 동시에 실행될 때 발생할 수 있는 세 가지 문제다.

**1. Dirty Read (오손 읽기)**

커밋되지 않은 데이터를 읽는 것. 상대방이 롤백하면 존재하지 않는 데이터를 읽은 셈이 된다.

```
트랜잭션 A                          트랜잭션 B
─────────                          ─────────
UPDATE accounts
SET balance = 500
WHERE id = 1;
(아직 커밋 안 함)
                                    SELECT balance FROM accounts
                                    WHERE id = 1;
                                    → 500 읽음 (커밋 안 된 값!)

ROLLBACK;
(balance는 원래 1000으로 복원)
                                    → B는 500이라고 믿고 있지만
                                      실제로는 1000. 데이터 불일치!
```

**2. Non-Repeatable Read (비반복 읽기)**

같은 쿼리를 두 번 실행했는데 사이에 다른 트랜잭션이 값을 수정하여 결과가 달라지는 현상.

```
트랜잭션 A                          트랜잭션 B
─────────                          ─────────
SELECT balance FROM accounts
WHERE id = 1;
→ 1000 읽음
                                    UPDATE accounts
                                    SET balance = 500
                                    WHERE id = 1;
                                    COMMIT;

SELECT balance FROM accounts
WHERE id = 1;
→ 500 읽음 ← 같은 쿼리인데 결과가 다름!
```

**3. Phantom Read (유령 읽기)**

같은 조건으로 조회했는데 사이에 다른 트랜잭션이 행을 삽입/삭제하여 행의 개수가 달라지는 현상.

```
트랜잭션 A                          트랜잭션 B
─────────                          ─────────
SELECT COUNT(*) FROM orders
WHERE status = 'PENDING';
→ 10건

                                    INSERT INTO orders (status)
                                    VALUES ('PENDING');
                                    COMMIT;

SELECT COUNT(*) FROM orders
WHERE status = 'PENDING';
→ 11건 ← 유령처럼 1건이 나타남!
```

#### 격리수준 (Isolation Level)

이 세 가지 이상현상을 어디까지 허용할 것인가를 정하는 것이 격리수준이다.

```
격리수준           Dirty Read    Non-Repeatable    Phantom Read
                                Read
───────────────────────────────────────────────────────────────
READ UNCOMMITTED    발생 O          발생 O            발생 O
READ COMMITTED      방지 ✓          발생 O            발생 O
REPEATABLE READ     방지 ✓          방지 ✓            발생 O
SERIALIZABLE        방지 ✓          방지 ✓            방지 ✓
                    ←── 동시성 높음          일관성 높음 ──→
```

#### Oracle vs MySQL 격리수준 차이

두 DBMS는 격리수준 지원 범위와 구현 방식이 다르다.

```
Oracle:
  - READ COMMITTED (기본)
  - SERIALIZABLE
  - READ UNCOMMITTED 미지원 (MVCC 기반이므로 Dirty Read 자체가 불가능)

MySQL InnoDB:
  - READ UNCOMMITTED
  - READ COMMITTED
  - REPEATABLE READ (기본) ← Locking Read에서 Gap Lock으로 Phantom 방지
  - SERIALIZABLE
```

Oracle은 Undo 기반 MVCC를 사용하므로, 항상 커밋된 데이터만 읽는다. READ UNCOMMITTED 격리수준을 별도로 제공하지 않는 것은 아키텍처상 Dirty Read가 원천적으로 불가능하기 때문이다.

MySQL InnoDB의 REPEATABLE READ는 두 가지 메커니즘을 함께 쓴다.

- 일반 SELECT(consistent nonlocking read)는 트랜잭션 시작 시점의 MVCC 스냅샷을 읽으므로 다른 트랜잭션의 변경이 보이지 않는다.
- Locking Read(`SELECT ... FOR UPDATE`/`LOCK IN SHARE MODE`)와 DML은 **Gap Lock / Next-key Lock**으로 범위 자체를 잠가 Phantom을 방지한다.

즉 Phantom 방지가 격리수준 자체에서 자동으로 보장되는 것이 아니라, 잠금 기반 읽기·쓰기 시에 Gap Lock이 같이 따라붙기 때문이다.

```
Gap Lock이란?

SELECT * FROM orders WHERE price BETWEEN 100 AND 200 FOR UPDATE;
  → 100~200 사이의 "간격"에도 락을 설정
  → 다른 트랜잭션이 이 범위에 INSERT하는 것을 방지
  → Phantom Read 방지

예시:
  현재 price 컬럼의 인덱스에 값이 [50, 100, 150, 200, 300]이 있다면
  Gap Lock은 (100, 150), (150, 200) 간격에 락을 건다.
  → 다른 트랜잭션이 price = 120인 행을 INSERT하려 하면 대기!
  → 이로써 REPEATABLE READ에서도 Phantom Read가 방지된다.
```

따라서 MySQL InnoDB에서는 잠금 기반 읽기·쓰기 패턴을 사용하면 REPEATABLE READ만으로도 대부분의 동시성 이상현상을 막을 수 있어 SERIALIZABLE까지 올릴 필요가 거의 없다. 다만 일반 SELECT만으로 Phantom을 막는다고 가정하면 안 된다 — 격리수준이 아니라 Locking Read가 핵심이다. 반면 Oracle에서 Phantom Read를 완전히 방지하려면 SERIALIZABLE 격리수준을 사용해야 한다.

#### 격리수준 선택 기준

```
일반 웹 서비스      → READ COMMITTED
  이유: 대부분의 조회는 최신 커밋 데이터를 보면 충분.
        동시성이 가장 좋음.

정산/리포트 생성    → REPEATABLE READ 또는 SERIALIZABLE
  이유: 리포트 생성 중 데이터가 바뀌면 숫자가 안 맞음.
        일관된 스냅샷이 필요.

계좌이체 등 금융    → SERIALIZABLE (해당 트랜잭션만)
  이유: 잔액 확인과 출금 사이에 다른 트랜잭션이
        끼어들면 안 됨.
```

---

### 12. 핵심 원칙

**1. 락과 성능은 반비례한다**
```
느린 쿼리 = 오래 지속되는 락 = 많은 대기 = 낮은 동시성
빠른 쿼리 = 빨리 해제되는 락 = 적은 대기 = 높은 동시성
```

**2. 커밋 타이밍의 균형**
```
너무 자주 커밋 → LGWR 대기 시간 누적 → 느림
너무 드물게 커밋 → 락 오래 유지 → 동시성 저하
```

**최적의 커밋 전략:**
- 트랜잭션의 **논리적 단위** 고려 (원자성이 필요한 범위)
- **성능 측정** 후 조절 (실제 환경에서 테스트)
- **데이터 중요도**에 따라 동기/비동기 선택

**예시:**
```sql
-- 주문 처리: 원자성 중요 → 한 번에 커밋
BEGIN TRANSACTION;
  UPDATE inventory SET qty = qty - 1 WHERE product_id = 100;
  INSERT INTO orders (product_id, qty) VALUES (100, 1);
  INSERT INTO payments (order_id, amount) VALUES (1001, 10000);
COMMIT;  -- 동기 커밋 (안전성 우선)

-- 로그 기록: 원자성 불필요 → 배치로 커밋
비동기 커밋 설정;
FOR i IN 1..100000 LOOP
  INSERT INTO access_logs VALUES (...);
  IF MOD(i, 1000) = 0 THEN
    COMMIT;  -- 1000건마다 커밋 (성능 우선)
  END IF;
END LOOP;
```

**3. 상황에 맞는 동시성 제어 선택**

```
충돌 많음 + 중요 데이터 → 비관적 락 + 동기 커밋
충돌 적음 + 일반 데이터 → 낙관적 락 + 동기 커밋
충돌 적음 + 로그 데이터 → 낙관적 락 + 비동기 커밋
```

**4. 모니터링과 튜닝**

정답은 없다. 실제 환경에서:
- 락 대기 시간 모니터링
- 커밋 빈도 조정
- 쿼리 성능 개선
- 지속적인 튜닝

으로 최적의 균형점을 찾아야 한다.

---

## 참고 문헌

- 조시형, 『친절한 SQL 튜닝』, 6장 DML 튜닝 전체 (p399-p497)
