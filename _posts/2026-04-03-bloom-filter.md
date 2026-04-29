---
layout: post
title: "Bloom Filter: 확률적 자료구조의 정수 — 원리, 수학, 변형, 실전 적용까지"
date: 2026-04-03
tags: [data-structure, CS, algorithm, bloom-filter, probabilistic, 확률적자료구조, 캐시, Redis, Guava]
category: CS
permalink: /bloom-filter/
excerpt: "Bloom Filter는 '이 원소가 집합에 있는가?'라는 질문에 O(1)로 답하되, 거짓 양성(false positive)을 허용하는 대신 거짓 음성(false negative)은 절대 발생하지 않는 확률적 자료구조다. 비트 배열과 해시 함수만으로 수백만 원소의 멤버십을 수 KB에 판정하는 원리, 오탐률 수학, Counting·Cuckoo·Scalable 변형, 그리고 Redis·Guava·DB 쿼리 최적화 실전까지 정리한다."
---

"이 사용자 이름은 이미 사용 중인가?", "이 URL은 이미 크롤링했는가?", "이 키가 DB에 존재하는가?"

이런 **멤버십 질의(membership query)**는 어디서든 등장한다. 해시 테이블이면 정확히 답할 수 있지만, 원소가 수억 개면 메모리가 수 GB 필요하다. 정확도를 조금 양보하면 메모리를 수십~수백 배 줄일 수 있는데, 그게 Bloom Filter다.

---

# 핵심 아이디어

Bloom Filter는 1970년 Burton Howard Bloom이 제안했다. 핵심은 단 두 줄로 요약된다.

1. **"없다"고 답하면 → 100% 없다** (false negative 없음)
2. **"있다"고 답하면 → 아마 있다** (false positive 가능)

이 비대칭성이 Bloom Filter의 존재 이유다. "없다"를 확신할 수 있으므로, **불필요한 작업을 조기에 차단**하는 데 쓴다.

---

# 구조

Bloom Filter의 내부는 놀라울 정도로 단순하다.

```
m비트 배열 (초기값 전부 0)
k개의 독립적인 해시 함수 h₁, h₂, ..., hₖ
```

그게 전부다. 포인터도, 연결 리스트도, 트리도 없다.

## 삽입 (Insert)

원소 x를 삽입할 때:

```
h₁(x) mod m → i₁번째 비트를 1로
h₂(x) mod m → i₂번째 비트를 1로
...
hₖ(x) mod m → iₖ번째 비트를 1로
```

예를 들어 m=16, k=3이고 "apple"을 삽입하면:

```
h₁("apple") mod 16 = 3
h₂("apple") mod 16 = 7
h₃("apple") mod 16 = 11

비트 배열:
[0 0 0 1 0 0 0 1 0 0 0 1 0 0 0 0]
       ↑           ↑           ↑
       3           7          11
```

"banana"도 삽입하면:

```
h₁("banana") mod 16 = 1
h₂("banana") mod 16 = 7   ← 이미 1인 비트와 겹침!
h₃("banana") mod 16 = 14

비트 배열:
[0 1 0 1 0 0 0 1 0 0 0 1 0 0 1 0]
   ↑   ↑           ↑           ↑  ↑
   1   3           7          11 14
```

7번 비트는 "apple"과 "banana" 모두 사용한다. 이 **비트 공유**가 메모리 절약의 원천이면서 동시에 오탐(false positive)의 원인이다.

## 조회 (Lookup)

원소 x가 존재하는지 확인할 때:

```
h₁(x), h₂(x), ..., hₖ(x) 위치의 비트를 모두 확인
→ 하나라도 0이면: "확실히 없다" (100% 정확)
→ 전부 1이면:     "아마 있다"   (오탐 가능)
```

"cherry"를 조회한다고 하자:

```
h₁("cherry") mod 16 = 1   → 비트[1] = 1 ✓
h₂("cherry") mod 16 = 3   → 비트[3] = 1 ✓
h₃("cherry") mod 16 = 14  → 비트[14] = 1 ✓

→ 전부 1이므로 "있을 수 있다"고 답한다.
→ 그러나 "cherry"는 삽입한 적 없다! → false positive
```

