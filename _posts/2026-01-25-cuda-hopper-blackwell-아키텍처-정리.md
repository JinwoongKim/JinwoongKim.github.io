---
title: "CUDA 최신 아키텍처 정리 (Hopper/Blackwell)"
categories: GPU
tags:
  - GPU
  - CUDA
  - Hopper
  - Blackwell
  - H100
  - B200
published: true
---

CUDA 강의 들으면서 추가로 찾아본 내용들을 정리해봤다. 강의 내용 자체보다는 Hopper, Blackwell 등 최신 아키텍처에서 달라진 점 위주로 적어둔다.

# Thread Block Cluster

기존에 배운 GPU 쓰레드 계층은 `Grid → Block → Warp → Thread` 였다.

Hopper 아키텍처부터는 여기에 **Cluster**라는 계층이 추가됐다.

```
기존:     Grid ─────→ Block ───→ Warp ───→ Thread
                        ↓
                       SM

Hopper+:  Grid ─→ Cluster ─→ Block ───→ Warp ───→ Thread
                     ↓
                 여러 SM이 협력
```

Cluster의 특징은 다음과 같다:
- 최대 16개 Block이 하나의 Cluster로 묶임
- **DSMEM (Distributed Shared Memory)**: 다른 SM의 Shared Memory를 직접 접근할 수 있음
- Global Memory 경유 대비 약 7배 정도 빠름 (181 cycles vs 956 cycles)

```cpp
// Hopper에서만 가능한 코드
__global__ __cluster_dims__(4, 1, 1) void cluster_kernel() {
    // 다른 Block의 Shared Memory 직접 접근
    int* remote_smem = cluster.map_shared_rank(local_smem, target_rank);
}
```

FlashAttention-2가 이 기능을 활용해서 성능이 좋아진 것으로 알려져 있다.

참고로 Volta 이전에는 Warp 내 모든 Thread가 같은 PC(Program Counter)를 공유했는데, Volta부터 Independent Thread Scheduling이 도입되면서 각 Thread가 독립적인 PC를 갖게 됐다. 그래서 divergence 페널티가 줄었다.
{: .notice--info}

# 하드웨어 제한 변화

Max threads per block = 1024는 Fermi(2010)부터 안 변했다. 15년째 동일.

|스펙|A100 (sm_80)|H100 (sm_90)|B200 (sm_100)|
|---|---|---|---|
|Max Threads/Block|1024|1024|1024|
|Max Blocks/SM|32|32|32|
|Shared Memory/SM|164 KB|228 KB|228 KB|
|Max Shared/Block|163 KB|227 KB|227 KB|

Thread 제한은 그대로인데, Shared Memory가 늘어난 게 중요하다. Hopper에서 228KB로 늘어나면서 FlashAttention 같은 tiling 알고리즘이 더 큰 tile을 쓸 수 있게 됐다.

## Occupancy에 대해

100% Occupancy가 항상 좋은 건 아니다.

```
Occupancy = Active Warps / Max Warps per SM
```

100% 맞추려면 레지스터를 적게 써야 하는데, 그러면 **register spilling**이 발생한다. 실제로 60-80%가 최적인 경우가 많다.

```cpp
// 요즘은 API가 알아서 최적값을 계산해준다
int blockSize, minGridSize;
cudaOccupancyMaxPotentialBlockSize(&minGridSize, &blockSize, kernel, 0, 0);
```

Nsight Compute로 roofline 분석해보면 무조건 100%가 좋은 게 아니라는 걸 확인할 수 있다.

# CUDA 컴파일 관련

## PTX vs SASS

```
.cu 소스 → PTX (가상 ISA) → SASS (실제 기계어)
              ↑                  ↑
         Forward Compatible   Architecture Specific
```

- **PTX**: NVIDIA의 중간 언어. 미래 GPU에서도 JIT 컴파일로 실행 가능
- **SASS**: 실제 GPU 기계어. 특정 아키텍처에서만 실행

PTX만 배포하면 JIT 컴파일 때문에 첫 실행이 600ms 정도 느려진다. 프로덕션에서는 타겟 아키텍처별 SASS와 미래용 PTX를 같이 넣는 게 좋다.

```bash
# 프로덕션 배포 시 권장 컴파일 옵션
nvcc -gencode arch=compute_80,code=sm_80 \   # A100용 SASS
     -gencode arch=compute_90,code=sm_90 \   # H100용 SASS
     -gencode arch=compute_100,code=sm_100 \ # B200용 SASS
     -gencode arch=compute_100,code=compute_100 \  # 미래용 PTX
     -o myapp kernel.cu
```

## Compute Capability 'a' 접미사

