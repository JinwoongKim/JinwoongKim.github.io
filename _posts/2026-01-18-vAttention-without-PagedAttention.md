---
title: "[논문리뷰] vAttention: PagedAttention 없이 LLM 서빙하기"
categories: papers
tags:
  - vAttention
  - vLLM
  - LLM
  - GPU
published: true
---

vLLM의 PagedAttention이 메모리 관리를 혁신했지만, 성능 오버헤드가 존재한다는 사실을 알고 계셨나요? Microsoft Research India 팀이 ASPLOS '25에 발표한 vAttention은 같은 목표를 더 효율적인 방법으로 달성합니다.

> **논문**: vAttention: Dynamic Memory Management for Serving LLMs without PagedAttention  
> **학회**: ASPLOS '25  
> **저자**: Microsoft Research India 팀

# 배경 지식

## KV Cache란?

LLM은 토큰을 **한 번에 하나씩** 순차적으로 생성합니다.

```
[프롬프트: "오늘 날씨가"] 
    → 모델 실행 → "좋다"
    → 모델 실행 → "고"  
    → 모델 실행 → "합니다"
```

매번 새 토큰을 생성할 때마다 이전 토큰들의 Key(K), Value(V) 벡터가 필요한데, 이걸 매번 다시 계산하면 낭비입니다. 그래서 **한번 계산한 K, V를 GPU 메모리에 저장**해두고 재사용합니다. 이게 바로 **KV Cache**입니다.

**특성:**
- 토큰당 수십~수백 KB (모델마다 다름)
- iteration마다 정확히 1토큰씩 선형 증가 (예측 가능)
- 추론 시 GPU 메모리의 대부분을 차지

## 메모리 관리가 어려운 이유

핵심 문제는 요청의 **최종 길이를 미리 모른다**는 것입니다.

```
Request A: 프롬프트 100 토큰 → 최종 500 토큰?
Request B: 프롬프트 50 토큰 → 최종 2000 토큰?
```

- 과하게 잡으면: 내부 단편화 → 배치 크기 ↓
- 부족하게 잡으면: OOM 또는 swap 필요

# PagedAttention (vLLM의 접근법)

## 핵심 아이디어

GPU 메모리에서 `cudaMalloc`은 **즉시 물리 메모리를 할당**합니다 (CPU의 demand paging과 다름).

```python
# CPU: 실제 접근할 때 물리 메모리 할당 (lazy)
cpu_buffer = malloc(1GB)

# GPU: 즉시 1GB 전부 할당 (eager)
gpu_buffer = cudaMalloc(1GB)
```

vLLM은 이 문제를 해결하기 위해 **user space에서 직접 블록 단위 메모리 관리**를 구현했습니다.

```
시점 1: 토큰 1~16개 → Block-0 할당
시점 2: 토큰 17~32개 → Block-1 할당
시점 3: 토큰 33~48개 → Block-2 할당
```

## 발생한 문제

각 블록을 따로 `cudaMalloc` 했으니까 **GPU 메모리에서 연속된 위치에 있지 않습니다**.

```
GPU 메모리:
[...][Block-0][...빈 공간...][Block-2][Block-1][...]
      0x1000                  0x5000   0x3000
```

Attention 계산할 때 **점프가 필요**합니다:

```
K[0~15]는 0x1000에서
K[16~31]는 0x3000에서  ← 점프!
K[32~47]는 0x5000에서  ← 또 점프!
```

그래서 **Block-Table**로 주소를 추적해야 합니다:

```c
// PagedAttention 커널
for (i = 0; i < seq_len; i++) {
    block_idx = i / BLOCK_SIZE;
    offset = i % BLOCK_SIZE;
    addr = block_table[block_idx];  // 추가 lookup!
    k = K[addr + offset];
}
```

**결과:** 매 토큰마다 소프트웨어가 주소 계산 → GPU 코어 사이클 소모 → **최대 42% 성능 저하**

# vAttention의 해결책

## 핵심 아이디어

> **가상 메모리는 연속으로 유지하고, 물리 메모리만 동적 할당**

```
Virtual Memory (커널이 보는 것)
[────────── 연속된 큰 공간 ──────────]

              ↓ CUDA VMM API가 매핑

Physical Memory (실제 GPU 메모리)
[Page][Page][...흩어져 있어도 OK...][Page]
```

## 비교

| | PagedAttention | vAttention |
|---|---|---|
| **가상 메모리** | 비연속 | 연속 ✓ |
| **물리 메모리** | 비연속 | 비연속 (동일) |
| **점프 처리** | 소프트웨어 (느림) | 하드웨어/GPU MMU (빠름) |
| **Block-Table** | 필요 | 불필요 |
| **기존 커널 수정** | 필요 | 불필요 ✓ |

## 점프 처리의 차이

**PagedAttention**: 커널 코드가 직접 주소 계산
```c
addr = block_table[block_idx];
k = K[addr + offset];
```

**vAttention**: 하드웨어가 알아서 변환
```c
k = K[i];  // GPU MMU가 가상→물리 변환
```

# 구현 과제와 해결책

## 문제 1: CUDA VMM API가 2MB 페이지만 지원

```
토큰당 KV Cache: ~128KB (Llama-3-8B 기준)
2MB 페이지 쓰면: 토큰 1개만 써도 2MB 점유 → 낭비
```

**해결:** NVIDIA 오픈소스 드라이버 (Unified Memory 관련 부분) 수정해서 **64KB 페이지 지원 추가**

| 기존 CUDA API | vAttention API |
|--------------|----------------|
| `cuMemMap` (2MB) | `vMemMap` (64KB) |
| `cuMemCreate` (2MB) | `vMemCreate` (64KB) |

## 문제 2: API 호출 latency가 높음

`cuMemMap` 한 번 호출: ~40μs → 매 iteration마다 호출하면 오버헤드

**해결:** 
- **백그라운드에서 미리 할당** (compute와 overlap)
- **Deferred reclamation**: 요청 끝나도 바로 회수 안 하고, 다음 요청이 재사용

# 성능 결과

- **Prefill throughput**: PagedAttention 대비 최대 1.26× 향상
- **End-to-end throughput**: 최대 1.23× 향상 (long-context)
- **Portability**: FlashAttention-3 등 새 커널 수정 없이 바로 사용 가능

# 핵심 정리

```
Static Allocation (기존)
  → 메모리 낭비 심함, 배치 크기 작음
  
PagedAttention (vLLM)
  → 동적 할당으로 메모리 효율 ↑
  → 단, user space에서 구현 → 가상 메모리까지 비연속 → 소프트웨어 오버헤드
  
vAttention
  → 동적 할당 (같은 목표)
  → CUDA VMM 활용 (더 나은 구현)
  → 가상 메모리 연속 유지 → 하드웨어가 점프 처리 → 빠름
```

**vAttention = PagedAttention의 "목표"를 더 효율적인 "수단"으로 달성**

# 결론

- PagedAttention의 혁신적인 아이디어는 유지하면서, 구현 방식을 CUDA VMM으로 개선
- 소프트웨어 오버헤드를 하드웨어로 옮겨 성능 향상 (최대 1.26×)
- 기존 커널 코드 수정 없이 사용 가능 → 이식성 ↑
- GPU 시스템 프로그래밍에서 "right level of abstraction"의 중요성을 보여줌

# 참고

- 논문: [arXiv:2405.04437](https://arxiv.org/abs/2405.04437)
- 코드: [github.com/microsoft/vattention](https://github.com/microsoft/vattention)
