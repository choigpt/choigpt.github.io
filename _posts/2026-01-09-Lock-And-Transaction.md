---
layout: post
title: "Lock과 Transaction"
date: 2026-01-09
categories: [Database]
tags: [lock, transaction, concurrency, database]
last_modified_at: 2026-01-23
---

## 참고 문헌
- 조시형, 『친절한 SQL 튜닝』, p466-p479

## 수정 이력
- 2026-01-23: 초안 작성

---

## Lock과 Transaction

데이터베이스에서 여러 사용자가 동시에 같은 데이터를 수정하려고 하면 어떻게 될까? 누군가는 기다려야 하고, 데이터의 일관성을 보장해야 한다. 이를 위해 데이터베이스는 **Lock(락)**이라는 메커니즘을 사용한다.

락에는 여러 종류가 있지만, 애플리케이션 개발 측면에서 가장 중요한 것은 **DML Lock**이다. DML(Data Manipulation Language)은 데이터를 조작하는 SQL 문으로, INSERT, UPDATE, DELETE, SELECT 등이 여기에 해당한다.

### 1. DML 로우 락 (Row Lock)

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

#### SELECT는 왜 다른 트랜잭션을 방해하지 않을까? - MVCC

많은 현대 데이터베이스(Oracle, PostgreSQL, MySQL InnoDB 등)는 **MVCC(Multi-Version Concurrency Control)** 모델을 사용한다.

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

### 2. 성능 최적화

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

### 3. 테이블 락 (Table Lock)

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

### 4. 트랜잭션 관리 주의사항

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

### 5. 커밋과 LGWR (Log Writer)

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
3. 갑자기 정전! 💥
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
3. 서버 프로세스가 즉시 사용자에게 응답: "커밋 성공!" ⚡
   |
4. LGWR는 백그라운드에서 천천히 디스크에 기록
```

**장점:**
- 응답 속도 매우 빠름 (디스크 I/O 대기 없음)
- 대량의 트랜잭션 처리 가능

**단점:**
- 디스크에 쓰기 전에 장애 발생 시 데이터 유실

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

**언제 사용하는가?**

1. **로그성 데이터**
   ```sql
   -- 웹 사이트 방문 기록
   INSERT INTO access_logs (user_id, page, timestamp) 
   VALUES (123, '/products', NOW());
   COMMIT;  -- 몇 건 유실되어도 큰 문제 없음
   ```

2. **대량 배치 작업**
   ```sql
   -- 100만 건의 통계 데이터 생성
   FOR i IN 1..1000000 LOOP
     INSERT INTO statistics VALUES (...);
     IF MOD(i, 1000) = 0 THEN
       COMMIT;  -- 비동기로 빠르게!
     END IF;
   END LOOP;
   ```

3. **임시 데이터**

**절대 사용하면 안 되는 경우:**
- 금융 거래 (계좌 이체, 결제 등)
- 주문 정보
- 사용자 인증 정보
- 모든 중요한 비즈니스 데이터

**비교표:**

| 구분 | 동기 커밋 | 비동기 커밋 |
|------|----------|-----------|
| 응답 시간 | 5~10ms (디스크 I/O 대기) | <1ms (즉시) |
| 데이터 안정성 | 매우 높음 (장애 시 복구 가능) | 낮음 (장애 시 유실 가능) |
| 처리량 | 낮음 (초당 수백~수천 TPS) | 높음 (초당 수만~수십만 TPS) |
| 사용 사례 | 금융, 주문, 인증 등 중요 데이터 | 로그, 통계, 임시 데이터 |
| 기본 설정 | O (대부분 DB의 기본값) | X (명시적으로 설정 필요) |

### 6. 트랜잭션 동시성 제어 방식

같은 데이터를 여러 사용자가 동시에 수정하려 할 때 어떻게 처리할 것인가?

#### 비관적 동시성 제어 (Pessimistic Concurrency Control)

**개념:** "다른 사람이 수정할 거야. 미리 잠가놓자!"

**동작:**

```sql
-- 트랜잭션 A: 주문 조회 및 락 획득
SELECT * FROM orders WHERE order_id = 1001 FOR UPDATE;
-- 이 순간 다른 트랜잭션은 이 행을 수정할 수 없음