|타겟|의미|호환성|
|---|---|---|
|sm_90|Hopper 표준|Forward Compatible|
|sm_90a|Hopper 전용 기능|H100에서만 실행|
|sm_100|Blackwell 표준|Forward Compatible|
|sm_100a|Blackwell 전용 기능|B200에서만 실행|
|sm_100f|Blackwell 패밀리 (CUDA 12.9)|10.x 내에서 호환|

**'a' 접미사**를 붙이면 TMA(Tensor Memory Accelerator), wgmma 같은 특수 기능을 쓸 수 있다. 대신 해당 GPU에서만 실행된다.

FlashAttention 커널이 sm_90a로 컴파일하는 이유가 TMA를 쓰려고 그런 것이다. 그래서 범용 라이브러리들은 sm_90이랑 sm_90a 둘 다 빌드하는 경우가 많다.

# 메모리 전송 최적화

## cudaMemcpy vs cudaMemcpyAsync

cudaMemcpy는 동기 함수라서 CPU가 블로킹된다.

```cpp
// 느린 방법 (CPU가 매번 대기)
cudaMemcpy(d_a, a, size, cudaMemcpyHostToDevice);  // CPU 블로킹
cudaMemcpy(d_b, b, size, cudaMemcpyHostToDevice);  // CPU 블로킹
kernel<<<...>>>();
cudaMemcpy(c, d_c, size, cudaMemcpyDeviceToHost);  // CPU 블로킹

// 빠른 방법 (비동기 + 스트림)
cudaMemcpyAsync(d_a, a, size, H2D, stream1);  // 논블로킹
cudaMemcpyAsync(d_b, b, size, H2D, stream1);
kernel<<<..., stream1>>>();
cudaMemcpyAsync(c, d_c, size, D2H, stream1);
```

## Pinned Memory

```cpp
// 일반 malloc (pageable) - 느림
int* a = (int*)malloc(size);

// Pinned memory - 빠름 (DMA 직접 전송)
int* a;
cudaMallocHost(&a, size);  // 또는 cudaHostAlloc
```

일반 메모리는 CPU가 임시 버퍼에 복사한 다음 GPU로 전송하는 2단계를 거친다. Pinned 메모리는 GPU가 직접 DMA로 접근해서 약 2배 빠르다. 대신 시스템 메모리 압박이 있을 수 있다.

PyTorch의 `pin_memory=True`가 바로 이 기능이다.
{: .notice--info}

## CUDA Graphs

커널 하나 런치하는 데 약 3-4μs가 걸린다. 작은 커널을 많이 쓰면 오버헤드가 연산보다 커질 수 있다.

```cpp
// CUDA Graph: 여러 커널을 하나의 그래프로 묶어서 한 번에 실행
cudaGraph_t graph;
cudaStreamBeginCapture(stream, cudaStreamCaptureModeGlobal);
    kernel1<<<...>>>();
    kernel2<<<...>>>();
    kernel3<<<...>>>();
cudaStreamEndCapture(stream, &graph);

cudaGraphExec_t instance;
cudaGraphInstantiate(&instance, graph, NULL, NULL, 0);

// 이후 반복 실행 시 오버헤드 거의 없음
cudaGraphLaunch(instance, stream);
```

실제로 ResNet-50 학습이 1.7배, Mask R-CNN 추론이 5배 빨라졌다는 벤치마크가 있다.

PyTorch 2.0의 torch.compile이 내부적으로 CUDA Graphs를 사용한다. 그래서 첫 iteration은 느린데 그 다음부터 빨라지는 것이다.

# 인덱싱 확장

1D 인덱싱만 배웠는데, 실제로는 2D, 3D도 많이 쓴다.

```cpp
// 1D
int idx = threadIdx.x + blockIdx.x * blockDim.x;

// 2D (행렬 연산)
int row = threadIdx.y + blockIdx.y * blockDim.y;
int col = threadIdx.x + blockIdx.x * blockDim.x;
int idx = row * width + col;

// 3D (3D convolution, 볼륨 데이터)
int x = threadIdx.x + blockIdx.x * blockDim.x;
int y = threadIdx.y + blockIdx.y * blockDim.y;
int z = threadIdx.z + blockIdx.z * blockDim.z;
int idx = z * width * height + y * width + x;
```

## GPU Utilization vs SM Utilization

nvidia-smi에 나오는 GPU utilization과 Nsight에서 보는 SM utilization은 다른 개념이다.

- **GPU Utilization**: 시간 기반. GPU가 얼마나 바쁜지
- **SM Utilization**: 리소스 기반. SM이 얼마나 활용되는지
- **Theoretical Occupancy**: 리소스 기반 계산값
- **Achieved Occupancy**: 실제 측정값

