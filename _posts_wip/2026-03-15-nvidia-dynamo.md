---
title: "NVIDIA Dynamo 딥다이브: vLLM/SGLang 위에 왜 또 레이어가 필요한가"
categories: GPU
tags:
  - GPU
  - NVIDIA-Dynamo
  - LLM-Serving
  - vLLM
  - SGLang
  - NIXL
  - KV-Cache
  - Disaggregated-Serving
  - Infrastructure
published: false
---

[Blackwell 글](https://jinwoongkim.net/gpu/blackwell/)을 쓸 때 Dynamo를 한 섹션으로 짧게 다뤘었다. 그때는 "클러스터 단위 오케스트레이션 레이어" 정도로만 소개했는데, 실제로 뜯어보니까 그 한마디로는 부족해서 별도 글로 정리하게 됐다.

vLLM이든 SGLang이든 잘 돌아가는데 왜 그 위에 또 뭘 얹어야 하는지, 그리고 GPU를 얼마나 더 알뜰하게 쓸 수 있는지를 중심으로 정리해본다.

---

# 한 줄 요약

> vLLM/SGLang은 **GPU 한 대(또는 TP 그룹)에서 최고의 성능**을 뽑는 엔진.  
> Dynamo는 **수십~수백 대의 GPU를 하나의 서빙 시스템처럼 묶어주는** 프레임워크.  
> GPU 한두 대면 필요 없고, 클러스터로 가면 없으면 답이 안 나옴.

---

# Part 1 — 알고만 있어도 되는 것들

## 1. Dynamo가 풀려는 문제 — 왜 서빙 엔진만으로는 부족한가

vLLM과 SGLang은 잘 만들어진 서빙 엔진이다. PagedAttention, RadixAttention, continuous batching, chunked prefill 등등 — 단일 GPU 또는 TP 그룹 안에서의 최적화는 이미 충분히 성숙하다.

문제는 GPU가 수십 장, 수백 장으로 늘어날 때 생긴다.

**Prefill/Decode 비대칭**: Prefill은 compute-bound, decode는 memory-bound. 성격이 완전히 다른 두 작업이 같은 GPU에서 돌아가니까 양쪽 다 최적이 아니다. 요약 요청이 몰리면 prefill만 터지고 decode GPU는 놀고, 반대로 long output이 몰리면 decode가 병목. vLLM/SGLang 단독으로는 "이 요청의 prefill은 저쪽에서, decode는 이쪽에서" 같은 분리가 안 된다.

**KV cache 중복 계산**: 같은 system prompt를 쓰는 요청 100개가 들어오면, round-robin 로드밸런서는 100개 GPU에 골고루 뿌린다. 동일한 prefix의 KV cache를 100번 계산하는 셈이다. "이 GPU에 이미 해당 KV cache가 있으니까 거기로 보내자"는 판단은 일반 로드밸런서로는 불가능.

**GPU 고정 할당**: Reasoning 모델(DeepSeek-R1 등)은 output 길이가 예측 불가능하다. 어떤 요청은 50 토큰, 어떤 요청은 5,000 토큰. GPU를 고정 비율로 할당하면 한쪽은 터지고 한쪽은 노는 상황이 반복된다.

이 세 가지는 서빙 엔진을 아무리 잘 만들어도 풀 수 없는 문제고, 클러스터 레벨에서 스케줄링과 라우팅을 해주는 별도의 레이어가 필요하다.

참고로 Dynamo는 Triton Inference Server의 후속작이다. Triton은 범용 모델 서빙(CV, NLP, 추천 등)에 잘 맞았지만, KV cache 관리, prefill/decode 분리 같은 LLM 특유의 요구사항에는 구조적 한계가 있었다.
{: .notice--info}

---

## 2. 핵심 아키텍처 — 4개 컴포넌트

Dynamo는 4개의 독립적인 컴포넌트로 구성되어 있다. 필요한 것만 골라 쓸 수 있고, 각각 독립적으로 스케일 가능하다.

### GPU Resource Planner

Prefill GPU와 decode GPU의 비율을 실시간으로 조정해준다.

각 GPU의 부하(큐 깊이, 배치 크기, KV cache 사용률)를 모니터링하면서, 사전에 설정한 SLA 타겟(TTFT P99, ITL P99)을 기준으로 GPU 할당을 바꾼다. 요약 요청이 몰려서 prefill이 터지면 놀고 있는 decode GPU를 prefill로 전환하거나, 아예 aggregated 모드(한 GPU에서 prefill + decode 다 하는 모드)로 돌릴 수도 있다.

핵심은 정적 할당이 아니라 동적 재배치라는 점.

### Smart Router (KV-aware)

클러스터 전체의 KV cache 위치를 추적하고, 새 요청이 들어오면 **KV cache overlap이 가장 큰 GPU**로 보내준다.

일반 로드밸런서는 GPU의 "부하"만 보는데, Smart Router는 "이 요청의 prefix와 이미 캐싱된 KV 블록이 얼마나 겹치는가"까지 계산한다. Overlap이 크면 prefill을 건너뛸 수 있으니까 TTFT가 줄어들고, 중복 계산도 사라진다.

멀티턴 대화, 공통 system prompt, RAG에서 같은 문서를 반복 참조하는 패턴 등에서 효과가 크다.

### KV Cache Manager (KVBM)

GPU HBM에 다 못 담는 KV cache를 계층적으로 오프로딩한다.

```
GPU HBM (가장 빠름, 가장 비쌈)
  ↓ 자주 안 쓰이는 KV 블록
CPU DDR (빠름, 저렴)
  ↓ 더 오래된 KV 블록
NVMe SSD (보통, 매우 저렴)
  ↓ 아카이브
Object Storage / 네트워크 스토리지
```

"B300이 288GB나 되는데 오프로딩이 왜 필요해?" 싶을 수 있는데, long-context(128K+) 서빙에서 동시 세션 수가 늘면 288GB도 모자란다. NVIDIA 벤치마크에서 H100 기준 멀티턴 10개 대화 × 80 유저 시나리오에서 system memory 오프로딩만으로 TTFT가 40% 개선됐다. Prefix caching을 이미 쓰고 있는 상태에서도.

### NIXL (NVIDIA Inference Transfer Library)

Disaggregated serving에서 prefill GPU → decode GPU로 KV cache를 옮겨야 하는데, 이 데이터 이동이 병목이 되면 분리한 의미가 없다.

NIXL은 NVLink, InfiniBand, PCIe, GDS 등 다양한 연결을 **단일 API**로 추상화해준다. 소스가 GPU HBM이든 CPU 메모리든 NVMe든 관계없이 같은 인터페이스로 데이터를 옮기고, 비동기 전송과 배치 최적화로 동기화 오버헤드를 최소화한다.

---

## 3. Disaggregated Serving — prefill과 decode를 왜 분리하는가

### 왜 분리하는가

LLM inference는 두 단계로 나뉜다:

| 단계 | 특성 | 병목 |
| --- | --- | --- |
| **Prefill** | Input 토큰 전체를 한 번에 처리, KV cache 생성 | Compute-bound (FLOPS) |
| **Decode** | 토큰을 하나씩 생성, KV cache 참조 | Memory-bound (bandwidth) |

비유하자면 prefill은 주방에서 재료 손질(CPU 집약적), decode는 손님 테이블에 한 접시씩 나르는 것(왔다갔다 bandwidth 집약적)에 가까운데, 같은 사람이 둘 다 하면 어느 쪽이든 비효율이 생긴다.

같은 GPU에서 둘을 돌리면(aggregated mode), 긴 입력을 처리하는 prefill이 진행 중일 때 decode 중인 다른 요청들의 토큰 생성이 지연된다 — TPOT 스파이크의 흔한 원인.

분리하면:
- Prefill GPU는 FLOPS 극대화에 집중
- Decode GPU는 bandwidth 극대화에 집중
- 각각의 GPU 수를 트래픽 패턴에 맞게 독립 조절

이 아이디어는 UCSD Hao AI Lab의 **DistServe** 논문에서 시작됐고, 18개월 만에 사실상 업계 표준이 됐다.

### 분리가 오히려 손해인 경우

만능은 아니다. KV cache를 prefill GPU에서 decode GPU로 전송하는 비용이 있기 때문에:

- 짧은 input + 짧은 output이면 전송 오버헤드가 더 클 수 있고
- GPU가 몇 장 없으면 분리할 여유가 없고
- 워크로드가 균일하면 고정 할당으로도 충분

Dynamo Planner가 이런 상황을 감지하면 알아서 aggregated 모드로 전환하거나, decode GPU를 prefill 역할로 재할당한다. 정적 설정이 아니라 동적 판단이라는 게 핵심.

---

## 4. KV-aware Routing — 왜 로드밸런서로는 안 되는가

### 기존 방식의 문제

```
사용자 A: system prompt(2K 토큰) + 질문 → GPU 1로 라우팅
사용자 B: 동일한 system prompt + 다른 질문 → GPU 2로 라우팅
사용자 C: 동일한 system prompt + 또 다른 질문 → GPU 3으로 라우팅
```

3개 GPU 모두 같은 2K 토큰에 대한 KV cache를 각각 계산한다. GPU 1에 이미 해당 KV cache가 있으니까 B, C도 GPU 1로 보내면 prefill을 건너뛸 수 있는데, round-robin은 이 정보가 없다.

### Smart Router의 동작

클러스터 전체의 KV cache 인덱스를 유지하고, 새 요청이 들어오면:

1. 요청의 prefix 추출
2. 각 GPU의 KV cache와의 overlap 계산
3. Overlap이 가장 큰 GPU로 라우팅 (부하와 함께 고려)

공통 system prompt 쓰는 API 서비스, 멀티턴 대화, RAG 같은 패턴에서 효과가 크다.

### KServe와의 관계

KServe와 Dynamo는 대체 관계가 아니라 다른 레이어.

| 역할 | KServe/Knative | Dynamo |
| --- | --- | --- |
| 모델 배포/롤링 업데이트 | O | X |
| HTTP 엔드포인트 관리 | O | O (OpenAI-compatible) |
| 트래픽 기반 autoscaling | O (Pod 단위) | O (GPU 역할 단위) |
| KV cache 기반 라우팅 | X | O |
| Prefill/Decode 분리 | X | O |

현실적으로 Dynamo를 쓰면 KServe의 라우팅/스케일링 기능 일부가 중복된다. Dynamo가 서빙 로직을 담당하고 KServe는 모델 lifecycle 관리(배포, 버전, canary)용으로만 쓰거나, Dynamo Operator로 대체하는 선택지가 있다.

---

## 5. KV Cache 계층적 오프로딩

KV cache 오프로딩은 vLLM이나 SGLang에서도 prefix caching으로 어느 정도 되긴 한다. Dynamo의 KVBM이 다른 점은 **클러스터 레벨**에서 작동한다는 것.

| 접근 | 범위 | 한계 |
| --- | --- | --- |
| vLLM prefix caching | 단일 워커 내 GPU HBM | GPU 메모리 초과 시 eviction |
| SGLang RadixAttention | 단일 워커 내 radix tree | 워커 간 캐시 공유 불가 |
| LMCache | 외부 스토리지 연동 가능 | 별도 구성 필요 |
| **Dynamo KVBM** | 클러스터 전체, 다단계 메모리 | Smart Router와 통합 |

핵심 차별점은 Smart Router와의 통합이다. 오프로딩된 KV cache의 위치를 Router가 알고 있으니까, "이 prefix의 KV cache가 GPU 3의 CPU 메모리에 있다 → GPU 3으로 라우팅하면 CPU → GPU로 빠르게 복원 가능"이라는 판단을 할 수 있다.

B300 288GB가 크다고 해도, 128K context + 동시 세션이 늘면 금방 모자라기 때문에, 동시 서빙 capacity를 극대화하려면 오프로딩은 어차피 필요하다.

---

## 6. GPU Utilization을 높이는 7가지 메커니즘

앞선 섹션에서 다룬 disaggregated serving, KV-aware routing, KV cache offloading도 결국 GPU를 더 알뜰하게 쓰기 위한 것들이다. 여기선 이미 다룬 것 포함해서, Dynamo가 GPU util을 높이기 위해 갖고 있는 카드 전체를 한 곳에 모아본다.

GPU가 노는 이유를 크게 세 가지로 나눠보면:

1. **Compute/Memory 불균형**: Prefill은 FLOPS를 다 쓰는데 bandwidth는 남고, decode는 그 반대
2. **낭비되는 계산**: 이미 계산된 KV cache를 또 계산하거나, 유저가 이미 떠난 요청을 계속 처리
3. **비탄력적 할당**: 트래픽이 줄어도 GPU가 묶여 있고, 워커가 죽으면 그 GPU가 통째로 놂

### 6-1. Disaggregated Serving (→ 섹션 3에서 상세)

Prefill과 decode를 분리해서 각 GPU가 자기 특성에 맞는 워크로드만 처리하게 만든다. Compute/Memory 불균형 문제를 구조적으로 없앤다.

### 6-2. Phase-specific Parallelism

Disaggregated serving을 하면 얻는 보너스인데, 종종 간과되는 부분이다. Prefill과 decode가 분리되면 **각 단계에 서로 다른 TP/EP degree**를 적용할 수 있다.

MoE 모델(DeepSeek-R1 671B)을 예로 들면:
- Prefill: Compute-bound → 높은 TP로 GEMM 병렬화가 유리
- Decode: Memory-bound → 넓은 EP로 expert를 분산해서 bandwidth 극대화가 유리

Aggregated 모드에서는 TP와 EP를 하나의 값으로 고정해야 하니까 양쪽 다 최적이 안 된다. 분리하면 prefill 워커에 TP=4, decode 워커에 EP=8 같은 식으로 독립 설정이 가능해진다.

GB200 NVL72 + Dynamo에서 MoE throughput이 Hopper 대비 15배까지 올라가는 핵심 원리 중 하나.

### 6-3. E/P/D Disaggregation — MoE의 Expert까지 분리

P/D disaggregation을 한 단계 더 확장한 개념인데, MoE에서 Expert 연산을 별도 워커로 분리해서 **Expert / Prefill / Decode 3단계**로 나누는 것.

```
요청 → [Prefill 워커] → KV cache 전송 → [Decode 워커]
                                              ↕
                                      [Expert 워커]
```

MoE에서 활성화되는 expert는 요청마다 다르니까, expert를 따로 빼면 더 효율적으로 분산 배치할 수 있다. 현재 Dynamo 로드맵에서 적극 개발 중이고, SGLang과 TRT-LLM에서 wideEP 스크립트가 이미 나와 있다.

아직 프로덕션에서 안정적으로 쓰기엔 이른 감이 있다. 지켜보면서 준비하는 단계.
{: .notice--warning}

### 6-4. KV-aware Routing (→ 섹션 4에서 상세)

같은 prefix의 KV cache를 여러 GPU에서 중복 계산하는 건 순수한 FLOPS 낭비. Smart Router가 이미 캐싱된 KV 블록이 있는 GPU로 보내면 prefill 자체를 건너뛸 수 있다.

최근 릴리즈에서는 **LoRA adapter 상태까지 라우팅에 반영**한다. 같은 base model에 여러 LoRA를 서빙할 때, adapter ID에 맞는 워커로 보내면 adapter 로딩/스왑 오버헤드도 줄일 수 있다.

### 6-5. Request Cancellation & Migration

유저가 응답 기다리다 탭을 닫으면, 그 요청은 더 이상 결과를 받을 곳이 없다. 그런데 기존 서빙 엔진에서는 decode가 끝날 때까지 GPU를 계속 점유한다. 클러스터가 커질수록 이런 "좀비 요청"이 누적되면서 전체 util을 깎아먹는다.

**Request Cancellation**: Frontend에서 연결 끊김을 감지하면, 취소 신호가 prefill → decode 워커까지 전파된다. 워커는 generation을 즉시 중단하고 GPU 리소스를 다른 요청에 양보.

**Request Migration**: 워커가 죽거나 과부하일 때, 진행 중인 요청을 다른 워커로 이전한다. `--migration-limit 3` 같은 설정으로 요청당 최대 마이그레이션 횟수를 지정 가능. 워커가 종료 신호를 받으면 frontend가 자동으로 다른 워커에서 이어받는다.

### 6-6. Speculative Decoding

Decode는 memory-bound라서 GPU FLOPS가 남는다. 이 남는 FLOPS를 활용하는 기법.

작은 draft model(Eagle3 등)이 여러 토큰을 빠르게 "추측"하고, 큰 target model이 한 번에 검증한다. 맞으면 그대로 쓰고, 틀리면 거기서부터 다시 생성. 한 번의 decode step에서 여러 토큰을 출력할 수 있어 TPOT이 줄어든다.

Dynamo에서는 두 가지 모드가 가능한데:
- Aggregated: 단일 워커에서 draft + target 모델을 같이 실행
- Disaggregated: Draft model을 별도 "draft" 워커로 분리하고, Dynamo가 통신 조율

vLLM 백엔드에서 Eagle3 기반이 이미 지원되고, TRT-LLM과 SGLang에서도 확대 중.

### 6-7. Multi-LoRA

고객사마다 fine-tuned 모델이 다르지만 base model은 같은 경우가 많다. 각 모델에 GPU를 따로 할당하면 트래픽 적은 모델의 GPU는 대부분 놀게 된다.

Multi-LoRA는 base model weight를 GPU에 올려두고, 요청에 따라 다른 LoRA adapter를 동적으로 적용한다. GPU당 서빙할 수 있는 "모델 수"가 LoRA adapter 수만큼 늘어나는 셈.

Dynamo에서는 단순히 엔진 레벨 기능을 넘어서:
- MinIO 같은 외부 스토리지에서 adapter를 동적 로드/언로드
- Router가 adapter ID까지 고려해서 이미 해당 adapter가 로드된 워커로 라우팅
- 여러 replica에서 동일 adapter에 대해 일관된 라우팅 보장

### 요약 테이블

| 메커니즘 | 무슨 낭비를 잡나 | 효과 큰 시나리오 | 성숙도 |
| --- | --- | --- | --- |
| Disaggregated Serving | Compute/Memory 불균형 | Long-context, reasoning 모델 | 프로덕션 |
| Phase-specific Parallelism | 단일 parallelism 비효율 | MoE 대형 모델 | 프로덕션 |
| E/P/D Disaggregation | MoE expert 비효율 | DeepSeek-R1 등 대형 MoE | 개발 중 |
| KV-aware Routing | KV cache 중복 계산 | 공통 system prompt, 멀티턴 | 프로덕션 |
| Request Cancel/Migration | 좀비 요청의 GPU 점유 | 대규모 API 서비스 | 프로덕션 |
| Speculative Decoding | Decode 시 남는 FLOPS | 짧은 output 다수 | 초기 지원 |
| Multi-LoRA | 모델당 GPU 고정 할당 | 고객별 fine-tuned 모델 다수 | 초기 지원 |

이 메커니즘들은 독립이 아니라 복합적으로 작용한다. Disaggregated serving으로 분리하고, phase-specific parallelism으로 각 단계 병렬화를 최적화하고, KV-aware routing으로 중복 계산을 없애고, request cancellation으로 낭비를 즉시 회수하는 식. 하나의 GPU에서 한 가지 기법을 적용하는 게 아니라, 클러스터 전체에서 여러 기법이 시너지를 내는 구조.

---

## 7. NIXL — 왜 NCCL만으로는 안 되는가

NCCL 쓰고 있는데 왜 또 다른 통신 라이브러리가 필요하지? 싶을 수 있는데, 역할이 다르다.

| 항목 | NCCL | NIXL |
| --- | --- | --- |
| 설계 목적 | Collective communication (AllReduce 등) | Point-to-point inference transfer |
| 주 사용처 | 학습, TP/PP gradient 동기화 | KV cache 전송, 메모리 계층 간 이동 |
| 통신 패턴 | All-to-all, broadcast, reduce | 1:1, 비동기, 배치 |
| 메모리 지원 | GPU HBM 위주 | HBM, DDR, NVMe, Object Storage 전부 |

NCCL은 "모든 GPU가 동시에 참여하는 집합 통신"에 최적화되어 있다. TP에서의 AllReduce 같은.

반면 disaggregated serving에서는 "prefill GPU 1번이 decode GPU 3번에게 KV cache 블록을 보내는" point-to-point 전송이 핵심이고, 소스가 GPU 메모리일 수도, CPU 메모리일 수도, SSD일 수도 있다. NCCL은 이런 이기종 메모리 간 전송을 지원하지 않는다.

NIXL은 **"Memory Section"**이라는 추상화로 이걸 해결한다. HBM이든 DDR이든 NVMe든 동일한 API로 다루고, 최적의 전송 경로(GPUDirect RDMA, GDS 등)를 알아서 선택.

---

## 8. Dynamo vs 경쟁 프레임워크

Disaggregated serving은 Dynamo만의 것이 아니다.

| 프레임워크 | 핵심 접근 | 서빙 엔진 | 강점 | 약점 |
| --- | --- | --- | --- | --- |
| **NVIDIA Dynamo** | 4-컴포넌트 모듈 아키텍처 | vLLM, SGLang, TRT-LLM | 완성도 높은 disagg., NIXL, Planner | NVIDIA 하드웨어에 최적화, 복잡도 |
| **llm-d** | K8s-native, vLLM 기반 | vLLM | Red Hat 주도, K8s 생태계 | vLLM 의존, 아직 초기 |
| **Ray Serve LLM** | Ray 생태계 위 P/D 분리 | vLLM + NIXL/LMCache | Ray 통합, RL 워크로드 연계 | Ray 의존성 |
| **SGLang 자체 PD** | 엔진 내장 disaggregation | SGLang | 별도 프레임워크 불필요 | 클러스터 스케줄링 제한적 |

### 멀티 엔진 운용

Dynamo는 **같은 클러스터에서 여러 서빙 엔진을 동시에** 돌릴 수 있다:

- 모델 A (Dense 70B) → vLLM 워커
- 모델 B (MoE 671B) → SGLang 워커
- 모델 C → TRT-LLM 워커

앞단의 OpenAI-compatible Frontend가 모델별로 적절한 워커 그룹으로 라우팅한다. 블랙웰 글에서 "vLLM과 SGLang 둘 다 준비해야 한다"고 했었는데, Dynamo가 이 "둘 다"를 하나의 클러스터에서 가능하게 해준다.

단, 하나의 모델에 대해 "prefill은 SGLang, decode는 vLLM"처럼 엔진을 섞는 건 안 된다. KV cache 포맷이 엔진마다 다르기 때문.
{: .notice--warning}

---

## 9. 언제 Dynamo가 필요하고, 언제 불필요한가

기술이 있다고 무조건 쓰는 게 답은 아니다. Dynamo는 복잡도를 추가하는 레이어이므로, 그만큼의 이점이 있을 때만 도입해야 한다.

**필요한 경우**:
- 멀티노드 서빙 (노드 간 요청 분배 필요)
- Long-context + 가변 output, reasoning 모델
- MoE Expert Parallelism
- 시간대별로 워크로드가 크게 바뀌는 API 서비스
- KV cache 재활용이 높은 패턴 (공통 system prompt, 멀티턴, RAG)

**불필요한 경우**:
- 단일 노드 TP2 서빙 — B300 288GB면 vLLM/SGLang만으로 충분
- 고정 워크로드 — input/output 길이가 일정하고 트래픽이 안정적이면 정적 할당으로 가능
- GPU 8장 이하 — disaggregation의 이점보다 운영 복잡도가 더 클 수 있음
- 프로토타이핑 — 빠르게 모델 띄워보는 단계에서는 오버엔지니어링

대략적으로 아래 질문 중 3개 이상 Yes면 적극 검토, 1개 이하면 엔진 단독이 현실적.

| 질문 | Yes → | No → |
| --- | --- | --- |
| GPU가 여러 노드에 분산? | Dynamo 검토 | 엔진 단독으로 충분할 가능성 높음 |
| Reasoning 모델 or long-context? | Disagg. PD 이점 큼 | Aggregated로 충분할 수 있음 |
| 동일 prefix 공유 요청 많음? | KV-aware routing 효과 큼 | 일반 LB로도 무방 |
| 트래픽이 시간대별로 크게 변동? | Planner 동적 재배치 필요 | 고정 할당 가능 |
| 동시 세션이 GPU 메모리 초과? | KVBM 오프로딩 필요 | 메모리 내 처리 가능 |

---

*다음 글(Part 2)에서는 실제로 Dynamo를 띄우고, disaggregated 모드를 설정하고, Kubernetes에서 운용하고, 관측성 스택과 통합하는 과정을 다룬다.*

---

*참고 자료:*

- [NVIDIA Dynamo 공식 페이지](https://www.nvidia.com/en-us/ai/dynamo/)
- [NVIDIA Dynamo GitHub](https://github.com/ai-dynamo/dynamo)
- [NVIDIA Dynamo 공식 문서 — Architecture](https://docs.nvidia.com/dynamo/latest/_sections/architecture.html)
- [NVIDIA 기술 블로그 — Introducing Dynamo](https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/)
- [NVIDIA 기술 블로그 — Dynamo GPU Autoscaling](https://developer.nvidia.com/blog/nvidia-dynamo-adds-gpu-autoscaling-kubernetes-automation-and-networking-optimizations/)
- [Hao AI Lab — Disaggregated Inference: 18 Months Later](https://haoailab.com/blogs/distserve-retro/)
- [NVIDIA Dynamo + llm-d 블로그](https://developer.nvidia.com/blog/nvidia-dynamo-accelerates-llm-d-community-initiatives-for-advancing-large-scale-distributed-inference/)
- [AWS — Dynamo on Amazon EKS](https://aws.amazon.com/blogs/machine-learning/accelerate-generative-ai-inference-with-nvidia-dynamo-and-amazon-eks/)
- [Azure AKS — Scaling multi-node LLM inference with Dynamo](https://blog.aks.azure.com/2025/10/24/dynamo-on-aks)
- [InfoQ — NVIDIA Dynamo Multi-Node Inference](https://www.infoq.com/news/2025/12/nvidia-dynamo-kubernetes/)
- [Dynamo Roadmap — GitHub Issue #762](https://github.com/ai-dynamo/dynamo/issues/762)
