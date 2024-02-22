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
- 이해를 돕기 위해 CUDA 코드를 같이 첨부하였는데, 단순 Pseudo-code 라서 복붙한다고 돌아가지는 않는다 . 전체 코드를 작성하면 복잡도가 올라가서 이해를 해칠 수 있을 것 같아 단순 하게 작성하였다.

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

```cuda
#define THREAD_NUM 32

__global__ void f(int *off_chip_array){
    int i = blockDim.x * blockIdx.x + threadIdx.x;
	
	// load sth
	__shared__ int on_chip_array[THREAD_NUM];
	on_chip_array[i] = off_chip_array[i];
	 
	// do sth
	on_chip_array[i] = on_chip_array[i] * on_chip_array[i];
	
	// store sth
	off_chip_array[i] = on_chip_array[i];
}

__global__ void g(int *off_chip_array){
    int i = blockDim.x * blockIdx.x + threadIdx.x;
	
	// load sth
	__shared__ int on_chip_array[THREAD_NUM];
	on_chip_array[i] = off_chip_array[i];
	 
	// do sth
	on_chip_array[i] = on_chip_array[i] ** on_chip_array[i];
	
	// store sth
	off_chip_array[i] = on_chip_array[i];
}

__global__ void f_and_g(int *off_chip_array){
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
- 따라서 다시 나누지 않을 것 같은 폭넓게 적용되는 함수들을 `kernel fusion` 하는 것이 일반적이고, 소프트웨어가 어느 정도 성숙하게 되면 `kernel fusion`의 빈도는 점점 낮아지게 된다.
	- 물론 성능 향상이 중요한 소프트웨어는 매번 번거롭더라도 최대한 fusion 하는 경우도 있다 (FlashAttention이 대표적)

### 1.B. Dynamic Parallelism
- 또 다른 기법 중 하나는 GPU 커널 함수 안에서 또 다른 커널 함수를 호출 할 수 있게 한 것이다. CPU에서 GPU 커널 함수를 부르는 것 보단, GPU 커널 함수 내에서 다른 GPU 커널 함수를 부르는 것이 오버헤드가 적다.
<p align="center">
<img width="400" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/77f1bd12-c438-4fc1-a72b-bef448c0f472">
<br>
출처 : https://www.sciencedirect.com/topics/computer-science/dynamic-parallelism
</p>
- Dynamic parallelism의 또 다른 장점은 병렬화 레벨을 다양하게 설정 할 수 있다는 것이다. 이 부분은 직접 써보진 않아서 잘 모르겠다. Dynamic parallelism in deep learning 등으로 검색해봐도 잘 안 나와서 그렇게 적극적으로 사용되는 기술은 아닌 것 같다고 생각한다.

<p align="center">
<img width="400" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/47b0361a-fdf3-48a5-8f67-cf46ba65de96">
<br>
출처 : https://hayunjong83.tistory.com/22
</p>

```cuda
__global__ ChildKernel(void* data){
//Operate on data
}
__global__ ParentKernel(void *data){
ChildKernel<<<16, 1>>>(data);
}
// In Host Code
ParentKernel<<<256, 64>>(data);
```
출처 : https://developer.download.nvidia.com/assets/cuda/files/CUDADownloads/TechBrief_Dynamic_Parallelism_in_CUDA.pdf

## 2. Avoid Warp Divergence
- [앞서 설명](http://jinwoongkim.net/gpu/%EC%95%8C%EC%93%B8G%EC%9E%A1-GPU-%EB%A9%94%EB%AA%A8%EB%A6%AC-%EB%B0%8F-%EC%93%B0%EB%A0%88%EB%93%9C-%EA%B5%AC%EC%A1%B0/#3-gpu-thread-hierarchy) 했듯이 GPU는 32개의 쓰레드가 그룹핑되어 Warp라고 불리고, 각 warp는 하나의 명령어를 처리 한다. 
- 이러한 구조를 SIMT (Single Instruction Multiple Threads) 라고 부르는데, 여러 쓰레드가 있어도 warp 당 하나의 명령어 밖에 수행 할 수 없기 때문에, 사실 쓰레드 보다 warp 를 최소 단위로 보는 경우도 많다.
	- 다만 volta 아키텍쳐 부터는 각각의 thread 가 독립적으로 업무를 수행 할 수 있도록 패치가 되었지만, warp divergence 의 비용이 0이 된 것은 아니다 ([참고](https://forums.developer.nvidia.com/t/warp-divergence-in-independent-thread-scheduling/188557/2)). 그리고 쓰레드 하나하나 다른 업무 줄 거면 GPU 같은 매니 코어 아키텍쳐에서 동작하는게 맞는지 부터 확인해보아야 한다. 따라서 GPU 프로그래밍에서 warp divergence 는 계속 주의해야 할 요소이다.
- 아래 코드는 64개의 쓰레드를 생성(Warp 2개)하고 쓰레드 ID 에 따른 분기를 태운 코드이다.
	- 단순화를 위해 블럭은 1개로 고정했다.
- `A` 함수는 쓰레드 ID 의 홀짝을 구분하여 업무를 할당하고, `B` 함수는 쓰레드 ID가 32보다 적은지 구분하여 업무를 할당하고 있다.

```cuda
__global__ void A(){
  int i = threadIdx.x;

  if ( i%2 == 0 ) {
    // do something
  } else {
    // and then do something else
  }
}

