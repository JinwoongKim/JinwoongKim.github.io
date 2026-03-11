---
title: Blackwell Hardware
categories: GPU
tags:
  - GPU
  - B300
  - Blackwell
  - Infrastructure
  - LLM-Serving
  - DataCenter
published: true
---

[1편](/blackwell-b300-왜-알아야-하는가)에서 B300이 Hopper와 설계 철학부터 다른 칩이라는 이야기를 했다. 이번 글에서는 그 칩을 **물리적으로 운용하기 위해** 왜 미리 준비해야 하는지를 다룬다.

GPU를 주문하고 나서 인프라를 맞추려 하면 늦다. 전력, 쿨링, 네트워크, 토폴로지 — 이 중 하나라도 빠지면 B300은 박스에서 꺼낼 수조차 없다.

- 1편: [Blackwell 아키텍처](/blackwell-b300-왜-알아야-하는가) — 왜 다른 칩인가
- **2편 (이 글): B300 하드웨어/인프라** — 왜 미리 준비해야 하는가
- 3편: 서빙 스택 (vLLM/SGLang/KServe) — 왜 지금부터 맞춰야 하는가

---

# 한 줄 요약

> GPU를 사는 건 돈 문제지만, GPU를 꽂을 수 있느냐는 **인프라 문제**다.
> B300은 역대 GPU 중 가장 큰 인프라 격차를 요구한다.

---

# 1. 전력 — 왜 "와트"가 첫 번째 체크 항목인가

## WHY

H100은 GPU당 700W였다. B300은 **1,400W**다. 정확히 2배.

8-GPU DGX B300 시스템 하나가 피크 기준 약 14kW를 먹는다. CPU, 메모리, 네트워킹까지 포함하면 그 이상이다. 같은 랙에 H100 서버 4대를 넣던 전력 예산으로는 B300 서버 2대도 힘들 수 있다.

이게 왜 "제일 먼저" 체크해야 하는 항목인가 하면:

- 전력 증설은 **리드타임이 가장 긴 작업**이다. 수전 용량 증설은 몇 달에서 반년 이상 걸릴 수 있다
- PDU(전력 분배 장치) 용량이 부족하면 물리적으로 서버를 켤 수 없다
- 전력 비용은 GPU 가동 비용의 상당 부분을 차지한다 — 전력 효율 계획 없이 도입하면 TCO가 예상을 초과한다

## WHAT

| 항목 | H100 | B200 | B300 |
|------|------|------|------|
| GPU당 TDP | 700W | 1,000W | **1,400W** |
| 8-GPU 시스템 전체 | ~10-11kW | ~14kW | **~14kW+** |

- DGX B300은 AC/PDU 방식과 DC/busbar 방식 두 가지 전원 옵션을 지원한다
- PSU 12개로 N+N 이중화 구성, 최소 6개 이상 필요
- 향후 Rubin 세대(2027)는 랙당 250-900kW까지 예측되고 있어, 전력 설계를 이번에 여유 있게 잡아두는 것이 장기적으로 유리하다

## 실무에서는

도입 전에 확인해야 할 것:
- 현재 랙당 전력 예산이 얼마인가?
- PDU가 14kW+ 단일 서버를 감당할 수 있는가?
- 전력 증설이 필요하다면 시설 담당 부서와 **GPU 발주보다 먼저** 협의를 시작해야 한다

---

# 2. 쿨링 — 왜 에어쿨링이 안 되는가

## WHY

1,400W짜리 칩 8개가 한 서버에 들어가면, 발열량이 물리적으로 공기만으로는 해결이 안 되기 때문이다.

Supermicro의 DLC-2 기술 기준으로 수랭(Direct Liquid Cooling)은 열의 98%를 액체로 제거한다. 에어쿨링으로는 이 수준의 열 방출이 불가능하다. 참고로 기존 엔터프라이즈 랙의 일반적인 발열 한계는 10-15kW 수준인데, B300 서버 하나가 이미 그 수준이다.

이게 왜 미리 준비해야 하는 문제인가:

