---
layout: post
title: "랜덤 포레스트 공부: 배깅, Bias-Variance, OOB, 특성 중요도까지"
date: 2026-01-18
tags: [machine-learning, random-forest, ensemble, bagging]
permalink: /random-forest/
excerpt: "랜덤 포레스트는 부트스트랩으로 다양한 트리를 만들고 다수결로 예측한다. Bias-Variance 분해, OOB 검증, 특성 중요도를 깊이 다룬다."
---

이전 글: [의사결정 트리](/decision-tree/)

- **Level 1** — 랜덤 포레스트의 핵심 원리. 배깅과 부트스트랩, 그리고 왜 트리를 여러 개 쓰면 좋아지는지를 Bias-Variance 분해로 이해한다.
- **Level 2** — OOB 검증과 특성 중요도. 별도 검증 데이터 없이 성능을 평가하고, 어떤 특성이 중요한지 두 가지 방법으로 측정한다.
- **Level 3** — 하이퍼파라미터 튜닝, 부스팅과의 비교, 실전 scikit-learn 코드까지.

---

# Level 1: 배깅과 Bias-Variance

## 트리 하나의 문제: 높은 Variance

[의사결정 트리](/decision-tree/) 글에서 다뤘듯, 깊은 트리 하나는 학습 데이터를 거의 외운다. 문제는 데이터가 조금만 바뀌어도 완전히 다른 트리가 만들어진다는 것이다.

```
데이터 A로 학습:           데이터 B로 학습 (1건만 다름):
      [날씨?]                    [습도?]
     /  |  \                    /     \
  맑음 흐림  비              높음    보통
   ...                        ...

→ 첫 분할 기준부터 달라진다.
→ 이것이 높은 Variance다.
```

Bias는 낮다(복잡한 패턴도 잡을 수 있다). 하지만 Variance가 너무 높아서 새 데이터에 일반화되지 않는다. 이 문제를 해결하는 것이 **배깅(Bagging)**이다.

## Bias-Variance 분해

모델의 기대 오차는 세 가지로 분해된다:

$$\text{E}[\text{오차}] = \text{Bias}^2 + \text{Variance} + \text{Noise}$$

```
Bias (편향):  모델이 구조적으로 놓치는 오차.
              너무 단순한 모델 → 높은 Bias → 과소적합

Variance (분산): 학습 데이터에 따라 예측이 흔들리는 정도.
                 너무 복잡한 모델 → 높은 Variance → 과적합

Noise: 데이터 자체에 내재된 불가피한 오차. 줄일 수 없다.
```

깊은 트리 하나는 Bias가 낮고 Variance가 높다. 배깅의 목표는 **Bias를 낮게 유지하면서 Variance만 줄이는 것**이다.

## 평균의 힘: 독립인 경우

트리 하나의 예측을 $f_i(x)$라 하자. 각 트리의 분산이 $\sigma^2$이고, $n$개의 트리가 **완전히 독립**이라면:

$$\bar{f}(x) = \frac{1}{n}\sum_{i=1}^{n} f_i(x)$$

$$\text{Var}(\bar{f}) = \frac{\sigma^2}{n}$$

트리 100개를 평균 내면 분산이 1/100로 줄어든다. 하지만 현실에서 트리들은 같은 데이터에서 학습하므로 완전히 독립이 아니다.

## 상관계수 rho를 포함한 Variance 공식

$n$개의 트리가 있고, 각 트리의 분산이 $\sigma^2$, 트리 쌍 사이의 평균 상관계수가 $\rho$일 때:

$$\text{Var}(\bar{f}) = \rho \sigma^2 + \frac{1-\rho}{n}\sigma^2$$

이 공식이 랜덤 포레스트의 핵심을 설명한다:

