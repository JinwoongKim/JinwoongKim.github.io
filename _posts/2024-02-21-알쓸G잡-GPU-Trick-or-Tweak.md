---
title: "[알쓸G잡] GPU Trick or Tweak"
categories: GPU
tags:
  - kernel_fusion
  - dynamic_parallelism
  - warp_divergence
  - memory_coalescing
published: true
---

- 앞서 다른 글에서도 언급했듯이 GPU는 구조적으로 CPU와 매우 다르다. [참조](http://jinwoongkim.net/gpu/%EC%95%8C%EC%93%B8G%EC%9E%A1-GPU-%EB%A9%94%EB%AA%A8%EB%A6%AC-%EB%B0%8F-%EC%93%B0%EB%A0%88%EB%93%9C-%EA%B5%AC%EC%A1%B0/#cpu-vs-gpu-%EA%B5%AC%EC%A1%B0-%EB%B9%84%EA%B5%90)
- 따라서, 기존의 CPU 기반 설계에선 문제가 되지 않았던 코드들이 GPU 에서 동작하면서 예기치 못한 성능 저하를 일으키는 경우가 종종 있다.
- 본 글에서는 그러한 성능 저하를 일으키는 여러 현상, 요소들을 어떻게 해결해야 하는지 몇 가지를 꼽아 소개하려고 한다.
	- 처음에 접하면서도 성능에 많은 영향을 주는 것들로 선별하였다. 
- 이해를 돕기 위해 CUDA 코드를 같이 첨부하였는데, 단순 Pseudo-code 라서 복붙한다고 돌아가지는 않는다 ^^
	- 전체 코드를 작성하면 복잡도가 올라가서 이해를 해칠 수 있을 것 같아 단순 하게 작성하였다.

## 1. Minimizing GPU Kernel Launch Overhead
- GPU 프로그래밍에서는 GPU 메모리로 데이터를 옮기는 것 뿐만 아니라, GPU 함수 (GPU Kernel 함수) 를 수행하는 것 또한 성능 저하의 요인이다.
- 그렇기 때문에 이를 해결하기 위해 다양한 최적화 기법이 존재하는데, 그 중 두 가지를 소개 하겠다.
### 1.A. Kernel fusion
- CPU 기반의 코드 들은 가독성 등의 문제로 함수를 잘게 쪼개는 경우가 흔했는데, 이러한 구조는 GPU 함수를 여러 번 호출해야 해서 [SM](http://jinwoongkim.net/gpu/%EC%95%8C%EC%93%B8G%EC%9E%A1-GPU-%EB%A9%94%EB%AA%A8%EB%A6%AC-%EB%B0%8F-%EC%93%B0%EB%A0%88%EB%93%9C-%EA%B5%AC%EC%A1%B0/#streaming-multiprocessor-sm-%EA%B5%AC%EC%A1%B0) 이나 코어 가동률이 낮아지게 되는 경우가 많았다.
- 이러한 문제를 해결 하는 가장 흔한 기법 중 하나는 여러 개의 함수를 하나로 합치는 것이다. 이를 `kernel fusion` 이라는 멋진 이름으로 부르는데, 사실 아이디어 자체는 단순하다.
	- 음식점에서 가위 주세요, 집게 주세요, 두 번 부르지 말고, 한 번에 가위랑 집게 주세요 하는 느낌
- 아래 그림은 `f` 라는 함수에서 `load`, `compute`, `store` 한 변수를 `g` 라는 함수에서 다시 `load`, `compute`, `store` 하는 것을 하나의 함수로 합치는 것을 도식화 하였다.

<p align="center">

<img width="900" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/3f5a2bd4-71ec-42d1-945f-14a6b0dc21f3">

출처 : https://github.com/huggingface/transformers/issues/13845

</p>
- 위의 그림을 간단한 CUDA pseudo-code 로 서술한 것이 아래 코드이다.

```c


#define THREAD_NUM 32

__global__ void f(int *off_chip_array){
    int i = blockDim.x * blockIdx.x + threadIdx.x;
	
	// load sth
	__shared__ int on_chip_array[THREAD_NUM];
	on_chip_array[i] = off_chip_array[i];
	 
	// do sth
	on_chip_array[i] = on_chip_array[i] * on_chip_array[i];
	
	// store sth
	off_chip_array[i] = on_chip_array[i];
}

__global__ void g(int *off_chip_array){
    int i = blockDim.x * blockIdx.x + threadIdx.x;
	
	// load sth
	__shared__ int on_chip_array[THREAD_NUM];
	on_chip_array[i] = off_chip_array[i];
	 
	// do sth
	on_chip_array[i] = on_chip_array[i] ** on_chip_array[i];
	
	// store sth
	off_chip_array[i] = on_chip_array[i];
}

__global__ void f_and_g(int *off_chip_array){
    int i = blockDim.x * blockIdx.x + threadIdx.x;
	
	// load sth
	__shared__ int on_chip_array[THREAD_NUM];
	on_chip_array[i] = off_chip_array[i];
	 
	// do sth
	on_chip_array[i] = on_chip_array[i] * on_chip_array[i];
	
	// do sth more
	on_chip_array[i] = on_chip_array[i] ** on_chip_array[i];
	
	// store sth
	off_chip_array[i] = on_chip_array[i];
}


int main() {
  // blah blah
  f<<<1,THREAD_NUM>>>(off_chip_array); // 블럭 1개, 쓰레드 32개 생성
  g<<<1,THREAD_NUM>>>(off_chip_array); // 블럭 1개, 쓰레드 32개 생성

  // fusion
  f_and_g<<<1,THREAD_NUM>>>(off_chip_array); // 블럭 1개, 쓰레드 32개 생성
  // blah blah..
}
```

- 위의 예제는 너무나도 단순하여, 오히려 이것을 두 개로 나눈 것이 이해가 안 될 정도이다.
- 하지만 실제 CUDA 코드에선 수십, 수백개의 쓰레드가 병렬처리를 비동기적으로 하면서, 여러 계층 및 종류로 나누어진 메모리를 다양하게 접근하므로 복잡도가 훨씬 올라가게 된다.
- 이때, 논리적 사고로는 나뉘어야 하는 코드가 실제 하드웨어에서는 비슷한 역할을 하게 되는 경우를 캐치하여 `kernel fusion` 을 하는 것이 일반적이다. 
- 보통 `kernel fusion` 을 하면 불필요한 Off-chip 메모리 접근이 사라지므로 성능이 매우 많이 올라가게 된다.
- 이러한 `kernel fusion` 도 단점이 있는데, 한 번 합쳤던 코드를 다시 나누기 어려워, 향후 상황에 맞게 기존 코드를 유동적으로 변경하긴 힘들다는 것이다. 또한 하드웨어 및 워크로드 종속적인 코드들을 작성해야해서, 주어진 상황에 맞게 `kernel fusion` 을 다 다르게 해야 하는 경우가 많다.
- 따라서 다시 나누지 않을 것 같은, 폭넓게 적용되는 파트를 `kernel fusion` 하는 것이 일반적이고, 소프트웨어가 어느 정도 성숙하게 되면 `kernel fusion`의 빈도는 점점 낮아지게 된다. 

### 1.B. Dynamic Parallelism
- 또 다른 기법 중 하나는 GPU 커널 함수 안에서 또 다른 커널 함수를 호출 할 수 있게 한 것이다. CPU에서 GPU 커널 함수를 부르는 것 보단, GPU 커널 함수 내에서 다른 GPU 커널 함수를 부르는 것이 오버헤드가 적다.
<p align="center">
<img width="400" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/77f1bd12-c438-4fc1-a72b-bef448c0f472">
<br>
출처 : https://www.sciencedirect.com/topics/computer-science/dynamic-parallelism
</p>
- https://developer.download.nvidia.com/assets/cuda/files/CUDADownloads/TechBrief_Dynamic_Parallelism_in_CUDA.pdf

## 2. Avoid Warp Divergence
- [앞서 설명](http://jinwoongkim.net/gpu/%EC%95%8C%EC%93%B8G%EC%9E%A1-GPU-%EB%A9%94%EB%AA%A8%EB%A6%AC-%EB%B0%8F-%EC%93%B0%EB%A0%88%EB%93%9C-%EA%B5%AC%EC%A1%B0/#3-gpu-thread-hierarchy) 했듯이 GPU는 32개의 쓰레드가 그룹핑되어 Warp라고 불리고, 각 warp는 하나의 명령어를 처리 한다. 
- 이러한 구조를 SIMT (Single Instruction Multiple Threads) 라고 부르는데, 여러 쓰레드가 있어도 warp 당 하나의 명령어 밖에 수행 할 수 없기 때문에, 사실 쓰레드 보다 warp 를 최소 단위로 보는 경우도 많다.
	- 다만 volta 아키텍쳐 부터는 각각의 thread 가 독립적으로 업무를 수행 할 수 있도록 패치가 되었지만, 아직 warp divergence 의 비용이 0이 된 것은 아니다. 그리고 쓰레드 하나하나 다른 업무 줄 거면 CUDA 코딩이 적합한지부터 판단해 봐야 한다.  [참고](https://forums.developer.nvidia.com/t/warp-divergence-in-independent-thread-scheduling/188557/2)

```c
__global__ void non_parallel(){
  int i = blockDim.x * blockIdx.x + threadIdx.x;

  if ( i%2 == 0 ) {
    // do something
  } else {
    // and then do something else
  }
}

__global__ void parallel(){
  int i = blockDim.x * blockIdx.x + threadIdx.x;

  if (i < 32 ) {
    // do something
  } else {
    // do something else in parallel
  }
}

int main() {
  // blah blah
  non_parallel<<<1,64>>>(); // 블럭 1개, 쓰레드 64개 생성
  parallel<<<1,64>>>(); // 블럭 1개, 쓰레드 64개 생성
  // blah blah..
}
```

- 따라서 위와 같이 홀수번(검은색), 짝수번(빨간색) 쓰레드 ID 를 가지는 녀석들을 분기문을 태운다면, 아래 처럼 일부 쓰레드가 코드를 수행할때 나머지 쓰레드가 다른 구문을 수행하는게 아니라, 기다리는 상황이 발생한다.

<p align="center">

<img width="900" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/53604d19-09df-4859-98b4-1881963299ac">

(좌) 일반적으로 생각하는 병렬화 (우) Warp divergence가 발생한 상황 (점선은 아무것도 하지 않는 쓰레드를 뜻함)
</p>

## 3. Access Memory Efficiently
### 3.A. How to achieve memory coalescing
- 모던 CPU, GPU는 128, 256 바이트로 정렬되어서 할당되기 때문에 메모리 할당은 이슈가 아니다. 하지만, 기존 시퀀셜 메모리 엑세스 패턴이 변형 없이 매니 코어 시스템 사용 될 때 성능 저하가 일어난다.
- 예를 들어 아래와 같이 `student` 이라는 구조체가 있고, `age`, `id` 를 변수로 갖고 있다고 생각해보자.

```c
struct student_AoS {
  int age;
  int id;
}

struct student_AoS students[32];

__global__ void access_with_AoS(){
  int i = blockDim.x * blockIdx.x + threadIdx.x;

  if ( students[i].age > 19) {
    printf("id is %d\n", students[i].id);
  }
}

struct student_SoA {
  int age[32];
  int id[32];
}

struct student_SoA students;

__global__ void access_with_SoA(){
  int i = blockDim.x * blockIdx.x + threadIdx.x;

  if ( students.age[i] > 19) {
    printf("id is %d\n", students.id[i]);
  }
}

int main() {
  // blah blah
  access_with_AoS<<<1,32>>>();
  access_with_SoA<<<1,32>>>();
  // blah blah..
}

```

<p align="center">
<img width="900" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/b6c5c2f0-db48-47e8-888a-8bee81d5bd95">
</p>

<p align="center">
<img width="900" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/0ff378b9-4cb5-42de-b925-a305b7f027bd">
</p>

### 3.B. Distributed Shared Memory

<p align="center">
<img width="700" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/ee38e738-6ae4-4852-8fab-a88c4cb971f6">
</p>