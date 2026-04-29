---
title: 1BRC Challenge — 10억 행 CSV를 Java로 얼마나 빨리 읽을 수 있을까
date: 2026-03-18
tags:
  - Java
  - Performance
  - IO
  - Optimization
  - 1BRC
  - HashMap
  - 병렬처리
permalink: /1brc-challenge/
excerpt: 1 Billion Row Challenge를 자바로 풀면서 거친 11단계 최적화 기록. readLine()부터 바이트 직접 스캔, 수동 파싱, 병렬 처리까지 — 각 단계마다 무엇이 병목이었고 무엇을 제거했는지 정리한다.
---

## 들어가며

유튜브에서 1BRC(1 Billion Row Challenge)를 소개하는 영상을 봤다. 10억 줄짜리 기상 관측 CSV를 읽고 station별 min/mean/max를 가장 빨리 출력하는 도전인데, 단순해 보여서 나도 한번 해보려고 [GitHub 레포](https://github.com/gunnarmorling/1brc)를 받았다.

솔직히 혼자 힘으로는 엄두가 안 났다. 결국 Codex를 켜놓고 1등([thomaswue](https://github.com/thomaswue/1brc-steps))의 코드를 따라가는 방식으로 진행했다. 1등은 GraalVM 팀 리더 Thomas Wuerthinger인데, GraalVM native image에 `sun.misc.Unsafe`, epsilon GC, 프로세스 포킹까지 동원해서 **316ms**에 끝냈다. 내 Windows 환경에서 그대로 재현할 수는 없지만, 그 코드를 한 줄씩 뜯어보면서 "왜 이렇게 했는지"를 이해하는 것 자체가 공부였다. 그러다 zero-copy 개념이 나왔고, 그게 [별도의 글](/zero-copy/)로 이어졌다.

이 글은 가장 느린 구현에서 출발해 11단계에 걸쳐 최적화한 과정을 기록한다. 벤치마크 수치는 따로 넣지 않았다 — 1등의 기록은 Hetzner 전용 서버 + 커스텀 이미지 기준이라 내 로컬에서 재현해도 의미가 없고, 각 단계에서 **무엇이 병목이었고 무엇을 제거했는지**에 집중하는 게 이 글의 목적이다.

---

## 알아야 할 핵심 개념

최적화 과정을 이해하려면 "같은 일을 하더라도 CPU, 메모리, 런타임이 무엇을 더 많이 하게 만드는가"를 보는 감각이 필요하다.

- **I/O vs CPU 병목**: 큰 파일 처리에서 처음엔 디스크가 느릴 것 같지만, 자바에서 바이트마다 메서드 호출, 문자열 생성, 파싱을 하면 CPU가 먼저 병목이 된다.
- **객체 생성 비용**: `String`, `substring()`, `Double.parseDouble()` 같은 경로는 편하지만 객체 할당과 GC 부담이 크다. "한 줄마다 새 객체를 만들지 않기"가 핵심이다.
- **파싱 비용**: `"12.3"`을 `Double.parseDouble()`로 읽는 건 범용적이라 느리다. 데이터 형식이 고정이면 바이트를 직접 읽어서 `123` 같은 정수로 만드는 게 훨씬 싸다.
- **해시 테이블 비용**: `HashMap<String, Stats>`는 편하지만 String 생성, 해시 계산, equals 비교가 누적된다. station 키를 바이트 기반으로 다루면 비용을 줄일 수 있다.
- **메모리 복사**: 이미 읽어온 바이트를 `lineBuffer`로 한 번 더 복사하면 메모리 대역폭과 CPU를 낭비한다. 원본 버퍼에서 바로 처리하는 게 좋다.
- **캐시 친화성**: 작은 원시 타입(`int`, `long`, `byte[]`) 위주로 연속 메모리를 다루면 CPU 캐시 효율이 좋아진다. 문자열/객체가 많아지면 포인터 추적이 늘어난다.
- **병렬 처리**: 파일을 여러 구간으로 나눠 각 스레드가 독립 처리하면 코어를 활용할 수 있다. 세그먼트 시작점은 줄 경계에 맞춰야 하고, 마지막에 결과 merge 비용이 든다.
- **환경 의존성**: 상위권 코드는 Linux, GraalVM native, Unsafe, mmap 전제가 많다. Windows + HotSpot JVM에서는 같은 코드가 꼭 더 빠르지 않다.

---

## 단계별 최적화 과정

### 1단계 — 바이트 수만 세기

```java
try (InputStream in = new BufferedInputStream(new FileInputStream(filePath))) {
    int b;
    while ((b = in.read()) != -1) {
        totalBytes++;
    }
}
```

- 1바이트씩 `read()` 호출 → 매우 비효율적
- 1BRC 계산이 아니라 단순 I/O 루프 측정 기준선

---

### 2단계 — 블록 단위 읽기

```java
byte[] buffer = new byte[1024 * 1024];
while ((n = in.read(buffer)) != -1) {
    totalBytes += n;
}
```

- **제거한 병목:** 바이트마다 `read()` 호출하는 메서드 호출 오버헤드
- 디스크/스트림 읽기 기준선 개선

---

### 3단계 — 줄 단위 읽기

```java
try (BufferedReader reader = new BufferedReader(new FileReader(filePath), 1024 * 1024)) {
    String line;
    while ((line = reader.readLine()) != null) {
        lineCount++;
        totalChars += line.length();
    }
}
```

- "파일 읽기"에서 "행 처리"로 진입
- **새로운 비용:** `String` 객체 생성 시작

---

### 4단계 — station/value 분리

```java
int separator = line.indexOf(';');
String station = line.substring(0, separator);
String value = line.substring(separator + 1);
```

- 1BRC 레코드 구조를 실제로 파싱
- **새로운 비용:** `substring()` 객체 생성

---

### 5단계 — 숫자 파싱 추가

```java
double parsedValue = Double.parseDouble(value);
valueSum += parsedValue;
```

- **새로운 비용:** 범용 부동소수 파싱
- 여기서 CPU 사용량이 크게 상승

---

### 6단계 — station별 집계

```java
Map<String, Stats> statsByStation = new HashMap<>();
Stats stats = statsByStation.computeIfAbsent(station, k -> new Stats());
stats.add(parsedValue);
```

- **진짜 1BRC 기본형 완성**
- 하지만 `String` 중심이라 아직 느림

---

### 7단계 — readLine() 제거, 바이트 직접 스캔

- `BufferedInputStream`으로 큰 청크를 읽고 `\n`과 `;`를 직접 찾음
- 처음엔 줄마다 `lineBuffer`에 복사해서 처리
- **제거한 병목:** `BufferedReader` 오버헤드
- **트레이드오프:** 구현 복잡도 상승

---

### 8단계 — Double.parseDouble() 제거, 수동 숫자 파싱

```java
private static int parseTemperature(byte[] buf, int start, int end) {
    int sign = 1;
    if (buf[start] == '-') { sign = -1; start++; }
    int value = 0;
    while (start < end) {
        byte c = buf[start++];
        if (c != '.') value = value * 10 + (c - '0');
    }
    return value * sign;
}
```

- `"12.3"`을 정수 `123`으로 바로 파싱
- **핵심 최적화 포인트** — 범용 파싱의 객체 생성·예외 처리·문자열 변환 비용을 모두 제거한다. [thomaswue의 단계별 레포](https://github.com/thomaswue/1brc-steps)에서도 이 단계에서 가장 큰 폭의 개선이 관찰된다.

---

### 9단계 — station 문자열 생성 최소화

- 매 줄 `new String(station)` 하지 않음
- 조회용 임시 키로 먼저 Map 검색 → 새 station일 때만 바이트 복사 후 저장
- **제거한 병목:** station 관련 객체 생성
- `HashMap<String, ...>`보다 싸짐

---

### 10단계 — 줄 버퍼 복사 제거

- 청크 안에 완전한 줄은 `readBuffer`에서 바로 처리
- 청크 끝에 걸린 미완성 줄만 `pendingLine`에 복사
- **제거한 병목:** 모든 바이트를 두 번 만지던 구조
- 메모리 복사 비용 절감

---

### 11단계 — 파일 청크 병렬 처리

- 파일을 여러 세그먼트로 나눔
- 세그먼트 시작은 줄 경계로 보정
- 스레드별 partial map 생성 → 마지막에 merge
- **멀티코어 활용 — 최종 형태**

---

## 현재 코드 상태

최종 `Main.java`의 특징:

1. **바이트 직접 스캔** — `readLine()` 없음
2. **수동 온도 파싱** — `Double.parseDouble()` 없음
3. **station 문자열 생성 최소화** — 새 station일 때만 생성
4. **청크 경계 줄만 복사** — 완전한 줄은 원본 버퍼에서 처리
5. **세그먼트 병렬 처리 + merge** — 멀티코어 활용

---

## 이해 순서 추천

이 과정을 공부용으로 소화하려면 이 순서가 좋다:

1. 왜 `readLine()` + `split()` + `Double.parseDouble()`가 느린지
2. 왜 객체 생성이 느린지
3. 왜 고정 형식 데이터는 수동 파싱이 유리한지
4. 왜 `HashMap<String, ...>`가 비싼지
5. 왜 "복사 제거"가 중요한지
6. 왜 병렬화는 마지막 단계인지
