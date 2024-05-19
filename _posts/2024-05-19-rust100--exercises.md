---
title: 러스트 학습을 위한 100가지 예제
categories: archive
tags:
  - rust
  - exercise
published: true
---
https://rust-exercises.com

- Rust의 핵심 개념을 한번에 하나씩 실습을 통해 학습하는 방식으로 구성
- 문법, 타입 시스템, 표준 라이브러리 및 생태계를 배울 수 있음
- Rust에 대한 사전 지식은 필요하지 않지만, 다른 프로그래밍 언어에 대한 기본 지식은 필요함
- 시스템 프로그래밍이나 메모리 관리에 대한 사전 지식도 필요하지 않음
- 처음부터 시작하여 작은 단계로 Rust 지식을 쌓아나갈 수 있음
- 과정이 끝나면 약 100개의 연습 문제를 해결하여 소규모에서 중규모의 Rust 프로젝트를 다룰 수 있는 자신감을 가질 수 있음

## 방법론

- 이 과정은 "실습을 통한 학습(Learn By Doing)" 원칙에 기반함
- 상호작용적이고 실습 중심으로 설계
- 4일 동안 교실 환경에서 진행되도록 설계됨
    - 각 참가자는 자신의 속도에 맞춰 학습하며, 경험 많은 강사가 안내하고 질문에 답변하는 형식
- 혼자서도 과정을 따라갈 수 있지만, 친구나 멘토의 도움을 받는 것을 추천함
- 모든 연습 문제의 해답은 GitHub 저장소의 솔루션 브랜치에서 찾을 수 있음

## 구조

- 화면 왼쪽에 과정이 섹션으로 나뉘어 있음
- 각 섹션은 Rust 언어의 새로운 개념이나 기능을 소개함
- 이해도를 확인하기 위해 각 섹션에는 해결해야 할 연습 문제가 있음
- 연습 문제는 동반 GitHub 저장소에서 찾을 수 있음
- 과정을 시작하기 전에 저장소를 로컬 머신에 클론해야 함
- SSH 키가 설정된 경우: `git clone git@github.com:mainmatter/100-exercises-to-learn-rust.git`
- HTTPS URL을 사용하는 경우: `git clone [https://github.com/mainmatter/100-exercises-to-learn-rust.git](https://github.com/mainmatter/100-exercises-to-learn-rust.git)`
- 진행 상황을 쉽게 추적하고 필요 시 메인 저장소에서 업데이트를 가져오기 위해 브랜치에서 작업하는 것을 추천함
- 모든 연습 문제는 exercises 폴더에 위치함
- 각 연습 문제는 Rust 패키지로 구성됨
- 패키지에는 연습 문제 자체, 수행할 작업에 대한 지침(src/lib.rs) 및 솔루션을 자동으로 확인하는 테스트 스위트가 포함됨

## 저자 소개

- 이 과정은 Mainmatter의 수석 엔지니어링 컨설턴트인 Luca Palmieri가 작성함
- Luca는 2018년부터 Rust를 사용해왔으며, TrueLayer와 AWS에서 일함
- "Zero to Production in Rust"의 저자로, Rust로 백엔드 애플리케이션을 구축하는 방법을 배우는 데 필수적인 자원임
- cargo-chef, Pavex 및 wiremock을 포함한 다양한 오픈 소스 Rust 프로젝트의 저자이자 유지 관리자임


https://news.hada.io/topic?id=14872&utm_source=slack&utm_medium=bot&utm_campaign=T03FE7QJV 에서 퍼옴


왠지 공부는 안 할거 같지만 누군가 물으면 전달용..?ㅋㅋ