- 수랭 인프라(CDU, 배관, manifold 등)는 **데이터센터 설계 단계에서 들어가야** 한다
- 기존 에어쿨링 데이터센터에 수랭을 추가하는 것은 가능하지만, 시간과 비용이 크다
- 쿨링 없이 GPU를 가동하면 thermal throttling으로 성능이 떨어지거나, 아예 셧다운된다

## WHAT

- **DGX B300**: 수랭 필수 (4U 수랭 / 8U 에어쿨링 옵션이 있으나, 에어쿨링은 TDP 제한)
- **HGX B300 (Supermicro)**: 2U 수랭 구성으로 랙당 최대 144 GPU (18노드) 가능
- CDU(Coolant Distribution Unit): Supermicro 기준 1.8MW 용량 CDU로 랙 단위 쿨링
- blind-mate manifold 연결 방식으로 노드 교체 시 배관 분리 불필요

## 실무에서는

에어쿨링만 있는 기존 IDC에 B300을 넣으려면:
- 수랭 CDU 설치 공간과 배관 경로 확보 필요
- 냉각수 공급/배출 용량 산정 필요
- 혹은 수랭이 준비된 코로케이션/클라우드를 대안으로 검토

솔직히 온프레미스에 수랭을 처음 도입하는 거라면 이 부분이 **가장 리드타임이 길고 비용이 큰 항목**이 될 수 있다. 클라우드에서 먼저 B300을 테스트하면서 온프레미스 수랭을 병행 준비하는 전략도 현실적이다.

---

# 3. 네트워크 — 왜 400Gbps로는 부족한가

## WHY

B300의 GPU 간 통신 속도가 올라갔기 때문에, **네트워크가 따라오지 못하면 GPU가 놀게 된다.**

HGX B300은 GPU당 1.8 TB/s NVLink 5 대역폭을 제공하고, 노드 간 통신은 ConnectX-8 SuperNIC으로 **800 Gb/s InfiniBand 또는 Ethernet**을 사용한다. 기존 H100 세대의 ConnectX-7 (400 Gb/s)으로는 B300의 처리량을 소화하지 못한다.

왜 이게 서빙에서도 중요한가:

- 멀티노드 서빙(TP/EP가 노드를 넘어가는 경우)에서 네트워크 대역폭이 직접적인 throughput 병목
- Disaggregated serving (prefill/decode 분리)을 하려면 노드 간 KV cache 전송이 빈번해지는데, 이때 네트워크가 느리면 전체 파이프라인이 막힌다
- GPU-NIC 비율이 1:1 (8 GPU : 8 NIC)로 설계되어 있어, NIC 수가 부족하면 비대칭 병목 발생

## WHAT

| 항목 | H100 세대 | B300 세대 |
|------|----------|----------|
| NIC | ConnectX-7 | **ConnectX-8 SuperNIC** |
| 속도 | 400 Gb/s | **800 Gb/s** |
| 스위치 | Quantum-2 | **Quantum-X800** |
| GPU:NIC 비율 | 다양 | **1:1 (GPU당 1 NIC)** |
| 노드 내 NVLink | 900 GB/s/GPU | **1.8 TB/s/GPU** |

- B300에서는 NIC이 GPU tray에 직접 통합되어 있고, PCIe Gen6로 GPU와 직결
- Leaf-spine 스위치 토폴로지 설계 시 800 Gb/s 대역폭 기준으로 재산정 필요

## 실무에서는

- 기존 400 Gb/s InfiniBand 패브릭을 그대로 쓸 수는 있지만, **B300의 성능을 온전히 활용하려면 800 Gb/s로 업그레이드 필요**
- 스위치, 케이블(광모듈 포함), NIC 전부 세대 교체가 필요하므로 비용과 리드타임 산정이 필수
- 서빙이 단일 노드 내에서 완결된다면(TP2로 대형 모델 서빙 등) 노드 간 네트워크 병목은 상대적으로 덜하다. 하지만 클러스터 스케줄링(KServe/Kubeflow)에서 노드 간 통신은 여전히 발생

---

# 4. 토폴로지 재설계 — 왜 기존 GPU 배치 전략이 안 통하는가