```
두 항을 분리해서 보자:

  항 1: ρσ²           → n을 아무리 키워도 사라지지 않는다
  항 2: (1-ρ)/n · σ²  → n이 커지면 0에 수렴한다

n → ∞ 이면:
  Var(f̄) → ρσ²

즉 트리 수를 아무리 늘려도 ρσ²보다 낮아질 수 없다.
→ ρ를 줄이는 것이 핵심이다!
```

**랜덤 포레스트가 각 분할에서 특성을 랜덤으로 뽑는 이유가 바로 여기에 있다.** 모든 특성을 다 쓰면 트리들이 비슷해지고(높은 $\rho$), 특성을 랜덤으로 제한하면 트리들이 다양해진다(낮은 $\rho$). $\rho$가 낮아지면 항 1이 줄어들어 분산 감소 효과가 극대화된다.

## 배깅 (Bagging = Bootstrap Aggregating)

배깅은 두 단계다:

**1. Bootstrap (복원 추출):** 원본 데이터 $n$개에서 $n$개를 **복원 추출**하여 새로운 학습 데이터를 만든다. 같은 샘플이 여러 번 뽑힐 수 있고, 한 번도 안 뽑힐 수도 있다.

**2. Aggregating (집계):** 각 부트스트랩 샘플로 모델을 학습하고, 예측을 모아서 합친다.

```
원본 데이터: [A, B, C, D, E]

부트스트랩 1: [A, A, C, D, E]  → 트리 1 학습
부트스트랩 2: [B, C, C, D, E]  → 트리 2 학습
부트스트랩 3: [A, B, D, D, E]  → 트리 3 학습
...

예측 시:
  분류 → 다수결 (Majority Voting)
  회귀 → 평균 (Average)
```

## 부트스트랩의 수학

$n$개 데이터에서 $n$번 복원 추출할 때, 특정 샘플 하나가 **한 번도 뽑히지 않을** 확률:

$$P(\text{안 뽑힘}) = \left(1 - \frac{1}{n}\right)^n$$

$n$이 충분히 크면:

$$\left(1 - \frac{1}{n}\right)^n \approx e^{-1} \approx 0.368$$

```
검증:
  n = 10:   (1 - 1/10)^10  = 0.9^10  = 0.349
  n = 100:  (1 - 1/100)^100          = 0.366
  n = 1000: (1 - 1/1000)^1000        = 0.368
  n → ∞:                             → e^{-1} = 0.3679...

결론: 각 부트스트랩 샘플에는 원본의 약 63.2%가 포함된다.
      나머지 약 36.8%는 빠진다 → 이것이 OOB(Out-Of-Bag) 데이터.
```

## 랜덤 포레스트 알고리즘 전체 과정

배깅 + **랜덤 특성 선택** = 랜덤 포레스트. Leo Breiman이 2001년에 제안했다.

```
입력: 학습 데이터 D (n개 샘플, p개 특성), 트리 수 B

for b = 1 to B:
  1. D에서 n개를 복원 추출 → 부트스트랩 샘플 D_b
     (원본의 약 63.2% 포함, 36.8%는 OOB)

  2. D_b로 의사결정 트리 T_b를 학습:
     각 노드에서:
       a. p개 특성 중 m개를 무작위로 선택
          분류: m = √p (기본값)
          회귀: m = p/3 (기본값)
       b. 선택된 m개 중에서 최적 분할 기준 선택
          (정보 이득 또는 지니 감소가 가장 큰 것)
       c. 두 자식 노드로 분할
     가지치기 없이 끝까지 키운다 (각 리프가 순수하거나 최소 샘플에 도달할 때까지)

  3. 트리 T_b 저장

예측 시:
  분류: T₁(x), T₂(x), ..., T_B(x) 중 다수결
  회귀: T₁(x), T₂(x), ..., T_B(x)의 평균
```

## 랜덤 특성 선택이 rho를 낮추는 이유

왜 특성을 제한하면 트리 사이의 상관계수 $\rho$가 낮아지는가?

