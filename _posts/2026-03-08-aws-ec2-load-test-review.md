---
title: AWS EC2 부하 테스트 중간 점검 — 5개 도메인 테스트 후 인프라 진단과 다음 계획
date: 2026-03-08
tags: [AWS, EC2, MySQL, Redis, HikariCP, k6, 성능최적화, 부하테스트, Java, 인프라, Wireshark, 네트워크]
permalink: /aws-ec2-load-test-midpoint/
excerpt: "알림, 채팅, 피드, 정산, 검색 5개 도메인의 EC2 부하 테스트를 마쳤다. 로컬과 EC2의 병목은 본질적으로 다르다. 로컬은 쿼리·코드 레벨, EC2는 커넥션 풀·네트워크·OS 레벨이다. 알림·채팅에서 커널 튜닝을 적용하고, Buffer Pool 9.5%, redo log 48MB 등 인프라 설정 문제를 확인했다. 다음 단계로 수평 확장, Wireshark 패킷 분석, OS 네트워크 수치 측정을 진행한다."
---

## 개요

[알림](/notification-aws-ec2-load-test/), [채팅](/chat-aws-ec2-load-test/), [피드](/feed-aws-ec2-load-test/), [정산](/settlement-aws-ec2-load-test/), [검색](/search-aws-ec2-load-test/) 5개 도메인의 EC2 부하 테스트를 완료했다. 이 글은 전체 테스트를 통해 확인된 패턴, 인프라 점검 결과, 그리고 다음 테스트 계획을 정리한다.

### EC2 인프라 구성

| 역할 | 인스턴스 타입 | 사양 | 디스크 |
|------|-------------|------|--------|
| App Server | c5.xlarge | 4 vCPU (Xeon 8124M), 8GB | 49GB |
| Infra (MySQL, Redis, Kafka) | c5.2xlarge | 8 vCPU, 16GB | 146GB |
| Load Generator (k6) | c5.xlarge | 4 vCPU, 8GB | 29GB |

---

## 5개 도메인 테스트 결과 요약

| 도메인 | 최대 VU | 주요 병목 | 최종 상태 |
|--------|---------|----------|----------|
| 알림 | 6,000 | MySQL write 붕괴 (mark p50 29s), SSE 전달률 63.2% | MongoDB 비교: mark p50 30ms, 커널/JVM 튜닝 후 iterations +20% |
| 채팅 | 4,500 | write-pool 24.7s 점유 (TX 내 Redis publish) | TX 분리 후 전 Phase 100%, 에러 1건, write-pool acquire 95% 개선 |
| 피드 | 3,000 | IN절 병목 (p95 9.55s), X-lock 경합 | 개인 피드 p95 2.00s (1,000VU), 피드 생성 p95 2.88s (3,000VU) |
| 정산 | 1,500 | COMPLETED 0건, row lock 10,832건 | p95 18.64s → 1.50s, 5xx 0% |
| 검색 | 2,000 | FULLTEXT ngram p95 24.11s | 600+ VU에서 커넥션 풀 포화 |

---

## 로컬과 EC2의 차이 — 반복된 패턴

5개 도메인 테스트에서 동일한 패턴이 반복됐다.

| 항목 | 로컬 | EC2 |
|------|------|-----|
| 네트워크 RTT | 0ms (동일 머신) | 0.5~1ms (VPC 내부) |
| 병목 유형 | 쿼리 실행 시간, 코드 레벨 버그 | 커넥션 풀 소진, 네트워크 지연, OS 제약 |
| HikariCP Pending | 관측되지 않음 | 알림 275, 채팅 173, 피드 268, 정산 131 |
| 커넥션 점유 시간 | 짧음 (RTT 0) | 길어짐 (매 쿼리에 RTT 추가) |
| 드러나는 문제 | WHERE IN 풀스캔, Redis Pub/Sub 풀 포화 | 풀 크기 부족, OS 커널 설정, Buffer Pool 부족 |

로컬에서는 네트워크 지연이 0이므로 커넥션이 빠르게 반환된다. 동일한 풀 크기라도 EC2에서는 각 쿼리에 RTT가 추가되어 커넥션 점유 시간이 길어지고, Pending이 발생한다. 로컬 테스트에서 발견되지 않던 인프라 레벨 병목이 EC2에서 드러나는 이유다.

