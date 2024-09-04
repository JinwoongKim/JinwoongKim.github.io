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
- 본 논문에서는, GPU 메모리가 관리가 그다지 효율적이지 않다는 점을 문제로 삼고 있다.
- 그러다 보니 낭비가 되는
# 3. Cause
- 비효율적인 메모리 관리
- 

# 4. Proposal

# 5. Implementation

# 6. Evaluation

# 7. Discussion