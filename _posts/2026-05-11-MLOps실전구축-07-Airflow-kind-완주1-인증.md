---
layout: post
title: "[MLOps 실전 구축] 7. Airflow + kind 완주 (1): 오퍼레이터와 인증"
series: "MLOps 실전 구축"
date: 2026-05-11
categories: [Data Engineer, MLops]
tags: [kubernetes, kind, airflow, kubernetespodoperator, troubleshooting]
---

부록 B에서 준비한 kind 클러스터 위에서, 드디어 `collect → preprocess → train`이 실제 K8s Pod로 끝까지 돌았다. 여기까지 오며 만난 문제들을 "문제 → 원인 → 해결"로 정리한다. 분량이 있어서 두 편으로 나눴다. 이번 편은 Airflow/K8s 연동 자체의 문제 둘이고, 다음 편은 실행 환경(이미지·디스크) 문제다.

## 문제 1: KubernetesExecutor가 필요한 줄 알았는데 아니었다

증상. Airflow에 KubernetesExecutor가 있다길래 실행기를 통째로 바꾸려 했다.

정리. `KubernetesExecutor`(Airflow 자신의 내부 실행 방식)와 `KubernetesPodOperator`(DAG의 특정 태스크만 Pod로 띄우는 오퍼레이터)는 서로 다른 축이다. 내가 원한 건 "태스크마다 격리된 Pod"였는데, 여기엔 `KubernetesPodOperator`만 있으면 충분하고 실행기는 LocalExecutor를 그대로 둬도 된다. 덕분에 안 해도 될 아키텍처 변경을 피했다.

## 문제 2: `Signature verification failed`

증상. 태스크가 시작하자마자 죽고, 스케줄러 로그에 인증 토큰 서명 검증 실패가 찍혔다.

원인. Airflow 3.x는 태스크 프로세스가 API 서버에 JWT로 서명된 토큰을 보내 "시작한다"고 보고하는 구조다. 그런데 JWT 시크릿을 설정하지 않으면 각 컨테이너(scheduler, apiserver)가 시작할 때마다 자기만의 랜덤 키를 생성해서, 서로의 서명을 검증하지 못한다.

해결. `.env`에 고정된 시크릿 값을 두고, 모든 airflow 서비스가 공유하는 공통 환경변수 앵커에 그 값을 주입해서 모든 컴포넌트가 같은 키를 쓰게 강제했다.

다음 편에서는 이미지가 낡아서 생긴 문제, 그리고 디스크가 꽉 차서 컨테이너들이 연쇄로 죽은 문제를 다룬다.
