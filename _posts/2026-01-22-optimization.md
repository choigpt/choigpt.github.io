---
layout: post
title: 최적화 기초 — Convex 함수와 전역 최솟값
date: 2026-01-22
tags: [optimization, mathematics, machine-learning]
permalink: /optimization/
excerpt: "Convex 함수에서는 지역 최솟값이 곧 전역 최솟값이다. 이차 미분이 0 이상(Hessian이 PSD)이면 convex이고, 이 성질이 머신러닝 최적화의 핵심 기반이다."
---

## 최적화 문제의 구조

최적화 문제는 다음과 같이 정의된다.

$$
\min_{\mathbf{x}} f(\mathbf{x}) \quad \text{subject to} \quad \mathbf{x} \in S
$$

- $$f(\mathbf{x})$$: 목적 함수 (objective function)
- $$S$$: 실행 가능 집합 (feasible set) — 제약 조건을 만족하는 $$\mathbf{x}$$의 집합

목적 함수를 최소화하는 $$\mathbf{x}^*$$를 찾는 것이 목표다. 일반적인 최적화 문제는 풀기 어렵지만, 목적 함수와 실행 가능 집합이 특수한 구조를 가지면 훨씬 쉬워진다. 그 핵심이 바로 **convexity**다.

## Convex 집합

집합 $$S$$가 convex하다는 것은, 집합 내 임의의 두 점을 잇는 선분이 항상 $$S$$ 안에 포함된다는 뜻이다.

$$
\mathbf{x}_1, \mathbf{x}_2 \in S \implies \theta \mathbf{x}_1 + (1 - \theta) \mathbf{x}_2 \in S \quad \text{for all } \theta \in [0, 1]
$$

$$\theta \mathbf{x}_1 + (1-\theta)\mathbf{x}_2$$는 두 점의 가중 합(weighted sum)이며, $$\theta$$가 0에서 1까지 변할 때 $$\mathbf{x}_1$$과 $$\mathbf{x}_2$$를 잇는 직선 위의 모든 점을 나타낸다. 이 점들이 전부 $$S$$에 들어가야 convex 집합이다.

예시를 들면, 원, 타원, 다면체는 convex 집합이다. 반면 도넛 모양이나 별 모양처럼 안쪽이 오목한 집합은 non-convex다.

## Convex 함수

함수 $$f$$가 convex하려면, 함수 위 임의의 두 점을 잇는 현(chord)이 항상 함수 그래프 위에 있어야 한다.

$$
f(\theta \mathbf{x}_1 + (1-\theta)\mathbf{x}_2) \leq \theta f(\mathbf{x}_1) + (1-\theta) f(\mathbf{x}_2) \quad \text{for all } \theta \in [0, 1]
$$

이를 **Jensen의 부등식**이라고도 한다.

### 일변수 함수의 판별

일변수 함수 $$f(x)$$가 두 번 미분 가능하면, 다음 조건으로 convexity를 판별할 수 있다.

$$
f''(x) \geq 0 \quad \text{for all } x
$$

이차 미분이 항상 0 이상이면 함수가 아래로 볼록(convex)하다. 예를 들어 $$f(x) = x^2$$은 $$f''(x) = 2 > 0$$이므로 convex다.

### 다변수 함수와 Hessian 행렬

다변수 함수 $$f(\mathbf{x})$$에서는 이차 미분의 역할을 **Hessian 행렬**이 한다.

$$
\mathbf{H} = \begin{bmatrix} \frac{\partial^2 f}{\partial x_1^2} & \frac{\partial^2 f}{\partial x_1 \partial x_2} & \cdots \\ \frac{\partial^2 f}{\partial x_2 \partial x_1} & \frac{\partial^2 f}{\partial x_2^2} & \cdots \\ \vdots & \vdots & \ddots \end{bmatrix}
$$

$$f$$가 convex일 조건은 Hessian이 **양반정치(positive semi-definite, PSD)**인 것이다.

$$
\mathbf{H} \succeq 0 \iff \mathbf{z}^T \mathbf{H} \mathbf{z} \geq 0 \quad \text{for all } \mathbf{z}
$$

PSD는 일변수에서 $$f''(x) \geq 0$$을 다변수로 일반화한 것이다. Hessian의 모든 고유값이 0 이상이면 PSD다.

## 핵심 정리: 지역 최솟값 = 전역 최솟값

Convex 최적화 문제(convex 함수 + convex 실행 가능 집합)에서는 다음이 성립한다.

> 지역 최솟값(local minimum)이 곧 전역 최솟값(global minimum)이다.

일반적인 non-convex 함수에서는 지역 최솟값이 여러 개 존재할 수 있고, 그중 일부만 전역 최솟값이다. 그래서 경사하강법이 지역 최솟값에 빠져 전역 최솟값을 찾지 못할 수 있다. 하지만 convex 함수에서는 그런 걱정이 없다. 어디서 출발하든 경사를 따라 내려가면 반드시 전역 최솟값에 도달한다.

## 머신러닝에서의 의미

머신러닝의 학습은 손실 함수(loss function)를 최소화하는 최적화 문제다.

- **선형 회귀**의 MSE 손실 함수는 $$\boldsymbol{\beta}$$에 대해 convex다. 따라서 경사하강법이나 정규 방정식으로 전역 최솟값을 확실히 찾을 수 있다.
- **로지스틱 회귀**의 cross-entropy 손실도 convex다. 같은 보장이 성립한다.
- **SVM**의 힌지 손실 역시 convex 최적화로 풀 수 있다.

손실 함수가 convex인지 아닌지에 따라 최적화의 난이도가 근본적으로 달라진다.

## Non-convex 문제: 신경망

신경망(neural network)의 손실 함수는 가중치에 대해 **non-convex**다. 여러 층의 비선형 활성화 함수가 겹치면서 손실 곡면에 수많은 지역 최솟값과 안장점(saddle point)이 생긴다.

그럼에도 불구하고 SGD(확률적 경사하강법)와 그 변형(Adam 등)이 실전에서 잘 작동하는 이유는 다음과 같다.

- 고차원 공간에서 대부분의 임계점은 안장점이지 지역 최솟값이 아니다.
- 여러 지역 최솟값의 손실 값이 서로 비슷한 경우가 많다.
- 미니배치의 노이즈가 안장점을 탈출하는 데 도움을 준다.

Convex 최적화의 이론적 보장은 없지만, 이런 경험적 특성 덕분에 딥러닝이 실용적으로 동작한다.
