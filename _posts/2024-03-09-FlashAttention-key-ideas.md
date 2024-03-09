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

이를 위해 GPU 관련 내용이나 여러 최적화 기법에서 사용되는 글들을 꾸준히 정리하고 있다. 나의 정리가 미약하게나마 도움이 되었으면 좋겠다.

본 글에서는 GPU-aware 딥러닝 최적화 논문 리뷰 첫 번째로써, 유명 논문인 FlashAttention의 주요 아이디어들을 정리 해보았다.

논문의 코드나 그래프를 하나하나 설명하기 보단, 이 논문들에서 어떤 문제들을 파악하였고, 그것의 원인이 무엇이며, 어떻게 해결하려고 하는지를 정리하여 공유하겠다.
{: .notice--info} 

## FlashAttention 1

FlashAttention 1 의 풀네임은 "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness" 이다. 지금은 프린스턴 대학의 조교수로 취임한 Tri Dao가 박사일때 2022 년에 낸 논문이며 500회 이상의 인용수를 기록하고 있다. ~~(내 전체 논문 인용수를 합쳐야 500이 넘는데..눈물..)~~

논문 자체는 ICML workshop 등, 워크샵에만 등재되어 있지만 실제 코드를 구현하여 대부분의 상황에서 잘 작동하게 만들었기 때문에 다양하게 활용되며 입소문이 퍼지게 된 케이스이다. 요즘 MIT의 song han 교수도 그렇고 tri dao도 github 에 코드를 공개 <sup> [[Link]](https://github.com/Dao-AILab/flash-attention/issues) </sup> <sup> [[Link]](https://github.com/mit-han-lab) </sup> 하고 활동을 열심히 하는 모습을 볼 수 있다. 내 전공인 시스템 분야와는 다르게 '내부적으로 어떻게 돌아가는지는 모르겠고, 그래서 내가 써 볼 수 있어? 쉽게?'가 중요한 분야 같다.

사설이 길었는데, 이 논문은 제목에서 알 수 있다시피 IO, read/write 관련 최적화 논문이며 

### 문제
- long sequence
	- requires time, memory O(N^2
- approximation some how leads / may causes problems

### 원인
- attention 은 매우 핵심
- 근데 앞서 말했듯이 sequence length에 N*N임
- ![[blog/images/Pasted image 20240309102820.png]]
- 그러다보니, GPU는 matmal 에 최적화, 근데 실제론 다른 곳에서 시간을 더 쓰고 있음
- ![[blog/images/Pasted image 20240309103137.png|200]]
- 그게 메모리 엑세스임
- 예전 자료에서도 설명했지만..
![[blog/images/Pasted image 20240309103611.png]]

### 해결
- tiling
	- restructure algorithm to load block by block from HBM to SRAM to compute attention
- recomputation
	- don't store attn, matrix from forward, recompute it in the backward

## tiling
![[blog/images/Pasted image 20240309105731.png]]


- Online normalizer softmax
- tiling
- recomputation
- kernel fusion

# 결과
스피드업
메모리 세이빙

![[blog/images/Pasted image 20240309110044.png]]
![[blog/images/스크린샷 2024-03-09 11.00.57.png]]

![[blog/images/Pasted image 20240309110131.png]]
## flash 2

tweak
more blocks
warp 순서 바꾸기

#####  참고
- https://www.youtube.com/watch?v=FThvfkXWqtE 