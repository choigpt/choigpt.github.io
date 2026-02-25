---
layout: post
title: "강화학습 기초"
date: 2026-02-05
tags: [강화학습, ML-AI]
category: ML-AI
---

## 핵심 개념

- **greedy action**: 현재 추정치 기준으로 가장 좋아 보이는 행동을 선택
- **ε-greedy**: 0~1 사이 값. ε 확률로 탐험(exploration), 1-ε 확률로 착취(exploitation)
- **exploration & exploitation**: 새로운 가능성 탐색 vs 현재 최선 활용의 균형
- **decaying ε-greedy**: 학습이 진행될수록 ε를 점점 줄여 착취 비중을 높임

## 가치 함수 업데이트

$$Q(s,a) \leftarrow (1-\alpha)Q(s,a) + \alpha(r + \gamma \max_{a'} Q(s',a'))$$

- `α`: 학습률
- `r`: 즉각적 보상
- `γ`: 할인율 (discount factor) — 미래 보상을 현재 가치로 환산

## MDP (Markov Decision Process)

- **state value function V(s)**: 상태 s에서 기대되는 누적 보상
- **action value function Q(s,a)**: 상태 s에서 행동 a를 취했을 때 기대되는 누적 보상
- **optimal policy**: 모든 상태에서 최대 가치를 선택하는 정책

벨만 방정식으로 이어짐.