__global__ void B(){
  int i = threadIdx.x;

  if (i < 32 ) {
    // do something
  } else {
    // do something else in parallel
  }
}

int main() {
  // blah blah
  A<<<1,64>>>(); // 블럭 1개, 쓰레드 64개 생성
  B<<<1,64>>>(); // 블럭 1개, 쓰레드 64개 생성
  // blah blah..
}
```

- 만약 위의 함수들의 로직이 CPU 에서 동작한다면 두 함수 모두 병렬처리가 될 것이다.
- 하지만, GPU 에선 `A` 함수는 `if` 문과 `else` 문이 순차적으로 실행 되고, `B` 함수는  `if` 문과 `else` 문이 동시에 수행될 것이다.
- 아래 그림에서 홀수번 ID는 검은색, 짝수번 ID 의 쓰레드는 빨간색으로 표현해보았다.
- 일반적으로 우리는 좌측과 같이 동시에 처리 될 것이라고 생각하지만, 앞서 설명했듯이 하나의 warp는 하나의 명령어만 수행하기 때문에 짝수번 ID를 가지는 쓰레드가 작업을 수행할때, 홀수번 ID를 가지는 쓰레드는 상대방을 기다리게 된다.

<p align="center">

<img width="900" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/bf5722eb-b17c-4ef5-88b8-b3d36a67029b">

(좌) 일반적으로 생각하는 병렬화 (우) 실제 GPU에서 동작하는 방식. 같은 warp 내 쓰레드는 분기가 될 수 없는 상황을 보여준다. (점선은 idle 상태를 의미함)
</p>

## 3. Access Memory Efficiently
### 3.A. How to achieve memory coalescing
- 모던 CPU, GPU는 128, 256 바이트로 정렬되어서 주소가 할당되기 때문에 메모리 할당은 이슈가 아니다. 하지만, 기존 순차적 메모리 엑세스 패턴이 변형 없이 매니 코어 시스템 사용 될 때 성능 저하가 일어난다.
- 예를 들어 아래와 같이 `student` 이라는 구조체가 있고, `age`, `id` 를 변수로 갖고 있다고 생각해보자.

```c
struct student {
  int age;
  int id;
}

struct student students[32];
```

- 이러한 코드 구조체는 매우 흔하며, 이러한 구조체들의 배열 (Array of Structure) 형태 또한 일반적인 방식이다.
- 하지만, 병렬 프로세싱에서 이러한 구조는 메모리 버스의 효율을 저하시키는데, 만약 나이가 19살 이상한 학생들에 대해서 어떤 작업을 해줘야 하는 코드를 작성해야 한다 해보자.
- 이때, 순차적으로 접근하면 문제가 되지 않지만, 최소 32개, 최대 수백 수천 개의 쓰레드가 아래의 Array of Structure 처럼 메모리를 접근하게 되면 메모리 버스가 낭비 되게 된다.

<p align="center">
<img width="900" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/0ff378b9-4cb5-42de-b925-a305b7f027bd">
</p>

- 따라서 병렬 프로세싱에선 위 그림의 아래 같이 Structure of Array 의 형태로 변환하는게 일반적이다.
- 아래는 SoA로 변환한 구조체를 접근하는 예제 코드


```cuda
struct student_SoA {
  int age[32];
  int id[32];
}

struct student_SoA students;

__global__ void access_with_SoA(){
  int i = blockDim.x * blockIdx.x + threadIdx.x;

  if ( students.age[i] > 19) {
    printf("id is %d\n", students.id[i]);
  }
}

int main() {
  // blah blah
  access_with_SoA<<<1,32>>>();
  // blah blah..
}
```

### 3.B. Distributed Shared Memory
- Hopper 부터 [Thread block cluster](http://jinwoongkim.net/gpu/알쓸G잡-GPU-메모리-및-쓰레드-구조/#3-gpu-thread-hierarchy) 라는 개념이 새로 생겼는데, 같은 SM 안에 있는 쓰레드 뿐만 아니라 같은 thread block cluster 에 있는 쓰레드 끼리는 shared memory 끼리 통신이 된다는 개념인데, H100 오면 써보고 더 자세히 공유 드리겠다.
- 다만, 이러한 개념이 있다는 것만 기억하시면 될 것 같다.
<p align="center">
<img width="700" alt="image" src="https://github.com/JinwoongKim/JinwoongKim.github.io/assets/12505517/ee38e738-6ae4-4852-8fab-a88c4cb971f6">
</p>