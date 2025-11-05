---
title: "[논문리뷰] Zero"
categories: papers
published: false
---
ZeRO: Memory Optimizations Toward Training Trillion Parameter Models
https://arxiv.org/abs/1910.02054

MS에서 19년도에 발표한 논문
당시에 읽어봤을때는 모델쪽을 정말 많이 몰라서(지금도 잘 모르지만;;) 잘 이해가 안됐었다.

요즘 https://huggingface.co/spaces/nanotron/ultrascale-playbook 으로 스터디를 하고 있어서 여기에서 나온 부분 다시 읽어보며 정리

---

기본적으로 Zero의 목적은 메모리 중복 제거를 통한 **메모리 최적화**이다. 이러한 메모리 최적화를 통해서 더 큰 모델을 학습하는 것이 최종 목표. 

하지만, 그렇기 때문에 커뮤니케이션이 늘어날 수가 있다. 따라서 네트워크 성능이 좋지 않은 경우, 더 큰 모델을 돌릴 수 있다는 것은 이점이지만, 학습 시간은 많이 늘어날 것으로 보여진다.

제로는 버전에 따라 아래처럼 파티셔닝을 하는데, 여기서 말하는 파티셔닝은 기본적으로 DP 축을 말하는 것

- ZeRO-1: optimizer state partitioning
- ZeRO-2: optimizer state + gradient partitioning
- ZeRO-3: optimizer state + gradient + parameter partitioning

기본적으로 DP는 GPU 개수에 따르니까, 즉, GPU가 4개 인 경우 DP=4, GPU를 따라서 자른다고 이해하면 됨
즉, GPU가 1개라서 DP가 1인 경우 딱히 나눌게 없고, GPU가 2개라서 DP가 2인 경우, ZeRO-1의 경우 optimizer를 반으로 나눠서 0번 GPU가 반, 1번 GPU가 나머지 반을 갖는 구조


1.  Model’s parameters (half precision; i.e., BF16/FP16): 2Ψ
2.  Model’s gradients (half precision; i.e., BF16/FP16): 2Ψ
3. Model’s parameters in FP32 and optimizer states: 4Ψ+(4Ψ+4Ψ)
4. Model’s gradients in FP32: 4Ψ (optional, only included if we want to accumulate gradients in FP32)

activation을 다루지 않는 것이 이상하다고 생각할 수 있는데, activation의 경우 각 DP 랭크 마다 다 다르기 때문에(파라미터는 같지만 들어오는 시퀀스가 다르기 때문) 중복이 아니라서 제거할게 없기 때문

1이 있는데 왜 3에 파라미터가 또 있지? → 옵티마이저에서 계산을 위해 FP32 파라미터 값이 필요함. 포워드/백워드는 FP16으로 해도 유실이 덜 일어나는데 아담은 아니라고 함;
4Ψ 2 개는 아담 옵티마이저 특성
4의 경우 grad_acc를 하면 4Ψ가 더 필요하다고 하는데, 저거 축적할떄 FP32로 해야 돼기 때문이라고 함. 특이한 사실은 grad_acc 값이 얼마든지 간에 4Ψ만 있으면 충분. 저거 계속 더하다가 나중에 grad_acc 값으로 나누기만 하면 된다고.. 신기.. 그래도 되나 진짜 ㅋㅋ

## 학습 순서
파라미터(FP16) → Forward pass → Activation 생성 → Backward pass → Activation + (이전 레이어)  Gradient를 이용해서 현재 레이어 Gradients생성 → Gradient FP32로 변환 → Optimizer가 states 등을 반영하여 계산 후 FP16으로 변경하여 파라미터에 반영



![[Pasted image 20251103161248.png]]

k는 옵티마이저마다 다름. 아담의 경우 12

## ZeRO - 1

![[Pasted image 20251103181632.png]]


1. 포워드 패스
	1. 걍 하면 됨
2. 백워드 패스
	1. 걍 하면 됨
3. gradient **필요한 만큼만 교환**
	1. Reduce-scatter
	2. 원래는 allreduce를 통해 전체 계산 후 교환
4. 옵티마이저 스텝
	1. 주어진 gradient 기반으로 각자의 parameter 업데이트
5. all-gather
	1. 파라미터 싱크
6. 포워드 패스