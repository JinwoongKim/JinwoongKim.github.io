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
- Nvidia GPU 에서 softmax 연산시 메모리 엑세스 최소화 및 병렬 프로세싱을 다룬 논문
- 여타 다른 논문에선 근사치로 softmax 를 계산해서 최적화하는데 여기선 아님. 정확한 값이 나옴 

## Attention & Softmax

<img width="600" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/3b757b71-4f83-487b-9b35-f3050fb58d41">
_수식 1. Attention 수식_

이 논문에서는 Attention을 다루진 않지만, 나는 Flash Attention 의 softmax 를 기준으로 이해하기 위해서 보는 것이므로, Flash Attention 논문에서 attention 을 가지고 왔다. 

여기서 내가 강조하고 싶은 것은 softmax의 S가 최근 LLM에서는 엄청 크다는 것이다. 따라서 아래에서 살펴볼 가장 기본적인 softmax는 수식으로는 아무런 문제가 없을 수 있으나, 실제로 컴퓨터에서 동작을 하면 오버/언더플로우가 발생 할 수 있다.

<img width="200" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/c936ec0b-fa65-4e78-a0f1-860935199bec">
_수식 2. Softmax 수식_
## Safe Softmax



## Online Safe Softmax