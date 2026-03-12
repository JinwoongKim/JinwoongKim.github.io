---
title: "B300 도입 준비: 아키텍처부터 프로덕션 서빙까지"
categories: GPU
tags:
  - GPU
  - CUDA
  - Blackwell
  - B300
  - NVFP4
  - vLLM
  - SGLang
  - KServe
  - Kubeflow
  - NVIDIA-Dynamo
  - LLM-Serving
  - DCGM
  - NCCL
  - Infrastructure
published: true
---

올해 팀에서 B300을 다수 도입할 예정이다. 학습도 하지만 주로 서빙 용도로, vLLM/SGLang/KServe 기반 Kubeflow 파이프라인 위에서 운용하게 된다.

도입 전에 "이 칩이 기존과 뭐가 다른지", "인프라를 왜 미리 준비해야 하는지", "서빙 스택을 어떻게 맞춰야 하는지"를 정리했다.

크게 두 파트로 나눴다:
- **Part 1 — 알고만 있어도 되는 것들**: 아키텍처, 하드웨어 스펙, SW 개념. 우리가 직접 건드리는 영역은 아니지만, 왜 프로덕션 설정이 그렇게 생겼는지 이해하는 데 필요하다.
- **Part 2 — 직접 해야 하는 것들**: DCGM, k8s 설정, vLLM/SGLang 파라미터, NCCL 튜닝, 관측성 스택. GPU 받기 전부터 준비 가능한 것들이다.

---

# 한 줄 요약

> B300은 Hopper의 "더 빠른 버전"이 아니라 **설계 철학 자체가 다른 칩**이다.
> 아키텍처를 모르면 성능을 못 뽑고, 인프라를 안 깔면 GPU를 못 꽂고, 소프트웨어를 안 맞추면 서빙이 안 뜬다.

---

# Part 1 — 알고만 있어도 되는 것들

## 1. 듀얼 다이 (Dual Reticle) — 하나인 척하는 두 개의 칩

B300은 단일 칩(monolithic)이 아니라 **두 개의 다이가 하나의 GPU로 동작**하는 구조다. 겉으로는 하나의 GPU처럼 보이지만, 내부적으로 두 다이 사이에 데이터가 오가는 cross-die 통신이 존재한다.

이걸 모르면 이런 일이 생긴다:

- 커널이 두 다이에 걸쳐 실행될 때 **예상치 못한 latency**가 나타남
- 프로파일링 결과에서 "왜 bandwidth가 이론치보다 낮지?" 하는 의문이 생김

**WHAT**: 2개의 reticle-size 다이, 총 ~208B 트랜지스터. 다이 간 연결은 **NV-HBI (10 TB/s)** — 충분히 빠르지만 0은 아니다. CUDA 레벨에서는 단일 GPU로 추상화된다.

서빙 워크로드에서는 CUDA 추상화가 대부분 처리해준다. 단, **커스텀 커널을 작성하거나 Nsight Compute로 프로파일링할 때** 두 다이 간 데이터 이동 패턴을 이해하고 있어야 병목을 정확히 짚을 수 있다.

---

## 2. NVFP4 — B300을 사는 진짜 이유

B300의 존재 이유가 NVFP4라고 해도 과언이 아니다.

- FP8로 돌리면 B300은 그냥 "약간 빠른 B200"일 뿐이다
- **NVFP4를 써야 비로소 B300의 가격 대비 성능이 정당화된다**

**WHAT**:
- NVFP4: NVIDIA 독자 4-bit 부동소수점 포맷 (IEEE FP4와 다름)
- Dense FP4: B300 ~14 PFLOPS vs B200 ~9 PFLOPS (약 55% 향상)
- FP4 사용 시 **메모리 footprint가 FP8 대비 절반** → 같은 GPU 수로 더 큰 모델 서빙 가능

vLLM 블로그의 DeepSeek-R1 서빙 데이터: B300에서 NVFP4를 사용하면 FP8 대비 GPU를 절반만 써도 더 높은 throughput이 나온다. **NVFP4를 못 쓰면 GPU를 두 배로 사야 같은 성능**이라는 뜻이다.

