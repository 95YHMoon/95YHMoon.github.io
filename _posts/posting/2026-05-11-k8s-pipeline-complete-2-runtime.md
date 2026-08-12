---
layout: post
title: "Airflow + kind로 SARIMA 파이프라인 완주 (2) — 실행 환경 문제"
date: 2026-05-11
categories: [Data Engineer,MLops]
tags: [kubernetes, kind, docker, troubleshooting]
---

[1편](/notations/2026-05-11-k8s-pipeline-complete-1-auth)에서 Airflow/K8s 인증 문제를 해결한 뒤에도 두 가지가 더 있었다.

## 문제 3: `python: can't open file '/app/src/collect.py'`

**증상**: Pod는 정상적으로 뜨는데, 컨테이너 안에 `collect.py`가 없다고 함.

**원인**: `mlops-pipeline` 이미지는 세션 아주 초반(`sarima_model.py`만 있던 시점)에 딱 한 번 빌드된 뒤로 재빌드된 적이 없었음. `docker-compose`의 `pipeline` 서비스는 `volumes: - ./src:/app/src`로 항상 최신 코드를 덮어써서(bind mount) 이 사실이 가려져 있었는데, `KubernetesPodOperator`는 이런 마운트 없이 **이미지에 실제로 구워진 파일만** 봐서 낡은 상태가 그대로 드러남.

**해결**: `docker compose build pipeline`으로 재빌드 → `kind load docker-image mlops-pipeline --name mlops-learn`으로 kind 안의 스냅샷도 갱신 (`kind load`는 그 순간 스냅샷이라, 호스트에서 이미지를 다시 만들어도 자동 반영 안 됨 — 매번 다시 로드해야 함).

## 문제 4: 여기저기서 500 / 컨테이너가 이미 죽어있음

**증상**: `train` 태스크에서 500 에러. 확인해보니 `mlflow-server`, `postgres`, `airflow-postgres` 등이 이미 죽어있었음.

**원인**: `docker system df` 확인 결과 이미지 28GB+ (81% reclaimable, 재빌드 반복하며 쌓인 dangling 이미지) — Docker Desktop 가상 디스크가 꽉 차서 `postgres`가 `No space left on device`로 패닉. 여기에 서비스들에 `restart: always`가 없어서 한 번 죽으면 자동 복구도 안 됨.

**해결**: `docker image prune -f` + `docker builder prune -f`로 공간 확보 → `docker compose up -d`로 전체 스택 재기동.

## 최종 확인

```
run_id    run_name      mape
7f9a2644  sarima-step2  12.98   ← K8s Pod(train)가 만든 run, 기존 정상값과 일치
```

## 앞으로 손볼 것 (기록만, 아직 미착수)

- 서비스에 `restart: always` 추가 — 컨테이너가 죽어도 자동 복구되게
- 정기적인 `docker image prune` 습관화, 또는 디스크 알람
- `host.docker.internal`로 MLflow에 접근하는 지금 방식은 임시방편 — 장기적으로는 kind 클러스터 안에 mlflow 접근 경로를 더 견고하게 만들 필요 있음