-- 비즈니스 로직 처리
-- ...

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
    // 락 획득 시도
    Order order = orderRepository.findByIdForUpdate(1001);
    // 성공 → 처리 진행
} catch (LockTimeoutException e) {
    // 실패 → 사용자에게 "다른 사용자가 처리 중입니다" 메시지
    return "현재 이 주문을 다른 사용자가 처리하고 있습니다.";
}
```

**장점:**
- 확실한 데이터 보호
- 동시 수정 방지 보장

**단점:**
- 락을 잡고 있는 동안 다른 사용자는 대기
- 교착상태(Deadlock) 발생 가능

**언제 사용?**
- 동시 수정이 빈번한 경우
- 데이터 충돌이 발생하면 안 되는 경우
- 예: 재고 관리, 좌석 예매

#### 낙관적 동시성 제어 (Optimistic Concurrency Control)

**개념:** "대부분은 충돌이 안 일어날 거야. 일단 진행하고, 마지막에 확인하자!"

**구현 방법 1: 버전 컬럼 사용**

```sql
-- 테이블 구조
CREATE TABLE orders (
    order_id INT,
    status VARCHAR(20),
    version INT,  -- 버전 컬럼 추가
    ...
);

-- 1단계: 데이터 조회 (락 없이)
SELECT order_id, status, version 
FROM orders WHERE order_id = 1001;
-- 결과: order_id=1001, status='PENDING', version=5

-- 2단계: 비즈니스 로직 처리 (시간이 오래 걸릴 수 있음)
-- ... 복잡한 계산 ...

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

**구현 방법 2: 타임스탬프 사용**

```sql
-- 테이블 구조
CREATE TABLE orders (
    order_id INT,
    status VARCHAR(20),
    updated_at TIMESTAMP,  -- 최종 수정 시간
    ...
);

-- 1. 조회
SELECT order_id, status, updated_at 
FROM orders WHERE order_id = 1001;
-- 결과: updated_at = '2026-01-23 10:30:00'

-- 2. 업데이트
UPDATE orders 
SET status = 'SHIPPED',
    updated_at = CURRENT_TIMESTAMP
WHERE order_id = 1001 
  AND updated_at = '2026-01-23 10:30:00';  -- 변경 안 됐는지 확인
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
    // 다른 사용자가 먼저 수정함
    return "충돌 발생. 다시 시도해주세요.";
}
```

**장점:**
- 락을 잡지 않으므로 동시성 높음
- 대기 시간 없음
- 교착상태 발생하지 않음

**단점:**
- 충돌 시 재시도 필요
- 충돌이 빈번하면 오히려 비효율적

**언제 사용?**
- 동시 수정이 드문 경우
- 읽기가 많고 쓰기가 적은 경우
- 예: 게시글 수정, 프로필 업데이트

**비관적 vs 낙관적 선택 기준:**

| 상황 | 권장 방식 | 이유 |
|-----|----------|------|
| 동시 수정 빈번 (재고, 좌석) | 비관적 | 충돌이 많아 재시도 비용이 큼 |
| 동시 수정 드묾 (게시글) | 낙관적 | 락으로 인한 대기가 낭비 |
| 짧은 트랜잭션 | 비관적 | 락 유지 시간이 짧아 영향 적음 |
| 긴 트랜잭션 | 낙관적 | 락을 오래 잡으면 다른 사용자 대기 |
| 실시간 시스템 | 낙관적 | 대기 없이 빠른 응답 |

### 7. 핵심 원칙

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
  -- 재고 차감
  UPDATE inventory SET qty = qty - 1 WHERE product_id = 100;
  -- 주문 생성
  INSERT INTO orders (product_id, qty) VALUES (100, 1);
  -- 결제 기록
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