---

## 3. Attention 가속 2배 — 진짜 병목은 GEMM이 아니었다

Long-context/reasoning 모델에서 attention의 softmax(exponential) 연산이 병목이다. 역대 GPU 세대에서 GEMM 속도는 올라갔지만, **SFU(Special Function Unit)는 거의 제자리**였다.

B300은 이 문제를 정면으로 해결했다. **SFU를 대폭 강화하여 attention 처리량을 2배로 올렸다.**

- DGX B300 기준 attention 성능: DGX B200 대비 2배
- 이 개선은 GEMM FLOPS 수치에는 안 잡힌다 — **스펙시트만 보면 놓치는 개선점**

Long-context 서빙(32K, 128K+)과 reasoning 모델(DeepSeek-R1 등 CoT 계열)에서 TTFT 체감 단축. 반대로 짧은 input/output의 단순 서빙에서는 체감이 크지 않다.

---

## 4. TMEM (Tensor Memory) — 새로운 메모리 계층

**WHAT**: SM당 256KB TMEM — warp-synchronous 중간 결과 저장용. 레지스터와 비슷한 속도, 명시적 할당 필요.

Blackwell 메모리 계층 (빠른 순): **TMEM/Register → Shared Memory → L2 Cache → HBM3e**

직접 CUDA 커널을 작성하지 않는 한 TMEM을 직접 만질 일은 없다. **FlashAttention 4, FlashInfer 같은 최적화 커널들이 TMEM 활용 여부에 따라 성능 차이가 크다.** 서빙 프레임워크를 고를 때 "Blackwell-optimized kernel을 쓰는가?"를 체크해야 하는 근거가 여기에 있다.

---

## 5. FP64 포기 — NVIDIA가 보내는 신호

B200의 FP64 성능은 37 TFLOPS였는데, B300은 **1.25 TFLOPS로 거의 30배 하락**했다.

> "B300은 HPC/과학계산용이 아니라, **AI inference 전용 칩**이다."

B300으로 학습 외에 과학계산이나 시뮬레이션도 돌리려는 계획이 있다면 재고 필요. 우리 팀의 서빙 위주 활용 방향이 B300 설계 방향과 맞다는 확인이기도 하다.

---

## 6. Compute Capability SM103 — 숫자 하나의 무게

소프트웨어 호환성의 생사가 이 숫자에 달려있다:
- H100 = SM90
- B200 = SM100
- **B300 = SM103**

SM90(Hopper)용 cubin은 SM103에서 돌아가지 않는다. PTX가 포함되어 있으면 JIT 컴파일로 돌아가긴 하지만 성능이 최적이 아니다.

**WHAT**:
- CUDA 13.0+에서 SM103 최적 지원, Driver 580+
- `sm_100a`, `compute_100a` 같은 아키텍처 특화 기능은 forward/backward 호환 안 됨

---

## 7. 전력 — 왜 GPU 발주보다 먼저 확인해야 하는가

H100은 GPU당 700W, B300은 **1,400W**다. 정확히 2배.

8-GPU DGX B300 시스템 하나가 피크 기준 약 14kW. 전력 증설은 **리드타임이 가장 긴 작업**이다. 수전 용량 증설은 몇 달에서 반년 이상 걸릴 수 있다.

| 항목 | H100 | B200 | B300 |
|------|------|------|------|
| GPU당 TDP | 700W | 1,000W | **1,400W** |
| 8-GPU 시스템 전체 | ~10-11kW | ~14kW | **~14kW+** |

도입 전에 확인: 랙당 전력 예산, PDU 용량, 증설 필요 시 시설 담당 부서와 **GPU 발주보다 먼저** 협의.

---

## 8. 쿨링 — 왜 에어쿨링이 안 되는가

1,400W짜리 칩 8개가 한 서버에 들어가면, 공기만으로는 열 방출이 물리적으로 불가능하다.

- **DGX B300**: 수랭 필수
- **HGX B300**: 2U 수랭 구성으로 랙당 최대 144 GPU 가능
- 쿨링 없이 GPU를 가동하면 thermal throttling 또는 셧다운

