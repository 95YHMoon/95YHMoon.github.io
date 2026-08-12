---
layout: post
title: "Airflow + kind로 SARIMA 파이프라인 완주 (1) — 오퍼레이터 선택과 인증 문제"
date: 2026-05-11
categories: [Data Engineer,MLops]
tags: [kubernetes, kind, airflow, kubernetespodoperator, troubleshooting]
---

[전편](/notations/2026-04-29-kind-multinode-airflow-connect)에서 준비해둔 kind 클러스터 + PV/PVC + `KubernetesPodOperator` DAG으로, 드디어 `collect → preprocess → train`이 실제 K8s Pod로 끝까지 돌았다. 여기까지 오는 동안 만난 문제들을 "문제→원인→해결"로 정리한다. 분량이 있어서 두 편으로 나눴다 — 이번 편은 Airflow/K8s 연동 자체의 문제 두 가지, [2편](/notations/2026-05-11-k8s-pipeline-complete-2-runtime)은 실행 환경(이미지/디스크) 문제.

## 문제 1: `KubernetesExecutor`가 필요한 줄 알았는데 아니었음

**증상**: Airflow에 KubernetesExecutor가 있다길래 Airflow 전체 실행기를 바꾸려 했음.

**정리**: `KubernetesExecutor`(Airflow 자신의 내부 실행 방식)와 `KubernetesPodOperator`(DAG의 특정 태스크를 Pod로 띄우는 오퍼레이터)는 서로 다른 축. 목표로 삼은 것(태스크마다 격리된 Pod)엔 `KubernetesPodOperator`만 있으면 충분하고, Airflow 실행기는 LocalExecutor를 그대로 둬도 된다. → 불필요한 아키텍처 변경을 피함.

## 문제 2: `Invalid auth token: Signature verification failed`

**증상**: 태스크가 시작하자마자 죽고, 스케줄러 로그에 인증 토큰 서명 검증 실패.

**원인**: Airflow 3.x는 태스크 프로세스가 `airflow-apiserver`의 Execution API에 JWT로 서명된 토큰을 보내 "시작한다"고 보고하는 구조인데, `AIRFLOW__API_AUTH__JWT_SECRET`을 설정 안 해두면 **각 컨테이너(scheduler, apiserver)가 시작할 때마다 자기만의 랜덤 키를 생성**해서 서로 검증이 안 됨.

**해결**: `.env`에 고정된 `AIRFLOW_JWT_SECRET` 값을 만들고, `x-airflow-common`(모든 airflow 서비스가 공유하는 앵커) 환경변수에 `AIRFLOW__API_AUTH__JWT_SECRET: ${AIRFLOW_JWT_SECRET}` 추가 — 모든 컴포넌트가 같은 키를 쓰게 강제.

다음 편에서 이미지가 낡아서 생긴 문제, 그리고 Docker Desktop 디스크가 꽉 차서 연쇄적으로 컨테이너들이 죽은 문제를 다룬다.