## WHY

288GB HBM3e라는 메모리 용량이 **기존의 모델 배치 전략의 전제를 바꾸기** 때문이다.

H100 (80GB) 시절에는 70B 모델을 서빙하려면 TP4~TP8이 기본이었고, MoE 모델은 더 많은 GPU가 필요했다. H200 (141GB)에서 좀 나아졌지만 여전히 대형 모델은 멀티 GPU가 필수였다.

B300은 GPU당 288GB이다. 8-GPU 노드면 **총 2.3TB**다. 이게 의미하는 것:

- DeepSeek-R1 (671B MoE) NVFP4 기준으로 **GPU 2장이면 전체 weight가 올라간다**
- 70B dense 모델은 FP16으로도 **단일 GPU에 올라간다** (여유 메모리로 KV cache와 대형 배치 처리)
- 기존에 TP8이 필요하던 모델이 TP2면 되고, TP4가 필요하던 모델이 TP1으로 가능

이게 왜 "토폴로지 재설계"를 요구하는가:

- TP degree가 줄어들면 **같은 GPU 수로 더 많은 모델 인스턴스를 띄울 수 있다**
- 즉, 클러스터 전체의 배치 전략(어떤 모델을 몇 장에, 몇 replica로 띄울지)이 근본적으로 달라진다
- EP(Expert Parallelism)와 DP(Data Parallelism)의 최적 조합도 바뀐다

## WHAT

vLLM 블로그의 DeepSeek 서빙 실험이 좋은 예시다:

- B300 TP2에서 DeepSeek-R1 NVFP4 서빙 가능 (기존 H200에서는 불가능한 구성)
- 같은 2-GPU 구성에서 EP2 vs TP2를 비교하면, EP2가 throughput ceiling이 더 높고 TTFT 성장이 더 가파르다
- 즉, **"일단 TP로"라는 관성적 접근이 아니라, 모델 구조에 따라 EP/TP/DP 조합을 처음부터 실험해야 한다**

## 실무에서는

도입 전에 답해야 할 질문들:

- 우리가 서빙하는 모델들의 NVFP4 weight size는 각각 얼마인가?
- 각 모델별 최소 GPU 수(TP degree)는 얼마로 줄어드는가?
- GPU가 남는다면, 그 GPU를 추가 replica로 쓸지 / 다른 모델에 할당할지?
- MoE 모델이라면 EP가 유리한지 TP가 유리한지?
- Kubeflow/KServe에서 리소스 할당 단위를 어떻게 재정의할지?

**핵심은 "같은 GPU 수 = 같은 성능"이 아니라는 것이다.** B300에서는 더 적은 GPU로 같거나 더 나은 성능을 낼 수 있고, 남는 GPU를 다른 워크로드에 돌릴 수 있다. 이 재배치 전략을 사전에 세우지 않으면 B300의 메모리 이점을 비용 절감으로 연결하지 못한다.

---

# 5. 폼팩터 선택 — DGX vs HGX vs GB300 NVL72, 왜 중요한가

## WHY

같은 B300 GPU라도 **어떤 시스템에 넣느냐에 따라 운용 방식이 완전히 다르기** 때문이다. GPU 스펙만 보고 "B300이면 다 같겠지"라고 생각하면 안 된다.

## WHAT

| 시스템 | 구성 | 특징 | 적합한 경우 |
|--------|------|------|-----------|
| **DGX B300** | 8x B300, Intel Xeon | NVIDIA 턴키 시스템, 인증 인력 설치, AC/DC 전원 옵션 | 빠른 도입, 관리 단순화 우선 |
| **HGX B300** | 8x B300, OEM 선택 | Supermicro/Dell/Lenovo 등이 통합, CPU/쿨링/섀시 선택 가능 | 유연한 구성, 기존 인프라 활용 |
| **GB300 NVL72** | 72x B300 + 36 Grace CPU | 랙 단위 단일 NVLink 도메인, 수랭 전용 | 초대형 모델, 하이퍼스케일 |

