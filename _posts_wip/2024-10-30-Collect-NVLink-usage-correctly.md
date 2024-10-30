---
title: NVLink 지표를 올바르게 수집하는 방법
categories: GPU
tags:
  - NVLink
  - 모니터링
  - 노하우
published: false
---

- GPU 클러스터 구축시, 혹은 개인 PC이더라도, 모니터링 시스템 구축하는 건 필수
- 제대로 쓰고 있는지 등을 알아야 이후 액션을 할 수 있기 때문
- 그 중에서도, 예전에 내가 테크 블로그 글에 올렸떤 NVLink같은걸 구축했다면 실제로 내가 이런걸 잘 쓰고 있나 확인 필요
	- 비싸기 때문..ㅋㅋ
- 보통 GPU인프라 모니터링시(NVdia 제품 기준) DCGM을 많이 이용함
	- 나중에 꼭 모니터링 해야 하는 GPU 주요 지표에 대해서 설명하겠음.
- 여기선 먼저, NVLink에 대해서 이야기 하겠음
	- 참고로 NVLink가 뭔지 잘 모른다면, 제가 올린 토스 테크 블로그 글을 참고 바람
- 기본적으로 DCGM에서 제공하는 NVLink는 ㅇㅇㅇ ㅁㅁㅁ 지표임
- 이러한 지표는, 대체로 해석이 안 됨
	- 일단 단위도 적혀 있지 않음
- 아래 코드는 CUDA로 구현한, 1GB 통신하는 코드임
- 요 코드를 수행후, 해당 지표를 보면 아래와 같이 나옴
- 그리고 2GB로 늘리고 지표를 봐도 아래처럼 나옴
-

https://grafana-service.tossinvest.bz/d/edxcekwheb4zkb/kubeflow-namespace-monitoring?orgId=1&refresh=5s&var-phase=k2_N-t34z&var-namespace=seonghyunkim&var-Service=seonghyunkim-h100-train-2&var-ib_local_name=All