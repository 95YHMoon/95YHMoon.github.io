---
layout: post
title: "[MLOps 실전 구축] 6. 방향 전환 — Docker 한 대로는 부족하다"
series: "MLOps 실전 구축"
date: 2026-04-28
categories: [Data Science, MLops]
tags: [airflow, kubernetes, kind, architecture]
---

* 목차
{:toc}

여기까지 네 단계 파이프라인이 로컬 docker-compose 위에서 돌았다. 이번 편은 그다음, 왜 Airflow와 K8s로 넘어가기로 했는지를 확정된 것만 추려서 남긴다.

## 이 작업의 이중 목적

이 파이프라인은 시계열 수요예측이라는 실제 필요 말고도, 취업 준비용 교두보이기도 하다. 타겟 직무는 MLOps/ML 엔지니어, 데이터 사이언티스트, 데이터 엔지니어 세 갈래인데, 이 중 데이터 엔지니어링과 MLOps는 하나의 축으로 묶어서 학습하기로 했다. 두 분야가 실무에서 수렴하는 추세라고 봤기 때문이다. 이 방향이 이후 기술 선택의 기준이 된다.

## 왜 Docker로 만족하지 않았나

몇 가지가 정리됐다.

먼저 Databricks 얘기. 흔히 "기능이 한정적"이라고들 하는데, 정확히는 Spark/Delta Lake라는 하나의 패러다임에 수직 통합된 플랫폼이라 그 틀 밖의 임의 워크로드에 대한 유연성이 낮은 것이다.

그리고 규모가 커지면 K8s는 사실상 필수불가결하다. 이기종 워크로드를 하나의 컴퓨트 풀에 효율적으로 bin-packing한다는 점 때문이다. Airflow에서 이 사고가 그대로 적용되는 지점이 KubernetesExecutor다. 태스크마다 K8s Pod를 그때그때 만들었다 없애는 방식이라, 격리가 없는 LocalExecutor나 고정 워커 풀로 유휴 자원을 낭비하는 CeleryExecutor보다 자원 효율이 높다.

결정적이었던 건 이 정리다. 지금 만들어둔 DockerOperator 구성은 사실 KubernetesExecutor의 축소판이다. 단일 도커 호스트에서만 돌고, 멀티노드 스케줄링이나 자원 limit 강제가 안 된다. 그러니 DockerOperator DAG을 끝까지 검증하는 것보다, 애초에 그 상위 개념인 K8s로 넘어가는 게 먼저라고 판단했다.

## 학습 진행 방식 (프로토콜)

방향 전환과 함께, 앞으로 새 기술 주제마다 지킬 순서도 정했다. 개념(왜 필요한지, 아는 것과 어떻게 연결되는지) → 최소 예시 → 성공 기준 합의 → 구현 순이고, 명령·코드는 설명과 함께 텍스트로 받되 실제 실행과 트러블슈팅은 직접 한다. 학습 목적상 중요한 명령은 대신 실행하지 않는다.

## 지금 상태와 다음 단계

Airflow를 docker-compose로 추가하고(LocalExecutor 기준), DockerOperator로 기존 `pipeline` 이미지를 형제 컨테이너로 실행하는 DAG까지는 붙였다. UI 기동과 DAG import까지는 확인했지만 끝까지 트리거해 도는지는 검증하지 않았다. 위에서 정리한 대로, 이 검증보다 K8s 전환이 먼저라고 봤기 때문이다.

로컬 K8s 도구로는 kind를 골랐다. minikube보다 멀티노드 시뮬레이션이 실제 클러스터에 가깝고, 가볍고, CI/DE 맥락에서도 흔히 쓰인다. 싱글노드 클러스터부터 시작해서, `kubectl get nodes`가 `Ready`를 보여주는 걸 첫 성공 기준으로 잡았다. (K8s 기본 개념 정리는 부록 B로 뺐다.)
