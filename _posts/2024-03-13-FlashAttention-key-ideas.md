---
title: "[논문리뷰] FlashAttention"
categories: review
tags:
  - FlashAttention
  - GPU
  - Optimization
published: false
---
앞선 글들에서 [GPU 구조](http://jinwoongkim.net/gpu/알쓸G잡-GPU-메모리-및-쓰레드-구조/) 및 [최적화](http://jinwoongkim.net/gpu/알쓸G잡-GPU-Trick-or-Tweak/), [소프트맥스 병렬화](http://jinwoongkim.net/papers/paper-review-online-softmax/) 등을 다루어 보았다. 이러한 글들을 다루게 된 계기는 여럿 있었지만 그 중 하나는 **GPU-aware 한 딥러닝 최적화 논문들을 리뷰하기 위함**이었다.

GPU-aware한 최적화 논문들은 GPU도 알아야 하고, 딥러닝도 알아야 해서 난이도가 꽤 높은 편이다. 그래서 그런지 전체를 아우르는 설명은 찾기 힘들고, 더더욱이나 한글로된 자료는 없었다.

나의 정리가 미약하게나마 도움이 되었으면 좋겠다.



본 글에서는 그 중 가장 유명한 논문 중 하나인 FlashAttention의 주요 아이디어들을 정리 해보았다.
## FlashAttention 1

### 문제

### 원인

### 해결



- Online normalizer softmax
- tiling
- recomputation
- kernel fusion

flash 2

tweak
more blocks
warp 순서 바꾸기