```
p개 특성 중 "소득"이 압도적으로 강한 특성이라 하자.

모든 특성을 다 쓰면 (일반 배깅):
  트리 1: 첫 분할 = 소득
  트리 2: 첫 분할 = 소득
  트리 3: 첫 분할 = 소득
  ...
  → 모든 트리가 비슷한 구조 → ρ가 높다

랜덤으로 √p개만 쓰면 (랜덤 포레스트):
  트리 1: 후보 = {소득, 나이, 학력}        → 소득으로 분할
  트리 2: 후보 = {부채, 지역, 결혼}        → 부채로 분할 (소득 없음!)
  트리 3: 후보 = {나이, 학력, 직업}        → 나이로 분할 (소득 없음!)
  트리 4: 후보 = {소득, 지역, 직업}        → 소득으로 분할
  ...
  → 트리마다 다른 구조 → ρ가 낮아진다
  → Var(f̄) = ρσ² + (1-ρ)/n · σ² 에서 ρ 감소 → 분산 크게 감소
```

이것이 배깅과 랜덤 포레스트의 차이다. 배깅은 부트스트랩만 하지만, 랜덤 포레스트는 특성도 랜덤으로 뽑아 $\rho$를 적극적으로 줄인다.

---

# Level 2: OOB 검증과 특성 중요도

## OOB (Out-Of-Bag) 검증

부트스트랩에서 약 36.8%의 데이터가 빠진다고 했다. 이 빠진 데이터를 **OOB(Out-Of-Bag) 데이터**라 부른다. 이것을 검증에 활용하면 별도의 검증 데이터 없이 성능을 평가할 수 있다.

### OOB 검증의 절차

```
트리 1:  학습 데이터 = {A, A, C, D, E}  → OOB = {B}
트리 2:  학습 데이터 = {B, C, C, D, E}  → OOB = {A}
트리 3:  학습 데이터 = {A, B, D, D, E}  → OOB = {C}
트리 4:  학습 데이터 = {A, C, E, E, B}  → OOB = {D}
트리 5:  학습 데이터 = {B, B, C, D, A}  → OOB = {E}

OOB 예측:
  샘플 A → 자기가 빠진 트리들(트리 2)로만 예측
  샘플 B → 자기가 빠진 트리들(트리 1)로만 예측
  샘플 C → 자기가 빠진 트리들(트리 3)로만 예측
  ...

트리가 500개면:
  샘플 A가 빠진 트리는 약 500 × 0.368 = 184개
  → 184개 트리의 다수결로 A의 OOB 예측을 구한다
  → 이 예측과 실제 레이블을 비교

모든 샘플에 대해 OOB 예측을 모으면 → OOB 정확도(또는 OOB 오차)
```

### OOB vs 교차 검증 (Cross-Validation)

```
교차 검증 (5-Fold CV):
  데이터를 5등분
  각 폴드마다:
    4/5로 학습, 1/5로 검증
  → 모델을 5번 학습해야 한다
  → 랜덤 포레스트 500개 트리 × 5폴드 = 트리 2500개 학습

OOB 검증:
  학습 과정에서 자동으로 계산
  → 추가 학습 비용 없음
  → 트리 500개만 학습하면 된다

OOB 오차 ≈ Leave-One-Out Cross-Validation 오차
  → 이론적으로 거의 동일한 추정량
  → 데이터가 부족할 때 특히 유용
```

### OOB 검증의 장단점

```
장점:
  1. 별도 검증 데이터가 필요 없다 → 모든 데이터를 학습에 쓸 수 있다
  2. 계산 비용이 거의 없다 → 학습 과정의 부산물
  3. 과적합 모니터링에 유용 → n_estimators를 늘릴수록 OOB 오차 추적
  4. 이론적으로 LOOCV와 유사한 추정

단점:
  1. 랜덤 포레스트(배깅 기반)에서만 사용 가능
  2. 부트스트랩 특성상 약간의 편향이 있을 수 있다
  3. 데이터가 매우 적을 때(n < 50)는 불안정할 수 있다
```

