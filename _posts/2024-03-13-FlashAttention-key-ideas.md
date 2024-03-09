---
title: "[논문리뷰] FlashAttention"
categories: review
tags:
  - FlashAttention
  - GPU
  - Optimization
published: true
---
앞선 글들에서 [GPU 구조](http://jinwoongkim.net/gpu/알쓸G잡-GPU-메모리-및-쓰레드-구조/) 및 [최적화](http://jinwoongkim.net/gpu/알쓸G잡-GPU-Trick-or-Tweak/), [소프트맥스 병렬화](http://jinwoongkim.net/papers/paper-review-online-softmax/) 등을 다루어 보았다. 이러한 글들을 다루게 된 계기는 여럿 있었지만 그 중 하나는 **GPU-aware 한 딥러닝 최적화 논문들을 리뷰하기 위함**이었다.

GPU-aware한 최적화 논문들은 GPU도 알아야 하고, 딥러닝도 알아야 해서 난이도가 꽤 높은 편이다. 그래서 그런지 전체를 아우르는 설명은 찾기 힘들고, 더더욱이나 한글로된 자료는 없었다.

나의 정리가 미약하게나마 도움이 되었으면 좋겠다.

본 글에서는 GPU-aware 딥러닝 최적화 논문 리뷰 첫 번째로써, 유명 논문인 FlashAttention의 주요 아이디어들을 정리 해보았다.

논문의 코드나 그래프를 하나하나 설명하기 보단, 이 논문들에서 어떤 문제들을 파악하였고, 그것의 원인이 무엇이며, 어떻게 해결하려고 하는지를 정리하여 공유하겠다.
{: .notice--info} 

## FlashAttention 1

FlashAttention 1 의 풀네임은 "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness" 이다. 지금은 프린스턴 대학의 조교수로 취임한 Tri Dao가 박사일때 2022 년에 낸 논문이며 500회 이상의 인용수를 기록하고 있다.

논문 자체는 ICML workshop 등, 워크샵에만 등재되어 있지만 실제 코드를 구현하여 대부분의 상황에서 잘 작동하게 만들었기 때문에 다양하게 활용되며 입소문이 퍼지게 된 케이스이다. 

제목에서 알 수 있다시피 IO, read/write 관련 최적화 논문이며 

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
