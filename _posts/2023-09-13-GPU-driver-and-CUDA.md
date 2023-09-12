---
title:  "Nvidia GPU Driver and CUDA toolkit"
categories: "GPU"
tag: ["CUDA", "GPU", "GPU Driver"]
---

##### 0. 다루는 내용
- Nvidia 드라이버는 최신화해야하는가?
- 해야한다면 어떻게 하는가?

### 1. 최신화 해야 하는가?

#### 그래픽 드라이버란?

말 그대로 ...

#### 쿠다란?

GPGPU를 가능케 하는..

요즘 인공지능하면서 많이 ㅎ는 ..

#### 최신화 필요성

최신 라이브러리나 프레임워크는 최신 그래픽 드라이버를 요구하는 경우가 많다.
인프라 문제도 해결이 된다


### 2. 어떻게 업그레이드?

CUDA 툴킷 및 그래픽 드라이버 구조

<img src="/images/CUDA-components.png">

#### 도커를 쓰는 경우

그래픽 드라이버만 업그레이드 하고, 도커 내 쿠다 매핑

#### 도커를 쓰지 않는 경우

보통 두 개 다 업그레이드 한다. 다만 CUDA는 이미 그래픽 드라이버를 내장하고 있다. (호환 참조: https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html) 다만 Tesla GPU의 경우, 해당 드라이버는 개발용은 괜찮지만 서비스용으로 적합하진 않다고 한다. TeslaGPU로 서빙을 하려면 최신 드라이버로 업그레이드 하라고 한다.


<img src="/images/forward-compatibility.png">

### 참고
- CUDA Compatibility nvidia doc https://docs.nvidia.com/deploy/cuda-compatibility/
- 최신 드라이버 버전 확인 https://www.nvidia.com/download/index.aspx?lang=en-us
