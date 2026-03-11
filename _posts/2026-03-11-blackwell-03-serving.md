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
  - Kubernetes
  - Observability
published: true
---

[1편](https://jinwoongkim.net/gpu/blackwell-01-architecture/)에서 B300의 아키텍처가 왜 다른지, [2편](https://jinwoongkim.net/gpu/blackwell-02-hardware/)에서 인프라를 왜 먼저 준비해야 하는지를 다뤘다. 이번 3편에서는 그 위에서 **실제로 서빙을 올리기 위한 소프트웨어 스택**을 왜 지금부터 맞춰야 하는지를 다룬다.

하드웨어가 준비되어도 소프트웨어가 안 맞으면 서빙은 안 뜬다. 특히 B300은 소프트웨어 호환성이 아직 빠르게 변하고 있는 시점이라, 도입 시점에 "어떤 버전의 어떤 프레임워크"를 쓸지 미리 파악하지 않으면 긴급 대응에 시간을 뺏기게 된다.

- 1편: [Blackwell 아키텍처](https://jinwoongkim.net/gpu/blackwell-01-architecture/) — 왜 다른 칩인가
- 2편: [B300 하드웨어/인프라](https://jinwoongkim.net/gpu/blackwell-02-hardware/) — 왜 미리 준비해야 하는가
- **3편 (이 글): 서빙 스택** — 왜 지금부터 맞춰야 하는가

---

# 한 줄 요약

> B300에서 서빙을 올리려면 **CUDA 버전, 프레임워크 버전, 환경변수 하나**까지 맞아야 한다.
> "기존 이미지 그대로 올리면 되겠지"는 통하지 않는다.

---

# 1. SM103 호환성 — 왜 컨테이너 이미지를 전부 다시 빌드해야 하는가

## WHY

1편에서 다룬 SM103 이야기의 실무 연장선이다. Kubeflow/KServe 파이프라인의 **모든 서빙 컨테이너가 SM103을 지원하는 CUDA 런타임 위에서 돌아야** 하기 때문이다.

기존 H100(SM90) 기반으로 빌드한 컨테이너 이미지를 B300에 그대로 올리면:

- cubin이 SM90 전용이면 **커널이 아예 실행되지 않는다**
- PTX가 포함되어 있으면 JIT 컴파일로 돌아가긴 하지만, **첫 기동 시 수 분의 JIT 오버헤드** + 네이티브 대비 성능 저하
- PyTorch/Triton 자체가 SM103을 인식하지 못하는 버전이면 에러로 즉사

이건 vLLM이든 SGLang이든 TensorRT-LLM이든 동일한 문제다. 서빙 프레임워크보다 **더 아래 레이어**(CUDA, Driver, PyTorch)에서 터지기 때문이다.

## WHAT — 최소 요구 스택

| 컴포넌트 | 최소 버전 | 비고 |
|---------|----------|------|
| NVIDIA Driver | 580+ | SM103 네이티브 지원 |
| CUDA Toolkit | 13.0+ | SM103 cubin 생성 가능 |
| PyTorch | nightly 또는 최신 stable | torch 2.9.1의 Triton이 SM103 PTXAS 미지원 이슈 있었음 |
| cuDNN | 9.x+ | Blackwell 최적화 커널 포함 |

## 실무에서는

- KServe의 커스텀 서빙 런타임 이미지를 **CUDA 13.0+ base로 재빌드**
- Kubeflow 파이프라인에서 사용하는 모든 GPU 컨테이너 이미지 점검
- CI/CD 파이프라인의 base image를 갱신하고, `-gencode=arch=compute_100a,code=sm_103` 플래그 추가
- **도입 전에 이미지 빌드 & 테스트를 완료해두면**, GPU 도착 즉시 서빙 가동 가능

---

# 2. NVFP4 활성화 — 왜 환경변수 하나가 성능을 2배로 바꾸는가

## WHY

1편에서 "NVFP4를 못 쓰면 B300을 살 이유가 없다"고 썼다. 이번에는 그걸 **실제로 활성화하는 방법**과 왜 그게 간단하지 않은지를 다룬다.

현재 vLLM에서 B300의 NVFP4 MoE 커널을 쓰려면 명시적으로 환경변수를 설정해야 한다:

```bash
VLLM_USE_FLASHINFER_MOE_FP4=1
```

이 한 줄을 안 넣으면 vLLM이 느린 fallback 경로로 빠진다. FP8로 돌거나, weight-only FP4 경로를 타거나, 최악의 경우 Marlin 커널로 떨어져서 B300의 네이티브 FP4 Tensor Core를 전혀 활용하지 못한다.

SGLang도 마찬가지로 FlashInfer의 Blackwell 전용 FP4 MoE 커널을 사용하며, 버전에 따라 활성화 방식이 다를 수 있다.

**문제는 이런 환경변수와 설정이 빠르게 변하고 있다는 것이다.** 지금 시점의 정보가 도입 시점에는 달라져 있을 수 있으므로, 도입 직전에 각 프레임워크의 Blackwell 관련 릴리즈 노트를 반드시 재확인해야 한다.

## WHAT — NVFP4 서빙의 현재 상태

vLLM 블로그의 DeepSeek 벤치마크 기준:

- **DeepSeek-R1 NVFP4 + TP2**: B300 GPU 2장이면 전체 weight 탑재 가능
- Prefill throughput: H200 대비 **8배**, Mixed-context: H200 대비 **10-20배**
- FP8 대비 GPU 절반으로 더 높은 throughput → **비용 효율이 핵심 차별점**

NVFP4 모델 weight는 NVIDIA가 직접 제공하는 경우가 늘고 있다 (`nvidia/DeepSeek-R1-0528-NVFP4` 등). 자체 모델은 quantization 파이프라인을 별도로 준비해야 한다.

## 실무에서는

- 우리가 서빙하는 모델 목록별로 **NVFP4 weight 존재 여부** 확인
- 자체 모델은 NVFP4 quantization 후 **정확도 검증**(eval benchmark)이 필수
- KServe 서빙 컨테이너의 환경변수에 FP4 관련 설정을 포함하도록 템플릿 갱신
- **NVFP4가 안 되는 모델은 FP8로 서빙하되, GPU 할당 계획을 그에 맞게 조정** (TP degree가 달라지므로)

---

# 3. vLLM vs SGLang — 왜 지금은 둘 다 준비해야 하는가

## WHY

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

## WHAT — 현시점 비교

| 항목 | vLLM | SGLang |
|------|------|--------|
| B300 공식 지원 | O (GB300/B300 검증 완료) | O (GB200/B300 명시) |
| NVFP4 | FlashInfer FP4 MoE 커널 지원 | FlashInfer FP4 커널 지원 |
| Attention 커널 | FlashAttention 3/4, cutlass_mla | FlashInfer, TRT-LLM NSA 커널 |
| Disaggregated serving | 지원 (NVIDIA Dynamo 연동) | 지원 (자체 PD disaggregation) |
| Blackwell 전용 Docker | `lmsysorg/sglang:*-blackwell` | NVIDIA NGC vLLM 이미지 |
| 생태계 | KServe 연동, OpenAI API 호환 | Anthropic API 호환 추가, 커뮤니티 성장 중 |

## 실무에서는

- **현재 Kubeflow/KServe에서 vLLM을 쓰고 있다면, 그대로 B300으로 가져가는 것이 리스크가 적다**
- 단, 성능 테스트 시 SGLang도 병행으로 돌려보고, 모델별로 throughput/latency 비교
- 장기적으로 두 프레임워크를 KServe 커스텀 런타임으로 **모델별로 선택 가능하게** 구성하는 것이 유연하다
- 특히 MoE 모델의 Expert Parallelism 지원 수준은 양쪽 모두 빠르게 변하고 있으므로, 도입 시점 기준으로 재평가 필요

---

# 4. NVIDIA Dynamo — 왜 새로운 레이어가 필요해졌는가

## WHY

vLLM과 SGLang이 "단일 노드 또는 소수 노드에서 모델을 서빙하는 엔진"이라면, NVIDIA Dynamo는 **"클러스터 단위에서 GPU 워커를 동적으로 관리하고 요청을 라우팅하는 오케스트레이션 레이어"**다.

기존에는 이런 역할을 KServe + Istio/Knative 조합으로 했다. 그런데 B300 시대의 서빙은 기존보다 훨씬 복잡해졌다:

- **Disaggregated serving**: prefill과 decode를 다른 GPU에서 돌리면, 그 사이에 KV cache를 옮겨야 한다
- **Dynamic GPU scheduling**: reasoning 모델은 output 길이가 예측 불가능해서, 고정 할당이 비효율적이다
- **KV-aware routing**: 같은 prefix를 가진 요청을 KV cache가 이미 있는 GPU로 보내면 중복 계산을 피할 수 있다

이런 LLM-specific한 스케줄링은 KServe의 일반적인 Knative autoscaling으로는 해결이 안 된다. Dynamo는 이 문제를 정면으로 다룬다.

## WHAT

- NVIDIA가 GTC 2025에서 오픈소스로 공개 (Apache 2.0, Triton Inference Server의 후속)
- 백엔드로 vLLM, SGLang, TensorRT-LLM 모두 지원
- 핵심 기능:
  - **Disaggregated prefill/decode**: prefill GPU와 decode GPU를 분리하여 각각 최적화
  - **KV Cache Manager**: 클러스터 전체의 KV cache를 글로벌하게 추적, NIXL로 고속 전송
  - **Smart Router**: KV cache hit율 + GPU load를 합산한 cost function으로 요청 라우팅
  - **Dynamic GPU allocation**: 부하에 따라 GPU 워커를 동적으로 증감
- Kubernetes 위에서 배포 가능 (AWS EKS, GKE에서 레퍼런스 존재)

## 실무에서는

Dynamo가 지금 당장 필요한가? 솔직히 말하면:

- **단일 노드에서 TP2로 서빙하는 구성이면, 당장은 Dynamo 없이도 된다**
- Dynamo가 진가를 발휘하는 건 **멀티노드 disaggregated serving, 대규모 MoE EP, 트래픽 변동이 큰 프로덕션 환경**
- 하지만 KServe의 autoscaling이 LLM 워크로드에 맞지 않는 문제(QPS 기반 스케일링의 한계)를 이미 겪고 있다면, Dynamo의 LLM-aware scheduling을 검토할 시점

도입 전략으로는:
1. **1차**: vLLM/SGLang + KServe로 B300 서빙 안정화 (기존 파이프라인 이식)
2. **2차**: Disaggregated serving 필요성이 생기면 Dynamo 도입 평가
3. **3차**: Dynamo + Kubernetes로 클러스터 단위 동적 스케줄링

급하게 도입할 필요는 없지만, **존재를 알고 있어야 한다.** "왜 우리 서빙이 prefill에서 병목이지?"라는 질문이 나올 때, disaggregated serving이라는 해법과 Dynamo라는 도구가 있다는 것을 아는 것 자체가 의미가 있다.

---

# 5. KServe/Kubeflow — 왜 기존 파이프라인을 손봐야 하는가

## WHY

KServe가 B300에서 안 돌아가는 것이 아니다. KServe는 서빙 엔진(vLLM/SGLang)을 감싸는 레이어이므로, **엔진이 돌면 KServe도 돈다.** 문제는 디테일에 있다.

B300 도입으로 바뀌는 것들이 KServe 설정 곳곳에 영향을 주기 때문이다:

- **리소스 할당**: GPU당 메모리가 288GB로 늘었으므로, 기존 `resources.limits`의 GPU 수가 바뀐다. TP8이 TP2가 되면 단일 InferenceService가 점유하는 GPU 수가 달라진다
- **컨테이너 이미지**: CUDA 13.0+ 기반 이미지로 교체 필요
- **환경변수**: NVFP4 관련 환경변수 (`VLLM_USE_FLASHINFER_MOE_FP4=1` 등)를 InferenceService spec에 추가
- **Health check**: 모델 로딩 시간이 달라질 수 있으므로 readiness probe timeout 조정 필요
- **Autoscaling 설정**: B300의 높은 throughput에 맞춰 scaling 임계값 재조정 필요

## WHAT — 체크리스트

```yaml
# KServe InferenceService 예시 — B300 대응 포인트
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: llm-serving
spec:
  predictor:
    containers:
    - name: kserve-container
      image: custom-vllm:cuda13.0-sm103  # (1) CUDA 13.0+ 이미지
      env:
      - name: VLLM_USE_FLASHINFER_MOE_FP4  # (2) NVFP4 활성화
        value: "1"
      resources:
        limits:
          nvidia.com/gpu: "2"  # (3) TP8 → TP2로 변경
      readinessProbe:
        initialDelaySeconds: 120  # (4) 모델 로딩 시간 조정
```

## 실무에서는

- KServe manifest 전수 점검: GPU 수, 이미지 태그, 환경변수
- Kubeflow 파이프라인의 GPU 스케줄링 로직이 B300 노드를 올바르게 인식하는지 확인 (node selector, toleration)
- **이 작업은 GPU 없이도 미리 할 수 있다** — manifest 수정, 이미지 빌드, dry-run까지 선행 가능
- GPU가 도착하면 실제 배포 + 벤치마크만 하면 된다

---

# 6. 버전 호환성 매트릭스 — 왜 이게 가장 골치 아픈 문제인가

## WHY

위에서 나온 모든 문제의 근본 원인은 하나다: **B300(SM103)은 아직 소프트웨어 생태계가 안정화되지 않은 하드웨어**이기 때문이다.

실제로 있었던 문제들:

- PyTorch stable (torch 2.9.1+cu130)의 Triton이 SM103용 PTXAS를 지원하지 않아서, nightly에서 먼저 수정됨
- vLLM에서 FP4 MoE 커널이 기본 비활성 상태여서, 명시적 환경변수 없이는 fallback 경로를 탐
- SGLang의 Blackwell 전용 Docker 이미지가 별도 태그(`*-blackwell`)로 분리되어 있어서, 기본 이미지로는 최적 성능 미달

이런 문제들은 시간이 지나면 해결되지만, **도입 시점에 어떤 버전 조합이 안정적인지**는 직접 확인해야 한다.

## WHAT — 현시점 기준 권장 스택

| 컴포넌트 | 권장 | 주의사항 |
|---------|------|---------|
| CUDA | 13.0+ | SM103 네이티브 지원 |
| Driver | 580+ | |
| PyTorch | nightly 또는 최신 stable | 도입 시점에 SM103 지원 여부 재확인 |
| vLLM | 최신 release | NVIDIA NGC 컨테이너 권장 |
| SGLang | 최신 release | `sglang:*-blackwell` 태그 이미지 |
| FlashInfer | Blackwell 지원 버전 | FP4 MoE 커널 포함 여부 확인 |
| NCCL | 시스템 번들 사용 | CUDA 13.0 번들 NCCL 권장 |

**이 표는 오늘 기준이다.** 3개월 후에는 달라져 있을 수 있다. 도입 직전에 각 프레임워크의 GitHub release notes와 NVIDIA NGC 카탈로그를 반드시 재확인해야 한다.

## 실무에서는

- **버전 매트릭스 문서를 만들고 유지보수하는 담당자를 지정**하는 것을 추천
- 도입 전 테스트 환경에서 전체 스택을 한 번 올려보는 "smoke test"가 필수
- 가능하면 클라우드(AWS P6-B300 등)에서 먼저 테스트하고, 검증된 스택을 온프레미스에 이식하는 전략이 리스크를 줄인다

---

---

# 7. DCGM — GPU 헬스 모니터링이 왜 가장 먼저여야 하는가

## WHY

서빙 프레임워크가 죽었을 때, 원인은 세 곳 중 하나다: **소프트웨어 버그, 설정 문제, 또는 GPU 하드웨어 이상.** 셋 중 하드웨어 이상은 DCGM 없이는 소프트웨어 버그처럼 보인다. ECC 에러가 쌓이거나 NVLink bandwidth가 떨어져도, 서빙 로그에는 단순 타임아웃이나 성능 저하로만 보이기 때문이다.

B300처럼 고단가 GPU일수록 하드웨어 이상을 조기 발견하는 것이 중요하다. XID 79(GPU fell off bus)나 XID 48(Double-Bit ECC Error)이 쌓이고 있는데 서빙만 재시작하다가 GPU를 날리는 케이스가 실제로 있다.

## WHAT — 기본 설정

```bash
# 헬스체크 — GPU 도착 즉시 실행
dcgmi discovery -l              # 인식된 GPU 목록
dcgmi diag -r 1                 # 빠른 헬스체크 (~2분)
dcgmi diag -r 3                 # 풀 다이어그노스틱 (~1시간, 출하 전 필수)
dcgmi stats --enable            # per-process GPU 메트릭 활성화

# XID 에러 실시간 확인
dcgmi health -g 0 -w 1
```

**`dcgm-exporter` → Prometheus → Grafana 파이프라인:**

```yaml
# dcgm-exporter DaemonSet (핵심 메트릭 필터)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: dcgm-exporter
spec:
  template:
    spec:
      containers:
      - name: dcgm-exporter
        image: nvcr.io/nvidia/k8s/dcgm-exporter:latest
        env:
        - name: DCGM_EXPORTER_COLLECTORS
          value: /etc/dcgm-exporter/default-counters.csv
        ports:
        - containerPort: 9400   # Prometheus scrape port
```

**반드시 모니터링해야 하는 지표:**

| 지표 | 의미 | 알람 임계값 |
|------|------|------------|
| `DCGM_FI_DEV_GPU_TEMP` | GPU 온도 | > 85°C |
| `DCGM_FI_DEV_ECC_SBE_VOL_TOTAL` | Single-Bit ECC 에러 누적 | > 1000/hr |
| `DCGM_FI_DEV_ECC_DBE_VOL_TOTAL` | Double-Bit ECC 에러 누적 | > 0 (즉시 알람) |
| `DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL` | NVLink 대역폭 | < 이론치 70% |
| `DCGM_FI_DEV_XID_ERRORS` | XID 에러 발생 | > 0 (유형별 대응) |
| `DCGM_FI_DEV_SM_ACTIVE` | SM 활용률 | 서빙 중 낮으면 병목 의심 |

## 실무에서는

- GPU 도착 즉시 `dcgmi diag -r 3` 실행 — 하드웨어 불량을 가장 빨리 잡는 방법
- XID 에러별 대응 매뉴얼을 미리 작성해두기: XID 13(그래픽스 엔진 예외)은 소프트웨어, XID 48/79는 하드웨어 교체 사유
- `dcgm-exporter`는 GPU 서버 세팅과 동시에 올리는 것을 기본값으로
- NVLink bandwidth 저하가 감지되면 `nvidia-smi nvlink -s` 로 포트별 상태 확인

---

# 8. K8s GPU 레이어 — device plugin과 topology 설정

## WHY

K8s에서 `nvidia.com/gpu: 1` 요청이 동작하려면 nvidia-device-plugin이 있어야 하고, B300의 NVLink bandwidth를 온전히 쓰려면 **NUMA topology를 인식하는 스케줄링**이 필요하다. 이 두 레이어가 잘못 세팅되면 GPU는 켜지지만 성능이 안 나온다.

DGX B300 기준으로 GPU와 CPU는 NVLink + PCIe 토폴로지로 연결되어 있는데, NUMA-aware 스케줄링 없이 배포하면 remote memory access가 발생해 latency가 튄다. 특히 TP2+ 설정에서 차이가 두드러진다.

## WHAT — 핵심 설정

```yaml
# nvidia-device-plugin ConfigMap — 자주 놓치는 옵션들
apiVersion: v1
kind: ConfigMap
metadata:
  name: nvidia-device-plugin-config
data:
  config.yaml: |
    version: v1
    flags:
      migStrategy: "none"          # 프로덕션 서빙은 MIG 비추 (아래 설명 참조)
      deviceListStrategy: "envvar"
      deviceIDStrategy: "uuid"
    sharing:
      timeSlicing:
        renameByDefault: false
        resources:
        - name: nvidia.com/gpu
          replicas: 1               # B300은 time-slicing 절대 금지
```

```yaml
# kubelet 설정 — Topology Manager 활성화
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cpuManagerPolicy: static
topologyManagerPolicy: single-numa-node   # NVLink bandwidth 보장 핵심 설정
topologyManagerScope: container
```

```yaml
# GPU 노드 taint & label 예시
apiVersion: v1
kind: Node
metadata:
  labels:
    nvidia.com/gpu.compute.major: "10"    # gpu-feature-discovery 자동 생성
    nvidia.com/gpu.memory: "288000"
    nvidia.com/gpu.product: "B300"
spec:
  taints:
  - key: nvidia.com/gpu
    effect: NoSchedule
```

**MIG에 대해:** B300은 288GB라서 MIG로 쪼개고 싶은 유혹이 있다. 하지만 서빙 프로덕션에서는 비추다. Tensor Parallelism과 충돌하고, FlashAttention이 MIG 인스턴스에서 제 성능을 못 내는 케이스가 아직 있다. MIG는 배치 추론이나 개발 환경 공유용으로 제한하는 것이 안전하다.

## 실무에서는

- `gpu-feature-discovery`를 device-plugin과 함께 배포 — node label 자동 생성
- `nvidia-container-toolkit` 버전이 CUDA 13.0을 지원하는지 확인 (1.17+)
- Topology Manager 활성화 후 기존 파드가 `TopologyAffinityError`로 Pending되는 경우 확인
- 노드 레이블 기반 nodeSelector를 KServe InferenceService에 명시적으로 지정

---

# 9. vLLM/SGLang 프로덕션 파라미터 튜닝

## WHY

"일단 올려보자"로 배포한 기본 설정은 B300의 성능을 절반도 못 뽑는 경우가 많다. `--gpu-memory-utilization`이 너무 높으면 워크로드 중간에 OOM이 터지고, `--max-num-seqs`가 너무 낮으면 B300의 배치 처리 능력을 못 쓴다. 기본값은 일반적인 상황을 위한 것이지, B300의 대용량 메모리와 높은 throughput을 위해 최적화된 것이 아니다.

## WHAT — vLLM 프로덕션 설정

```bash
vllm serve deepseek-r1 \
  --tensor-parallel-size 2 \
  --max-num-seqs 256 \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.90 \
  --enable-chunked-prefill \
  --max-num-batched-tokens 8192 \
  --kv-cache-dtype fp8 \
  --disable-log-requests
```

**파라미터별 의미:**

| 파라미터 | 권장값 | 왜 |
|---------|--------|----|
| `--gpu-memory-utilization` | 0.88~0.92 | 0.95+ 는 KV cache 할당 시 OOM 리스크 |
| `--max-num-seqs` | 128~512 | B300의 배치 처리 능력 활용 — 낮으면 throughput 낭비 |
| `--enable-chunked-prefill` | 항상 활성화 | long-context 요청이 decode를 블록킹하는 문제 방지 |
| `--kv-cache-dtype` | `fp8` | NVFP4가 안 되는 모델의 fallback — KV cache 메모리 절반으로 감소 |
| `--disable-log-requests` | 프로덕션에서 항상 | 요청마다 로그 쓰는 overhead 제거 |

**SGLang 주요 설정:**

```bash
python -m sglang.launch_server \
  --model deepseek-r1 \
  --tp 2 \
  --max-running-requests 512 \
  --mem-fraction-static 0.88 \
  --chunked-prefill-size 8192 \
  --enable-mixed-chunk \
  --disable-radix-cache-rebuild-on-init  # 재시작 시 cold start 단축
```

## 실무에서는

- `--max-num-seqs`와 `--gpu-memory-utilization`은 실측 기반으로 결정 — vLLM benchmark 스크립트로 sweet spot 찾기
- KServe의 readinessProbe `initialDelaySeconds`를 모델 로딩 시간 기반으로 조정 (DeepSeek-R1 NVFP4 기준 60~120초)
- 모델별로 최적 파라미터가 다르므로, **InferenceService 템플릿에 파라미터를 하드코딩하지 말고 ConfigMap으로 분리**
- 프로덕션 배포 전 `locust` 또는 vLLM benchmark로 부하 테스트 필수

---

# 10. NCCL 튜닝 — 멀티노드에서 성능이 안 나올 때

## WHY

TP(Tensor Parallelism)가 단일 노드를 벗어나는 순간, NCCL 설정이 성능을 결정한다. NVLink는 단일 노드 내에서만 쓸 수 있고, 노드 간에는 InfiniBand(또는 RoCE) RDMA를 써야 한다. NCCL이 이 경로를 제대로 잡지 못하면, 노드 간 AllReduce가 TCP를 타거나 잘못된 NIC를 잡아서 bandwidth가 수십 배 떨어진다.

이건 에러로 죽지 않고 **느리게 돌아가기 때문에** 놓치기 쉽다. `NCCL_DEBUG=INFO`로 실행 로그를 보기 전까지는 이유를 모른다.

## WHAT — 필수 환경변수

```bash
# 단일 노드 (NVLink 사용)
export NCCL_P2P_DISABLE=0           # NVLink P2P 반드시 활성화
export NCCL_SHM_DISABLE=0           # Shared Memory 허용

# 멀티노드 (InfiniBand 사용)
export NCCL_IB_DISABLE=0
export NCCL_IB_HCA=mlx5_0,mlx5_1   # 사용할 HCA 명시 (서버마다 다름)
export NCCL_SOCKET_IFNAME=eth0      # 폴백 소켓 인터페이스
export NCCL_IB_GID_INDEX=3          # RoCE v2 사용 시

# 디버깅 시에만
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=INIT,NET
```

**설정 전 반드시 측정:**

```bash
# nccl-tests로 AllReduce bandwidth 측정
git clone https://github.com/NVIDIA/nccl-tests
cd nccl-tests && make

# 단일 노드
./build/all_reduce_perf -b 1G -e 1G -f 2 -g 2

# 멀티노드 (mpirun)
mpirun -np 4 -H node1:2,node2:2 \
  ./build/all_reduce_perf -b 1G -e 1G -f 2 -g 2
```

B300 DGX 기준으로 단일 노드 NVLink 4.0 이론치는 AllReduce ~900GB/s (bisection bandwidth 기준). 실측이 이론치의 80% 미만이면 토폴로지 설정 또는 드라이버 문제다.

## 실무에서는

- 멀티노드 서빙 배포 전 nccl-tests를 자동화 테스트 스위트에 포함
- KServe에서 멀티노드 서빙 시 `NCCL_*` 환경변수를 InferenceService spec에 명시
- InfiniBand 환경이면 `ib_write_bw`, `ib_read_bw`로 fabric 레벨도 검증
- `NCCL_TOPO_DUMP_FILE`로 NCCL이 인식한 토폴로지를 파일로 저장하고 점검

---

# 11. 관측성 스택 — 병목을 찾으려면 무엇을 봐야 하는가

## WHY

"서빙이 느리다"는 말은 원인이 아니다. Prefill이 느린 건지, Decode가 느린 건지, KV cache가 꽉 찬 건지, GPU SM이 놀고 있는 건지 — 지표 없이는 구분이 안 된다. 그리고 이 지표들은 **하나의 대시보드에 모여있어야** 상관관계를 볼 수 있다.

## WHAT — 최소 관측성 스택

```
┌──────────────────────────────────────────────┐
│              Grafana Dashboard               │
├─────────────────┬────────────────────────────┤
│   Prometheus    │     (scrape targets)        │
├─────────────────┼────────────────────────────┤
│  dcgm-exporter  │  vLLM /metrics endpoint    │
│  (GPU 하드웨어)  │  (서빙 레이어 지표)          │
└─────────────────┴────────────────────────────┘
```

**vLLM `/metrics` 핵심 지표:**

| 지표 | 의미 | 해석 |
|------|------|------|
| `vllm:num_requests_running` | 실시간 배치 크기 | 낮으면 under-utilized |
| `vllm:gpu_cache_usage_perc` | KV cache 사용률 | 90%+ 지속되면 OOM 전조 |
| `vllm:time_to_first_token_seconds` | TTFT P99 | Prefill 성능 지표 |
| `vllm:time_per_output_token_seconds` | TPOT P99 | Decode 성능 지표 |
| `vllm:num_requests_waiting` | 큐 대기 요청 수 | 지속 증가 → 용량 부족 |

**Grafana 대시보드 — 최소 패널 구성:**

```
┌──────────────┬──────────────┬──────────────┐
│  GPU SM Util │  TTFT P99    │ Cache Usage  │
├──────────────┼──────────────┼──────────────┤
│ NVLink BW    │  TPOT P99    │ Req Waiting  │
├──────────────┼──────────────┼──────────────┤
│  GPU Temp    │  Throughput  │  XID Errors  │
└──────────────┴──────────────┴──────────────┘
```

**병목 진단 패턴:**

| 증상 | 지표 패턴 | 원인 추정 |
|------|----------|----------|
| Latency 갑자기 튐 | TTFT P99 증가, SM Util 정상 | Prefill 배치 과부하 → chunked-prefill 설정 |
| Throughput이 낮음 | SM Util 낮음, 큐 비어있음 | `max-num-seqs` 너무 작음 |
| OOM으로 재시작 반복 | Cache Usage 90%+ 후 크래시 | `gpu-memory-utilization` 너무 높음 |
| 멀티노드 느림 | TPOT 정상, NVLink BW 낮음 | NCCL 경로 문제 |

## 실무에서는

- GPU 세팅과 동시에 dcgm-exporter + Prometheus + Grafana 파이프라인 구성 — 사후가 아니라 처음부터
- vLLM 서빙 컨테이너에 `--disable-log-stats` 를 **쓰지 말 것** — stats가 꺼지면 `/metrics`도 유의미한 값을 못 냄
- 알람 룰은 최소한 3개만 먼저: `gpu_cache_usage_perc > 90`, `XID_errors > 0`, `TTFT_P99 > SLA 임계값`
- SGLang도 `/metrics` 엔드포인트를 제공하므로, vLLM과 같은 Prometheus 설정 재사용 가능

---

# 정리: 서빙 스택 도입 로드맵

| 단계 | 할 일 | GPU 필요 여부 | 시기 |
|------|-------|-------------|------|
| **0단계** | 버전 매트릭스 조사, 이미지 빌드, manifest 수정, K8s GPU 레이어 설정 준비 | 불필요 | **지금 즉시** |
| **1단계** | 관측성 스택 구성 (dcgm-exporter + Prometheus + Grafana), nccl-tests 기준치 측정 | GPU 도착 즉시 | GPU 도착 직후 |
| **2단계** | 클라우드 B300에서 smoke test (vLLM + SGLang), 프로덕션 파라미터 튜닝 | 클라우드 | GPU 발주 후 |
| **3단계** | 모델별 NVFP4 정확도 검증 & 벤치마크, TTFT/TPOT SLA 설정 | 클라우드 또는 온프레미스 | GPU 도착 전후 |
| **4단계** | KServe 서빙 안정화, 모델별 최적 프레임워크 확정 | 온프레미스 | GPU 도착 후 |
| **5단계** | Disaggregated serving / Dynamo 평가 (필요시) | 온프레미스 | 안정화 후 |

**0단계는 GPU 없이도 할 수 있다.** 지금 당장 시작할 수 있고, 시작해야 한다.

---

# 마치며

세 편에 걸쳐 B300 도입 준비를 정리했다.

- **1편**: 아키텍처를 모르면 성능을 못 뽑는다
- **2편**: 인프라를 안 깔면 GPU를 못 꽂는다
- **3편**: 소프트웨어를 안 맞추면 서빙이 안 뜬다

세 가지가 모두 맞아떨어져야 B300이 제값을 한다. 그리고 세 가지 모두 **GPU가 도착하기 전에 준비할 수 있는 것들**이다.

B300은 서빙 엔지니어에게 기회이기도 하다. NVFP4로 GPU 효율이 크게 올라가고, 288GB 메모리로 모델 배치 유연성이 높아지며, attention 가속으로 long-context 서빙이 현실적이 된다. 이 기회를 온전히 잡으려면, 지금부터 준비해야 한다.

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