## 특성 중요도: MDI vs Permutation

랜덤 포레스트의 큰 장점 중 하나는 **어떤 특성이 예측에 중요한지** 알 수 있다는 것이다. 두 가지 방법이 있다.

### 방법 1: 불순도 기반 중요도 (MDI, Mean Decrease in Impurity)

각 트리에서 각 특성이 분할에 사용될 때 **지니 불순도(또는 엔트로피)를 얼마나 감소시켰는지** 합산한다.

```
트리 1:
  노드 A: "소득"으로 분할 → 지니 감소 = 0.15
  노드 B: "나이"로 분할   → 지니 감소 = 0.08
  노드 C: "소득"으로 분할 → 지니 감소 = 0.05

트리 2:
  노드 A: "부채"로 분할   → 지니 감소 = 0.12
  노드 B: "소득"으로 분할 → 지니 감소 = 0.10

모든 트리에서 합산 후 정규화:
  소득: (0.15 + 0.05 + 0.10) / 전체 = 0.30 / 0.50 = 0.60
  부채: 0.12 / 0.50 = 0.24
  나이: 0.08 / 0.50 = 0.16

특성        MDI 중요도    시각화
─────────────────────────────────
소득         0.60        ████████████████████
부채         0.24        ████████
나이         0.16        █████
```

**MDI의 장점과 단점:**

```
장점:
  - 빠르다: 학습 과정에서 자동으로 계산된다 (추가 비용 없음)
  - scikit-learn의 feature_importances_ 속성으로 바로 접근 가능

단점:
  - 높은 카디널리티(고유값이 많은) 특성에 편향된다
    예: "고객ID"(10만개 고유값) vs "성별"(2개 고유값)
    → 고유값이 많으면 분할 기회가 많아서 중요도가 과대평가됨
    → "고객ID"가 "소득"보다 중요하게 나올 수 있다 (과적합 신호!)
  - 학습 데이터에 대한 중요도이므로 과적합된 특성도 높게 나온다
  - 상관된 특성이 있으면 중요도가 분산된다
    예: "소득"과 "자산"이 높은 상관 → 둘 다 중간 정도로 나옴
```

### 방법 2: 순열 중요도 (Permutation Importance)

특성 하나의 값을 **무작위로 섞은 뒤** 성능이 얼마나 떨어지는지를 측정한다.

```
절차:
  1. OOB 데이터(또는 검증 데이터)로 기본 성능 계산
     기본 정확도 = 85%

  2. 특성 "소득"의 값을 데이터 내에서 무작위로 섞는다 (Permutation)
     → "소득"과 레이블 사이의 관계가 파괴된다
     → 다시 예측 → 정확도 = 62%
     → 23% 하락! → "소득"은 매우 중요

  3. "소득"을 원래대로 복원하고, "지역"을 섞는다
     → 다시 예측 → 정확도 = 84%
     → 1% 하락 → "지역"은 덜 중요

  4. 모든 특성에 대해 반복

특성        정확도 하락    시각화
─────────────────────────────────
소득         23%          ████████████████████████
부채         12%          ████████████
나이          8%          ████████
학력          4%          ████
지역          1%          █
결혼          0%
```

**Permutation Importance의 장점과 단점:**

```
장점:
  - 카디널리티 편향이 없다
  - 검증 데이터에서 측정하면 과적합된 특성을 걸러낸다
  - 모델에 독립적이다 (RF뿐 아니라 어떤 모델이든 적용 가능)

단점:
  - 느리다: 특성마다 예측을 다시 해야 한다 (특성 p개 × 반복 횟수)
  - 상관된 특성 문제는 여전하다:
    "소득"을 섞어도 상관된 "자산"이 정보를 제공 → 중요도 과소평가
  - 랜덤성이 있으므로 여러 번 반복하여 평균을 구해야 안정적
```

