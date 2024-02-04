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
Flash Attention2 논문을 읽다 보니 이해가 안 되는 부분이 많았다. 팀 내 ML 연구원들은 GPU 파트를 이해 못 했는데 난 오히려 GPU 파트 보단 ML 파트가 이해가 안 됐다;;

그 중에서도 Flash Attention 2 의 핵심 아이디어인 softmax 병렬 계산이 도통 이해가 안 돼서 관련 논문을 열심히 읽었다. 인용수가 22밖에 안돼서 그런가 자료가 매우 없어서 아쉬웠고, 그래서 나의 부족한 글이라도 누군가가에게 도움이 돼지 않을까 하여 올려본다.

논문 URL : https://arxiv.org/abs/1805.02867

# tl;dr
- Softmax 메모리 최적화 및 병렬 프로세싱을 다룬 논문
	- 병렬로 연산을 하지만 근사치가 아닌 정확한 값을 도출 해낸다.
	- 메모리가 제곱배수가 아닌 선형으로 증가한다.

## 1. Motivation - Attention requires O(NxN) memory

<p align="center">
<img width="600" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/3b757b71-4f83-487b-9b35-f3050fb58d41">
<br>
<em>수식 1. Attention</em>
</p>

이 논문에서는 Attention을 다루진 않지만, 나는 Flash Attention 의 softmax 를 기준으로 이해하기 위해서 보는 것이므로, Flash Attention 논문에서 attention 을 가지고 왔다. 

여기서 강조하고 싶은 것은 위의 attention matrix,  S가 최근 long sequence가 유행하는 LLM에서는 점 점 커지고 있다는 것이다. 심지어 N x N 이기 때문에 (여기서 N은 인풋 길이, 토큰 수) 선형이 아니라 제곱으로 그 크기가 증가하고 있다.

본 논문에서는, 이러한 많은 메모리 및 연산 시간을 요구하는 비싼 softmax 연산을 어떻게 메모리, 시간 적으로 최적화 할지 다룬다.

## 2. Potential Problem of Standard(Naive) Softmax

먼저, 기본 Softmax 수식을 살펴보겠다. 아래 수식은 수학적으로는 \아무런 문제가 없을 수 있으나, _**실제로 컴퓨터에서 동작을 하면 오버/언더플로우가 발생 할 수 있다.**_ 따라서 `TensorFlow`, `PyTorch` 등은 뒤에서 설명할 `Safe Softmax` 를 사용한다.

<p align="center">
<img width="200" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/c936ec0b-fa65-4e78-a0f1-860935199bec">
<br>
<em>수식 2. (Naive) Softmax</em>
</p>


여기서 한 가지 더 살펴볼 점은, **메모리 엑세스 횟수**이다.
아래 `<코드1>` 에서 볼 수 있듯이 standard softmax 는 각각의 값을 구하기 위해서 메모리를 총 3번 엑세스 하는데, 
1. `(3번째 줄)` <em>d<sub>j</sub></em> 를 구할때 모든 <em>e<sup>x<sub>j </sub></sup></em> 에 대해 한 번 씩 `(Load)`
2. `(6번째 줄, 우항)` <em>y<sub>i</sub></em> 를 구할때 또 <em>e<sup>x<sub>i</sub></sup></em> 한번씩 `(Load)`
3. `(6번째 줄, 좌항)` 그리고 <em>y<sub>i</sub></em> 에 값을 저장할때 또 한 번씩 `(Store)`
이렇게 softmax 를 구할때 총 _O(3V)_ 번 메모리에 접근한다.  `(V=N)`

<p align="center">
<img width="300" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/27ae5fe9-2b4d-45d1-8c87-2b4df54d2dc6">
<br>
<em> 코드 1. Softmax </em>
</p>

## 3. Safe Softmax

앞서 기본 형태의 softmax는 오버/언더플로우 문제가 있다고 언급했고, 그걸 해결하기 위한 것이 이 safe softmax 라는 것을 말했다. 아래 수식을 보면 Safe softmax 는 오버/언더플로우 문제를 해결하기 위해 단순히 최댓값을 빼주어서 scale down 해준다.


