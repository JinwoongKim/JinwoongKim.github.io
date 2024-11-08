---
title: vLLM 주요 아이디어
categories: papers
tags:
  - vLLM
published: true
---

주위에서 하도 vLLM, vLLM 해서, 뭔가 하고 읽어 봤다. 키 아이디어만 빠르게 읽고 정리해 보았다.

논문제목 : Efficient Memory Management for Large Language Model Serving with PagedAttention

# 1. 문제
KV cache 를 사용하는 LLM 기반의 인퍼런스의 경우, 배치 사이즈를 키웠을때 KV cache가 필요로 하는 메모리의 양이 크게 증가하여 **배치사이즈 키우는데 많은 제약이 걸린다.**

즉, 본 논문은 단일 추론 속도를 빠르게 하는데 초점이 맞춰져 있기 보단 (메인 아이디어만 생각하면 오히려 느려질 수 있다.) 메모리 효율화를 통해 배치사이즈를 늘리는데 있다.

<p align="center"> <img width="400" src="https://github.com/user-attachments/assets/386da356-a5ce-4e86-a24d-cb99b59aa961"></p>
# 2. 원인
이러한 문제를 야기하는 **원인으로는 KV cache 의 비효율적인 메모리 관리**를 꼽는다.

크게 두 가지 특성이 이러한 것들을 야기하는데,
1. 연속성 : 하나의 request를 위해 꼭 **연속**된 주소의 메모리를 할당 받는다는 것
2. 점유성: 할당받은 메모리를 하나의 request가 잠시라도 공유하지 않고 **독점** 하는 것
이 있다.

예를 들어, 2048개의 토큰을 ‘생성할 수 도’있는 모델은, 2048개의 토큰을 저장 할 수 있는 ‘**연속된 메모리’**를 할당받고, request의 전체 생애 동안 해당 메모리(2048개를 저장 할 ‘수도 있는’ 공간)를 자신만이 **‘독점’ 점유**한다.
- 만약 parallel sampling을 하거나 beam search를 하게 된다면 더욱 메모리 사용량이 증가한다.

이러한 관리체계가 메모리 효율을 어떻게 떨어뜨릴까?
1. 파편화
	- 이러한 메모리 관리 체계는 크게 3가지의 메모리 낭비(파편화)를 일으키는데, 각각 `reserved`, `internal fragmentation`, `external fragmentation` 라 부른다.
	- `reserved` 은 **최종적으로 사용되긴 하지만, 그전엔 아무도 못쓰는 공간**을 칭한다. 우리가 실제로 토큰을 생성하더라도 한번에 모든 토큰이 생성되는 것이 아니기 때문에, 마지막으로 생성되는 토큰의 경우 마지막 순간에만 잠시 토큰을 GPU 메모리에 저장하기 위해 자신의 자리를 계속 ‘예약’하고 있기 때문에 이렇게 부르고 있다.
	- `internal fragmentation` 은 **할당 받았지만 쓰지 않고 릴리즈 하는 공간**을 의미한다. 예를 들어 출력 길이가 2k인 모델인 경우, 2k 만큼의 공간을 미리 할당 받았지만, 상황에 따라서 2k까지 토큰이 생성되지 않을 수도 있다. 이때 이렇게 할당 받았지만 안 쓰여지고 릴리즈 되는 공간을 말한다.
	- `external fragmentation`은 **메모리 할당 사이사이의 뜨는 공간**을 칭한다.
	- 예를 들어, 6인 테이블이 하나 있는데 5명의 손님이 예약을 하고 한 명씩 10분 간격으로 등장을 하고 2명이 노쇼를 했다고 생각해보자.
		- 그렇다면, 아무도 올 가능성이 없는 빈자리 하나는 `external fragmentation` . 올 수 있었지만, 안 온 노쇼 2자리는 `internal fragmentation`, 모두 오긴 하지만 하나씩 자리가 채워지는 3자리는 `reserved` 가 된다.
		<p align="center"> <img width="600" src="https://github.com/user-attachments/assets/114c5a69-2bfa-4fd5-a886-fb1f9e8fea83"></p>
		- 본 논문에서는 이러한 비효율적인 메모리 관리 체계를 지적하며, 이러한 메모리 낭비가 상당하다고 보여주고 있다. 아래 그림 참조
 <p align="center"> <img width="400" src="https://github.com/user-attachments/assets/dfc72af7-1936-4ce8-a851-3eec2beb4046"></p>
2.  잦은 메모리 복사
	- Parallel sampling 및 beam search의 경우, 프롬프트 또는 기존 생성한 토큰들의 KV 값이 같은 경우가 종종 있다고 함
		- 그럼에도 불구하고 기존 방식들은 새로운 토큰을 생성하기 위해, 기존 프롬프트나 토큰들의 KV 값이 연속된 메모리에 있어야 하기 때문에, 각자 복사를 해서 사용했다고 함

