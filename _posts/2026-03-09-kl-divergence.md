---
layout: post
title: KL Divergence — 두 확률분포의 정보량 차이 측정
date: 2026-03-09
tags: [statistics, probability, 정보이론, 머신러닝]
permalink: /kl-divergence/
excerpt: "KL Divergence는 두 확률분포가 얼마나 다른 정보를 담고 있는지를 측정한다. 크로스 엔트로피에서 엔트로피를 빼면 된다. 비대칭이라는 한계가 있고, 이를 보완하는 Jensen-Shannon Divergence도 정리한다."
---

## KL Divergence란

**KL Divergence**(Kullback-Leibler Divergence)는 두 확률분포 사이의 정보량 차이를 측정하는 지표다. 딥러닝, 머신러닝, 데이터 사이언스에서 크로스 엔트로피만큼 자주 등장한다.

수식으로 보면 **두 분포의 크로스 엔트로피에서 엔트로피를 뺀 것**이다.

$$D_{KL}(P \| Q) = H(P, Q) - H(P)$$

이산 확률분포에서는 이렇게 전개된다.

$$D_{KL}(P \| Q) = \sum_{x} P(x) \log \frac{P(x)}{Q(x)}$$

---

## 직관적 의미

기준 분포 P가 있을 때, Q가 P와 얼마나 다른 정보를 가지고 있는지를 측정한다. 기준 분포의 불확실성(엔트로피)을 제거하고, **순수하게 두 분포 간의 정보 차이만** 남긴다.

- 두 분포가 유사하면 값이 작아진다
- 두 분포가 동일하면 KL = 0
- 값의 범위는 **0 ~ ∞**

---

## 비음수성과 비대칭성

KL Divergence는 항상 0 이상이고 (**Gibbs 부등식**), 등호는 정확히 $P = Q$일 때 성립한다. "두 분포가 같으면 0, 다를수록 양수"라는 거리감이 여기서 나온다. 다만 일반적인 거리(metric)와 달리 대칭은 성립하지 않는다.

$$D_{KL}(P \| Q) \neq D_{KL}(Q \| P)$$

기준 분포가 바뀌면, 같은 두 분포라도 KL 값이 달라진다. P를 기준으로 Q를 본 것과 Q를 기준으로 P를 본 것이 다르다. 특정 분포를 기준으로 잡아 상대적으로 계산하기 때문이다.

이 비대칭성 때문에 모델 업데이트 시 문제가 발생하기도 한다. 예를 들어 강화학습의 PPO에서 정책 업데이트 폭을 제한할 때, 어느 분포를 기준으로 잡느냐에 따라 결과가 달라진다.

---

## 보완: Jensen-Shannon Divergence

KL Divergence의 비대칭성을 보완하는 것이 **Jensen-Shannon Divergence**(JSD)다.

$$JSD(P \| Q) = \frac{1}{2} D_{KL}(P \| M) + \frac{1}{2} D_{KL}(Q \| M)$$

여기서 $M = \frac{1}{2}(P + Q)$, 두 분포의 평균이다.

양쪽 방향의 KL을 구해서 평균을 내는 방식이라 **대칭**이고, 값의 범위도 **0 ~ 1**(log base 2 기준)로 제한되어 더 부드럽게 분포 차이를 측정할 수 있다.

KL Divergence와 Jensen-Shannon Divergence를 비교하면 다음과 같다. 대칭성 측면에서 KL Divergence는 비대칭이고, JSD는 대칭이다. 값의 범위는 KL Divergence가 0에서 무한대인 반면, JSD는 0에서 1(log₂ 기준) 사이로 제한된다. KL Divergence는 기준 분포가 필요하지만, JSD는 두 분포의 평균을 사용하므로 기준 분포가 불필요하다. KL Divergence는 크로스 엔트로피 손실, VAE, PPO 등에서 쓰이고, JSD는 GAN(원래 논문)이나 분포 유사도 비교에 쓰인다.

---

## 정리

KL Divergence는 두 확률분포의 정보량 차이를 측정하는 기본 도구다. 크로스 엔트로피 손실 함수가 결국 KL Divergence를 최소화하는 것이고, VAE의 잠재 분포 정규화에도 쓰인다. 비대칭이라는 한계가 있으므로, 대칭 비교가 필요하면 Jensen-Shannon Divergence를 쓴다.
