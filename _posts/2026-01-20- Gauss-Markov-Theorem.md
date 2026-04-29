---
title: 가우스-마르코프 정리 — OLS가 최적인 조건
date: 2026-01-20
tags: [statistics, machine-learning, regression]
permalink: /gauss-markov-theorem/
excerpt: "가우스-마르코프 정리는 특정 가정 하에서 OLS 추정량이 선형 비편향 추정량 중 분산이 가장 작은 BLUE임을 보장한다. 데이터로 파라미터를 추정하는 것 자체가 머신러닝이다."
---

## 선형 회귀 모델

선형 회귀는 종속 변수 $$y$$를 독립 변수 $$\mathbf{x}$$의 선형 결합으로 모델링한다.

$$
y_i = \beta_0 + \beta_1 x_{i1} + \beta_2 x_{i2} + \cdots + \beta_k x_{ik} + \epsilon_i
$$

행렬 표기로 쓰면

$$
\mathbf{y} = \mathbf{X}\boldsymbol{\beta} + \boldsymbol{\epsilon}
$$

여기서 $$\boldsymbol{\beta}$$는 우리가 추정해야 할 파라미터 벡터이고, $$\boldsymbol{\epsilon}$$은 오차항이다. 정규분포를 가정하면 파라미터는 $$\mu$$와 $$\sigma^2$$로 요약되는데, 이 파라미터를 데이터에서 추정하는 과정 자체가 하나의 머신러닝 모델이다.

## OLS 추정량

OLS(Ordinary Least Squares)는 잔차 제곱합을 최소화하는 $$\boldsymbol{\beta}$$를 구한다.

$$
\hat{\boldsymbol{\beta}} = \arg\min_{\boldsymbol{\beta}} \sum_{i=1}^{n} (y_i - \mathbf{x}_i^T \boldsymbol{\beta})^2
$$

이를 풀면 닫힌 형태의 해를 얻는다.

$$
\hat{\boldsymbol{\beta}} = (\mathbf{X}^T \mathbf{X})^{-1} \mathbf{X}^T \mathbf{y}
$$

여기서 sample은 training data에 대응하고, 추정(estimation)은 training 과정에 대응한다. 데이터로부터 파라미터를 학습한다는 점에서 통계적 추정과 머신러닝은 본질적으로 같은 작업이다.

## 가우스-마르코프 가정

가우스-마르코프 정리가 성립하려면 다음 네 가지 가정이 필요하다.

**1. 선형성 (Linearity)**

모델이 파라미터에 대해 선형이다. 즉, $$\mathbf{y} = \mathbf{X}\boldsymbol{\beta} + \boldsymbol{\epsilon}$$ 형태를 따른다.

**2. 외생성 (Exogeneity)**

오차항의 조건부 기댓값이 0이다.

$$
E[\epsilon_i \mid \mathbf{X}] = 0
$$

독립 변수와 오차항이 상관되지 않아야 한다는 뜻이다.

**3. 등분산성 (Homoscedasticity)**

오차항의 분산이 모든 관측치에서 동일하다.

$$
\text{Var}(\epsilon_i \mid \mathbf{X}) = \sigma^2 \quad \text{(상수)}
$$

**4. 비자기상관 (No Autocorrelation)**

서로 다른 관측치의 오차항이 상관되지 않는다.

$$
\text{Cov}(\epsilon_i, \epsilon_j \mid \mathbf{X}) = 0 \quad (i \neq j)
$$

가정 3과 4를 행렬로 합치면 $$\text{Var}(\boldsymbol{\epsilon} \mid \mathbf{X}) = \sigma^2 \mathbf{I}$$이다.

## BLUE: Best Linear Unbiased Estimator

가우스-마르코프 정리의 결론은 다음과 같다.

> 위 네 가지 가정이 성립하면, OLS 추정량 $$\hat{\boldsymbol{\beta}}$$는 **BLUE**이다.

BLUE의 각 요소를 분해하면 다음과 같다.

BLUE의 각 요소는 다음과 같다. B(Best)는 최적이라는 뜻으로, 분산이 가장 작다는 의미다. L(Linear)은 선형이라는 뜻으로, $$\hat{\boldsymbol{\beta}}$$가 $$\mathbf{y}$$의 선형 함수임을 의미한다. U(Unbiased)는 비편향이라는 뜻으로, $$E[\hat{\boldsymbol{\beta}}] = \boldsymbol{\beta}$$가 성립한다. E(Estimator)는 추정량이라는 뜻으로, 데이터로부터 파라미터를 추정한다는 의미다.

핵심은 "Best"의 의미다. 선형이면서 비편향인 추정량의 집합 안에서, OLS 추정량의 분산이 가장 작다. 임의의 선형 비편향 추정량 $$\tilde{\boldsymbol{\beta}}$$에 대해

$$
\text{Var}(\tilde{\beta}_j) \geq \text{Var}(\hat{\beta}_j) \quad \text{for all } j
$$

가 성립한다. 분산이 작다는 것은 추정의 정밀도가 높다는 뜻이므로, 같은 조건이라면 OLS를 쓰는 것이 가장 효율적이다.

## 가정이 깨질 때

현실 데이터에서는 가우스-마르코프 가정이 완벽히 성립하지 않는 경우가 많다.

- **이분산성(Heteroscedasticity)**: 오차 분산이 관측치마다 다르면 OLS는 여전히 비편향이지만, 더 이상 "Best"가 아니다. 이 경우 WLS(가중 최소제곱)나 robust standard error를 사용한다.
- **자기상관**: 시계열 데이터에서 흔히 발생하며, GLS(일반화 최소제곱)로 대응한다.
- **내생성**: 독립 변수와 오차항이 상관되면 OLS 추정량 자체가 편향된다. 도구 변수(IV) 등의 기법이 필요하다.

가정이 성립하는지 먼저 확인하고, 위반 시 적절한 대안을 선택하는 것이 올바른 모델링의 출발점이다.