수랭 인프라(CDU, 배관, manifold)는 데이터센터 설계 단계에서 들어가야 한다. 클라우드에서 먼저 B300을 테스트하면서 온프레미스 수랭을 병행 준비하는 전략도 현실적이다.

---

## 9. 네트워크 — 왜 400Gbps로는 부족한가

B300 세대는 **ConnectX-8 SuperNIC (800 Gb/s)** 이다. 기존 H100 세대의 ConnectX-7 (400 Gb/s)으로는 B300의 처리량을 소화하지 못한다.

| 항목 | H100 세대 | B300 세대 |
|------|----------|----------|
| NIC | ConnectX-7 | **ConnectX-8 SuperNIC** |
| 속도 | 400 Gb/s | **800 Gb/s** |
| 스위치 | Quantum-2 | **Quantum-X800** |
| GPU:NIC 비율 | 다양 | **1:1** |
| 노드 내 NVLink | 900 GB/s/GPU | **1.8 TB/s/GPU** |

기존 400 Gb/s InfiniBand를 그대로 쓸 수는 있지만, **B300의 성능을 온전히 활용하려면 800 Gb/s로 업그레이드 필요**. 서빙이 단일 노드 내에서 완결된다면(TP2로 대형 모델 서빙 등) 노드 간 네트워크 병목은 상대적으로 덜하다.

---

## 10. 토폴로지 재설계 — 288GB가 바꾸는 배치 전략

288GB HBM3e가 기존의 모델 배치 전략의 전제를 바꾼다.

- H100 (80GB): 70B 모델 서빙에 TP4~TP8 기본
- **B300 (288GB)**: DeepSeek-R1 (671B MoE) NVFP4 기준으로 GPU 2장이면 전체 weight 탑재

TP degree가 줄어들면 **같은 GPU 수로 더 많은 모델 인스턴스**를 띄울 수 있다. 도입 전에 답해야 할 질문:

- 우리 모델들의 NVFP4 weight size별 최소 TP degree는 얼마인가?
- GPU가 남는다면 추가 replica로 쓸지, 다른 모델에 할당할지?
- MoE 모델은 EP가 유리한지 TP가 유리한지?

---

## 11. 폼팩터 선택 — DGX vs HGX vs GB300 NVL72

| 시스템 | 구성 | 특징 | 적합한 경우 |
|--------|------|------|-----------|
| **DGX B300** | 8x B300, Intel Xeon | NVIDIA 턴키 시스템 | 빠른 도입, 관리 단순화 우선 |
| **HGX B300** | 8x B300, OEM 선택 | CPU/쿨링/섀시 선택 가능 | 유연한 구성, 기존 인프라 활용 |
| **GB300 NVL72** | 72x B300 + 36 Grace CPU | 랙 단위 단일 NVLink 도메인 | 초대형 모델, 하이퍼스케일 |

서빙이 주 용도인 경우 DGX B300 또는 HGX B300 (8-GPU 노드)가 현실적인 선택지다. GB300 NVL72는 수랭 전용에 인프라 요구가 훨씬 크다.

---

## 12. SM103 호환성 — 왜 컨테이너 이미지를 전부 다시 빌드해야 하는가

기존 H100(SM90) 기반으로 빌드한 컨테이너 이미지를 B300에 그대로 올리면:

- cubin이 SM90 전용이면 **커널이 아예 실행되지 않는다**
- PTX가 포함되어 있으면 JIT 컴파일로 돌아가긴 하지만, **첫 기동 시 수 분의 JIT 오버헤드** + 네이티브 대비 성능 저하
- PyTorch/Triton 자체가 SM103을 인식하지 못하는 버전이면 에러로 즉사

### 최소 요구 스택

| 컴포넌트 | 최소 버전 | 비고 |
|---------|----------|------|
| NVIDIA Driver | 580+ | SM103 네이티브 지원 |
| CUDA Toolkit | 13.0+ | SM103 cubin 생성 가능 |
| PyTorch | nightly 또는 최신 stable | torch 2.9.1의 Triton이 SM103 PTXAS 미지원 이슈 있었음 |
| cuDNN | 9.x+ | Blackwell 최적화 커널 포함 |