1, 3, 14 비트는 각각 "banana", "apple", "banana"가 세팅한 것이다. 다른 원소들이 세팅한 비트가 우연히 조합되어 오탐이 발생했다.

## 삭제

**기본 Bloom Filter는 삭제를 지원하지 않는다.** 비트를 0으로 돌리면 다른 원소의 비트까지 지워버리기 때문이다. 위 예시에서 "apple"을 삭제하려고 비트 7을 0으로 만들면, "banana"의 조회까지 깨진다(false negative 발생). 삭제가 필요하면 뒤에서 다루는 Counting Bloom Filter를 사용한다.

---

# 오탐률 수학

## 공식 유도

m비트 배열에 k개 해시 함수로 n개 원소를 삽입했을 때, 특정 비트가 여전히 0일 확률:

$$
P(\text{비트} = 0) = \left(1 - \frac{1}{m}\right)^{kn}
$$

$$ m $$이 충분히 크면 $$ (1 - 1/m)^m \approx e^{-1} $$이므로:

$$
P(\text{비트} = 0) \approx e^{-kn/m}
$$

따라서 특정 비트가 1일 확률:

$$
P(\text{비트} = 1) \approx 1 - e^{-kn/m}
$$

오탐(false positive)은 k개 비트가 **모두** 우연히 1인 경우이므로:

$$
\boxed{P(\text{FP}) \approx \left(1 - e^{-kn/m}\right)^k}
$$

이것이 Bloom Filter의 **오탐률 공식**이다.

## 최적의 해시 함수 개수

오탐률을 최소화하는 k를 구하면:

$$
k_{\text{opt}} = \frac{m}{n} \ln 2 \approx 0.693 \cdot \frac{m}{n}
$$

이때 최소 오탐률:

$$
P(\text{FP})_{\min} = \left(\frac{1}{2}\right)^k = (0.6185)^{m/n}
$$

## 실용 수치

| 원소 수 (n) | 원하는 오탐률 | 필요한 비트 (m) | 해시 함수 수 (k) | 메모리 |
|---|---|---|---|---|
| 100만 | 1% | 958만 비트 | 7 | **1.14 MB** |
| 100만 | 0.1% | 1437만 비트 | 10 | 1.71 MB |
| 1억 | 1% | 9.58억 비트 | 7 | **114 MB** |
| 1억 | 0.1% | 14.37억 비트 | 10 | 171 MB |

비교: HashSet에 100만 개의 문자열(평균 20바이트)을 저장하면 약 **50~80 MB** 필요하다. Bloom Filter는 오탐률 1%에 **1.14 MB**로 같은 멤버십 질의를 처리한다. **50배 이상** 절약이다.

## 비트 수 계산 공식

원하는 오탐률 p와 원소 수 n이 주어졌을 때 필요한 비트 수:

$$
m = -\frac{n \ln p}{(\ln 2)^2}
$$

---

# 해시 함수 전략

k개의 독립적인 해시 함수가 필요하지만, 실제로는 **2개의 해시 함수로 k개를 시뮬레이션**한다 (Kirsch-Mitzenmacker 기법, 2006):

$$
g_i(x) = h_1(x) + i \cdot h_2(x) \mod m \quad (i = 0, 1, ..., k-1)
$$

이 방법은 독립 해시 함수 k개를 쓰는 것과 수학적으로 동일한 오탐률을 보장한다.

```java
// Java 구현 예시
long hash1 = murmur3_128(key).asLong();
long hash2 = murmur3_128(key, seed=hash1).asLong();

for (int i = 0; i < k; i++) {
    long combinedHash = hash1 + i * hash2;
    int index = Math.floorMod(combinedHash, m);
    bitArray.set(index);
}
```

실무에서 많이 쓰는 해시 함수:
- **MurmurHash3:** Guava Bloom Filter의 기본 선택. 빠르고 분포가 균일하다.
- **xxHash:** MurmurHash보다 빠르고 품질도 좋다. 최근 추세.
- **FNV-1a:** 간단하지만 분포 품질이 조금 떨어진다.

---

# 직접 구현

