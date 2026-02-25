---
layout: post
title: "벨만 방정식 (Bellman Equation)"
date: 2026-02-07
tags: [강화학습, ML-AI, 벨만방정식]
category: ML-AI
---

벨만 방정식은 강화학습에서 **가치 함수를 재귀적으로 표현**하는 핵심 수식이다. 현재 상태의 가치를 즉각적 보상과 다음 상태의 가치로 분해한다.

## 상태 가치 함수 (V 함수)

현재 상태 s에서 정책 π를 따랐을 때 기대되는 누적 보상이다.

$$V^\pi(s) = \mathbb{E}_\pi \left[ R_{t+1} + \gamma V^\pi(S_{t+1}) \mid S_t = s \right]$$

- `R_{t+1}`: 바로 다음 스텝에서 받는 즉각적 보상
- `γ`: 할인율 (0~1)
- `V(S_{t+1})`: 다음 상태의 가치

## 행동 가치 함수 (Q 함수)

상태 s에서 행동 a를 취했을 때 기대되는 누적 보상이다.

$$Q^\pi(s, a) = \mathbb{E}_\pi \left[ R_{t+1} + \gamma Q^\pi(S_{t+1}, A_{t+1}) \mid S_t = s, A_t = a \right]$$

## V와 Q의 관계

$$V^\pi(s) = \sum_a \pi(a|s) \cdot Q^\pi(s, a)$$

$$Q^\pi(s, a) = \sum_{s'} P(s'|s,a) \left[ R + \gamma V^\pi(s') \right]$$

## 최적 벨만 방정식

$$V^*(s) = \max_a \sum_{s'} P(s'|s,a) \left[ R + \gamma V^*(s') \right]$$

$$Q^*(s, a) = \sum_{s'} P(s'|s,a) \left[ R + \gamma \max_{a'} Q^*(s', a') \right]$$

Q-learning에서 사용하는 업데이트 식이 바로 이 최적 벨만 방정식에서 나온다.