---

## 13. vLLM vs SGLang — 왜 지금은 둘 다 준비해야 하는가

Hopper 시절에는 vLLM이 사실상 표준이었고, SGLang은 리서치 프로젝트에 가까웠다. B300 시대에는 **판이 바뀌고 있다.**

- B300에서 SGLang이 vLLM보다 전 구간에서 더 높은 throughput을 보이는 경우가 늘고 있다
- vLLM은 생태계(KServe 연동, OpenAI 호환 API)에서 여전히 우위
- 모델에 따라 어느 쪽이 유리한지가 다를 수 있다

**"하나만 쓰면 된다"가 아니라 "둘 다 돌려보고 모델별로 최적을 고르는"** 시대가 됐다.

| 항목 | vLLM | SGLang |
|------|------|--------|
| B300 공식 지원 | O (GB300/B300 검증 완료) | O (GB200/B300 명시) |
| NVFP4 | FlashInfer FP4 MoE 커널 지원 | FlashInfer FP4 커널 지원 |
| Attention 커널 | FlashAttention 3/4, cutlass_mla | FlashInfer, TRT-LLM NSA 커널 |
| Disaggregated serving | 지원 (NVIDIA Dynamo 연동) | 지원 (자체 PD disaggregation) |
| Blackwell 전용 Docker | NVIDIA NGC vLLM 이미지 | `lmsysorg/sglang:*-blackwell` |
| 생태계 | KServe 연동, OpenAI API 호환 | Anthropic API 호환 추가, 커뮤니티 성장 중 |

---

## 14. NVIDIA Dynamo — 왜 새로운 레이어가 필요해졌는가

vLLM/SGLang이 "서빙 엔진"이라면, Dynamo는 **"클러스터 단위 오케스트레이션 레이어"**다.

B300 시대의 서빙이 복잡해진 이유:
- **Disaggregated serving**: prefill과 decode를 다른 GPU에서 돌리면 그 사이에 KV cache를 옮겨야 한다
- **Dynamic GPU scheduling**: reasoning 모델은 output 길이가 예측 불가능해서 고정 할당이 비효율적이다
- **KV-aware routing**: 같은 prefix를 가진 요청을 KV cache가 이미 있는 GPU로 보내면 중복 계산을 피할 수 있다

이런 LLM-specific한 스케줄링은 KServe의 Knative autoscaling으로는 해결이 안 된다.

**Dynamo가 지금 당장 필요한가?** 단일 노드에서 TP2로 서빙하는 구성이면 당장은 없어도 된다. Disaggregated serving, 대규모 MoE EP, 트래픽 변동이 큰 환경에서 진가가 나온다.

---

## 15. 버전 호환성 매트릭스

B300(SM103)은 아직 소프트웨어 생태계가 안정화되지 않은 하드웨어다.

| 컴포넌트 | 권장 | 주의사항 |
|---------|------|---------|
| CUDA | 13.0+ | SM103 네이티브 지원 |
| Driver | 580+ | |
| PyTorch | nightly 또는 최신 stable | 도입 시점에 SM103 지원 여부 재확인 |
| vLLM | 최신 release | NVIDIA NGC 컨테이너 권장 |
| SGLang | 최신 release | `sglang:*-blackwell` 태그 이미지 |
| FlashInfer | Blackwell 지원 버전 | FP4 MoE 커널 포함 여부 확인 |
| NCCL | 시스템 번들 사용 | CUDA 13.0 번들 NCCL 권장 |

> **이 표는 오늘 기준이다.** 도입 직전에 각 프레임워크의 GitHub release notes와 NVIDIA NGC 카탈로그를 반드시 재확인해야 한다.

---

# Part 2 — 직접 해야 하는 것들

## 16. DCGM — GPU 헬스 모니터링

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

B300에서 NVLink 4.0 bandwidth 저하가 감지되면 토폴로지 문제일 가능성이 높다.

---

## 17. K8s GPU 레이어 — device plugin부터 topology까지

### nvidia-device-plugin 설정에서 자주 놓치는 것

