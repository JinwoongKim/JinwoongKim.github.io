---
title: "[논문리뷰] Efficient Memory Management for Large Language Model Serving with PagedAttention"
categories: papers
tags:
  - vLLM
  - 논문리뷰
published: false
---
vLLM 논문 리뷰
별생각 없었는데 써보는 사람들이 생각보다 호환이 잘 된다고 하고
SOSP에 등재되었다고 해서 찾아봄
Ion Stoica, AMPLab,... 아주 쟁쟁하네 하며 봤는데 novelty가 엄청 높진 않다..

# 1. Goal
- Main goal은 GPU 메모리 효율을 높여서 배치 사이즈를 늘리는 것
- 부차적으론, 여러가지 최적화 기법을 통해 latency도 낮췄다.
# 2. Problem
- KV cache 를 사용하는 LLM 기반의 인퍼런스의 경우 배치 사이즈를 쉬이 키우지 못하는 것을 문제로 지적하고 있다.
# 3. Cause
- 이러한 문제를 야기하는 원인으로는 KV cache 의 비효율적인 메
- 

# 4. Proposal

# 5. Implementation

# 6. Evaluation

# 7. Discussion