## MIG (Multi-Instance GPU)

A100, H100에서 하나의 GPU를 여러 개로 쪼개서 사용할 수 있다.

```bash
# H100을 7개의 독립 GPU로 분할
sudo nvidia-smi mig -cgi 19,19,19,19,19,19,19 -C
```

|구성|SM 수|메모리|용도|
|---|---|---|---|
|1g.10gb|~14 SMs|10 GB|추론 서빙|
|3g.40gb|~42 SMs|40 GB|중형 모델 학습|
|7g.80gb|전체|80 GB|대형 모델 학습|

H100 하나로 작은 추론 서버 7개를 돌릴 수 있다. MIG를 쓰면 GPU 메모리 격리도 되고, 한 인스턴스가 죽어도 다른 건 영향 안 받는다.

# 대용량 데이터 처리

## CUDA Streams로 Overlap

Chunking을 할 때 그냥 순차적으로 하면 느리다. CUDA Streams를 쓰면 전송과 연산을 겹칠 수 있다.

```cpp
for (int i = 0; i < NUM_CHUNKS; i++) {
    int offset = i * CHUNK_SIZE;
    
    // Stream 0: chunk i 전송
    cudaMemcpyAsync(d_a, h_a + offset, size, H2D, stream[i % 2]);
    
    // Stream 1: chunk i-1 계산 (동시에!)
    if (i > 0) kernel<<<..., stream[(i-1) % 2]>>>();
}
```

```
시간 →
Stream 0: [Copy A0][Copy A1][Copy A2]...
Stream 1:          [Compute0][Compute1][Compute2]...
                   ↑ 오버랩!
```

## Unified Memory

명시적 복사 없이 CPU/GPU가 같은 메모리를 접근할 수도 있다.

```cpp
int* data;
cudaMallocManaged(&data, size);  // CPU/GPU 둘 다 접근 가능

kernel<<<...>>>(data);  // GPU가 필요할 때 자동으로 페이지 마이그레이션
```

Grace Hopper에서는 CPU-GPU 간 900 GB/s NVLink-C2C로 연결되어 있어서 더 빠르다.

# 메모리/인터커넥트 스펙 비교

## 메모리 대역폭

|GPU|메모리 종류|용량|대역폭|
|---|---|---|---|
|V100|HBM2|32 GB|900 GB/s|
|A100|HBM2e|80 GB|2.0 TB/s|
|H100|HBM3|80 GB|3.35 TB/s|
|H200|HBM3e|141 GB|4.8 TB/s|
|B200|HBM3e|192 GB|8.0 TB/s|

B200이 A100보다 4배 빠른 이유는 FLOPS보다 메모리 대역폭 차이가 크기 때문이다.

## NVLink 세대별 진화

|세대|GPU|GPU당 대역폭|연결 가능 GPU|
|---|---|---|---|
|NVLink 2.0|V100|300 GB/s|8 (DGX-1)|
|NVLink 3.0|A100|600 GB/s|8 (DGX A100)|
|NVLink 4.0|H100|900 GB/s|8 (DGX H100)|
|NVLink 5.0|B200|1.8 TB/s|72 (DGX GB200 NVL72)|

DGX GB200 NVL72는 72개 GPU가 하나의 메모리처럼 동작하고, 총 130 TB/s 대역폭을 제공한다.

LLM 학습할 때 GPU 8개 넘어가면 NVLink보다 느린 네트워크를 타야 하는데, Blackwell NVL72는 72개가 전부 NVLink로 연결돼서 all-reduce가 훨씬 빠르다.

# Tensor Core 세대별 비교

|세대|아키텍처|지원 정밀도|FP16 TFLOPS|
|---|---|---|---|
|1세대|Volta|FP16|125|
|2세대|Turing|FP16, INT8|130|
|3세대|Ampere|FP16, BF16, TF32, INT8|312|
|4세대|Hopper|+ FP8|2,000|
|5세대|Blackwell|+ FP4|9,000|

FP8이 중요한 이유:
- FP16 대비 2배 throughput, 메모리 절반
- 학습 정확도 거의 동일 (Transformer Engine이 자동 관리)

```python
# PyTorch에서 FP8 사용 (H100+)
import transformer_engine.pytorch as te

model = te.Linear(1024, 1024)  # 자동으로 FP8 사용
```

요즘 LLM 학습은 대부분 FP8을 쓴다. Llama 3 학습도 FP8이었다. FP8은 E4M3이랑 E5M2 두 종류가 있는데, forward는 E4M3, backward는 E5M2 쓰는 게 일반적이다.
