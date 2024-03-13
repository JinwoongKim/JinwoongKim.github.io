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

GPU-aware한 최적화 논문들은 GPU도 알아야 하고, 딥러닝도 알아야 해서 복잡한 편이다. 그래서 그런지 전체를 아우르는 설명은 찾기 힘들고, 더더욱이나 한글로된 자료는 없었다.

본 글에서는 앞서 정리한 GPU 및 딥러닝 관련 글을 기반으로, GPU-aware 딥러닝 최적화 논문 리뷰를 해보겠다.

오늘 리뷰할 논문은, 아주 유명한 논문 중 하나 인 **FlashAttention** 1, 2 이다. 

논문의 코드나 그래프를 하나하나 설명하기 보단, 주요 아이디어 들을 공유하는 방식으로 서술하였다.
{: .notice--info} 

# FlashAttention 1 (22.06.24)

## TMI
FlashAttention 1 의 풀네임은 "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness" 이다. 지금은 프린스턴 대학의 조교수로 취임한 Tri Dao가 박사일때 2022 년에 낸 논문이며 500회 이상의 인용수를 기록하고 있다. ~~(내 전체 논문 인용수를 합쳐야 500이 넘는데..눈물..)~~

논문 자체는 ICML workshop 등, 워크샵에만 등재되어 있지만 실제 코드를 구현하여 대부분의 상황에서 잘 작동하게 만들었기 때문에 다양하게 활용되며 입소문이 퍼지게 된 케이스이다. 요즘 MIT의 song han 교수도 그렇고 tri dao도 github 에 코드를 공개 <sup> [[Link]](https://github.com/Dao-AILab/flash-attention/issues) </sup> <sup> [[Link]](https://github.com/mit-han-lab) </sup> 하고 활동을 열심히 하는 모습을 볼 수 있다. 내 전공인 시스템 분야와는 다르게 '내부적으로 어떻게 돌아가는지는 모르겠고, 그래서 내가 써 볼 수 있어? 쉽게?'가 중요한 분야 같다.

사설이 길었는데, 이 논문은 제목에서 알 수 있다시피 IO, read/write 관련 최적화 논문이며, 그 중에서도 GPU의 메모리 구조를 파악하고 최적화 한 논문이다. [앞선 글](http://jinwoongkim.net/gpu/%EC%95%8C%EC%93%B8G%EC%9E%A1-GPU-%EB%A9%94%EB%AA%A8%EB%A6%AC-%EB%B0%8F-%EC%93%B0%EB%A0%88%EB%93%9C-%EA%B5%AC%EC%A1%B0/#gpu-%EB%82%B4%EB%B6%80-%EA%B5%AC%EC%A1%B0) 에서 설명했듯이 GPU의 메모리는 (단순화하면) 작지만 빠른 on-chip과 느리지만 큰 off-chip 으로 구성되어 있다. 저자는 이러한 구조를 이해하지 않고 GPU 프로그래밍을 하면 최대 성능을 이끌어 낼 수 없다고 하며, 이러한 구조에 최적화된 형태의 알고리즘을 제안한다.

## 문제
- 트랜스포머 기반 LLM 모델의 경우 문자열의 길이가 매우 제한적이다.
- 문자열의 길이가 제한적이라 더 긴 문장이나 이미지 등을 학습 할 수 없다.
## 원인
- 어텐션 행렬의 시간, 공간 복잡도가 N<sup>2</sup> (N = 토큰 갯 수) 라서, 문자열의 길이를 늘리기 어렵다. → Long latency, OOM
	- <img src="images/Pasted image 20240309102820.png">
## 해결
- Tiling (소프트맥스 병렬화를 통한 속도 향상)
	- SRAM 의 사이즈에 맞게 어텐션 행렬을 자른 후, SM에 할당, 블락 병렬로 수행
	- <img src="images/Pasted image 20240309105731.png">
	- ![[blog/images/Pasted image 20240309105731.png|500]]
	- HBM 접근 횟수 감소가 목표
	- ![[blog/images/Pasted image 20240313174625.png]]
- Recomputation (어탠션 행렬을 저장하지 않고, backward 일때 다시 계산)
	- We store the softmax normalization factor from the forward pass to quickly recompute attention on-chip in the backward pass, which is faster than the standard approach of reading the intermediate attention matrix from HBM
- One more, kernel fusioni
	- GPU는 matmal 에 최적화, 근데 실제론 다른 곳에서 시간을 더 쓰고 있음
	- ![[blog/images/스크린샷 2024-03-13 17.47.25.png]]
## 결과
- 스피드업
- 메모리 세이빙 → 더 긴 문자열
- 등등

![[blog/images/Pasted image 20240309110044.png]]
![[blog/images/스크린샷 2024-03-09 11.00.57.png]]

![[blog/images/Pasted image 20240309110131.png]]
![[blog/images/Pasted image 20240309110150.png]]

![[blog/images/Pasted image 20240309110209.png]]
![[blog/images/Pasted image 20240309110230.png]]
###  참고
- https://www.youtube.com/watch?v=FThvfkXWqtE 
- https://www.youtube.com/watch?v=gMOAud7hZg4


# FlashAttention 2 (23.07.18)

## 문제
- FlashAttention 이 아직 최적화가 덜 되었다.
	- 아직 GPU 를 많이 못 쓰고 있다.
		- Forward pass 이론 최대 FLOPs/s의 30-50%, backward pass는 25-35%
## 원인
- 알고리즘적, 하드웨어(GPU)적 최적화가 덜 진행 됨
- 자세한 건 아래에서 설명
- 매우 큰 A100의 matmul/non-matmul 성능 차이
	- 312 TFLOPS/s of FP16/BE16 matmul
	- 19.5 TFLOPS/s of non-matmul FP32
## 해결
- Tweak algorithm
	- non-matmul 연산 최소화
	- ![[blog/images/Pasted image 20240313204524.png]]
	- 첫 번째 tweak
		- FlashAttention 1
		- ![[blog/images/Pasted image 20240313202855.png]]
		- FlashAttention 2
		- ![[blog/images/Pasted image 20240313202910.png]]
	- 두 번째 tweak
		- ![[blog/images/Pasted image 20240313211551.png]]
- Parallelism
	- 
- Work Partitioning Between Warps
	- ![[blog/images/Pasted image 20240313211646.png]]


## 결과
### A100
![[blog/images/Pasted image 20240314082805.png]]
### H100
![[blog/images/Pasted image 20240314082823.png]]


참고
https://www.youtube.com/watch?v=IoMSGuiwV3g
https://www.youtube.com/watch?v=foG0ebzuw34&t=4080s
https://www.youtube.com/watch?v=IoMSGuiwV3g