```yaml
config:
  flags:
    migStrategy: "mixed"
    deviceListStrategy: "envvar"
    nvidiaDriverRoot: "/run/nvidia/driver"
  sharing:
    timeSlicing:
      resources:
      - name: nvidia.com/gpu
        replicas: 1    # B300은 time-slicing 금지 — 서빙 프로덕션에서 의미 없음
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
- GPU 노드에 taint + toleration 설정
- `nvidia-container-toolkit` 버전 확인 — CUDA 13.0 지원 여부
- **MIG 비추**: 288GB라서 쪼개고 싶은 유혹이 있지만, Tensor Parallelism 충돌 + FlashAttention MIG 미지원 케이스가 아직 있다

---

## 18. vLLM/SGLang 프로덕션 파라미터 튜닝

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

## 19. NCCL 튜닝 — 멀티노드에서 성능이 안 나오면 여기를 봐야 한다

```bash
export NCCL_DEBUG=INFO
export NCCL_IB_DISABLE=0                   # InfiniBand 사용 환경
export NCCL_IB_HCA=mlx5_0,mlx5_1
export NCCL_SOCKET_IFNAME=eth0
export NCCL_P2P_DISABLE=0                 # NVLink P2P 반드시 활성화
export NCCL_TOPO_FILE=/path/to/topo.xml
```

B300 DGX 기준으로 `nccl-tests`로 **AllReduce bandwidth를 먼저 측정**하고, 이론치(NVLink 4.0 ~1.8TB/s bisection) 대비 80% 이상 안 나오면 토폴로지나 드라이버 문제다.

---

## 20. OS/Infra 기본 설정

### 커널 파라미터

```bash
# /etc/sysctl.conf
vm.nr_hugepages = 4096
kernel.numa_balancing = 0       # NUMA auto-balancing 끄기
net.core.rmem_max = 134217728
```

### GPU Persistence Mode

```bash
nvidia-smi -pm 1   # 안 켜면 첫 요청마다 GPU init overhead 발생
# 부팅 시 자동 적용하려면 systemd service로 등록
```

---

## 21. 관측성 스택 — 없으면 병목 찾기 불가능

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

vLLM `/metrics` 엔드포인트에서 핵심 4개:

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
| **0단계** | 전력/쿨링/네트워크 인프라 협의, 버전 매트릭스 조사, 이미지 빌드, manifest 수정, DCGM/관측성 스택 구성 | 불필요 | **지금 즉시** |
| **1단계** | 클라우드 B300에서 smoke test (vLLM + SGLang 양쪽) | 클라우드 | GPU 발주 후 |
| **2단계** | 모델별 NVFP4 정확도 검증 & throughput/latency 벤치마크 | 클라우드 또는 온프레미스 | GPU 도착 전후 |
| **3단계** | KServe 서빙 안정화, 모델별 최적 프레임워크 확정 | 온프레미스 | GPU 도착 후 |
| **4단계** | Disaggregated serving / Dynamo 평가 (필요 시) | 온프레미스 | 안정화 후 |

**0단계는 GPU 없이도 할 수 있다.** 지금 당장 시작할 수 있고, 시작해야 한다.

---

*참고 자료:*
- [NVIDIA Blackwell Ultra Technical Blog](https://developer.nvidia.com/blog/inside-nvidia-blackwell-ultra-the-chip-powering-the-ai-factory-era)
- [NVIDIA Blackwell Compatibility Guide](https://docs.nvidia.com/cuda/blackwell-compatibility-guide/)
- [NVIDIA DGX B300 공식 페이지](https://www.nvidia.com/en-us/data-center/dgx-b300/)
- [vLLM - DeepSeek on GB300 Blog](https://blog.vllm.ai/2026/02/13/gb300-deepseek.html)
- [SGLang GitHub - Blackwell Roadmap](https://github.com/sgl-project/sglang/issues/7227)
- [NVIDIA Dynamo GitHub](https://github.com/ai-dynamo/dynamo)
- [SemiAnalysis InferenceX v2 - Blackwell vs AMD](https://newsletter.semianalysis.com/p/inferencex-v2-nvidia-blackwell-vs)