```java
import java.util.BitSet;

public class BloomFilter<T> {

    private final BitSet bitArray;
    private final int bitSize;       // m
    private final int hashCount;     // k

    /**
     * @param expectedInsertions 예상 삽입 원소 수 (n)
     * @param falsePositiveRate  원하는 오탐률 (p)
     */
    public BloomFilter(int expectedInsertions, double falsePositiveRate) {
        // m = -(n * ln(p)) / (ln2)^2
        this.bitSize = optimalBitSize(expectedInsertions, falsePositiveRate);
        // k = (m/n) * ln2
        this.hashCount = optimalHashCount(expectedInsertions, bitSize);
        this.bitArray = new BitSet(bitSize);
    }

    /** 원소 삽입 */
    public void put(T element) {
        long hash64 = element.hashCode() * 0x9E3779B97F4A7C15L; // 해시 확산
        int hash1 = (int) hash64;
        int hash2 = (int) (hash64 >>> 32);

        for (int i = 0; i < hashCount; i++) {
            int index = Math.floorMod(hash1 + i * hash2, bitSize);
            bitArray.set(index);
        }
    }

    /** 원소 존재 가능성 확인 */
    public boolean mightContain(T element) {
        long hash64 = element.hashCode() * 0x9E3779B97F4A7C15L;
        int hash1 = (int) hash64;
        int hash2 = (int) (hash64 >>> 32);

        for (int i = 0; i < hashCount; i++) {
            int index = Math.floorMod(hash1 + i * hash2, bitSize);
            if (!bitArray.get(index)) {
                return false; // 하나라도 0이면 확실히 없다
            }
        }
        return true; // 전부 1이면 "아마 있다"
    }

    /** 현재 채워진 비트 비율 */
    public double fillRatio() {
        return (double) bitArray.cardinality() / bitSize;
    }

    private static int optimalBitSize(int n, double p) {
        return (int) Math.ceil(-n * Math.log(p) / (Math.log(2) * Math.log(2)));
    }

    private static int optimalHashCount(int n, int m) {
        return Math.max(1, (int) Math.round((double) m / n * Math.log(2)));
    }
}
```

```java
// 사용 예시
BloomFilter<String> filter = new BloomFilter<>(1_000_000, 0.01); // 100만 개, 오탐률 1%

filter.put("apple");
filter.put("banana");

filter.mightContain("apple");   // true (정확)
filter.mightContain("cherry");  // false (정확) 또는 true (오탐, 1% 확률)
```

---

# 변형들

기본 Bloom Filter의 한계를 극복하기 위해 여러 변형이 존재한다.

## Counting Bloom Filter

**문제:** 기본 Bloom Filter는 삭제를 지원하지 않는다.

**해결:** 각 비트를 **카운터**(보통 4비트)로 교체한다.

```
기본:     [0] [1] [0] [1] [1] [0]     ← 비트
Counting: [0] [2] [0] [1] [3] [0]     ← 카운터
```

- 삽입: 해당 위치 카운터 +1
- 삭제: 해당 위치 카운터 -1
- 조회: 해당 위치 카운터가 모두 > 0이면 "아마 있다"

**대가:** 메모리가 4배(4비트 카운터) 증가한다. 카운터 오버플로우 문제도 있다.

## Cuckoo Filter

Fan et al. (2014)이 제안한 **실용적 대안**이다.

```
Cuckoo Hash Table + Fingerprint 저장

버킷 0: [fp_a] [fp_b]
버킷 1: [fp_c] [    ]
버킷 2: [fp_d] [fp_e]
버킷 3: [    ] [    ]
```

- 원소의 fingerprint(짧은 해시)를 Cuckoo Hash Table에 저장
- **삭제 지원**: fingerprint를 제거하면 된다
- **공간 효율**: 오탐률 3% 이하에서 Bloom Filter보다 공간 효율적
- **조회 속도**: 최대 2번의 메모리 접근 (Bloom Filter는 k번)

| 비교 | Bloom Filter | Cuckoo Filter |
|------|-------------|---------------|
| 삭제 | 불가 | 가능 |
| 오탐률 < 3% | 더 효율적 | 덜 효율적 |
| 오탐률 > 3% | 덜 효율적 | **더 효율적** |
| 조회 속도 | k번 메모리 접근 | 최대 2번 |

## Scalable Bloom Filter