# 3. 제안
- 이러한 문제는 여기서 처음 발생한게 아니다. 대부분의 메모리를 직접 관리해야하는 시스템에서는 흔하게 발생한 일이고, 우리가 아는 보편적인 OS에서는 이를 가상 메모리와 페이징으로 이를 해결 했다.
- vLLM은 여기서 영감을 받아 KV cache를 **비연속적 paged 메모리에 저장하는 PagedAttention** 알고리즘을 제안하고, 이를 기반으로 동작하는 **분산서빙엔진 vLLM**를 디자인하고 구현함

## **PagedAttention 알고리즘**
### chunking
- KV cache 를 기존처럼 연속된 주소에 할당하지 않고, GPU 메모리의 이곳저곳에 저장함
- 기존에는 N만큼의 공간이 필요할때, N만큼의 메모리가 연속되어 비어 있지 않으면 할당을 못했는데, 이젠 파편화 되었더라도 N만큼의 메모리가 있다면 할당 가능
	- 엄밀히 말하면 파편화된 각 메모리가 KV block 보단 커야됨
- 대신, 해당 KV cache 들을 접근할때 이를 연결해줄 매핑 테이블이 필요.
- 이를 여기선 KV block 테이블이라고 함
<p align="center"> <img width="600" src="https://github.com/user-attachments/assets/0b04f51f-e8c7-48ca-b278-2d721c1253e5"></p>
### sharing
- Parallel sampling 예시
	- 기존 시스템에선, 프롬프트가 같은 경우에도, 해당 프롬프트의 KV 값을 복사를 해서 진행해었다고 함
	- 여기선 logical block은 각자 소유하되, physical block은 하나만 두어 해결
	- 만약 블럭의 마지막에 다른 토큰이 들어가게 된다면, 그때 블럭을 하나 더 복사하여 해결

<p align="center"> <img width="600" src="https://github.com/user-attachments/assets/5f349f48-d451-47f5-9792-4faea7812d9a"></p>
    
- Beam search 예시
	- block 11이랑 12의 경우 block 0, 1, 3, 7 의 KV 값이 필요한데, 기존 시스템에선 각각 복사본이 필요했다면, 여기선 같은 블럭을 같이 참조

<p align="center"> <img width="600" src="https://github.com/user-attachments/assets/fa11a8c1-82b6-4f16-b18a-80b88db72287"></p>
    

## **vLLM**
- 위에서 제안한 PagedAttention 알고리즘을 통해 동작하는 서빙 엔진
	<p align="center"> <img width="600" src="https://github.com/user-attachments/assets/4f3e3e0e-3520-4684-91ac-cd2b9f948d4d"></p>    

- 다만 vLLM이 극복해야 할 문제들이 아직 남아 있는데,
    - 첫째로 미리 메모리를 할당받지 않아서 생기는 OOM 문제는 스케쥴링 알고리즘으로 해결하였는데, 아래 서술하였다.
	    -  Scheduling and Preemption (논문 4.5 섹션)
		    - swapping : CPU로 evict 했다가 다시 가져오는 방식
		        - evict 하는 단위는 request 내 모든 KV block들. 어차피 한 번에 접근해야 하므로..
		        - 이 방식은 하드웨어 성능에 의존적임
		    - recomputation
		        - 다시 계산하는 방식
		        - 다만, 10개의 토큰을 생성하다가 evict 된 경우, 기존 프롬프트에 생성된 토큰을 연결하여 프롬프트로 처리. 해당 10개의 토큰에 대해선 또 다시 생성을 안해도 되는 장점이 있어, evict 횟수만큼 처리 시간이 증가하진 않는다.
    - 둘째로 스케쥴러, 매핑 테이블 등 운영비용과 이로인한 irregular memory access patterns GPU 커널 최적화로 해결하였다.
	    - 논문에선 `4. Implementation, kernel-level optimization`을 보면 된다.
    - 마지막으로 적절한 KV block 사이즈를 찾는 것은, 경험적 실험 결과 공유 및 유저가 변경 가능하도록 파라미터화하였다.

# 4. Conclusion
- 엄청 자세히 읽진 않았고, 키 아이디어만 공유하여 약간은 어설픈 글이 되었다.
	- 사내에서 공유한 것을 북- 긁어다 놔서 더 부족해 보이기도 한다.
	- 그래도 누군가에겐 도움이 되었으면 해서 올린다.
- SOSP에 등재되었다고 하여 기대했는데, novelty가 그리 있진 않아서 살짝 아쉬웠다. 하지만, 그만큼 이미 있는 기술을 잘 적용하는게 매우 중요하다는 것을 다시 한 번 느끼게 되었다.