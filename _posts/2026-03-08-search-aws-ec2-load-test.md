---
title: 검색 부하 테스트 — AWS EC2에서 MySQL FULLTEXT ngram의 성능 한계
date: 2026-03-08
tags: [AWS, EC2, MySQL, FULLTEXT, Redis, HikariCP, k6, 성능최적화, 부하테스트, Java, 검색]
permalink: /search-aws-ec2-load-test/
excerpt: "EC2에 배포한 검색 API에서 시드 데이터 이중 인코딩 문제를 발견하고 수정했다. MySQL FULLTEXT(ngram) 검색은 단일 키워드 p95 24.11s, 스파이크 p95 45.40s로, 600+ 동시 사용자에서 HikariCP 300 풀이 포화된다. 필터 전용 쿼리(인덱스 스캔)는 13ms로 빠르며, FULLTEXT 쿼리 자체의 비용이 병목이다."
---

## 개요

[검색 엔진 추상화 글](/search-engine-abstraction-es-vs-fulltext/)에서 MySQL FULLTEXT(ngram)과 Elasticsearch를 비교하고, 단일 인스턴스 환경에서는 FULLTEXT를 선택했다. 이번 테스트에서는 EC2 환경에서 실제 부하를 걸어 FULLTEXT의 성능 한계를 확인한다.

## EC2 인프라 구성

| 역할 | 인스턴스 타입 | 사양 |
|------|-------------|------|
| Load Generator (k6) | c5.xlarge | 4 vCPU, 8GB |
| App Server | c5.xlarge | 4 vCPU, 8GB |
| Infra (MySQL, Redis) | c5.2xlarge | 8 vCPU, 16GB |

## 시드 데이터 이중 인코딩 문제

테스트 초기에 검색 결과가 0건이었다. 원인: MySQL 시드 데이터가 이중 인코딩(Double Encoding)되어 있었다.

### 발생 과정

1. seed SQL의 "축구동호회" → UTF-8 바이트: `EC B6 95 EA B5 AC ...`
2. Docker 내 mysql CLI의 기본 charset: `latin1`
3. MySQL이 UTF-8 바이트를 latin1 문자로 해석: `EC` → `ì`(U+00EC), `B6` → `¶`(U+00B6) ...
4. 컬럼이 `utf8mb4`이므로 이 latin1 문자들을 다시 UTF-8로 인코딩하여 저장
5. 결과: "축구동호회 1"이 7글자/17바이트여야 하는데 → 17글자/34바이트로 저장

### 왜 MySQL CLI에서는 검색되었는가

- Docker 내 mysql CLI도 `latin1` 기본 → 검색어도 같은 이중 인코딩 → 매칭 성공
- JDBC (`utf8mb4` 연결) → 검색어가 정상 UTF-8 → 이중 인코딩된 데이터와 불일치 → 0건

### 수정

```sql
ALTER TABLE club MODIFY name VARCHAR(255) CHARACTER SET latin1;
ALTER TABLE club MODIFY name VARBINARY(255);
ALTER TABLE club MODIFY name VARCHAR(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
ALTER TABLE club ADD FULLTEXT INDEX ft_club_search (name, description) WITH PARSER ngram;
```

수정 후 JDBC FULLTEXT 검색: 0건 → 5,000건 정상 반환. 이 문제를 진단하고 수정하는 데 상당한 시간이 소요됐다. mysql CLI에서는 정상 검색되기 때문에 JDBC 쪽 문제로 오인하기 쉬웠고, charset 변환 과정을 역추적해야 원인이 드러났다. 정산 도메인의 시드 데이터 버그와 마찬가지로, 부하 테스트에서 테스트 환경 자체의 정합성 확보가 최적화 이전에 필요한 작업이다.

## 테스트 결과

### API 응답 시간

| API | p95 | avg |
|-----|-----|-----|
| 단일 키워드 검색 | 24.11s | 11.86s |
| 복합 키워드 검색 | 12.34s | 8.34s |
| 키워드+지역 | 18.64s | 12.56s |
| 키워드+관심사 | 21.72s | 12.43s |
| 키워드+풀필터 | 18.66s | 14.69s |
| 필터 전용(MySQL 인덱스 스캔) | 13ms | 9ms |
| 추천 모임 | 19.27s | 2.95s |
| 팀메이트 모임 | 511ms | 131ms |
| 스파이크(2,000 VU) | 45.40s | 26.84s |

### 검색 정확도

| 지표 | 값 |
|------|-----|
| 단일 키워드 평균 결과 수 | 1.9건 |
| 단일 키워드 빈 결과율 | 90.7% |
| 복합 키워드 빈 결과율 | 95.6% |

