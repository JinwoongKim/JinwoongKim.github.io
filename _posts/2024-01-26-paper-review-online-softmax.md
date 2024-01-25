---
title: "[논문리뷰] Online normalizer calcuation for softmax by Nvidia (2018)"
categories: papers
tags:
  - 논문리뷰
  - softmax
  - Nvidia
  - FlashAttention
published: false
---
Flash Attention2 논문을 읽다 보니 이해가 안 되는 부분이 많았다. 팀내 연구원들은 GPU 파트를 이해 못 했는데 난 오히려 GPU 파트 보단 딥러닝파트가 이해가 안 됐다;;

그 중에서도 Flash Attention 2의 핵심 아이디어인 softmax 병렬 계산이 도통 이해가 안 돼서 관련 논문을 열심히 읽었다. 인용수가 22밖에 안돼서 그런가 자료가 매우 없어서 아쉬웠고, 그래서 나의 부족한 리뷰라도 누군가가에게 도움이 돼지 않을까 하여 올려본다.

논문 URL : https://arxiv.org/abs/1805.02867

# tl;dr
- Nvidia GPU 에서 softmax 연산시 메모리 엑세스 최소화 및 병렬 프로세싱을 다룬 논문
- 여타 다른 논문에선 근사치로 softmax 를 계산해서 최적화하는데 여기선 아님. 정확한 값이 나옴 

## softmax

![image](https://private-user-images.githubusercontent.com/12505517/299832146-c936ec0b-fa65-4e78-a0f1-860935199bec.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MDYyMjQ4MTksIm5iZiI6MTcwNjIyNDUxOSwicGF0aCI6Ii8xMjUwNTUxNy8yOTk4MzIxNDYtYzkzNmVjMGItZmE2NS00ZTc4LWEwZjEtODYwOTM1MTk5YmVjLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAxMjUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMTI1VDIzMTUxOVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTkyZWFlOTcwNGMzNTA5MDk0NDRmMjZhYmZjY2ZlZTUwMzlmNGEwOWVlY2EyY2RlZWVkZjRjYWM2YTVkMTMxNjQmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.YC7sdQQ0tFta1xi4f5Ha1IEea9Ebd93oKZcnugDoflU){: width="100" height="100"}

## safe-softmax

## online safe-softmax