- DGX B300: 새 폼팩터로 NVIDIA MGX 랙 호환, 기존 엔터프라이즈 랙에도 배치 가능
- HGX B300: 2U 수랭 구성 시 랙당 최대 144 GPU 밀도 달성 가능
- GB300 NVL72: 72 GPU가 하나의 NVLink 도메인으로 묶여 130 TB/s 대역폭. 사실상 별도 아키텍처

## 실무에서는

서빙이 주 용도인 우리 팀 기준으로는:

- **DGX B300 또는 HGX B300 (8-GPU 노드)가 현실적인 선택지**다
- NVFP4 기준으로 대부분의 모델이 TP2면 충분하므로, 8-GPU 노드 하나에서 4개의 독립 서빙 인스턴스를 띄울 수 있다
- GB300 NVL72는 trillion-parameter급 모델을 단일 랙에서 처리해야 하는 하이퍼스케일러 용도이고, 수랭 전용이므로 인프라 요구가 훨씬 크다
- HGX는 OEM에 따라 가격/납기/서포트가 다르므로 벤더 비교가 필요하다. DGX는 NVIDIA 직접 지원이라는 장점이 있지만 가격이 더 높다 (HGX ~$430K vs DGX는 그 이상으로 추정)

---

# 정리: 인프라 체크리스트

도입 전에 반드시 확인해야 할 항목을 우선순위 순으로 정리한다.

| 순위 | 항목 | 왜 먼저인가 | 리드타임 |
|------|------|-----------|---------|
| 1 | **전력 용량** | 증설 리드타임이 가장 길다 | 수개월 |
| 2 | **쿨링 방식** | 수랭 미비 시 물리적으로 불가 | 수개월 |
| 3 | **네트워크 패브릭** | 800Gbps 장비 납기와 설계 | 수주~수개월 |
| 4 | **폼팩터 결정** | 벤더 선정과 발주에 선행 | 수주 |
| 5 | **토폴로지 설계** | GPU 도착 후 즉시 서빙 가동 | 수주 |

**1~3번은 GPU 발주와 동시에, 혹은 더 먼저 시작해야 한다.** GPU가 도착했는데 전력이 부족하거나 수랭이 없으면 아무것도 할 수 없다.

4~5번은 GPU 도착 전에 완료하면 도착 즉시 서빙을 올릴 수 있다. 특히 5번(토폴로지 설계)은 3편에서 다룰 소프트웨어 스택과 직결되므로, 하드웨어와 소프트웨어를 병렬로 준비하는 것이 핵심이다.

---

# 마치며

B300 도입은 "GPU를 사는 것"이 아니라 **"GPU를 운용할 수 있는 환경을 만드는 것"**이다. 전력 2배, 수랭 필수, 800Gbps 네트워크, 그리고 288GB 메모리가 바꿔놓는 토폴로지 재설계까지 — GPU 스펙보다 인프라 스펙을 먼저 확인해야 한다.

다음 편에서는 이 하드웨어 위에서 실제로 서빙을 올리기 위한 소프트웨어 스택(vLLM/SGLang/KServe)을 왜 지금부터 맞춰야 하는지 다룬다.

---

*참고 자료:*
- [NVIDIA DGX B300 공식 페이지](https://www.nvidia.com/en-us/data-center/dgx-b300/)
- [NVIDIA DGX B300 User Guide](https://docs.nvidia.com/dgx/dgxb300-user-guide/introduction-to-dgxb300.html)
- [NVIDIA HGX B300 Platform](https://www.nvidia.com/en-us/data-center/hgx/)
- [Supermicro HGX B300 & GB300 NVL72](https://www.supermicro.com/en/accelerators/nvidia)
- [Introl - Blackwell Ultra Infrastructure Requirements](https://introl.com/blog/nvidia-blackwell-ultra-b300-infrastructure-requirements-2025)
- [vLLM - DeepSeek on GB300](https://blog.vllm.ai/2026/02/13/gb300-deepseek.html)
- [IntuitionLabs - HGX Physical Requirements](https://intuitionlabs.ai/articles/nvidia-hgx-data-center-requirements)