### MDI vs Permutation: 어떤 것을 쓸 것인가

```
빠른 탐색 단계:
  MDI를 쓴다. 학습과 동시에 계산되므로 비용이 없다.
  단, 카디널리티가 높은 특성이 있으면 결과를 의심한다.

최종 보고 / 모델 해석 단계:
  Permutation Importance를 쓴다. 검증 데이터에서 측정.
  더 신뢰할 수 있는 결과.

둘 다 한계가 있는 상황:
  상관된 특성이 많으면 SHAP을 고려한다.
  SHAP은 게임이론 기반으로 특성 기여도를 더 정밀하게 분해한다.
```

---

# Level 3: 하이퍼파라미터, 부스팅 비교, 실전 코드

## 핵심 하이퍼파라미터 튜닝 가이드

### n_estimators (트리 수)

```
트리 수와 성능:

OOB 오차
  0.30 ─ ●
         |  \
  0.25 ─     \
         |     \
  0.20 ─       \
         |       \──────────────────  ← 수렴 (더 늘려도 개선 미미)
  0.15 ─
         |
         ├──┬──┬──┬──┬──┬──┬──┬──→ 트리 수
        50 100 200 300 500 700 1000

권장:
  - 시작: 100~500
  - 트리를 늘린다고 과적합이 생기지는 않는다 (배깅의 성질)
  - 다만 학습 시간과 메모리가 선형으로 증가한다
  - OOB 오차가 수렴하면 그 지점에서 멈춘다
  - 실전에서는 500이면 대부분 충분하다
```

### max_features (분할 시 후보 특성 수)

```
이 파라미터가 ρ(트리 간 상관계수)를 직접 제어한다.

max_features 크면:
  → 각 트리가 강한 특성을 잘 활용 → 개별 트리 성능 높음
  → 하지만 트리들이 비슷해짐 → ρ 높음 → 앙상블 효과 감소

max_features 작으면:
  → 각 트리가 약한 특성에 의존할 수 있음 → 개별 트리 성능 낮음
  → 하지만 트리들이 다양해짐 → ρ 낮음 → 앙상블 효과 증가

기본값:
  분류: "sqrt" (√p)  — p가 100이면 10개씩 후보
  회귀: 통상 권장값 p/3
        ⚠ sklearn ≥1.1의 RandomForestRegressor.max_features 기본값은 1.0(전체)으로 변경.
           예전 기본값 "auto"(=p)와 사실상 동일. 분산 감소 효과를 보려면 명시적으로 p/3 또는 "sqrt"로 설정.

튜닝 범위: [sqrt(p), log2(p), 0.3p, 0.5p, p]
  → p가 아닌 값부터 시도, p는 일반 배깅과 동일
```

### max_depth (최대 깊이)

```
기본값: None (제한 없이 끝까지 키움)

랜덤 포레스트에서는 보통 제한하지 않는다.
  → 배깅이 Variance를 줄여주므로 각 트리를 깊게 키워도 괜찮다
  → 깊은 트리 = 낮은 Bias, 배깅 = 낮은 Variance → 둘 다 확보

제한이 필요한 경우:
  - 메모리 부족
  - 추론 속도가 중요한 경우
  - 데이터가 매우 적을 때 (과적합 방지)

튜닝 범위: [None, 10, 15, 20, 30]
```

### min_samples_leaf (리프 노드 최소 샘플 수)

```
기본값: 1

min_samples_leaf를 높이면:
  → 리프 노드가 적어도 그만큼의 샘플을 가져야 함
  → 트리가 덜 깊어짐 → 개별 트리의 Variance 감소
  → 약간의 Bias 증가를 대가로 더 안정적

분류: 1~5
회귀: 5~20 (회귀에서는 리프의 평균이므로 샘플이 많을수록 안정)

노이즈가 많은 데이터에서 특히 효과적이다.
```

### 하이퍼파라미터 요약표