> [!NOTE] 왜 최댓값일까?
> 사실 난 이 부분이 직관적으로 이해가 되진 않았다. 그런데 후에 곰곰이 생각해보니, 결국 오버/언더플로우가 발생하는 이유는 상대적으로 큰 수치 때문이니까 그럴 것 같다는 생각이 들었다. 


<p align="center">
<img width="200" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/d823cdbd-61a8-4e71-b609-48a8deb5d2d1">
<br>
<em>수식 3. Safe Softmax</em>
</p>

여기서 주목해야 할 점은 최댓값을 구해줘야 함으로써 **메모리 접근 수가 한 번 더 늘었다는 것**이다. 아래 `<코드2>` 에서 볼 수 있듯이이 3번째 줄에서 최댓값을 구하기 위해 O(V)만큼 메모리를 더 접근해야 한다. 따라서 하나의 softmax 값을 구하기 위해 이전에 3번만 접근하면 됐다면 이젠 4번 접근을 해야 하는 상황인 것이다.
<p align="center">
<img width="300" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/00d65fd5-b6aa-44f9-8a7b-2303812d8d46">
<br>
<em>코드 2. Safe Softmax</em>
</p>

메모리 한 번 더 접근하는 게 무슨 큰 이슈일까? -> 사실 GPU에선 매우 큰 이슈이다.

아래 `<그림1>`은 FlashAttention 논문에서 가져온 그림인데, GPU는 **작지만 빠른 SRAM**과 **크지만 느린 HBM** 으로 메모리가 구성되어있다. 그리고, softmax matrix는 너무 커서 SRAM에 모두 두기 힘들다. 따라서 크지만 느린 HBM 으로의 메모리 연산이 하나 더 증가하는 것은 전체 성능에 많은 영향을 끼친다.

<p align="center">
<img width="450" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/b37fe3d2-6095-469a-a947-379980acf627">
<br>
<em>그림 1. GPU 메모리 계층 및 대역폭, 사이즈 </em>
</p>


## Online Safe Softmax

<p align="center">
<img width="600" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/72075e54-36d6-4885-b8f0-347d627fee4b">
<br>
<em>코드 3. Online Safe Softmax</em>
</p>

위의 <코드3>에 본 논문의 핵심 아이디어가 서술되어있다. 주목 할 부분은 3-6번째 줄인데, <코드2>의 2-8줄과 비교해보면 for loop 이 하나 없다는 걸 파악 할 수 있다.

기존 <코드2>에서는 max 값을 구하고, 그 값을 이용해서 dj 를 구했다면, 여기서는 실시간으로 현재 최댓값을 이용해서 dj  를 구하고 나중에 그 값을 정상화(?) 해준다.

이렇게 해주면 메모리 접근 횟수가 한 번 줄어들게 된다. 좀 더 자세히 설명하자면 기존 <코드2>에서는 max 값을 구하려고 모든 xj 에 접근하면서 메모리 접근 한 번씩, 그리고 dj 를 구하느라 다시 한 번 xj에 접근하는데, 위의 코드(<코드3>)에서는 xj 에 한 번 엑세스하고 dj 를 구해 버리므로 메모리 접근 횟수가 한 번 줄어든다. (총 4에서 3으로 감소)

반면 연산 자체는 증가한 것 아닌가? 이런식으로 생각 할 수 있다. 그런데 그건 서술 안되어 있음;;

핵심은, 모든 x의 max 값을 구하지 않고, dj 값을 구하는 것인데 아래 예제에서 설며하


<수식 3> 으로 돌아가보자. 수식 3을 풀어서 서술하면 아래와 같다.

$$
e^{x_i}-maxx \over e

$$



t


참고 :
https://velog.io/@d2h10s/LaTex-Markdown-수식-작성법
https://github.com/JinwoongKim/JinwoongKim.github.io/issues/1


https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/ba400caf-3bd0-42f4-8436-5514d28cba30