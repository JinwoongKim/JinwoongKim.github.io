---
title: "[논문리뷰] Zero"
categories: papers
published: false
---

ZeRO: Memory Optimizations Toward Training Trillion Parameter Models
https://arxiv.org/abs/1910.02054

MS에서 19년도에 발표한 논문
당시에 읽어봤을때는 모델쪽을 정말 많이 몰라서(지금도 잘 모르지만;;) 잘 이해가 안됐었다.

요즘 https://huggingface.co/spaces/nanotron/ultrascale-playbook 으로 스터디를 하고 있어서 여기에서 나온 부분 다시 읽어보며 정리

---

기본적으로 Zero의 목적은 메모리 중복 제거를 통한 **메모리 최적화**이다. 이러한 메모리 최적화를 통해서 더 큰 모델을 학습하는 것이 최종 목표. 

하지만, 그렇기 때문에 커뮤니케이션이 늘어날 수가 있다. 따라서 네트워크 성능이 좋지 않은 경우, 더 큰 모델을 돌릴 수 있다는 것은 이점이지만, 학습 시간은 많이 늘어날 것으로 보여진다.

제로는 버전에 따라 아래처럼 파티셔닝을 하는데, 여기서 말하는 파티셔닝은 기본적으로 DP 축을 말하는 것이다.

- ZeRO-1: optimizer state partitioning
- ZeRO-2: optimizer state + gradient partitioning
- ZeRO-3: optimizer state + gradient + parameter partitioning