```
파라미터            기본값      튜닝 범위          영향
─────────────────────────────────────────────────────────────
n_estimators       100        100~1000          많을수록 안정 (과적합 없음)
max_features       "sqrt"     sqrt, log2, 0.3~1 ρ를 제어 (핵심!)
max_depth          None       None, 10~30       보통 None이 좋음
min_samples_leaf   1          1~20              노이즈 데이터에서 높임
min_samples_split  2          2~20              leaf와 유사한 효과
bootstrap          True       True/False        False면 일반 배깅 아님
n_jobs             1          -1 (전체 코어)     병렬화 (속도)
oob_score          False      True              OOB 점수 계산
```

## 랜덤 포레스트 vs 부스팅: 언제 무엇을 쓸까

```
랜덤 포레스트가 더 나은 경우:
  1. 빠른 프로토타이핑 / 베이스라인
     → 튜닝 없이도 꽤 좋은 성능
     → 하이퍼파라미터에 둔감하다

  2. 과적합이 걱정될 때
     → 트리를 아무리 많이 넣어도 과적합이 생기지 않는다
     → 부스팅은 트리를 너무 많이 넣으면 과적합

  3. 병렬 학습이 필요할 때
     → 각 트리가 독립 → 완벽한 병렬화 (n_jobs=-1)
     → 부스팅은 순차적 → 병렬화 어려움

  4. 노이즈가 많은 데이터
     → 부스팅은 노이즈까지 순차적으로 맞추려 한다
     → RF는 평균 내면서 노이즈가 상쇄된다

  5. 특성 중요도가 필요할 때
     → OOB + Permutation Importance가 자연스럽게 지원

부스팅(XGBoost/LightGBM)이 더 나은 경우:
  1. 최대 성능이 필요할 때
     → Bias도 줄이므로 보통 RF보다 정확도가 높다
     → Kaggle 대회에서 거의 항상 부스팅이 이긴다

  2. 데이터가 충분히 크고 깨끗할 때
     → 노이즈가 적으면 부스팅의 과적합 위험이 낮다

  3. 학습 데이터와 테스트 데이터의 분포가 비슷할 때
     → 분포 변화(distribution shift)가 있으면 부스팅이 취약

  4. 정밀한 튜닝에 시간을 투자할 수 있을 때
     → 부스팅은 learning_rate, n_estimators, max_depth 등
        파라미터 조합에 민감
```

```
실전 워크플로:

1단계: 랜덤 포레스트로 베이스라인
  → 튜닝 거의 없이 빠르게 성능 확인
  → 특성 중요도로 데이터 이해
  → OOB 점수로 검증

2단계: LightGBM/XGBoost로 성능 끌어올리기
  → Optuna 등으로 하이퍼파라미터 탐색
  → RF 대비 2~5% 정확도 향상이 일반적

3단계: (필요시) 앙상블
  → RF + LightGBM 예측을 평균/스태킹
```

## 실전 scikit-learn 코드

### 기본 분류

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, classification_report

# 데이터 준비
data = load_breast_cancer()
X_train, X_test, y_train, y_test = train_test_split(
    data.data, data.target, test_size=0.2, random_state=42
)

# 랜덤 포레스트 학습
rf = RandomForestClassifier(
    n_estimators=500,       # 트리 500개
    max_features='sqrt',    # 분류 기본값
    max_depth=None,         # 끝까지 키움
    min_samples_leaf=1,     # 기본값
    oob_score=True,         # OOB 점수 계산
    n_jobs=-1,              # 모든 코어 사용
    random_state=42
)
rf.fit(X_train, y_train)

# OOB 점수 (별도 검증 데이터 없이!)
print(f"OOB 정확도: {rf.oob_score_:.4f}")