**문제:** 삽입 원소 수 n을 미리 알아야 한다.

**해결:** 필터가 가득 차면 새 필터를 추가하여 **체이닝**한다. 각 후속 필터는 더 낮은 오탐률을 사용하여 전체 오탐률이 원하는 수준을 유지하도록 한다.

```
Filter₁ (FP: 0.5%)  →  Filter₂ (FP: 0.25%)  →  Filter₃ (FP: 0.125%)
      가득 참                  가득 참                 아직 여유
```

조회 시 모든 필터를 확인한다. 하나라도 "있다"고 하면 "있다"로 판정한다.

## 그 외 변형

- **Partitioned Bloom Filter:** 비트 배열을 k개 파티션으로 분할. 각 해시 함수가 자기 파티션만 사용. CPU 캐시 친화적.
- **Compressed Bloom Filter:** 네트워크 전송 시 비트 배열을 압축. 채워진 비트 비율이 50%일 때 최적 (≈ 최대 엔트로피).
- **Quotient Filter:** 삭제 지원 + 디스크 친화적. Merge 가능.

---

# 실전 적용

## 1. DB 쿼리 최적화 — 존재하지 않는 키 필터링

가장 고전적인 사용 사례다. LevelDB, RocksDB, HBase, Cassandra가 사용한다.

```
클라이언트 → GET key_123
                    ↓
           Bloom Filter 확인
                    ↓
        "확실히 없다" → 바로 NOT FOUND 반환 (디스크 I/O 0)
        "있을 수 있다" → 디스크에서 실제 조회
```

LSM-Tree 기반 DB에서 SSTable마다 Bloom Filter를 유지한다. 키가 어느 SSTable에도 없으면 디스크를 읽지 않는다. 이것만으로도 읽기 성능이 수십 배 개선된다.

## 2. 캐시 침투(Cache Penetration) 방어

```
공격: 존재하지 않는 키를 대량으로 조회 → 캐시 미스 → DB 직접 공격

방어:
요청 → Bloom Filter → "이 키는 DB에 존재하지 않는다" → 즉시 거부
                     → "존재할 수 있다" → 캐시 확인 → DB 조회
```

DB에 존재하는 키를 Bloom Filter에 미리 로드해두면, 존재하지 않는 키에 대한 DB 접근을 원천 차단한다.

## 3. 웹 크롤러 URL 중복 제거

```java
BloomFilter<String> visited = BloomFilter.create(
    Funnels.stringFunnel(UTF_8), 
    10_000_000,  // 예상 URL 수
    0.001        // 오탐률 0.1%
);

while (!queue.isEmpty()) {
    String url = queue.poll();
    if (visited.mightContain(url)) continue; // 이미 방문했을 가능성
    
    visited.put(url);
    crawl(url);
}
```

HashSet으로 1000만 URL을 저장하면 ~500 MB. Bloom Filter는 ~17 MB로 같은 일을 한다.

## 4. Redis Bloom Filter

Redis 4.0+에서 RedisBloom 모듈로 Bloom Filter를 네이티브 지원한다.

```bash
# 필터 생성: 100만 원소, 오탐률 0.01
BF.RESERVE username_filter 0.01 1000000

# 삽입
BF.ADD username_filter "john_doe"
BF.ADD username_filter "jane_doe"

# 조회
BF.EXISTS username_filter "john_doe"    # (integer) 1
BF.EXISTS username_filter "unknown"     # (integer) 0 → 확실히 없다

# 여러 개 한번에
BF.MADD username_filter "user1" "user2" "user3"
BF.MEXISTS username_filter "user1" "user4"
```

**Spring Boot에서 Redis Bloom Filter 사용:**

```java
@Service
public class UsernameService {
    
    private final StringRedisTemplate redis;
    
    public boolean isUsernameTaken(String username) {
        // BF.EXISTS 호출
        Object result = redis.execute((RedisCallback<Object>) connection ->
            connection.execute("BF.EXISTS", 
                "username_filter".getBytes(), 
                username.getBytes()));
        
        if (Long.valueOf(0).equals(result)) {
            return false; // 확실히 사용 가능
        }
        // Bloom Filter가 "있다"고 하면 DB에서 재확인 (오탐 가능)
        return userRepository.existsByUsername(username);
    }
}
```