---

## 인프라 점검 결과

전체 테스트 완료 후 3대 서버의 설정을 점검했다.

### 커널 튜닝 (App 서버, k6 서버)

초기 점검 시 아래 설정이 기본값 상태였다. 알림·채팅 도메인 테스트 과정에서 권장값으로 적용했다.

| 설정 | 초기값 | 적용값 | 영향 |
|------|--------|--------|------|
| net.core.rmem_max | 212KB | 16MB | TCP 수신 버퍼 부족 → 고부하 시 패킷 드롭 |
| net.core.wmem_max | 212KB | 16MB | TCP 송신 버퍼 부족 |
| tcp_keepalive_time | 7,200s | 60s | 비활성 커넥션 2시간 유지 → 리소스 낭비 |
| tcp_slow_start_after_idle | 1 | 0 | idle 후 재시작 시 cwnd 초기화 → 전송 지연 |
| tcp_fin_timeout | 60s | 15s | TIME_WAIT 소켓 60초 점유 |

알림 도메인에서 적용 후 SSE 연결 p95가 708ms → 90ms(-87%)로 개선됐고, 채팅 도메인에서도 p95 28~45% 개선에 기여했다. 피드·정산·검색 도메인은 커널 튜닝 적용 전에 테스트를 수행했다.

### MySQL 설정

| 설정 | 현재 | 권장 | 비고 |
|------|------|------|------|
| innodb_buffer_pool_size | 3GB | 8~10GB | 데이터+인덱스 32GB 대비 9.5%. 권장 70~80% |
| innodb_redo_log_capacity | 48MB | 512MB~1GB | 쓰기 부하 시 체크포인트 빈번 |
| innodb_flush_method | fsync | O_DIRECT | 이중 버퍼링 방지 (재시작 필요) |

Buffer Pool 3GB는 전체 데이터+인덱스 32GB의 9.5%다. InnoDB 권장치 70~80%에 크게 미달한다. 피드 테스트에서 Buffer Pool free pages가 142K → 28K로 감소한 것, 정산 테스트에서 Slow Query가 발생한 것에 이 설정이 기여했을 가능성이 있다.

### Redis 설정

| 설정 | 현재 | 권장 |
|------|------|------|
| maxmemory | 512MB | 1~2GB |
| eviction policy | allkeys-lru | 유지 |

Infra 서버 여유 RAM 8.2GB 중 Redis에 512MB만 할당되어 있다. 피드 테스트에서 Redis 메모리가 563MB까지 증가하여 eviction이 발생한 사례가 있었다.

### 정상 항목

- ulimit -n: 65,535 (파일 디스크립터 충분)
- ip_local_port_range: 1024~65,535 (k6 포트 충분)
- MySQL max_connections: 800 (초기 400 → 채팅·정산에서 500 → 피드에서 800으로 단계적 확대)
- HikariCP R/W 분리: 정상 동작
- innodb_flush_log_at_trx_commit: 2 (성능 우선 설정)
- innodb_io_capacity: 4,000/20,000 (SSD 환경에 적절)
- innodb_lock_wait_timeout: 3s (고부하 대응 설정)
- Kafka: 12개 Docker 컨테이너 전부 healthy

---

## 다음 계획

### 1. 인프라 튜닝 적용 후 재테스트

| 우선순위 | 변경 | 재시작 필요 |
|---------|------|-----------|
| 높음 | App/k6 서버 커널 튜닝 — 알림·채팅에서 적용 완료, 나머지 도메인 재테스트 시 효과 확인 | 불필요 |
| 높음 | MySQL Buffer Pool 3GB → 8~10GB | 필요 |
| 중간 | MySQL innodb_redo_log_capacity 48MB → 512MB | 필요 |
| 중간 | Redis maxmemory 512MB → 1~2GB | 불필요 |
| 낮음 | MySQL innodb_flush_method → O_DIRECT | 필요 |

커널 튜닝과 Buffer Pool 확대 후 동일 시나리오를 재실행하여, 인프라 설정이 응답 시간과 HikariCP Pending에 미치는 영향을 측정한다.

