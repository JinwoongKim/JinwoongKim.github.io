---
title: "[논문리뷰] Online normalizer calcuation for softmax by Nvidia (2018)"
categories: papers
tags:
  - 논문리뷰
  - softmax
  - Nvidia
  - FlashAttention
published: false
---
Flash Attention2 논문을 읽다 보니 이해가 안 되는 부분이 많았다. 팀내 연구원들은 GPU 파트를 이해 못 했는데 난 오히려 GPU 파트 보단 딥러닝파트가 이해가 안 됐다;;

그 중에서도 Flash Attention 2의 핵심 아이디어인 softmax 병렬 계산이 도통 이해가 안 돼서 관련 논문을 열심히 읽었다. 인용수가 22밖에 안돼서 그런가 자료가 매우 없어서 아쉬웠고, 그래서 나의 부족한 리뷰라도 누군가가에게 도움이 돼지 않을까 하여 올려본다.

논문 URL : https://arxiv.org/abs/1805.02867

# tl;dr
- 메모리 엑세스 최
## softmax

## safe-softmax

## online safe-softmax