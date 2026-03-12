---
title: Blackwell Serving
categories: GPU
tags:
  - GPU
  - B300
  - Blackwell
  - vLLM
  - SGLang
  - KServe
  - Kubeflow
  - NVIDIA-Dynamo
  - LLM-Serving
  - DCGM
  - NCCL
published: false
---

[1편](https://jinwoongkim.net/gpu/blackwell-01-architecture/)에서 B300의 아키텍처가 왜 다른지, [2편](https://jinwoongkim.net/gpu/blackwell-02-hardware/)에서 인프라를 왜 먼저 준비해야 하는지를 다뤘다. 이번 3편에서는 그 위에서 **실제로 서빙을 올리기 위한 소프트웨어 스택**을 다룬다.

크게 두 파트로 나눴다:
- **Part 1 — 알고만 있어도 되는 것들**: 아키텍처 배경지식, 프레임워크 비교, 버전 매트릭스. 우리가 직접 건드리는 영역은 아니지만, 왜 프로덕션 설정이 그렇게 생겼는지 이해하는 데 필요하다.
- **Part 2 — 직접 해야 하는 것들**: DCGM, k8s 설정, vLLM/SGLang 파라미터, NCCL 튜닝, 관측성 스택. GPU 받기 전부터 준비 가능한 것들이다.

- 1편: [Blackwell 아키텍처](https://jinwoongkim.net/gpu/blackwell-01-architecture/) — 왜 다른 칩인가
- 2편: [B300 하드웨어/인프라](https://jinwoongkim.net/gpu/blackwell-02-hardware/) — 왜 미리 준비해야 하는가
- **3편 (이 글): 서빙 스택** — 무엇을 알아야 하고 무엇을 직접 해야 하는가

---

# 한 줄 요약

> B300에서 서빙을 올리려면 **CUDA 버전, 프레임워크 버전, 환경변수 하나**까지 맞아야 한다.
> "기존 이미지 그대로 올리면 되겠지"는 통하지 않는다.

---

# Part 1 — 알고만 있어도 되는 것들

## 1. SM103 호환성 — 왜 컨테이너 이미지를 전부 다시 빌드해야 하는가

1편에서 다룬 SM103 이야기의 실무 연장선이다. Kubeflow/KServe 파이프라인의 **모든 서빙 컨테이너가 SM103을 지원하는 CUDA 런타임 위에서 돌아야** 하기 때문이다.

기존 H100(SM90) 기반으로 빌드한 컨테이너 이미지를 B300에 그대로 올리면:

- cubin이 SM90 전용이면 **커널이 아예 실행되지 않는다**
- PTX가 포함되어 있으면 JIT 컴파일로 돌아가긴 하지만, **첫 기동 시 수 분의 JIT 오버헤드** + 네이티브 대비 성능 저하
- PyTorch/Triton 자체가 SM103을 인식하지 못하는 버전이면 에러로 즉사

이건 vLLM이든 SGLang이든 TensorRT-LLM이든 동일한 문제다. 서빙 프레임워크보다 **더 아래 레이어**(CUDA, Driver, PyTorch)에서 터지기 때문이다.

### 최소 요구 스택

| 컴포넌트 | 최소 버전 | 비고 |
|---------|----------|------|
| NVIDIA Driver | 580+ | SM103 네이티브 지원 |
| CUDA Toolkit | 13.0+ | SM103 cubin 생성 가능 |
| PyTorch | nightly 또는 최신 stable | torch 2.9.1의 Triton이 SM103 PTXAS 미지원 이슈 있었음 |
| cuDNN | 9.x+ | Blackwell 최적화 커널 포함 |

---

## 2. NVFP4 — 왜 환경변수 하나가 성능을 2배로 바꾸는가

1편에서 "NVFP4를 못 쓰면 B300을 살 이유가 없다"고 썼다. 이번에는 그걸 **실제로 활성화하는 방법**과 왜 그게 간단하지 않은지를 다룬다.

현재 vLLM에서 B300의 NVFP4 MoE 커널을 쓰려면 명시적으로 환경변수를 설정해야 한다:

```bash
VLLM_USE_FLASHINFER_MOE_FP4=1
```

이 한 줄을 안 넣으면 vLLM이 느린 fallback 경로로 빠진다. FP8로 돌거나, weight-only FP4 경로를 타거나, 최악의 경우 Marlin 커널로 떨어져서 B300의 네이티브 FP4 Tensor Core를 전혀 활용하지 못한다.

SGLang도 마찬가지로 FlashInfer의 Blackwell 전용 FP4 MoE 커널을 사용하며, 버전에 따라 활성화 방식이 다를 수 있다.

**문제는 이런 환경변수와 설정이 빠르게 변하고 있다는 것이다.** 지금 시점의 정보가 도입 시점에는 달라져 있을 수 있으므로, 도입 직전에 각 프레임워크의 Blackwell 관련 릴리즈 노트를 반드시 재확인해야 한다.

### NVFP4 서빙의 현재 상태

vLLM 블로그의 DeepSeek 벤치마크 기준:

- **DeepSeek-R1 NVFP4 + TP2**: B300 GPU 2장이면 전체 weight 탑재 가능
- Prefill throughput: H200 대비 **8배**, Mixed-context: H200 대비 **10-20배**
- FP8 대비 GPU 절반으로 더 높은 throughput → **비용 효율이 핵심 차별점**

NVFP4 모델 weight는 NVIDIA가 직접 제공하는 경우가 늘고 있다 (`nvidia/DeepSeek-R1-0528-NVFP4` 등). 자체 모델은 quantization 파이프라인을 별도로 준비해야 한다.

---

## 3. vLLM vs SGLang — 왜 지금은 둘 다 준비해야 하는가

Hopper 시절에는 vLLM이 사실상 표준이었고, SGLang은 리서치 프로젝트에 가까웠다. B300 시대에는 **판이 바뀌고 있다.**

최근 벤치마크들을 보면:

- B300에서 SGLang이 vLLM보다 전 구간에서 더 높은 throughput을 보이는 경우가 늘고 있다
- SGLang은 GB200/B300 공식 지원을 명시하고, Blackwell 전용 커널 최적화를 적극적으로 진행 중이다
- SGLang의 RadixAttention이 B300의 대용량 메모리에서 KV cache를 더 효율적으로 관리하는 것으로 보인다

하지만 동시에:

- vLLM은 생태계(KServe 연동, OpenAI 호환 API, 커뮤니티 규모)에서 여전히 우위다
- vLLM도 GB300에서 DeepSeek 서빙 최적화 결과를 공개하며 빠르게 따라잡고 있다
- 모델에 따라 어느 쪽이 유리한지가 다를 수 있다

즉, **"하나만 쓰면 된다"가 아니라 "둘 다 돌려보고 모델별로 최적을 고르는"** 시대가 됐다.

### 현시점 비교

| 항목 | vLLM | SGLang |
|------|------|--------|
| B300 공식 지원 | O (GB300/B300 검증 완료) | O (GB200/B300 명시) |
| NVFP4 | FlashInfer FP4 MoE 커널 지원 | FlashInfer FP4 커널 지원 |
| Attention 커널 | FlashAttention 3/4, cutlass_mla | FlashInfer, TRT-LLM NSA 커널 |
| Disaggregated serving | 지원 (NVIDIA Dynamo 연동) | 지원 (자체 PD disaggregation) |
| Blackwell 전용 Docker | NVIDIA NGC vLLM 이미지 | `lmsysorg/sglang:*-blackwell` |
| 생태계 | KServe 연동, OpenAI API 호환 | Anthropic API 호환 추가, 커뮤니티 성장 중 |

---

## 4. NVIDIA Dynamo — 왜 새로운 레이어가 필요해졌는가

vLLM과 SGLang이 "단일 노드 또는 소수 노드에서 모델을 서빙하는 엔진"이라면, NVIDIA Dynamo는 **"클러스터 단위에서 GPU 워커를 동적으로 관리하고 요청을 라우팅하는 오케스트레이션 레이어"**다.

기존에는 이런 역할을 KServe + Istio/Knative 조합으로 했다. 그런데 B300 시대의 서빙은 기존보다 훨씬 복잡해졌다:

- **Disaggregated serving**: prefill과 decode를 다른 GPU에서 돌리면, 그 사이에 KV cache를 옮겨야 한다
- **Dynamic GPU scheduling**: reasoning 모델은 output 길이가 예측 불가능해서, 고정 할당이 비효율적이다
- **KV-aware routing**: 같은 prefix를 가진 요청을 KV cache가 이미 있는 GPU로 보내면 중복 계산을 피할 수 있다

이런 LLM-specific한 스케줄링은 KServe의 일반적인 Knative autoscaling으로는 해결이 안 된다. Dynamo는 이 문제를 정면으로 다룬다.

- NVIDIA가 GTC 2025에서 오픈소스로 공개 (Apache 2.0, Triton Inference Server의 후속)
- 백엔드로 vLLM, SGLang, TensorRT-LLM 모두 지원
- 핵심 기능:
  - **Disaggregated prefill/decode**: prefill GPU와 decode GPU를 분리하여 각각 최적화
  - **KV Cache Manager**: 클러스터 전체의 KV cache를 글로벌하게 추적, NIXL로 고속 전송
  - **Smart Router**: KV cache hit율 + GPU load를 합산한 cost function으로 요청 라우팅
  - **Dynamic GPU allocation**: 부하에 따라 GPU 워커를 동적으로 증감

**Dynamo가 지금 당장 필요한가?** 솔직히 말하면:

- **단일 노드에서 TP2로 서빙하는 구성이면, 당장은 Dynamo 없이도 된다**
- Dynamo가 진가를 발휘하는 건 **멀티노드 disaggregated serving, 대규모 MoE EP, 트래픽 변동이 큰 프로덕션 환경**
- 하지만 KServe의 autoscaling이 LLM 워크로드에 맞지 않는 문제를 이미 겪고 있다면, 도입 검토 시점이다

---

## 5. 버전 호환성 매트릭스

B300(SM103)은 아직 소프트웨어 생태계가 안정화되지 않은 하드웨어다. 실제로 있었던 문제들:

- PyTorch stable의 Triton이 SM103용 PTXAS를 지원하지 않아서, nightly에서 먼저 수정됨
- vLLM에서 FP4 MoE 커널이 기본 비활성 상태여서, 환경변수 없이는 fallback 경로를 탐
- SGLang의 Blackwell 전용 Docker 이미지가 별도 태그(`*-blackwell`)로 분리되어 있어서, 기본 이미지로는 최적 성능 미달

### 현시점 기준 권장 스택

| 컴포넌트 | 권장 | 주의사항 |
|---------|------|---------|
| CUDA | 13.0+ | SM103 네이티브 지원 |
| Driver | 580+ | |
| PyTorch | nightly 또는 최신 stable | 도입 시점에 SM103 지원 여부 재확인 |
| vLLM | 최신 release | NVIDIA NGC 컨테이너 권장 |
| SGLang | 최신 release | `sglang:*-blackwell` 태그 이미지 |
| FlashInfer | Blackwell 지원 버전 | FP4 MoE 커널 포함 여부 확인 |
| NCCL | 시스템 번들 사용 | CUDA 13.0 번들 NCCL 권장 |

> **이 표는 오늘 기준이다.** 3개월 후에는 달라져 있을 수 있다. 도입 직전에 각 프레임워크의 GitHub release notes와 NVIDIA NGC 카탈로그를 반드시 재확인해야 한다.

---

# Part 2 — 직접 해야 하는 것들

## 6. DCGM — GPU 헬스 모니터링

프로덕션에서 GPU가 죽으면 제일 먼저 DCGM을 봐야 한다. 설정하지 않으면 장애 원인 파악 자체가 불가능하다.

### 기본 설치 & 헬스체크

```bash
dcgmi discovery -l
dcgmi diag -r 1       # 빠른 헬스체크
dcgmi diag -r 3       # 풀 다이어그노스틱 (GPU 입고 시 반드시 실행)
dcgmi stats --enable  # per-process GPU 메트릭 수집
```

### Prometheus 연동

```bash
# dcgm-exporter 배포 (DaemonSet)
helm install dcgm-exporter nvidia/dcgm-exporter \
  --set serviceMonitor.enabled=true
```

### 핵심 알람 지표

| 지표 | 의미 | 알람 조건 |
|------|------|----------|
| `DCGM_FI_DEV_GPU_TEMP` | GPU 온도 | > 85°C |
| `DCGM_FI_DEV_ECC_SBE_VOL_TOTAL` | Single-bit ECC 에러 누적 | 급격히 증가 시 |
| `DCGM_FI_DEV_ECC_DBE_VOL_TOTAL` | Double-bit ECC 에러 | > 0 이면 교체 검토 |
| `DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL` | NVLink 대역폭 | 이론치 80% 이하 시 |
| `DCGM_FI_DEV_XID_ERRORS` | XID 에러 | XID 79(GPU fell off bus), XID 48(DBE) 즉시 대응 |

B300에서 NVLink 4.0 bandwidth 저하가 감지되면 토폴로지 문제일 가능성이 높다. DCGM이 없으면 이걸 서빙 성능 저하로 뒤늦게 발견하게 된다.

---

## 7. K8s GPU 레이어 — device plugin부터 topology까지

### nvidia-device-plugin 설정에서 자주 놓치는 것

```yaml
config:
  flags:
    migStrategy: "mixed"          # MIG 혼용 환경 시
    deviceListStrategy: "envvar"
    nvidiaDriverRoot: "/run/nvidia/driver"
  sharing:
    timeSlicing:
      resources:
      - name: nvidia.com/gpu
        replicas: 1               # B300은 time-slicing 금지 — 서빙 프로덕션에서 의미 없음
```

### Topology Manager — NVLink bandwidth 보장을 위해 필수

```yaml
# kubelet config
topologyManagerPolicy: single-numa-node
cpuManagerPolicy: static
```

`single-numa-node`를 안 쓰면 GPU와 CPU가 다른 NUMA 노드에 할당될 수 있다. NVLink bandwidth를 제대로 쓰려면 GPU-CPU-메모리가 같은 NUMA 노드에 있어야 한다.

### 체크리스트

- `gpu-feature-discovery` DaemonSet 배포 → node label 자동 생성 (`nvidia.com/gpu.compute.major=10`)
- GPU 노드에 taint + toleration 설정 (다른 워크로드가 GPU 노드에 스케줄링되는 것 방지)
- `nvidia-container-toolkit` 버전 확인 — CUDA 13.0 지원 여부
- **MIG 비추**: B300은 288GB라서 MIG로 쪼개고 싶은 유혹이 있지만, Tensor Parallelism 충돌 + FlashAttention MIG 미지원 케이스가 아직 있다. 서빙 프로덕션이면 MIG 쓰지 말 것

---

## 8. vLLM/SGLang 프로덕션 파라미터 튜닝

### vLLM 런 예시

```bash
vllm serve deepseek-r1 \
  --tensor-parallel-size 2 \
  --max-num-seqs 256 \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.90 \     # 0.95 이상이면 OOM 리스크
  --enable-chunked-prefill \          # long context에서 prefill latency 분산
  --max-num-batched-tokens 8192 \
  --kv-cache-dtype fp8 \              # NVFP4 안 되는 모델 fallback
  --disable-log-requests              # 프로덕션에서 로그 overhead 제거
```

### 자주 실수하는 것들

- `--gpu-memory-utilization` 너무 높게 잡으면 KV cache OOM이 워크로드 중간에 터진다
- `--max-num-seqs`가 너무 작으면 B300의 throughput을 못 뽑는다 — 배치 사이즈가 핵심이다
- chunked prefill 비활성 시 long-context 요청 하나가 decode 전체를 블록킹한다

### KServe InferenceService에 반영할 것

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: llm-serving
spec:
  predictor:
    containers:
    - name: kserve-container
      image: custom-vllm:cuda13.0-sm103       # (1) CUDA 13.0+ 이미지
      env:
      - name: VLLM_USE_FLASHINFER_MOE_FP4     # (2) NVFP4 활성화
        value: "1"
      resources:
        limits:
          nvidia.com/gpu: "2"                 # (3) TP8 → TP2로 변경 (288GB 덕분)
      readinessProbe:
        initialDelaySeconds: 120              # (4) 모델 로딩 시간 조정
```

---

## 9. NCCL 튜닝 — 멀티노드에서 성능이 안 나오면 여기를 봐야 한다

```bash
export NCCL_DEBUG=INFO                     # 초기 디버깅 시
export NCCL_IB_DISABLE=0                   # InfiniBand 사용 환경
export NCCL_IB_HCA=mlx5_0,mlx5_1         # NIC 바인딩 명시
export NCCL_SOCKET_IFNAME=eth0            # 소켓 인터페이스
export NCCL_P2P_DISABLE=0                # NVLink P2P 반드시 활성화
export NCCL_TOPO_FILE=/path/to/topo.xml   # 커스텀 토폴로지 (필요 시)
```

B300 DGX 기준으로 `nccl-tests`로 **AllReduce bandwidth를 먼저 측정**하고, 이론치(NVLink 4.0 ~1.8TB/s bisection) 대비 80% 이상 안 나오면 토폴로지나 드라이버 문제다.

---

## 10. OS/Infra 기본 설정

### 커널 파라미터

```bash
# /etc/sysctl.conf
vm.nr_hugepages = 4096          # pinned memory 성능 향상
kernel.numa_balancing = 0       # NUMA auto-balancing 끄기 (GPU는 직접 관리)
net.core.rmem_max = 134217728   # RDMA 네트워크 버퍼
```

### GPU Persistence Mode — 안 켜면 첫 요청마다 GPU init overhead 발생

```bash
nvidia-smi -pm 1
# 부팅 시 자동 적용을 원하면 systemd service로 등록
```

---

## 11. 관측성 스택 — 없으면 병목 찾기 불가능

```
┌─────────────────────────────────────┐
│         Grafana Dashboard           │
├────────────┬────────────────────────┤
│ Prometheus │  (scrape targets)      │
├────────────┼────────────────────────┤
│dcgm-exporter│ vLLM /metrics        │
│ (GPU 지표) │ (throughput/latency)  │
└────────────┴────────────────────────┘
```

vLLM `/metrics` 엔드포인트에서 아래 4개가 핵심이다:

| 지표 | 의미 |
|------|------|
| `vllm:num_requests_running` | 실시간 배치 크기 |
| `vllm:gpu_cache_usage_perc` | KV cache 사용률 |
| `vllm:time_to_first_token_seconds` | TTFT P99 |
| `vllm:time_per_output_token_seconds` | TPOT P99 |

**이 4개 + DCGM GPU util + NVLink bandwidth**를 하나의 대시보드에 올려놓는 것이 프로덕션 최소 요건이다.

---

# 도입 로드맵

| 단계 | 할 일 | GPU 필요 여부 | 시기 |
|------|-------|-------------|------|
| **0단계** | 버전 매트릭스 조사, 이미지 빌드, manifest 수정, DCGM/관측성 스택 구성 | 불필요 | **지금 즉시** |
| **1단계** | 클라우드 B300에서 smoke test (vLLM + SGLang 양쪽) | 클라우드 | GPU 발주 후 |
| **2단계** | 모델별 NVFP4 정확도 검증 & throughput/latency 벤치마크 | 클라우드 또는 온프레미스 | GPU 도착 전후 |
| **3단계** | KServe 서빙 안정화, 모델별 최적 프레임워크 확정 | 온프레미스 | GPU 도착 후 |
| **4단계** | Disaggregated serving / Dynamo 평가 (필요 시) | 온프레미스 | 안정화 후 |

**0단계는 GPU 없이도 할 수 있다.** 지금 당장 시작할 수 있고, 시작해야 한다.

---

# 마치며

세 편에 걸쳐 B300 도입 준비를 정리했다.

- **1편**: 아키텍처를 모르면 성능을 못 뽑는다
- **2편**: 인프라를 안 깔면 GPU를 못 꽂는다
- **3편**: 소프트웨어를 안 맞추면 서빙이 안 뜬다

세 가지가 모두 맞아떨어져야 B300이 제값을 한다. 그리고 세 가지 모두 **GPU가 도착하기 전에 준비할 수 있는 것들**이다.

---

*참고 자료:*
- [vLLM - DeepSeek on GB300 Blog](https://blog.vllm.ai/2026/02/13/gb300-deepseek.html)
- [SGLang GitHub - Blackwell Roadmap](https://github.com/sgl-project/sglang/issues/7227)
- [SGLang - GB200 Deployment Blog](https://lmsys.org/blog/2025-06-16-gb200-part-1/)
- [NVIDIA Dynamo 공식 페이지](https://www.nvidia.com/en-us/ai/dynamo/)
- [NVIDIA Dynamo Technical Blog](https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/)
- [NVIDIA Dynamo GitHub](https://github.com/ai-dynamo/dynamo)
- [NVIDIA Blackwell Compatibility Guide](https://docs.nvidia.com/cuda/blackwell-compatibility-guide/)
- [B300 Software Stack (Verda)](https://verda.com/blog/nvidia-b200-and-b300-gpu-architecture-and-software-stack)
- [SemiAnalysis InferenceX v2 - Blackwell vs AMD](https://newsletter.semianalysis.com/p/inferencex-v2-nvidia-blackwell-vs)
