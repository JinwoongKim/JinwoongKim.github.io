---
title: "[알쓸G잡] EFficient GPU?"
categories: GPU
tags:
  - kernel_fusion
  - dynamic_parallelism
  - warp_divergence
  - memory_coalescing
published: true
---

# 1. GPU Trick or Tweak

## A. Minimizing GPU Kernel Launch Overhead
- GPU 프로그래밍에서는 GPU 메모리로 데이터를 옮기는 것 뿐만 아니라, GPU 함수 (GPU Kernel 함수) 를 수행하는 것 또한 성능 저하의 요인이다.
- 그렇기 때문에 이를 해결하기 위해 다양한 최적화 기법이 존재하는데, 그 중 두 가지를 소개 하겠다.
### A.1. Kernel fusion
- 가장 흔한 기법 중 하나는 여러 개의 함수를 하나로 합치는 것이다. 이를 kernel fusion 이라는 멋진 이름으로 부르는데, 사실 아이디어 자체는 단순하다.
- 초기에 CPU 기반의 함수들을 GPU 기반으로 옮기는 작업을 할때, GPU 구조를 고려하지 않은 경우가 많아서 SM이나 코어 가동률이 낮아지게 되는 경우가 많았다.
### A.2. Dynamic Parallelism
- 또 다른 기법 중 하나는 GPU 커널 함수 안에서 또 다른 커널 함수를 호출 할 수 있게 한 것이다. CPU에서 GPU 커널 함수를 부르는 것 보단, GPU 커널 함수 내에서 다른 GPU 커널 함수를 부르는 것이 오버헤드가 적다.
<p align="center">
<img width="400" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/77f1bd12-c438-4fc1-a72b-bef448c0f472">
<br>
출처 : https://www.sciencedirect.com/topics/computer-science/dynamic-parallelism
</p>


## B. Avoid Warp Divergence
- 앞서 설명 했듯이 GPU는 32개의 쓰레드가 그룹핑되어 Warp라고 불림. 
- Warp 는 안에 32개의 쓰레드가 있어도 동시에 하나의 명령어 밖에 수행 할 수 없기 때문에, 사실 쓰레드 보다 warp 를 최소 단위로 보는 경우가 많음

```c
__global__ void test(int *c, int* a, int* b, int num){
  int i = blockDim.x * blockIdx.x + threadIdx.x;

  if i%2 == 0 {
    // do something
  } else {
    // do something else
  }
}

int main() {
  // blah blah
  test<<<1,32>>>(d_c, d_a, d_b, 1); // 블럭 1개, 쓰레드 32개 생성
  // blah blah..
}
```

- 따라서 위와 같이 홀수번(검은색), 짝수번(빨간색) 쓰레드 ID 를 가지는 녀석들을 분기문을 태운다면, 아래 처럼 일부 쓰레드가 코드를 수행할때 나머지 쓰레드가 다른 구문을 수행하는게 아니라, 기다리는 상황이 발생한다.

<p align="center">

<img width="900" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/72719429-a99f-4493-8c19-e000a2a8988c">

(좌) 일반적으로 생각하는 병렬화 (우) Warp divergence가 발생한 상황
</p>

## C. Access Memory Efficiently
### C.1. How to achieve memory coalescing?

<p align="center">
<img width="900" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/b6c5c2f0-db48-47e8-888a-8bee81d5bd95">
</p>

### C.2 Distributed Shared Memory

<p align="center">
<img width="700" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/ee38e738-6ae4-4852-8fab-a88c4cb971f6">
</p>



# 2. 그 외
- 유니파이드 메모리
- 스트림
- 싱크로나이제이션
- 호환성
	- CPU 와의 호환성
	- GPU 와의 호환성
- Warp ILP
- 컨트롤 유닛 연산 관련
- unrolling