## 5. Google Guava Bloom Filter

```java
import com.google.common.hash.BloomFilter;
import com.google.common.hash.Funnels;

// 생성: 100만 개, 오탐률 3%
BloomFilter<String> filter = BloomFilter.create(
    Funnels.stringFunnel(Charsets.UTF_8),
    1_000_000,
    0.03
);

filter.put("hello");
filter.mightContain("hello");  // true
filter.mightContain("world");  // false (대부분), true (3% 확률)

// 실제 오탐률 확인
filter.expectedFpp();  // 현재 삽입된 원소 기준 예상 오탐률
```

## 6. 그 외 실전 사례

| 사용처 | 목적 |
|--------|------|
| **Chrome 브라우저** | 악성 URL 목록(Safe Browsing). 로컬 Bloom Filter로 1차 필터링 후, 의심되면 Google 서버에 확인 |
| **Medium** | 추천 피드에서 이미 읽은 글 필터링 |
| **Bitcoin** | SPV 노드가 관심 있는 트랜잭션만 필터링 (BIP 37) |
| **Akamai CDN** | "한 번만 요청된 URL"을 캐시하지 않음. 두 번째 요청부터 캐시 |
| **Squid 프록시** | 캐시 다이제스트 교환으로 인접 프록시에 캐시 여부 확인 |

---

# Bloom Filter vs 대안

| | Bloom Filter | HashSet | Cuckoo Filter | Quotient Filter |
|---|---|---|---|---|
| 메모리 | **최소** | 최대 | 중간 | 중간 |
| 오탐 | 있음 | **없음** | 있음 | 있음 |
| 삭제 | 불가 | 가능 | **가능** | 가능 |
| 디스크 친화 | 보통 | 나쁨 | 보통 | **좋음** |
| 구현 복잡도 | 매우 낮음 | 낮음 | 중간 | 높음 |

**결정 기준:**
- 삭제 불필요 + 메모리 최소화 → **Bloom Filter**
- 삭제 필요 + 오탐률 > 3% → **Cuckoo Filter**
- 정확성 필수 → **HashSet**
- 디스크 기반 + Merge 필요 → **Quotient Filter**

---

# 주의사항

1. **n을 과소 추정하면 안 된다.** 예상보다 많은 원소가 들어오면 오탐률이 급격히 증가한다. 비트 배열의 대부분이 1로 채워지면 모든 조회에 "있다"고 답하게 된다.

2. **해시 함수의 독립성이 중요하다.** 해시 함수 간 상관관계가 있으면 이론적 오탐률보다 높아진다. MurmurHash3 + Kirsch-Mitzenmacker 기법이면 충분하다.

3. **직렬화/역직렬화를 고려하라.** Bloom Filter를 파일이나 네트워크로 전송해야 할 때, BitSet의 직렬화 크기를 확인하자. 비트 채움률이 50%에 가까울 때 압축 효율이 가장 나쁘다.

4. **멀티스레드 환경에서는 동기화가 필요하다.** BitSet은 스레드 안전하지 않다. `AtomicLongArray`로 구현하거나 동시 쓰기를 직렬화해야 한다. 읽기만 하는 경우에는 불변 Bloom Filter를 사용하면 된다.

---

# 정리

Bloom Filter는 **"없다"를 확실히 말할 수 있는 능력**이 핵심이다. 이 한 가지 보장만으로도 디스크 I/O 절감, 네트워크 요청 감소, 캐시 침투 방어 같은 실질적 성능 개선을 만들어낸다.

| 기억할 것 | |
|---|---|
| 본질 | 확률적 집합 멤버십 테스트 |
| 보장 | false negative는 **절대 없다** |
| 대가 | false positive가 **가끔 있다** |
| 메모리 | HashSet 대비 **수십 배** 절약 |
| 최적 k | $$ 0.693 \times m/n $$ |
| 오탐률 | $$ (1 - e^{-kn/m})^k $$ |
| 삭제 필요 시 | Counting Bloom Filter 또는 Cuckoo Filter |

단순한 비트 배열과 해시 함수의 조합이 이렇게 강력한 도구가 된다는 것 — 자료구조의 매력은 여기에 있다.