빈 결과율이 높은 이유: 이중 인코딩 수정 후 매칭 데이터가 50K → 5K로 감소. k6 테스트 키워드가 실제 데이터와 불일치.

### 인프라

| 지표 | 값 |
|------|-----|
| HikariCP Active Max | 300 (풀 포화) |
| HikariCP Pending Max | 165 |
| JVM Heap Max | 1,888MB |
| CPU Max | 100% |
| 5xx 에러 | 29,966건 |
| Thresholds | 7 PASS / 22 FAIL |

### Redis 캐시 상태 비교 (Run 1 캐시 포화 vs Run 2 정상)

| 항목 | Run 1 (캐시) | Run 2 (정상) |
|------|------------|------------|
| Redis RAM | 563MB | 22MB |
| Redis CPU | 95% | 0.56% |
| MySQL CPU | 222% | 229% |
| MySQL QPS | 6,568 | 7,220 |

Run 1에서 Redis 캐시가 563MB까지 적재되어 CPU 95%를 차지. Run 2에서는 22MB로 정상 동작. Redis 캐시가 과적되면 캐시 자체가 병목이 되고, 캐시가 정상 동작할 때는 DB 병목을 감추는 효과가 있다. 캐시 유무에 따라 측정되는 병목 지점이 달라지므로, 캐시 적용 전후를 분리하여 측정해야 실제 DB 성능을 파악할 수 있다.

## 병목 분석

1. **FULLTEXT 쿼리 비용**: ngram 파서 + `ORDER BY relevance DESC` + 69K 클럽 = 각 쿼리 수십~수백ms. 600+ 동시 사용자가 FULLTEXT 검색을 실행하면 커넥션 풀이 포화.
2. **필터 전용 쿼리와의 차이**: 필터 전용(인덱스 스캔) 13ms vs FULLTEXT 24.11s. 차이의 원인은 FULLTEXT 쿼리 자체의 연산 비용.
3. **HikariCP 포화**: FULLTEXT 쿼리가 오래 걸리면서 300개 커넥션 전부 점유, pending 165 발생.

## FULLTEXT의 구조적 한계

MySQL FULLTEXT(ngram)은 다음 특성을 갖는다:

- **ngram 토큰화**: 2-gram 기본 → "축구" → ["축구"] 토큰 생성, 역인덱스 검색
- **relevance 정렬**: TF-IDF 기반 스코어 계산이 전체 매칭 행에 대해 실행
- **단일 스레드**: 하나의 FULLTEXT 쿼리는 단일 CPU 코어에서 실행

600+ 동시 FULLTEXT 쿼리는 c5.2xlarge(8 vCPU)에서도 CPU-bound 병목이 된다. 이는 인덱스 추가나 커넥션 풀 확대로 해결되지 않는다.

## 개선 방안

| # | 방안 | 효과 | 비고 |
|---|------|------|------|
| 1 | Elasticsearch 도입 | 분산 검색, 샤딩으로 수평 확장 | 인프라 비용 증가 |
| 2 | Redis 검색 캐싱 강화 | 반복 쿼리 DB 우회 | 캐시 메모리 관리 필요 |
| 3 | 검색 결과 사전 인덱싱 | 자주 검색되는 키워드 결과 캐싱 | 실시간성 감소 |

## 정리

| 항목 | 필터 전용 (인덱스 스캔) | FULLTEXT (ngram) |
|------|----------------------|-----------------|
| p95 | 13ms | 24.11s |
| 동시 사용자 한계 | 1,000+ VU | ~600 VU |
| HikariCP 상태 | 여유 | 포화 (300/300, pending 165) |
| 병목 원인 | 없음 | 쿼리 연산 비용 |

> **MySQL FULLTEXT(ngram)은 단일 인스턴스에서 600+ 동시 검색 시 커넥션 풀을 포화시킨다.** 필터 전용 쿼리는 13ms인 반면 FULLTEXT는 24초로, 검색 연산 자체가 병목이다. 검색 트래픽이 이 수준을 초과하면 Elasticsearch 같은 분산 검색 엔진 도입이 필요하다.

## 시리즈 탐색

**◀ 이전 글**
[정산 부하 테스트 — AWS EC2에서 시드 데이터 버그 수정과 5라운드 최적화](/settlement-aws-ec2-load-test/)

**▶ 다음 글**
[AWS EC2 부하 테스트 중간 점검 — 5개 도메인 테스트 후 인프라 진단과 다음 계획](/aws-ec2-load-test-midpoint/)