# 테스트 정확도
y_pred = rf.predict(X_test)
print(f"테스트 정확도: {accuracy_score(y_test, y_pred):.4f}")
print(classification_report(y_test, y_pred, target_names=data.target_names))
```

### 특성 중요도 비교 (MDI vs Permutation)

```python
import numpy as np
from sklearn.inspection import permutation_importance

# 방법 1: MDI (불순도 기반)
mdi_importance = rf.feature_importances_

# 방법 2: Permutation Importance (검증 데이터 기반)
perm_result = permutation_importance(
    rf, X_test, y_test,
    n_repeats=30,           # 30번 반복하여 안정적 추정
    random_state=42,
    n_jobs=-1
)

# 상위 10개 특성 비교
feature_names = data.feature_names

print("=== MDI (불순도 기반) 상위 10 ===")
mdi_sorted = np.argsort(mdi_importance)[::-1][:10]
for idx in mdi_sorted:
    print(f"  {feature_names[idx]:25s} {mdi_importance[idx]:.4f}")

print("\n=== Permutation (순열 기반) 상위 10 ===")
perm_sorted = np.argsort(perm_result.importances_mean)[::-1][:10]
for idx in perm_sorted:
    print(f"  {feature_names[idx]:25s} "
          f"{perm_result.importances_mean[idx]:.4f} "
          f"+/- {perm_result.importances_std[idx]:.4f}")
```

### OOB 오차로 n_estimators 결정

```python
import matplotlib.pyplot as plt
from sklearn.ensemble import RandomForestClassifier

oob_errors = []
n_range = range(50, 1001, 50)

for n in n_range:
    rf_temp = RandomForestClassifier(
        n_estimators=n,
        oob_score=True,
        n_jobs=-1,
        random_state=42
    )
    rf_temp.fit(X_train, y_train)
    oob_errors.append(1 - rf_temp.oob_score_)

plt.figure(figsize=(8, 5))
plt.plot(list(n_range), oob_errors, marker='o', markersize=3)
plt.xlabel('n_estimators')
plt.ylabel('OOB Error')
plt.title('OOB Error vs Number of Trees')
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
# → 오차가 수렴하는 지점을 찾아 n_estimators로 설정
```

### 하이퍼파라미터 튜닝 (GridSearchCV)

```python
from sklearn.model_selection import GridSearchCV

param_grid = {
    'n_estimators': [200, 500],
    'max_features': ['sqrt', 'log2', 0.3],
    'max_depth': [None, 15, 25],
    'min_samples_leaf': [1, 3, 5],
}

grid_search = GridSearchCV(
    RandomForestClassifier(random_state=42, n_jobs=-1),
    param_grid,
    cv=5,
    scoring='accuracy',
    n_jobs=-1,
    verbose=1
)
grid_search.fit(X_train, y_train)

print(f"최적 파라미터: {grid_search.best_params_}")
print(f"최적 CV 정확도: {grid_search.best_score_:.4f}")
print(f"테스트 정확도: {grid_search.score(X_test, y_test):.4f}")
```

---

## 이 글에서 가져가야 할 것

```
Level 1:
  Var(f̄) = ρσ² + (1-ρ)/n · σ²
  → 트리 수 n을 늘리면 두 번째 항이 줄어든다
  → 특성을 랜덤으로 뽑으면 ρ가 줄어 첫 번째 항도 줄어든다
  → 부트스트랩에서 약 36.8%가 빠진다: (1-1/n)^n ≈ e^{-1}

Level 2:
  OOB 데이터로 교차 검증 없이 성능을 평가할 수 있다
  MDI는 빠르지만 카디널리티 편향이 있다
  Permutation Importance는 느리지만 더 신뢰할 수 있다

Level 3:
  핵심 튜닝 파라미터는 max_features (ρ 제어)
  RF는 튜닝 없이도 좋고, 과적합이 적고, 병렬화가 쉽다
  최대 성능이 필요하면 부스팅으로 넘어간다
```

다음 글: [그래디언트 부스팅과 XGBoost](/gradient-boosting-xgboost/)
