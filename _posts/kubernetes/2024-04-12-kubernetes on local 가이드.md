---
layout: post
title:  "kubernetes on local 가이드"
date:   2024-04-12 19:41:06 +0900
categories: guide, kubernetes
---

클라우드 서비스를 사용하면 비용이 많이 나오기 때문에 로컬 환경에서 쿠버네티스를 사용해보기 위해서 필요한 절차를 정리한 가이드입니다.

## Step 1 docker desktop 설치

[docker desktop 설치 링크](https://www.docker.com/products/docker-desktop/)

## Step 2 docker desktop kubernetes cluster 설치

[쿠버네티스 클러스터 구축하기](https://gurumee92.tistory.com/300)

## Step 3 Lens 설치

> Lens란 쿠버네티스 클러스터에서 관리하는 리소스를 쉽게 관리하도록 도와주는 GUI 툴
> 

![image](https://github.com/kSideProject/kpring/assets/67232422/87551ac6-0902-4714-9862-15e7c7a2f43a)

1. 도커 데스크탑에서 lens extensions를 설치한다.

![image](https://github.com/kSideProject/kpring/assets/67232422/e41bf314-f51c-4f66-bebe-123e5386ed51)

2. 링크를 타고 lens를 설치한다.

![image](https://github.com/kSideProject/kpring/assets/67232422/c14cabb7-b3b5-474d-8159-4673be868d7d)

3. 설치된 Lens에서 docker-desktop 클러스터를 확인한다.

## Step 4 helm 설치

> helm이란 여러개로 구성된 쿠버네티스 리소스를 관리해주는 패키지 매니저입니다.
> 

맥에서는 brew를 통해서 쉽게 설치가 가능

윈도우 helm 설치하기
[Windows 10에서 Helm 설치](https://sseokseok.tistory.com/7)
[Helm | 헬름 사용하기](https://helm.sh/ko/docs/intro/using_helm/)
[Helm Chart를 이용하여 nginx 설치하기](https://public-cloud.tistory.com/74)

## Fin.
관련한 어려움 점이나 궁금한 점이 있다면 댓글로 알려주세요!