### 2. 수평 확장 (다중 앱 인스턴스)

피드 Phase 9(1,000~3,000 VU)에서 HikariCP 300 풀 포화와 CPU 100%가 발생했다. 이는 단일 인스턴스의 물리적 한계이므로 수평 확장을 시도한다.

- App Server 2대 구성 + ALB(Application Load Balancer) 도입
- 세션 공유: Redis 기반 Spring Session
- 커넥션 풀: 인스턴스당 150으로 분할 (총 300 유지)
- WebSocket/SSE: sticky session 또는 Redis Pub/Sub 기반 브로드캐스트

단일 인스턴스 대비 수평 확장의 처리량 변화를 동일 시나리오로 비교한다.

### 3. Wireshark 패킷 분석

EC2 테스트에서 반복적으로 관찰된 현상 — HikariCP Pending, 커넥션 점유 시간 증가, SSE 전달률 저하 — 의 네트워크 레벨 원인을 Wireshark로 분석한다.

측정 대상:
- App ↔ MySQL 간 TCP RTT 분포
- 커넥션 수립(3-way handshake) 소요 시간
- 고부하 시 TCP retransmission, duplicate ACK 발생 빈도
- SSE 스트림의 패킷 전달 지연 패턴
- Kafka producer ↔ broker 간 메시지 전달 지연

App 서버에서 `tcpdump`로 캡처한 pcap 파일을 Wireshark에서 분석하여, 네트워크 지연이 애플리케이션 성능에 미치는 영향을 정량화한다.

### 4. OS 네트워크 수치 측정

커널 튜닝 적용 전후의 OS 레벨 네트워크 지표를 수집한다.

| 지표 | 수집 방법 | 측정 목적 |
|------|----------|----------|
| TCP 소켓 상태 분포 | `ss -s` | TIME_WAIT, ESTABLISHED 비율 |
| 수신/송신 버퍼 사용량 | `ss -tm` | 버퍼 오버플로우 여부 |
| 패킷 드롭 | `netstat -s` | 버퍼 부족으로 인한 드롭 수 |
| TCP retransmission | `/proc/net/snmp` | 재전송 비율 |
| 커넥션 수립 지연 | `ss -ti` | RTT, cwnd, retrans |
| NIC 통계 | `ethtool -S` | rx/tx errors, drops |

부하 테스트 중 1초 간격으로 수집하여, 커널 튜닝 전후 차이를 비교한다. 특히 `tcp_slow_start_after_idle=0` 적용 전후의 cwnd 변화와, TCP 버퍼 확대 전후의 패킷 드롭 수 변화를 확인한다.

---

## 정리

| 테스트 단계 | 검증 대상 | 상태 |
|------------|----------|------|
| 로컬 부하 테스트 | 쿼리 최적화, 코드 레벨 버그 | 완료 |
| EC2 단일 인스턴스 | 커넥션 풀, 네트워크 지연, 인프라 설정 | 완료 |
| 인프라 튜닝 | 커널, Buffer Pool, redo log | 커널: 알림·채팅에서 적용 완료, Buffer Pool·redo log: 피드에서 일부 적용 |
| 수평 확장 | 다중 인스턴스 + ALB | 예정 |
| 네트워크 분석 | Wireshark, OS 수치 | 예정 |

> **부하 테스트는 단계별로 다른 레벨의 병목을 드러낸다.** 로컬에서는 쿼리와 코드, EC2 단일 인스턴스에서는 커넥션 풀과 인프라 설정, 수평 확장에서는 세션 공유와 네트워크 분산이 각각의 과제가 된다. 다음 단계에서는 Wireshark와 OS 네트워크 수치로 네트워크 레벨까지 가시성을 확보한다.

---

## 시리즈 탐색

**◀ 이전 글**
[검색 부하 테스트 — AWS EC2에서 MySQL FULLTEXT ngram의 성능 한계](/search-aws-ec2-load-test/)

**▶ 다음 글**
[부하 테스트 코드 리뷰 — k6 시나리오의 10가지 함정과 수정](/k6-load-test-code-review/)
