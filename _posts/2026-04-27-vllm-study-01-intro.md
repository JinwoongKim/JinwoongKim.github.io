---
title: "vLLM 스터디 1회차 — 도입부 & Engine 진입"
categories: LLM-Serving
tags:
  - vLLM
  - LLM-Serving
  - PagedAttention
  - Continuous-Batching
  - Inference
  - Study
published: true
---

새로운 스터디를 시작했다. Aleksa Gordic의 [Inside vLLM: Anatomy of a High-Throughput LLM Inference System](https://www.aleksagordic.com/blog/vllm)을 돌아가면서 발표하는데, 단순 발표가 아니라 **현재 vLLM 코드베이스까지 같이 따라가면서** 보기로 했다.

> - 원문 commit: `42172ad` (2025-08-09)
> - 현재 분석한 vllm main: `c0879d948` — 약 8개월 더 진행됨
> - 따라서 파일 위치 / 함수명이 옮겨진 부분은 따로 짚어준다

링크는 모두 [vllm-project/vllm](https://github.com/vllm-project/vllm) upstream main 기준이다.

---

## 1. 제목 & 부제

> **Inside vLLM: Anatomy of a High-Throughput LLM Inference System**
> *From paged attention, continuous batching, prefix caching, specdec, etc. to multi-GPU, multi-node dynamic serving at scale*

번역하면 — *vLLM 내부: 고처리량 LLM 추론 시스템 해부 / paged attention, continuous batching, prefix caching, 추측 디코딩(specdec)부터 멀티 GPU·멀티 노드 동적 서빙까지*.

핵심 키워드는 **High-Throughput**. vLLM은 "단일 요청 latency"가 아니라 **동시에 많은 요청을 GPU에 꾹꾹 채워 넣어 throughput을 끌어올리는 것**이 목표인 시스템이다. 부제에 등장한 5개 키워드(paged attention / continuous batching / prefix caching / specdec / multi-GPU·node serving)가 사실상 앞으로의 스터디 목차다.

---

## 2. 글의 구성 — 역피라미드(inverse-pyramid)

> This post is the first in a series. It starts broad and then layers in detail (following an inverse-pyramid approach) so you can form an accurate high-level mental model of the complete system without drowning in minutiae.

처음엔 넓게 시작해서 점점 디테일을 쌓아가는 **역피라미드** 방식. 사소한 디테일에 빠지지 않고 전체 시스템의 고수준 mental model을 갖게 하는 것이 목표.

스터디 진행에도 그대로 적용한다 — *"왜 이 컴포넌트가 필요한가 → 어떻게 동작하는가 → 코드 어디에 있는가"* 순서. 처음부터 PagedAttention CUDA 커널부터 보면 청중을 잃는다.

---

## 3. 시리즈 5파트 + 코드 매핑

블로그 시리즈는 5개 파트로 구성된다.

1. **LLM engine & engine core** — scheduling, paged attention, continuous batching 등 fundamentals
2. **Advanced features** — chunked prefill, prefix caching, guided & speculative decoding, disaggregated P/D
3. **Scaling up** — single-GPU → multi-GPU
4. **Serving layer** — distributed / concurrent web scaffolding
5. **Benchmarks and auto-tuning** — latency / throughput 측정

각 파트가 현재 vllm 레포의 어디에 매핑되는지 정리:

| 블로그 파트 | vllm 디렉토리 / 핵심 파일 |
|---|---|
| ① Engine & Engine Core | [vllm/v1/engine/](https://github.com/vllm-project/vllm/tree/main/vllm/v1/engine), [vllm/v1/core/](https://github.com/vllm-project/vllm/tree/main/vllm/v1/core), [vllm/v1/core/sched/](https://github.com/vllm-project/vllm/tree/main/vllm/v1/core/sched) |
| ② Advanced features | [vllm/v1/attention/](https://github.com/vllm-project/vllm/tree/main/vllm/v1/attention) (paged attention)<br>[vllm/v1/core/kv_cache_manager.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/kv_cache_manager.py) (prefix caching)<br>[vllm/v1/spec_decode/](https://github.com/vllm-project/vllm/tree/main/vllm/v1/spec_decode) (speculative decoding)<br>[vllm/v1/structured_output/](https://github.com/vllm-project/vllm/tree/main/vllm/v1/structured_output) (guided decoding) |
| ③ Scaling up | [vllm/v1/executor/](https://github.com/vllm-project/vllm/tree/main/vllm/v1/executor)<br>[vllm/v1/worker/](https://github.com/vllm-project/vllm/tree/main/vllm/v1/worker)<br>[vllm/distributed/](https://github.com/vllm-project/vllm/tree/main/vllm/distributed) |
| ④ Serving layer | [vllm/entrypoints/api_server.py](https://github.com/vllm-project/vllm/blob/main/vllm/entrypoints/api_server.py)<br>[vllm/entrypoints/openai/](https://github.com/vllm-project/vllm/tree/main/vllm/entrypoints/openai) |
| ⑤ Benchmarks | [benchmarks/](https://github.com/vllm-project/vllm/tree/main/benchmarks)<br>[vllm/benchmarks/](https://github.com/vllm-project/vllm/tree/main/vllm/benchmarks) |

---

## 4. 분석 기준

> Analysis is based on commit `42172ad` (August 9th, 2025).
> Target audience: anyone curious about how state-of-the-art LLM engines work, as well as those interested in contributing to vLLM, SGLang, etc.
> I'll focus on the V1 engine. I also explored V0 (now deprecated)…

타겟 청중은 *state-of-the-art LLM 엔진의 동작이 궁금하거나, vLLM/SGLang 같은 프로젝트에 기여하고 싶은 사람*. **V1 엔진**에만 집중한다. V0는 이미 deprecated.

⚠️ 우리 레포 main은 블로그보다 약 8개월 더 진행됐기 때문에 파일이 옮겨지거나 함수명이 바뀐 부분이 있다. 차이가 보일 때마다 *"blog 시점 vs 현재"* 로 짚어주면 청중 관점에서 가치가 크다.

---

## 5. 실행 예제 — offline 추론 6줄

```python
from vllm import LLM, SamplingParams

prompts = [
    "Hello, my name is",
    "The president of the United States is",
]

sampling_params = SamplingParams(temperature=0.8, top_p=0.95)

def main():
    llm = LLM(model="TinyLlama/TinyLlama-1.1B-Chat-v1.0")
    outputs = llm.generate(prompts, sampling_params)

if __name__ == "__main__":
    main()
```

📂 현재 레포 실제 파일: [examples/basic/offline_inference/basic.py](https://github.com/vllm-project/vllm/blob/main/examples/basic/offline_inference/basic.py)
(블로그에선 단순히 `basic.py`라 적혀있지만 현재는 경로가 정리됐고, 모델도 OPT-125m으로 바뀌어 있다.)

이 짧은 코드가 앞으로 수십 페이지 분석의 **러닝 예제**가 된다. 모든 컴포넌트는 `LLM(...)` 생성자 안에서 만들어지고, `llm.generate(...)` 한 번 호출로 동작한다.

---

## 6. 환경 변수 — V1 / 단일 프로세스 모드

```bash
# Engine V1 사용 (현재 main에선 default)
VLLM_USE_V1="1"

# EngineCore 멀티프로세싱을 끔 → 한 프로세스 안에서 sync 실행
VLLM_ENABLE_V1_MULTIPROCESSING="0"
```

`VLLM_ENABLE_V1_MULTIPROCESSING=0`이면 EngineCore가 **별도 프로세스가 아니라 호출자와 같은 프로세스**에서 돌게 된다. 디버거로 step-through하면서 흐름을 따라가기 좋다.

이 변수는 코드 경로를 분기시킨다. 끄면 `InprocClient`, 켜면 `SyncMPClient` / `AsyncMPClient` 가 사용된다 ([vllm/v1/engine/core_client.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/engine/core_client.py)의 `EngineCoreClient.make_client()`).

```python
# core_client.py 요약
class EngineCoreClient(ABC):
    @staticmethod
    def make_client(multiprocess_mode, asyncio_mode, ...):
        # multiprocess_mode == False  → InprocClient  (학습용)
        # multiprocess_mode == True   → Sync/AsyncMPClient  (운영용)
        ...

class InprocClient(EngineCoreClient):       # 같은 프로세스
    def __init__(self, *args, **kwargs):
        self.engine_core = EngineCore(*args, **kwargs)  # 직접 생성

class MPClient(EngineCoreClient):           # 별도 프로세스 + ZMQ IPC
    ...
class SyncMPClient(MPClient): ...
class AsyncMPClient(MPClient): ...
```

> 모든 `VLLM_*` 환경변수 정의는 [vllm/envs.py](https://github.com/vllm-project/vllm/blob/main/vllm/envs.py)에 모여 있다.

---

## 7. 분석 시작 시 가정 — 단순화 사다리

> This configuration is:
> - **offline** (no web/distributed system scaffolding)
> - **synchronous** (single blocking process)
> - **single-GPU** (DP/TP/PP/EP = 1)
> - **standard transformer** (no hybrid models like Jamba)

4가지 단순화 가정에서 출발해서, 한 단계씩 풀어가며 vLLM 전체로 확장하는 게 시리즈 전체의 진행 방식이다.

| 가정 | 풀리는 시점 (블로그 파트) | 관련 코드 |
|---|---|---|
| offline (웹 X) | Part 4 (Serving layer) | [vllm/entrypoints/openai/](https://github.com/vllm-project/vllm/tree/main/vllm/entrypoints/openai) |
| synchronous | Part 1 끝 ~ Part 3 | [vllm/v1/engine/async_llm.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/engine/async_llm.py)<br>[vllm/v1/engine/core_client.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/engine/core_client.py) (Async 계열) |
| single-GPU (DP/TP/PP/EP = 1) | Part 3 (Scaling up) | [vllm/distributed/](https://github.com/vllm-project/vllm/tree/main/vllm/distributed)<br>[vllm/v1/executor/](https://github.com/vllm-project/vllm/tree/main/vllm/v1/executor) |
| standard transformer (no hybrid) | (시리즈 후속편) | [vllm/v1/core/kv_cache_coordinator.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/kv_cache_coordinator.py)<br>[vllm/v1/core/single_type_kv_cache_manager.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/single_type_kv_cache_manager.py) |

---

## 8. "두 가지 일" — Constructor & generate

> In this example we do two things, we:
> 1. Instantiate an engine
> 2. Call generate on it to sample from the given prompts
>
> Let's start analyzing the constructor.

이제 본격적으로 코드로 들어간다.

---

## 🔬 코드 들여다보기: `LLM.__init__` 첫 진입

### ① `LLM.__init__` 시그니처가 알려주는 것

파일: [vllm/entrypoints/llm.py](https://github.com/vllm-project/vllm/blob/main/vllm/entrypoints/llm.py) — 클래스 `LLM`, `__init__` 시작은 라인 212.

```python
def __init__(
    self,
    model: str,
    *,
    runner: RunnerOption = "auto",
    tokenizer: str | None = None,
    tokenizer_mode: TokenizerMode | str = "auto",
    skip_tokenizer_init: bool = False,
    trust_remote_code: bool = False,
    # ... 분량 많음 ...
    tensor_parallel_size: int = 1,
    dtype: ModelDType = "auto",
    quantization: QuantizationMethods | None = None,
    seed: int = 0,
    gpu_memory_utilization: float = 0.92,
    cpu_offload_gb: float = 0,
    enforce_eager: bool = False,
    disable_custom_all_reduce: bool = False,
    # ...
    compilation_config: int | dict[str, Any] | CompilationConfig | None = None,
    logits_processors: list[str | type[LogitsProcessor]] | None = None,
    **kwargs: Any,
) -> None:
```

읽는 법:

- `tensor_parallel_size=1`, `gpu_memory_utilization=0.92` 같은 default → **"단일 GPU + GPU 메모리의 92%를 vLLM에 쓴다"** 가 디폴트
- `enforce_eager=False` → 기본은 **CUDA Graph 캡처 사용**. 디버깅할 땐 보통 `enforce_eager=True`로 켠다
- `compilation_config`는 int / dict / CompilationConfig 객체 모두 받음 → 단계적 진화 흔적
- `**kwargs`로 EngineArgs의 거의 모든 필드를 그대로 흘려보냄 → "사용자 친화적 LLM 클래스"는 사실상 **EngineArgs 빌더의 얇은 wrapper**

### ② Body — Config로 변환되어 EngineArgs로

```python
# vllm/entrypoints/llm.py:340 부근
engine_args = EngineArgs(
    model=model,
    runner=runner,
    tokenizer=tokenizer,
    tensor_parallel_size=tensor_parallel_size,
    gpu_memory_utilization=gpu_memory_utilization,
    enforce_eager=enforce_eager,
    compilation_config=compilation_config_instance,
    # ... 위에서 받은 인자 거의 전부 ...
    **kwargs,
)

log_non_default_args(engine_args)

# 핵심: 진짜 엔진은 LLMEngine 가 만든다
self.llm_engine = LLMEngine.from_engine_args(
    engine_args=engine_args,
    usage_context=UsageContext.LLM_CLASS,
)
```

> 💡 `LLM` 클래스는 사용자 입력을 받아 `EngineArgs`로 묶고, `LLMEngine.from_engine_args`를 호출하는 게 전부. 실제 엔진 본체는 `LLMEngine`이며 그 자식이 `EngineCore`.

### ③ `LLMEngine.from_engine_args` — config 만들고 LLMEngine 생성

파일: [vllm/v1/engine/llm_engine.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/engine/llm_engine.py) — `from_engine_args` @ 라인 152.

```python
@classmethod
def from_engine_args(
    cls, engine_args: EngineArgs,
    usage_context: UsageContext = UsageContext.ENGINE_CONTEXT,
    stat_loggers: list[StatLoggerFactory] | None = None,
    enable_multiprocessing: bool = False,
) -> "LLMEngine":
    # 1) EngineArgs → VllmConfig 로 정규화
    vllm_config = engine_args.create_engine_config(usage_context)

    # 2) Executor 클래스 결정 (single GPU vs Ray vs MP)
    executor_class = Executor.get_class(vllm_config)

    # 3) 환경변수 보고 multiprocessing 토글
    if envs.VLLM_ENABLE_V1_MULTIPROCESSING:
        enable_multiprocessing = True

    return cls(
        vllm_config=vllm_config,
        executor_class=executor_class,
        log_stats=not engine_args.disable_log_stats,
        usage_context=usage_context,
        stat_loggers=stat_loggers,
        multiprocess_mode=enable_multiprocessing,
    )
```

- **VllmConfig**: 모든 하위 config(model, parallel, scheduler, cache, …)의 통합 컨테이너. 앞으로 vLLM 전체에서 가장 많이 보게 될 객체
- **Executor.get_class()**: GPU 한 장이면 `UniProcExecutor`, 여러 장이면 `MultiprocExecutor` / `RayDistributedExecutor` 등으로 분기. [vllm/v1/executor/](https://github.com/vllm-project/vllm/tree/main/vllm/v1/executor) 참고
- **multiprocess_mode**: 위에서 본 InprocClient ↔ MPClient 분기 스위치

### ④ `LLMEngine.__init__` — 엔진의 골격

파일: [vllm/v1/engine/llm_engine.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/engine/llm_engine.py) — `__init__` 시작 라인 50.

```python
def __init__(self, vllm_config, executor_class, log_stats, ...):
    self.vllm_config   = vllm_config
    self.model_config  = vllm_config.model_config
    self.observability_config = vllm_config.observability_config

    # renderer: prompt → token 시퀀스 변환 (chat template, tokenizer 적용)
    self.renderer = renderer = renderer_from_config(self.vllm_config)

    # input_processor: EngineInput → EngineCoreRequest 변환
    self.input_processor = InputProcessor(self.vllm_config, renderer)

    # output_processor: EngineCoreOutputs → RequestOutput 으로 풀어줌 (detokenize 등)
    self.output_processor = OutputProcessor(
        renderer.tokenizer,
        log_stats=self.log_stats,
        stream_interval=self.vllm_config.scheduler_config.stream_interval,
        tracing_enabled=tracing_endpoint is not None,
    )

    # 핵심: EngineCore 클라이언트 — Inproc 또는 MPClient
    self.engine_core = EngineCoreClient.make_client(
        multiprocess_mode=multiprocess_mode,
        asyncio_mode=False,
        vllm_config=vllm_config,
        executor_class=executor_class,
        log_stats=self.log_stats,
    )
```

🧱 **LLMEngine의 4대 컴포넌트**

1. **renderer** — prompt → token (chat template/tokenizer 적용)
2. **input_processor** — EngineInput → `EngineCoreRequest`
3. **engine_core** (Client) — 실제 추론 루프 보유. Inproc/MP 두 종류
4. **output_processor** — `EngineCoreOutputs` → `RequestOutput` (detokenize 등)

데이터 흐름:
`prompt` → renderer → input_processor → engine_core → output_processor → `RequestOutput`

### ⑤ `LLMEngine.step` — 한 사이클의 전부

파일: [vllm/v1/engine/llm_engine.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/engine/llm_engine.py) — `step` @ 라인 287.

```python
def step(self) -> list[RequestOutput | PoolingRequestOutput]:
    # 1) EngineCore 한 번 돌리고 결과 받아오기
    outputs = self.engine_core.get_output()

    # 2) detokenize 등 후처리
    processed_outputs = self.output_processor.process_outputs(
        outputs.outputs,
        engine_core_timestamp=outputs.timestamp,
        iteration_stats=iteration_stats,
    )
    self.output_processor.update_scheduler_stats(outputs.scheduler_stats)

    # 3) stop string 등으로 끝난 요청 abort
    self.engine_core.abort_requests(processed_outputs.reqs_to_abort)

    # 4) stats 기록
    # ...

    return processed_outputs.request_outputs
```

이 4단계가 **vLLM의 한 "iteration"** 이고, 이게 반복돼서 모든 요청이 끝날 때까지 돌아간다. `llm.generate()`는 결국 `has_unfinished_requests()` 동안 `step()`을 반복 호출하는 루프다.

### ⑥ `EngineCore` — 진짜 추론 루프의 본체

파일: [vllm/v1/engine/core.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/engine/core.py) — 클래스 `EngineCore` @ 라인 91.

```python
class EngineCore:
    """Inner loop of vLLM's Engine."""

    def __init__(self, vllm_config, executor_class, log_stats, ...):
        # 1) 모델 실행기 — 워커들에게 모델/메모리/배치 분배
        self.model_executor = executor_class(vllm_config)

        # 2) KV cache 메모리 측정 + 페이지 단위 블록 풀 구성
        kv_cache_config = self._initialize_kv_caches(vllm_config)

        # 3) 스케줄러 — Continuous batching의 심장
        Scheduler = vllm_config.scheduler_config.get_scheduler_cls()
        self.scheduler = Scheduler(
            vllm_config=vllm_config,
            kv_cache_config=kv_cache_config,
            structured_output_manager=self.structured_output_manager,
            ...
        )

        # 4) (옵션) speculative decoding
        self.use_spec_decode = vllm_config.speculative_config is not None

        # 5) (옵션) prefix caching 용 hash 함수 셋업
        if vllm_config.cache_config.enable_prefix_caching or kv_connector is not None:
            ...
            self.request_block_hasher = get_request_block_hasher(...)

        # 6) step 함수 결정 (PP 켜져 있으면 batch_queue 버전)
        self.step_fn = (
            self.step if self.batch_queue is None
            else self.step_with_batch_queue
        )
```

🎯 **여기서 보이는 것**: 블로그 시리즈의 다음 키워드들이 `EngineCore.__init__`에 그대로 박혀있다.

- **scheduler** → continuous batching
- **kv_cache_config / `_initialize_kv_caches`** → paged attention 메모리 풀
- **request_block_hasher** → prefix caching
- **use_spec_decode** → speculative decoding
- **batch_queue / step_with_batch_queue** → pipeline parallelism

1회차에서는 *"이 다섯이 EngineCore 안에 다 모여있다"* 는 것만 인지하면 충분. 각각의 디테일은 다음 회차에서 한 줄씩 풀어간다.

### ⑦ 전체 흐름 요약 (call chain)

```
# 사용자 코드
LLM(model=...)                                  ─┐
                                                 │
# entrypoints/llm.py:212                         │
LLM.__init__                                     │
  └─ EngineArgs(...)                             │
  └─ LLMEngine.from_engine_args(engine_args)     │
                                                 │
# v1/engine/llm_engine.py:152                    │ "엔진을 만든다"
LLMEngine.from_engine_args                       │
  └─ engine_args.create_engine_config()          │
  └─ Executor.get_class(vllm_config)             │
  └─ LLMEngine(vllm_config, ...)                 │
                                                 │
# v1/engine/llm_engine.py:50                     │
LLMEngine.__init__                               │
  ├─ renderer / input_processor / output_processor
  └─ EngineCoreClient.make_client(...)           │
       └─ InprocClient(...) ─┐                   │
                              ▼                  │
# v1/engine/core.py:91                           │
       EngineCore.__init__                       │
         ├─ model_executor (Executor)            │
         ├─ _initialize_kv_caches()              │
         ├─ Scheduler(...)                       │
         ├─ spec decode / prefix hash / batch_q  │
         └─ step_fn 결정                        ─┘

# 사용자 코드
llm.generate(prompts, sampling_params)           ─┐
  └─ LLM._validate_and_add_requests(...)         │
  └─ while llm_engine.has_unfinished_requests(): │ "generate 호출"
       step_outputs = llm_engine.step()          │
                                                 │
LLMEngine.step()                                 │
  └─ engine_core.get_output()                    │
  └─ output_processor.process_outputs(...)       │
  └─ engine_core.abort_requests(...)            ─┘
```

---

## 🧩 LLM Engine constructor — 엔진의 부속들

> The main components of the engine are:
> - vLLM config (contains all of the knobs for configuring model, cache, parallelism, etc.)
> - processor (turns raw inputs → EngineCoreRequests via validation, tokenization, and processing)
> - engine core client (in our running example we're using InprocClient which is basically == EngineCore; we'll gradually build up to DPLBAsyncMPClient which allows serving at scale)
> - output processor (converts raw EngineCoreOutputs → RequestOutput that the user sees)

엔진의 주요 컴포넌트는 — (1) **vLLM config** (모델/캐시/병렬화 등 모든 설정 노브), (2) **processor** (raw 입력 → 검증·토크나이즈·전처리 → `EngineCoreRequest`), (3) **engine core client** (러닝 예제에선 `InprocClient` ≈ `EngineCore`, 이후 점진적으로 대규모 서빙용 `DPLBAsyncMPClient`까지 확장), (4) **output processor** (raw `EngineCoreOutputs` → 사용자에게 보이는 `RequestOutput`).

> 📝 V0가 deprecated 되면서 클래스 이름·시그니처는 계속 바뀌는 중. 중요한 건 정확한 이름이 아니라 *core idea*.

### ① 4대 컴포넌트 — 현재 코드 매핑

| 블로그 표현 | 현재 vllm 클래스 | 위치 |
|---|---|---|
| `vLLM config` | `VllmConfig` | [vllm/config/](https://github.com/vllm-project/vllm/tree/main/vllm/config) |
| `processor` | `InputProcessor` (+ `Renderer`) | [v1/engine/input_processor.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/engine/input_processor.py) |
| `engine core client` | `InprocClient` → `SyncMPClient` → `AsyncMPClient` → `DPLBAsyncMPClient` | [v1/engine/core_client.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/engine/core_client.py) |
| `output processor` | `OutputProcessor` | [v1/engine/output_processor.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/engine/output_processor.py) |

### ② Engine Core 내부 — sub components

> Engine core itself is made up of several sub components:
> - Model Executor (drives forward passes; UniProcExecutor → MultiProcExecutor)
> - Structured Output Manager (guided decoding)
> - Scheduler (decides which requests go into the next engine step)
>   - policy: FCFS or priority
>   - waiting / running queues
>   - KV cache manager — heart of paged attention

| sub-component | 현재 vllm 위치 |
|---|---|
| Model Executor (UniProc) | [v1/executor/uniproc_executor.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/executor/uniproc_executor.py) (라인 26: `class UniProcExecutor`) |
| Model Executor (MultiProc) | [v1/executor/](https://github.com/vllm-project/vllm/tree/main/vllm/v1/executor) |
| Structured Output Manager | [v1/structured_output/](https://github.com/vllm-project/vllm/tree/main/vllm/v1/structured_output) |
| Scheduler | [v1/core/sched/scheduler.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/sched/scheduler.py) (라인 67) |
| KV Cache Manager | [v1/core/kv_cache_manager.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/kv_cache_manager.py) (라인 106) |
| Block Pool / `free_block_queue` | [v1/core/block_pool.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/block_pool.py) (라인 130: `class BlockPool`, 라인 168: `self.free_block_queue = FreeKVCacheBlockQueue(...)`) |

Scheduler 내부를 실제 코드로 보면 *policy / waiting / running* 이 그대로 변수명에 박혀있다 — 글과 코드의 일대일 매핑.

```python
# vllm/v1/core/sched/scheduler.py:67~170
class Scheduler(SchedulerInterface):
    def __init__(self, ...):
        ...
        self.max_num_running_reqs = self.scheduler_config.max_num_seqs

        # policy 에 따라 큐 구현이 달라짐 (FCFS deque / priority heap)
        self.waiting          = create_request_queue(self.policy)
        self.skipped_waiting  = create_request_queue(self.policy)  # async 의존/제약 때문에 스킵된 요청
        self.running: list[Request] = []
```

### ③ KV Cache Manager — `free_block_queue` 의 정체

> The KV-cache manager maintains a `free_block_queue` — a pool of available KV-cache blocks (often on the order of hundreds of thousands, depending on VRAM size and block size). During paged attention, the blocks serve as the indexing structure that map tokens to their computed KV cache blocks.

KV cache manager는 `free_block_queue`(가용 KV cache 블록 풀)를 들고 있다. VRAM 크기와 block size에 따라 **수십만 단위**까지 만들어진다. paged attention 동안 이 블록들이 *"토큰 → 계산된 KV cache 블록"* 으로 매핑하는 **indexing structure** 역할을 한다.

![](../images/2026-04-27-vllm-study-01-intro/engine-architecture.png)
![](/images/2026-04-27-vllm-study-01-intro/engine-architecture.png)

그림 해석: *위쪽* 에 engine core client → engine core(model executor / scheduler / SOM) → output processor의 흐름, *중간* 에 CPU상의 `block_pool` 인덱스 구조, *아래* 에 GPU상의 paged KV cache memory 블록들. **인덱스(CPU)와 실제 메모리(GPU)의 분리**가 paged attention의 핵심이다.

### ④ 표준 트랜스포머의 block size 공식

> Block size for a standard transformer layer (non-MLA) is computed as follows:
> `2 (key/value) * block_size (default=16) * num_kv_heads * head_size * dtype_num_bytes (e.g. 2 for bf16)`

- **2** = K, V 둘 다 저장
- **block_size = 16** = 한 블록에 토큰 16개분 KV가 들어감 (vLLM 디폴트)
- **num_kv_heads × head_size** = GQA에서 group된 KV head 차원
- **dtype_num_bytes** = bf16/fp16이면 2, fp8이면 1

예) Llama-3-8B, bf16, num_kv_heads=8, head_size=128, num_layers=32 → 레이어당 한 블록 = `2 · 16 · 8 · 128 · 2 = 65,536 B = 64 KiB`. 32 레이어면 **한 블록(=토큰 16개) 당 2 MiB**. 80 GiB GPU면 단순 계산으로 block 수가 수만~수십만 개 단위 — 그림의 "수십만"의 근거.

### ⑤ Model Executor 내부 — Worker의 3대 절차

> During model executor construction, a Worker object is created, and three key procedures are executed.

Model Executor를 만들 때 `Worker` 객체가 생성되고 **3대 절차**가 실행된다. 이후 `MultiProcExecutor`가 들어오면 이 같은 절차가 GPU별 worker 프로세스에서 각각 독립적으로 돈다.

코드: [vllm/v1/worker/gpu_worker.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/worker/gpu_worker.py) — `class Worker` @ 라인 105.

**(1) `init_device` — 라인 219**

- CUDA device 할당 (예: `"cuda:0"`) + 모델 dtype 지원 여부 검증 (bf16 등)
- 요청한 `gpu_memory_utilization` (예: 0.8 → VRAM 80%) 만큼 메모리 잡을 수 있는지 확인
- 분산 설정 셋업 (DP / TP / PP / EP …)
- `model_runner` 인스턴스화 — sampler, KV cache, forward용 버퍼(`input_ids`, positions 등) 보유
- `InputBatch` 인스턴스화 — CPU 측 forward 버퍼, KV cache indexing용 block table, sampling metadata 등

관련 클래스: [GPUModelRunner](https://github.com/vllm-project/vllm/blob/main/vllm/v1/worker/gpu_model_runner.py) (라인 394) · [InputBatch](https://github.com/vllm-project/vllm/blob/main/vllm/v1/worker/gpu_input_batch.py) (라인 81)

**(2) `load_model` — 라인 318**

- 모델 아키텍처 인스턴스화
- 모델 weight 로드
- `model.eval()` 호출 (PyTorch inference 모드)
- Optional: `torch.compile()` 호출

**(3) Initialize KV cache — `determine_available_memory` + `compile_or_warm_up_model`**

- 레이어별 KV cache spec 얻기. 과거엔 항상 `FullAttentionSpec`(homogeneous transformer)이었는데, hybrid model(sliding window, Transformer/SSM Jamba 등)이 등장하면서 복잡해짐 — Jenga 논문 참고. [kv_cache_interface.py:164 `class FullAttentionSpec`](https://github.com/vllm-project/vllm/blob/main/vllm/v1/kv_cache_interface.py)
- **dummy / profiling forward pass** 를 한 번 돌리고 GPU 메모리 스냅샷 → 가용 VRAM 안에 KV cache 블록 몇 개가 들어가는지 계산. [gpu_worker.py:332 `determine_available_memory`](https://github.com/vllm-project/vllm/blob/main/vllm/v1/worker/gpu_worker.py)
- KV cache 텐서 **allocate / reshape / attention 레이어에 bind**
- attention metadata 준비 (예: backend = FlashAttention) — 추후 forward 중 커널이 소비
- `--enforce-eager` 가 아니면 **warmup batch size 별로 dummy run하고 CUDA Graph 캡처**. CUDA Graph는 GPU 작업 시퀀스를 DAG로 기록해 두고, 실제 forward에서 launch/replay만 하므로 kernel launch overhead를 잘라내고 latency를 줄임. [gpu_worker.py:552 `compile_or_warm_up_model`](https://github.com/vllm-project/vllm/blob/main/vllm/v1/worker/gpu_worker.py)

🎯 **요약**: Worker init은 결국 *"GPU 잡기 → 모델 로드 → KV cache 풀 잡고 CUDA Graph 캡처"* 의 3 step. **가용 VRAM 측정 → 블록 수 결정** 이 paged attention 메모리 계획의 핵심 포인트.

> Now that we have the engine initialized let's proceed to the generate function.

여기까지가 엔진 생성자(constructor) 분석. 다음 단계는 `generate()` 함수.

---

## ❓ 발표 전 Q&A 체크리스트

발표 전, 이 4가지에 답할 수 있어야 안전하다.

**Q1.** V1 엔진과 V0 엔진의 가장 큰 구조적 차이는?
> 힌트: V1은 EngineCore를 별도 프로세스로 분리(IPC) + 스케줄러 단순화. V0의 sequence-level scheduling vs V1의 request/iteration-level scheduling 차이.

**Q2.** `VLLM_ENABLE_V1_MULTIPROCESSING=0`으로 끄면 사라지는 컴포넌트는 구체적으로 무엇인가?
> → `MPClient` / `SyncMPClient` / `AsyncMPClient` 경로와 ZMQ 기반 IPC가 사라지고, `InprocClient`가 직접 `EngineCore`를 들고 있게 된다 ([core_client.py:274](https://github.com/vllm-project/vllm/blob/main/vllm/v1/engine/core_client.py)).

**Q3.** DP / TP / PP / EP 각각이 vLLM 어디서 어떻게 enable되는가?
> → `EngineArgs.tensor_parallel_size`, `pipeline_parallel_size`, `data_parallel_size`, `enable_expert_parallel` 등을 통해 `VllmConfig.parallel_config`로 모이고, `Executor.get_class()`가 적절한 Executor로 분기.

**Q4.** 왜 hybrid model(Jamba 등)이 KV cache allocator를 더 복잡하게 만드는가?
> → Mamba(SSM)의 state는 *고정 크기*이고 attention KV는 *seq_len에 비례*해서 늘어난다. 두 종류의 메모리 풀을 한 모델 안에서 동시에 관리해야 한다 ([kv_cache_coordinator.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/kv_cache_coordinator.py)의 존재 이유).

---

## 📌 다음 회차 예고 — `generate()` 함수

- **대상 분량**: 블로그 본문 `generate()` 따라가는 부분. 엔진 생성자는 끝났고, 이제 실제 요청이 들어왔을 때 어떻게 동작하는지를 본다
- **코드 영역**: [entrypoints/llm.py](https://github.com/vllm-project/vllm/blob/main/vllm/entrypoints/llm.py)의 `generate()` / `_validate_and_add_requests` → [v1/engine/llm_engine.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/engine/llm_engine.py)의 `add_request` / `step` → [v1/core/sched/scheduler.py](https://github.com/vllm-project/vllm/blob/main/vllm/v1/core/sched/scheduler.py)의 `schedule()`
- **핵심 학습 목표**:
  1. 요청이 어떻게 `EngineCoreRequest`로 변환돼 큐에 들어가는지
  2. Scheduler가 한 iteration에 어떤 결정을 내리는지 (continuous batching의 정의)
  3. token budget / preemption / waiting → running 승격 로직

---

> 원문: [aleksagordic.com/blog/vllm](https://www.aleksagordic.com/blog/vllm)
> 분석 기준: vllm-project/vllm main `c0879d948`
