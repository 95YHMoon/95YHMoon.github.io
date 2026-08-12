---
layout: post
title: "[MLOps 실전 구축] 1. docker-compose로 환경 세우기"
series: "MLOps 실전 구축"
date: 2026-04-23
categories: [Data Engineer, MLops]
tags: [mlflow, docker-compose, postgresql, mlops]
---

* 목차
{:toc}

## 이 시리즈가 풀려는 문제

시계열 수요예측 모델은 이미 하나 있다. 문제는 이 모델의 삶이 처음부터 끝까지 주피터 노트북 안에서만 이루어진다는 것이다. 데이터는 사람이 ERP 화면에서 손으로 엑셀을 내려받고, 전처리와 학습은 각각 다른 노트북을 위에서 아래로 실행해야 한다. 서빙은 아예 없다. 노트북이 뱉은 숫자를 엑셀로 저장해 눈으로 확인하는 게 끝이다.

이걸 수집 → 전처리 → 학습 → 서빙 네 단계의 파이프라인으로 다시 세우고, MLflow로 엮어서 로컬에서 MLOps를 직접 해보려 한다. 남이 짜준 걸 받아 쓰는 게 아니라 "왜 이런 모양이 됐는지"를 이해하면서 가는 게 목적이라 과정을 남긴다.

1편에서 하는 건 딱 하나, tracking 서버 환경을 docker-compose로 세우는 일이다.

## 시작 전에 확인한 것

뭘 만들기 전에 기존 코드부터 뒤져봤다. 세 가지가 분명해졌다.

첫째, 데이터 수집은 자동화되어 있지 않았다. ERP에서 매출 이력을 가져오는 조회(GET) API가 없었고, 원본은 결국 사람이 웹 화면에서 수동으로 export한 엑셀이었다. (인증정보가 여러 코드에 평문으로 박혀 있는 것도 눈에 띄었지만, 이번 범위는 아니라 일단 넘겼다.)

둘째, 전처리와 학습이 서로 다른 노트북에 흩어져 있었다. 한 노트북이 연도별 시트를 합쳐 월별로 리샘플링하면, 다른 노트북이 그 결과를 읽어 모델을 학습하는 식이다.

셋째, MLflow는 어느 환경에도 깔려 있지 않았다.

참고로 기존 학습 모델은 TensorFlow 기반 Transformer였는데, 더는 유지보수되지 않는 구버전 패키지에 묶여 있었다. 이 제약이 나중에 학습 모델 자체를 다시 고민하게 만든다(2편).

## 왜 conda가 아니라 docker-compose인가

처음엔 단순하게 생각했다. 기존 conda 환경을 복제하고 거기에 MLflow만 얹으면 되지 않나. 그런데 이러면 "내 컴퓨터에서만 되는" 재현성 문제가 그대로 남는다.

이왕 MLOps를 제대로 경험할 거라면, tracking 서버와 학습·서빙 코드가 서로 다른 프로세스, 다른 네트워크로 나뉜 구조를 한번 만들어보는 게 낫겠다 싶었다. 그래서 docker-compose로 방향을 틀고 구성을 셋으로 나눴다.

- **postgres** — MLflow의 backend store. 실험·파라미터·메트릭·모델 레지스트리 메타데이터가 쌓이는 곳.
- **mlflow-server** — tracking 서버 겸 model registry UI.
- **pipeline** — 전처리·학습·서빙용 이미지. 2편에서 다룬다.

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

`docker compose up`을 실행하자 5000번 포트가 이미 사용 중이라며 죽었다. 범인은 macOS의 AirPlay 수신 기능이었다. Monterey부터 기본으로 5000번을 깔고 앉는다는 걸 이번에 알았다. 맥에서 MLflow나 개발 서버를 5000번에 띄우다 이걸 겪는 사람이 꽤 있는 모양이다.

AirPlay를 끄는 대신 호스트 쪽 포트만 바꿨다. 컨테이너 안은 그대로 5000번, 호스트에서는 5001번으로 붙게 `ports: ["5001:5000"]`로 매핑했다.

## 여기까지의 결과

postgres와 mlflow-server가 붙었고, `http://localhost:5001`에서 MLflow UI가 정상 응답한다. tracking 서버 환경은 이걸로 준비됐다.

## 다음 편

환경은 섰는데 두 가지가 남았다. 흩어져 있던 `.env`를 정리하는 일, 그리고 더 근본적인 질문 하나 — 학습 대상으로 잡아둔 Transformer를 그대로 밀고 갈 수 있을까. 2편에서 이 둘을 정리한다.
