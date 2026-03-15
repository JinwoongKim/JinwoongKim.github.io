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

[이전 글](https://jinwoongkim.net/gpu/blackwell/)에서 B300 도입 준비를 정리하면서 Dynamo를 한 섹션으로 짧게 다뤘다. "클러스터 단위 오케스트레이션 레이어"라고만 소개했는데, 실제로 뜯어보니 그 한마디로는 부족했다.

이 글에서는 Dynamo의 아키텍처를 제대로 파고든다. 핵심 질문은 하나다:

> **"vLLM이든 SGLang이든 잘 돌아가는데, 왜 그 위에 또 레이어를 얹어야 하는가?"**

---

# 한 줄 요약

> vLLM/SGLang은 **한 대의 GPU(또는 TP 그룹)에서 최고의 성능**을 뽑는 엔진이다.  
> Dynamo는 **수십~수백 대의 GPU가 하나의 서빙 시스템처럼 동작**하게 만드는 프레임워크다.  
> 단일 노드에서는 필요 없고, 클러스터로 가면 없으면 안 된다.

---

# Part 1 — 알고만 있어도 되는 것들

## 1. Dynamo가 풀려는 문제 — 왜 서빙 엔진만으로는 부족한가

vLLM과 SGLang은 뛰어난 서빙 엔진이다. PagedAttention, RadixAttention, continuous batching, chunked prefill — 단일 GPU 또는 TP 그룹 안에서의 최적화는 이미 충분히 성숙했다.

문제는 **스케일**이다. GPU가 수십 장, 수백 장으로 늘어나면 엔진 혼자서는 답이 안 나오는 문제 세 가지가 생긴다:

**Prefill/Decode 비대칭**: Prefill은 compute-bound, decode는 memory-bound다. 같은 GPU에서 둘을 돌리면 양쪽 다 최적이 아니다. 요약 요청이 몰리면 prefill GPU만 포화되고 decode GPU는 놀고, 반대로 long output이 몰리면 decode가 병목이다. 하지만 vLLM/SGLang 단독으로는 "이 요청의 prefill은 저쪽 GPU에서, decode는 이쪽 GPU에서" 같은 분리가 안 된다.

**KV cache 중복 계산**: 같은 system prompt를 쓰는 요청 100개가 들어오면, round-robin 로드밸런서는 100개 GPU에 골고루 분산한다. 결과적으로 동일한 prefix의 KV cache를 100번 계산한다. "이 GPU에 이미 해당 prefix의 KV cache가 있으니까 거기로 보내자"라는 판단은 일반 로드밸런서가 할 수 없다.

**GPU 고정 할당의 비효율**: Reasoning 모델(DeepSeek-R1 등)은 output 길이가 예측 불가능하다. 어떤 요청은 50 토큰, 어떤 요청은 5,000 토큰이다. GPU를 고정 비율로 할당하면 트래픽 패턴에 따라 한쪽은 과부하, 한쪽은 유휴 상태가 된다.

이 세 가지는 "더 좋은 서빙 엔진"으로 풀 수 있는 문제가 아니다. **클러스터 레벨의 스케줄링과 라우팅** 레이어가 필요하다. Dynamo가 바로 이 레이어다.

### Triton Inference Server와의 관계

Dynamo는 NVIDIA Triton Inference Server의 후속작이다. Triton은 범용 모델 서빙(CV, NLP, 추천 등)에 잘 맞았지만, LLM 특유의 요구사항 — KV cache 관리, prefill/decode 분리, reasoning 모델의 가변 output — 을 처리하기엔 구조적 한계가 있었다. Dynamo는 처음부터 LLM inference에 맞춰 설계됐다.

---

## 2. 핵심 아키텍처 — 4개 컴포넌트

Dynamo의 아키텍처는 4개의 독립적인 컴포넌트로 구성된다. 각각이 독립적으로 스케일 가능하고, 필요한 것만 골라 쓸 수 있다.

### GPU Resource Planner

Prefill GPU와 decode GPU의 비율을 실시간으로 조정한다.

동작 방식: 각 GPU의 현재 부하(큐 깊이, 배치 크기, KV cache 사용률)를 모니터링하면서, 사전에 설정한 SLA 타겟(TTFT P99, ITL P99)을 기준으로 GPU 할당을 변경한다. 예를 들어 요약 요청이 몰려서 prefill GPU가 포화되면, 유휴 상태인 decode GPU 일부를 prefill로 전환하거나, aggregated 모드(prefill + decode를 같은 GPU에서 처리)로 돌릴 수 있다.

핵심은 **정적 할당이 아니라 동적 재배치**라는 점이다. Reasoning 모델처럼 워크로드 패턴이 예측 불가능한 환경에서 특히 중요하다.

### Smart Router (KV-aware)

클러스터 전체의 KV cache 위치를 추적하고, 새로운 요청이 들어오면 **KV cache overlap이 가장 큰 GPU**로 라우팅한다.

일반 로드밸런서(round-robin, least-connection)는 GPU의 "부하"만 본다. Smart Router는 부하에 더해서 "이 요청의 prefix와 이미 캐싱된 KV 블록이 얼마나 겹치는가"를 계산한다. Overlap이 크면 해당 GPU에서 prefill을 건너뛸 수 있으니 TTFT가 줄어들고, 전체적으로 중복 계산이 사라진다.

멀티턴 대화, 공통 system prompt, RAG에서 같은 문서를 반복 참조하는 패턴 등에서 효과가 크다.

### KV Cache Manager (KVBM)

GPU HBM에 다 담지 못하는 KV cache를 **계층적으로 오프로딩**한다.

```
GPU HBM (가장 빠름, 가장 비쌈)
  ↓ 자주 안 쓰이는 KV 블록
CPU DDR (빠름, 저렴)
  ↓ 더 오래된 KV 블록
NVMe SSD (보통, 매우 저렴)
  ↓ 아카이브
Object Storage / 네트워크 스토리지
```

"B300이 288GB나 되는데 오프로딩이 왜 필요해?"라고 생각할 수 있다. 하지만 long-context 서빙(128K+)에서 동시 세션 수가 늘어나면 288GB도 부족하다. NVIDIA 벤치마크에서 H100 기준 10개 멀티턴 대화 × 80 유저 시나리오에서 system memory 오프로딩만으로 TTFT가 40% 개선됐다. Prefix caching을 이미 쓰고 있어도 추가 이점이 있었다.

### NIXL (NVIDIA Inference Transfer Library)

Disaggregated serving에서 prefill GPU → decode GPU로 KV cache를 옮겨야 하는데, 이 데이터 이동이 병목이 되면 분리한 의미가 없다. NIXL이 이 문제를 담당한다.

NIXL은 NVLink, InfiniBand, PCIe, GDS(GPUDirect Storage) 등 다양한 연결을 **단일 API**로 추상화한다. 소스/목적지가 GPU HBM이든 CPU 메모리든 NVMe든 관계없이 같은 인터페이스로 데이터를 옮긴다. 비동기 전송과 배치 최적화로 동기화 오버헤드를 최소화한다.

---

## 3. Disaggregated Serving — prefill과 decode를 왜 분리하는가

Disaggregated serving은 Dynamo의 핵심 기능이자, Dynamo가 존재하는 가장 큰 이유다.

### 왜 분리하는가

LLM inference는 두 단계로 나뉜다:

| 단계 | 특성 | 병목 |
| --- | --- | --- |
| **Prefill** | Input 토큰 전체를 한 번에 처리, KV cache 생성 | Compute-bound (FLOPS) |
| **Decode** | 토큰을 하나씩 생성, KV cache 참조 | Memory-bound (bandwidth) |

같은 GPU에서 둘을 돌리면(aggregated mode), prefill의 compute 요구와 decode의 bandwidth 요구가 서로를 방해한다. 특히 긴 입력을 처리하는 prefill이 진행 중이면 decode 중인 다른 요청들의 토큰 생성이 지연된다 — TPOT 스파이크의 흔한 원인이다.

분리하면 각 단계를 독립적으로 스케일할 수 있다:

- Prefill GPU는 FLOPS 극대화에 집중 (높은 배치 크기, 긴 시퀀스)
- Decode GPU는 bandwidth 극대화에 집중 (많은 동시 요청, 낮은 latency)
- 각각의 GPU 수를 트래픽 패턴에 맞게 독립 조절

이 아이디어는 UCSD Hao AI Lab의 **DistServe** 논문에서 시작됐고, 18개월 만에 사실상 업계 표준이 됐다. NVIDIA Dynamo, llm-d, Ray Serve LLM, SGLang, vLLM 모두 어떤 형태로든 P/D disaggregation을 지원한다.

### 분리가 오히려 손해인 경우

Disaggregated serving이 만능은 아니다. KV cache를 prefill GPU에서 decode GPU로 전송하는 비용이 존재하기 때문이다.

- **짧은 input + 짧은 output**: 전송 오버헤드가 계산 절감보다 큰 경우
- **GPU가 적은 환경**: 분리할 만큼 GPU가 없으면 aggregated가 더 효율적
- **균일한 워크로드**: Input/output 길이가 일정하면 고정 할당으로도 충분

Dynamo Planner가 이런 상황을 감지하면 자동으로 aggregated 모드로 전환하거나, decode GPU를 prefill 역할로 재할당할 수 있다. 이게 정적 설정이 아니라 동적 판단이라는 게 핵심이다.

---

## 4. KV-aware Routing — 왜 로드밸런서로는 안 되는가

### 기존 방식의 문제

KServe + Knative, 또는 일반적인 Kubernetes Ingress 기반 로드밸런싱은 "어떤 GPU가 덜 바쁜가"만 본다. LLM 서빙에서 이게 문제인 이유:

```
사용자 A: system prompt(2K 토큰) + 질문 → GPU 1로 라우팅
사용자 B: 동일한 system prompt(2K 토큰) + 다른 질문 → GPU 2로 라우팅
사용자 C: 동일한 system prompt(2K 토큰) + 또 다른 질문 → GPU 3으로 라우팅
```

3개 GPU 모두 동일한 2K 토큰에 대한 KV cache를 각각 계산한다. GPU 1에 이미 해당 system prompt의 KV cache가 있으니 B와 C도 GPU 1로 보내면 prefill을 건너뛸 수 있는데, round-robin은 이 정보가 없다.

### Smart Router의 동작

Dynamo의 Smart Router는 클러스터 전체의 KV cache 인덱스를 유지한다. 새 요청이 들어오면:

1. 요청의 prefix를 추출
2. 각 GPU의 KV cache와의 overlap을 계산
3. Overlap이 가장 큰 GPU로 라우팅 (부하 조건과 함께 고려)

이게 효과적인 워크로드:
- 동일한 system prompt를 공유하는 API 서비스 (대부분의 프로덕션 서빙)
- 멀티턴 대화에서 이전 컨텍스트 재활용
- RAG에서 같은 문서를 여러 쿼리가 참조하는 패턴

### KServe와의 관계

KServe와 Dynamo는 **대체 관계가 아니라 다른 레이어**다.

| 역할 | KServe/Knative | Dynamo |
| --- | --- | --- |
| 모델 배포/롤링 업데이트 | O | X |
| HTTP 엔드포인트 관리 | O | O (OpenAI-compatible) |
| 트래픽 기반 autoscaling | O (Pod 단위) | O (GPU 역할 단위) |
| KV cache 기반 라우팅 | X | O |
| Prefill/Decode 분리 | X | O |

현실적으로는 Dynamo를 쓰면 KServe의 라우팅/스케일링 기능 일부가 중복된다. 프로덕션에서는 Dynamo가 서빙 로직을 담당하고, KServe는 모델 lifecycle 관리(배포, 버전, canary) 용도로만 쓰거나 아예 Dynamo의 Kubernetes Operator로 대체하는 선택지가 있다.

---

## 5. KV Cache 계층적 오프로딩 — GPU 메모리 벽을 넘는 법

KV cache 오프로딩은 vLLM이나 SGLang에서도 prefix caching으로 어느 정도 지원한다. Dynamo의 KVBM(KV Block Manager)이 다른 점은 **클러스터 레벨**에서 작동한다는 것이다.

### 기존 접근과의 차이

| 접근 | 범위 | 한계 |
| --- | --- | --- |
| vLLM prefix caching | 단일 워커 내 GPU HBM | GPU 메모리 초과 시 eviction |
| SGLang RadixAttention | 단일 워커 내 radix tree | 워커 간 캐시 공유 불가 |
| LMCache | 외부 스토리지 연동 가능 | 별도 구성 필요 |
| **Dynamo KVBM** | 클러스터 전체, 다단계 메모리 | Smart Router와 통합 |

Dynamo KVBM의 핵심 차별점은 **Smart Router와의 통합**이다. 오프로딩된 KV cache의 위치를 Router가 알고 있으니, "이 prefix의 KV cache가 GPU 3의 CPU 메모리에 있다 → GPU 3으로 라우팅하면 CPU → GPU로 빠르게 복원 가능"이라는 판단을 할 수 있다.

### B300에서도 필요한 이유

B300은 288GB HBM3e를 탑재하고 있다. 큰 메모리지만, long-context 서빙에서는 여전히 부족할 수 있다:

- DeepSeek-R1 (671B MoE) NVFP4 weight만으로 GPU 2장 × 288GB의 상당 부분을 차지
- 128K context에서 동시 세션 수가 늘어나면 KV cache가 남은 HBM을 빠르게 소진
- 오프로딩 없이는 동시 세션 수를 제한하거나, KV cache eviction으로 인한 재계산 비용을 감수해야 함

결국 "GPU 메모리가 크니까 괜찮겠지"가 아니라, **동시 서빙 capacity를 극대화하려면 오프로딩은 필수**다.

---

## 6. GPU Utilization을 높이는 7가지 메커니즘

앞선 섹션들에서 Dynamo의 핵심 컴포넌트를 다뤘다. 여기서는 시야를 넓혀서, **Dynamo가 GPU utilization을 높이기 위해 동원하는 메커니즘 전체를 한 곳에 정리**한다. 이미 앞에서 다룬 것(disaggregated serving, KV-aware routing, KV cache offloading)도 포함하되, 아직 다루지 않은 것들을 중심으로 설명한다.

GPU가 노는 이유는 크게 세 가지다:

1. **Compute/Memory 불균형**: Prefill은 FLOPS를 다 쓰는데 bandwidth는 남고, decode는 그 반대
2. **낭비되는 계산**: 이미 계산된 KV cache를 다시 계산하거나, 유저가 이미 떠난 요청을 계속 처리
3. **비탄력적 할당**: 트래픽이 줄어도 GPU가 묶여 있고, 워커가 죽으면 그 GPU가 통째로 유휴

Dynamo는 이 세 가지 낭비를 각각 다른 메커니즘으로 공략한다.

### 6-1. Disaggregated Serving (→ 섹션 3에서 상세)

**풀려는 낭비: Compute/Memory 불균형**

Prefill과 decode를 분리해서 각각의 GPU가 자기 특성에 맞는 워크로드만 처리한다. Prefill GPU는 FLOPS를 끝까지 쓰고, decode GPU는 bandwidth를 끝까지 쓴다. 같은 GPU에서 둘을 돌릴 때 생기는 "한쪽이 놀고 있는" 상황을 구조적으로 제거한다.

### 6-2. Phase-specific Parallelism — prefill과 decode에 다른 병렬화 전략

**풀려는 낭비: 동일한 parallelism을 양쪽에 적용하는 비효율**

Disaggregated serving의 핵심 이점 중 하나인데, 종종 간과된다. Prefill과 decode를 분리하면 **각 단계에 서로 다른 TP/EP degree**를 적용할 수 있다.

예를 들어 MoE 모델(DeepSeek-R1 671B)을 서빙한다고 하자:

- **Prefill**: Compute-bound → 높은 TP로 GEMM 연산을 병렬화하는 게 유리
- **Decode**: Memory-bound → 넓은 EP(Expert Parallelism)로 expert를 최대한 분산해서 bandwidth를 극대화하는 게 유리

Aggregated 모드에서는 TP와 EP를 하나의 값으로 고정해야 하므로, 양쪽 모두에 최적인 설정을 잡을 수 없다. Disaggregated 모드에서는 prefill 워커에 TP=4, decode 워커에 EP=8 같은 식으로 독립 설정이 가능하다.

이것이 GB200 NVL72 + Dynamo 조합에서 MoE 모델 throughput이 Hopper 대비 15배까지 올라가는 핵심 원리 중 하나다.

### 6-3. E/P/D Disaggregation — MoE의 Expert까지 분리

**풀려는 낭비: MoE에서 Expert 처리의 비효율**

P/D disaggregation(Prefill/Decode 분리)을 한 단계 더 확장한 개념이다. MoE 모델에서는 Expert 연산 자체가 별도의 리소스 특성을 가지므로, **Expert / Prefill / Decode를 3단계로 분리**한다.

```
요청 → [Prefill 워커] → KV cache 전송 → [Decode 워커]
                                              ↕
                                      [Expert 워커] (MoE expert 연산)
```

MoE 모델에서 활성화되는 expert는 요청마다 다르다. Expert를 별도 워커로 분리하면, 각 expert를 더 효율적으로 분산 배치하고, expert 간 통신을 NVLink/NVSwitch로 최적화할 수 있다.

현재 Dynamo 로드맵에서 E/P/D disaggregation은 적극 개발 중이며, SGLang과 TRT-LLM 백엔드에서 wideEP 스크립트가 이미 제공되고 있다.

### 6-4. KV-aware Routing (→ 섹션 4에서 상세)

**풀려는 낭비: KV cache 중복 계산**

같은 prefix의 KV cache를 여러 GPU에서 중복으로 계산하는 것은 순수한 FLOPS 낭비다. Smart Router가 이미 캐싱된 KV 블록이 있는 GPU로 요청을 보내면, prefill 연산 자체를 건너뛸 수 있다. 건너뛴 만큼의 compute cycle이 다른 요청을 처리하는 데 쓰인다.

최근 릴리즈에서는 **LoRA adapter 상태까지 라우팅에 반영**한다. 같은 base model에 여러 LoRA를 서빙하는 환경에서, 요청의 adapter ID에 맞는 워커로 보내면 adapter 로딩/스왑 오버헤드를 줄일 수 있다.

### 6-5. Request Cancellation & Migration — 낭비되는 compute 즉시 회수

**풀려는 낭비: 이미 불필요한 요청에 GPU를 쓰는 것**

유저가 응답을 기다리다 연결을 끊으면(브라우저 탭 닫기, API 클라이언트 타임아웃 등), 해당 요청은 더 이상 결과를 받을 곳이 없다. 하지만 기존 서빙 엔진에서는 이런 요청이 decode가 끝날 때까지 GPU를 계속 점유한다.

Dynamo는 두 가지 메커니즘으로 이 낭비를 제거한다:

**Request Cancellation**: Frontend에서 유저 연결 끊김을 감지하면, 해당 요청의 취소 신호가 prefill → decode 워커까지 전파된다. 워커는 진행 중인 generation을 즉시 중단하고, 그 GPU 리소스(KV cache 슬롯, 배치 내 자리)를 다른 요청에 양보한다.

**Request Migration**: 워커가 죽거나 과부하 상태일 때, 진행 중인 요청을 다른 워커로 이전한다. `--migration-limit 3` 같은 설정으로 요청당 최대 마이그레이션 횟수를 지정할 수 있다. 워커가 종료 신호를 받으면 `GeneratorExit` 예외를 발생시키고, frontend가 자동으로 다른 워커에서 요청을 이어받게 한다.

이 두 메커니즘이 없으면, 클러스터 규모가 커질수록 "이미 필요 없는 요청"과 "죽은 워커에 묶인 요청"이 차지하는 GPU 자원이 누적되어 전체 utilization을 깎아먹는다.

### 6-6. Speculative Decoding — Decode GPU의 유휴 FLOPS 활용

**풀려는 낭비: Decode 시 남는 compute capacity**

Decode는 memory-bound이므로 GPU의 FLOPS가 남는다. Speculative decoding은 이 남는 FLOPS를 활용하는 기법이다.

작은 draft model(예: Eagle3)이 여러 토큰을 빠르게 "추측"하고, 큰 target model이 한 번의 forward pass로 추측을 검증한다. 검증을 통과한 토큰은 그대로 출력하고, 틀린 지점부터 다시 생성한다. 결과적으로 한 번의 decode step에서 여러 토큰을 출력할 수 있어 TPOT이 줄어든다.

Dynamo에서의 speculative decoding은 두 가지 모드로 동작한다:

- **Aggregated 모드**: 단일 워커 안에서 draft + target 모델이 함께 실행
- **Disaggregated 모드**: Draft model을 별도 컴포넌트("draft" 워커)로 분리하고, Dynamo의 서비스 디스커버리와 라우팅이 draft ↔ target 간 통신을 조율

vLLM 백엔드에서 Eagle3 기반 speculative decoding이 이미 지원되며, TRT-LLM과 SGLang에서도 확대 중이다.

### 6-7. Multi-LoRA — 하나의 GPU에서 여러 모델 서빙

**풀려는 낭비: 모델당 GPU를 따로 할당하는 비효율**

고객사마다 fine-tuned 모델이 다르지만, base model은 같은 경우가 많다. 각 fine-tuned 모델에 GPU를 따로 할당하면, 트래픽이 적은 모델의 GPU는 대부분 유휴 상태다.

Multi-LoRA는 하나의 base model weight를 GPU에 올려두고, 요청에 따라 다른 LoRA adapter를 동적으로 적용한다. GPU당 서빙할 수 있는 "모델 수"가 사실상 LoRA adapter 수만큼 늘어나는 셈이다.

Dynamo의 Multi-LoRA 지원은 단순히 엔진 레벨 기능을 넘어선다:

- **Adapter lifecycle 관리**: MinIO 같은 외부 스토리지에서 adapter를 동적으로 로드/언로드
- **KV-aware routing + LoRA**: Router가 adapter ID까지 고려해서, 해당 adapter가 이미 로드된 워커로 요청을 라우팅
- **Deterministic adapter ID**: 여러 replica에서 동일한 adapter에 대해 일관된 라우팅 보장

### 전체 그림 — 어디서 얼마나 GPU가 절약되는가

| 메커니즘 | 풀려는 낭비 | 효과가 큰 시나리오 | 성숙도 |
| --- | --- | --- | --- |
| Disaggregated Serving | Compute/Memory 불균형 | Long-context, reasoning 모델 | 프로덕션 |
| Phase-specific Parallelism | 단일 parallelism의 비효율 | MoE 대형 모델 | 프로덕션 |
| E/P/D Disaggregation | MoE expert 처리 비효율 | DeepSeek-R1 등 대형 MoE | 개발 중 |
| KV-aware Routing | KV cache 중복 계산 | 공통 system prompt, 멀티턴 대화 | 프로덕션 |
| Request Cancel/Migration | 불필요한 요청의 GPU 점유 | 대규모 API 서비스 | 프로덕션 |
| Speculative Decoding | Decode 시 남는 FLOPS | 짧은 output 다수 생성 | 초기 지원 |
| Multi-LoRA | 모델당 GPU 고정 할당 | 고객별 fine-tuned 모델 다수 | 초기 지원 |

이 메커니즘들은 서로 독립적이 아니라 **복합적으로 작용**한다. Disaggregated serving으로 prefill/decode를 분리하고, phase-specific parallelism으로 각 단계의 병렬화를 최적화하고, KV-aware routing으로 중복 계산을 제거하고, request cancellation으로 낭비를 즉시 회수하는 식이다. 하나의 GPU에서 단일 기법으로 얻는 개선이 아니라, 클러스터 전체에서 여러 기법이 시너지를 내는 구조다.

---

## 7. NIXL — 왜 NCCL만으로는 안 되는가

NCCL을 이미 쓰고 있는데 왜 또 다른 통신 라이브러리가 필요한가?

### NCCL vs NIXL — 역할이 다르다

| 항목 | NCCL | NIXL |
| --- | --- | --- |
| 설계 목적 | Collective communication (AllReduce, AllGather 등) | Point-to-point inference data transfer |
| 주 사용처 | 학습, TP/PP에서의 gradient/activation 동기화 | KV cache 전송, 메모리 계층 간 데이터 이동 |
| 통신 패턴 | All-to-all, broadcast, reduce | 1:1, 비동기, 배치 |
| 메모리 지원 | GPU HBM 위주 | HBM, DDR, NVMe, Object Storage 전부 |
| 네트워크 | NVLink, InfiniBand | NVLink, InfiniBand, PCIe, GDS 등 통합 |

NCCL은 "모든 GPU가 동시에 참여하는 집합 통신"에 최적화되어 있다. TP에서의 AllReduce 같은 패턴이다.

반면 disaggregated serving에서는 "prefill GPU 1번이 decode GPU 3번에게 KV cache 블록을 보내는" point-to-point 전송이 핵심이다. 이 전송의 소스가 GPU 메모리일 수도 있고, CPU 메모리일 수도 있고, SSD일 수도 있다. NCCL은 이런 이기종 메모리 간 전송을 지원하지 않는다.

NIXL은 이 문제를 **"Memory Section"이라는 추상화**로 해결한다. HBM이든 DDR이든 NVMe든 Object Storage든 동일한 API로 다루고, 최적의 전송 경로(GPUDirect RDMA, GDS 등)를 자동으로 선택한다.

---

## 8. Dynamo vs 경쟁 프레임워크 — 포지셔닝 정리

Disaggregated serving은 Dynamo만의 것이 아니다. 여러 프레임워크가 비슷한 문제를 다른 방식으로 풀고 있다.

| 프레임워크 | 핵심 접근 | 서빙 엔진 | 강점 | 약점 |
| --- | --- | --- | --- | --- |
| **NVIDIA Dynamo** | 4-컴포넌트 모듈 아키텍처 | vLLM, SGLang, TRT-LLM | 가장 완성도 높은 disagg. 구현, NIXL, Planner | NVIDIA 하드웨어에 최적화, 초기 복잡도 |
| **llm-d** | K8s-native, vLLM 기반 | vLLM | Red Hat 주도, K8s 생태계 통합 | vLLM 의존, 아직 초기 |
| **Ray Serve LLM** | Ray 생태계 위 P/D 분리 | vLLM + NIXL/LMCache | Ray 클러스터와 통합, RL 워크로드 연계 | Ray 의존성, K8s 외 환경에서 복잡 |
| **SGLang 자체 PD** | 엔진 내장 disaggregation | SGLang | 별도 프레임워크 불필요, 단순 | 클러스터 스케일 스케줄링 제한적 |

### 멀티 엔진 운용 — Dynamo만의 차별점

Dynamo는 **같은 클러스터에서 여러 서빙 엔진을 동시에 운용**할 수 있다. 예를 들어:

- 모델 A (Dense 70B) → vLLM 워커
- 모델 B (MoE 671B) → SGLang 워커
- 모델 C → TRT-LLM 워커

앞단의 OpenAI-compatible Frontend가 모델별로 적절한 워커 그룹으로 라우팅한다. 이전 글에서 "vLLM과 SGLang 둘 다 준비해야 한다"고 했는데, Dynamo가 이 "둘 다"를 하나의 클러스터에서 가능하게 만든다.

단, 하나의 모델에 대해 "prefill은 SGLang, decode는 vLLM"처럼 엔진을 섞는 건 안 된다. KV cache 포맷이 엔진마다 다르기 때문이다.

---

## 9. 언제 Dynamo가 필요하고, 언제 불필요한가

기술이 있다고 무조건 쓰는 게 정답은 아니다. Dynamo는 복잡도를 추가하는 레이어이므로, 그 복잡도에 상응하는 이점이 있을 때만 도입해야 한다.

### 필요한 경우

- **멀티노드 서빙**: GPU가 2개 이상의 노드에 걸쳐 있고, 노드 간 요청 분배가 필요할 때
- **Disaggregated PD가 의미 있는 워크로드**: Long-context input + 가변 output, reasoning 모델
- **MoE 모델의 Expert Parallelism**: 노드 간 expert 분산 시 통신 최적화
- **트래픽 변동이 큰 환경**: 시간대별로 워크로드 특성이 바뀌는 API 서비스
- **KV cache 재활용이 높은 패턴**: 공통 system prompt, 멀티턴 대화, RAG

### 불필요한 경우

- **단일 노드 TP2 서빙**: B300 288GB로 모델이 2장에 들어가면, 노드 내에서 vLLM/SGLang만으로 충분
- **고정 워크로드**: Input/output 길이가 일정하고 트래픽이 안정적이면 정적 할당으로 가능
- **GPU가 적은 환경**: 8장 이하에서 disaggregation의 이점보다 운영 복잡도가 더 클 수 있음
- **프로토타이핑/개발**: 빠르게 모델을 띄워보는 단계에서는 오버엔지니어링

### 판단 기준 요약

| 질문 | Yes → | No → |
| --- | --- | --- |
| GPU가 여러 노드에 분산되어 있는가? | Dynamo 검토 | 엔진 단독으로 충분할 가능성 높음 |
| Reasoning 모델 또는 long-context를 서빙하는가? | Disagg. PD의 이점 큼 | Aggregated로 충분할 수 있음 |
| 동일 prefix를 공유하는 요청이 많은가? | KV-aware routing 효과 큼 | 일반 LB로도 무방 |
| 트래픽 패턴이 시간대별로 크게 변하는가? | Planner의 동적 재배치 필요 | 고정 할당 가능 |
| 동시 서빙 세션이 GPU 메모리를 초과하는가? | KVBM 오프로딩 필요 | 메모리 내에서 처리 가능 |

위 질문 중 3개 이상이 Yes라면 Dynamo 도입을 적극 검토할 만하다. 1개 이하라면 엔진 단독 운용이 더 현실적이다.

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
