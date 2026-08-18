---
layout: post
title: "[MLOps 실전 구축] 01_환경 세팅_Docker Compose"
series: "MLOps 실전 구축"
date: 2026-04-23
categories: [Data Engineer, MLops]
tags: [mlflow, docker-compose, postgresql, mlops]
---

* 목차
{:toc}

## 문제

시계열 수요예측 모델은 이미 다루어보았으나. 모델을 선형이 아닌 순환형으로, 끊임없이 다루기 위한 시스템이 필요했다. 

이걸 수집 → 전처리 → 학습 → 서빙 네 단계의 파이프라인으로 다시 세우고, MLflow로 엮어서 로컬에서 MLOps를 직접 실행해 보는것을 목표로, 실제 비즈니스 환경을 구성해 보는 것을 목표로 했다.

먼저, 서버 환경을 docker-compose로 세우는 일부터 진행했다..

## 시작하기에 앞서

기존 코드를 리뷰해 보았고, 세 가지가 필요해졌다.

첫째, 데이터 수집의 자동화. ERP에서 매출 이력을 가져오는 조회(GET) API가 없었고, 원본은 결국 사람이 웹 화면에서 수동으로 export한 엑셀파일. 

둘째, 서로 다른 노트북에 흩어져있는 전처리와 학습. 한 노트북이 연도별 시트를 합쳐 월별로 리샘플링하면, 다른 노트북이 그 결과를 읽어 모델을 학습하는 식이다.

셋째, MLflow는 어느 환경에도 깔려 있지 않았다.



- **postgres** — MLflow의 backend store. 실험·파라미터·메트릭·모델 레지스트리 메타데이터가 쌓이는 곳.
- **mlflow-server** — tracking 서버 겸 model registry UI.
- **pipeline** — 전처리·학습·서빙용 이미지.


## 환경 구성

tracking 서버는 학습용 무거운 패키지를 짊어질 이유가 없다. MLflow 본체와 postgres 드라이버면 된다.

```dockerfile
# mlflow-server.Dockerfile
FROM python:3.11-slim
RUN pip install --no-cache-dir mlflow psycopg2-binary
EXPOSE 5000
```

```yaml
# docker-compose.yml (요지)
services:
  postgres:
    image: postgres:16
    environment: { POSTGRES_DB: mlflow, POSTGRES_USER: mlflow, POSTGRES_PASSWORD: mlflow }
    volumes: [ pg_data:/var/lib/postgresql/data ]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mlflow"]
      interval: 5s
      retries: 5

  mlflow-server:
    build: { context: ., dockerfile: docker/mlflow-server.Dockerfile }
    depends_on:
      postgres: { condition: service_healthy }
    ports: [ "5001:5000" ]
    volumes: [ mlartifacts:/mlartifacts ]
    command: >
      mlflow server
      --backend-store-uri postgresql://mlflow:mlflow@postgres:5432/mlflow
      --serve-artifacts --artifacts-destination /mlartifacts
      --host 0.0.0.0 --port 5000

volumes: { pg_data: {}, mlartifacts: {} }
```

특히 신경 쓴 곳이 세 군데 있다.

**healthcheck와 `condition: service_healthy`.** 컨테이너가 "떠 있다"는 것과 "접속이 된다"는 건 다른 얘기다. DB가 아직 초기화 중일 때 mlflow-server가 붙으려다 실패하는 일이 종종 생기는데, `pg_isready`로 진짜 준비됐는지 확인한 다음에야 붙게 하면 이걸 막을 수 있다.

**`--serve-artifacts`.** 이 옵션이 없으면 클라이언트가 서버와 똑같은 파일시스템 경로를 마운트하고 있어야 한다. 켜두면 클라이언트는 서버에 HTTP 요청만 던지고, 실제 저장은 서버가 알아서 한다. 컨테이너마다 볼륨 경로를 억지로 맞출 필요가 없어진다.

**named volume.** 컨테이너를 내렸다 올려도 실험 기록과 아티팩트가 날아가지 않게 하려는 것이다.

## 트러블슈팅: 포트 5000 충돌

`docker compose up`을 실행하자 5000번 포트가 이미 사용 중. 기본적으로 macOS의 AirPlay 수신 기능은 5000번을 깔고 있다.

 `ports: ["5001:5000"]`로 매핑하여 해결

## 여기까지의 결과

postgres와 mlflow-server가 붙었고, `http://localhost:5001`에서 MLflow UI가 정상 응답한다. tracking 서버 환경은 이걸로 준비됐다.


