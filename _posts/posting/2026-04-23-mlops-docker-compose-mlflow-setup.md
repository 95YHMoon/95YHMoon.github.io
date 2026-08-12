---
layout: post
title: "로컬 MLflow로 MLOps 파이프라인 만들기 (1) — docker-compose 환경 세팅"
date: 2026-04-23
categories: [Data Engineer,MLops]
tags: [mlflow, docker-compose, postgresql, tensorflow, transformer, sales-forecast]
---

## 배경

시계열 수요예측 모델은 이미 하나 있었다. Transformer 기반이고, Keras/TensorFlow로 짜여 있다. 문제는 이 모델의 삶이 처음부터 끝까지 주피터 노트북 안에서만 이루어진다는 점이었다. 데이터는 Ecount ERP 화면에서 사람이 손으로 엑셀을 내려받고, 전처리는 노트북 하나(`05_매출데이터_김홍은_전처리.ipynb`)를 위에서 아래로 실행해야 하고, 학습도 또 다른 노트북(`06_transfomer_매출데이터v2.ipynb`)을 돌려야 한다. 서빙은 아예 없다 — 노트북이 뱉은 숫자를 엑셀로 저장해서 눈으로 확인하는 게 전부다.

이번에 하려는 건 이 네 단계, 수집 → 전처리 → 학습 → 서빙을 MLflow로 엮어서 로컬에서 MLOps를 직접 손으로 경험해보는 것이다. 남이 짜준 파이프라인을 받아 쓰는 게 아니라 왜 이런 모양이 됐는지 이해하면서 가는 게 목적이라, 그 과정을 기록으로 남겨둔다.

## 기존 코드부터 뒤져봤다

뭘 만들기 전에 이미 있는 코드를 먼저 살펴봤다. 몇 가지가 눈에 띄었다.

Ecount API 중에 매출 이력을 `GET`으로 가져오는 코드는 어디에도 없었다. `SalesUpload.py`, `testApi.ipynb`, `mkorder.ipynb` — 이름은 다 다른데 하나같이 주문을 Ecount로 업로드(POST)하는, 방향이 반대인 코드들이었다. 실제 매출 원본은 결국 사람이 Ecount 웹 화면에서 엑셀로 export한 파일(`김홍은_나은교육_출고수량.xlsx` 등)이었다. 이 과정에서 `API_CERT_KEY` 같은 인증정보가 세 개 파일에 평문으로 박혀 있는 것도 봤는데, 지금 하려는 작업 범위는 아니라서 일단 눈에 담아만 두고 넘어갔다. 나중에 `.env`로 옮겨야 할 일이다.

전처리는 `05_매출데이터_김홍은_전처리.ipynb`가 담당한다. 연도별 시트를 하나로 합치고, 타깃 품목만 걸러내고, 정규식으로 스텝/월을 뽑아 월별로 리샘플링해서 `{project}_sales_resample_v2.xlsx`를 만들어낸다. 학습 노트북이 읽는 입력이 바로 이 파일이다.

학습은 `06_transfomer_매출데이터v2.ipynb`가 맡는다. 멀티헤드 어텐션과 month embedding을 쓰는 Transformer로, `yolo`라는 conda 환경(TensorFlow 2.13, tensorflow-addons 0.21) 위에서 돌아가게 짜여 있었다. tensorflow-addons는 최신 TF에서 더는 지원하지 않는 패키지라, 이 조합을 그대로 재현해야 한다는 제약이 하나 걸렸다.

그리고 mlflow는 — 이 컴퓨터의 어떤 conda 환경을 뒤져도 없었다.

## conda 대신 docker-compose를 고른 이유

처음엔 단순하게 생각했다. `conda create -n sales-mlops --clone yolo`로 기존 환경을 복제하고 거기에 mlflow만 얹으면 되지 않을까. 그런데 다시 생각해보니 이 방식엔 결국 "내 맥에서만 되는" 재현성 문제가 그대로 남는다. 게다가 이왕 MLOps를 제대로 경험해볼 거라면, tracking 서버(백엔드 저장소, 아티팩트 저장소)와 학습·서빙 코드가 서로 다른 프로세스, 다른 네트워크로 나뉘어 있는 구조를 한번 만들어보는 게 더 남는 게 있을 것 같았다.

그래서 docker-compose로 방향을 틀었다. 구성은 이렇게 잡았다.

- `postgres` — mlflow의 backend store. 실험, 파라미터, 메트릭, 모델 레지스트리 메타데이터가 여기 쌓인다.
- `mlflow-server` — mlflow tracking 서버이자 model registry UI.
- `pipeline` (다음 편에서 다룬다) — TensorFlow 2.13, tensorflow-addons, mlflow client가 들어간 학습·전처리·서빙용 이미지.

## 디렉토리 구조

```
Ecount/
  mlops/
    docker-compose.yml
    docker/
      mlflow-server.Dockerfile   # mlflow + psycopg2 만 있는 가벼운 이미지
      pipeline.Dockerfile        # (예정) TF2.13 + tensorflow-addons + mlflow client
    src/                         # (예정) collect.py / preprocess.py / train.py / serving_model.py
    configs/                     # (예정) 스텝별 하이퍼파라미터 설정
```

## mlflow-server.Dockerfile

tracking 서버는 TensorFlow 같은 무거운 학습용 패키지를 짊어질 이유가 없다. mlflow 본체와 postgres 드라이버(`psycopg2-binary`), 이 둘이면 충분하다.

```dockerfile
FROM python:3.11-slim

RUN pip install --no-cache-dir mlflow==2.14.1 psycopg2-binary==2.9.9

EXPOSE 5000
```

## docker-compose.yml

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: mlflow
      POSTGRES_USER: mlflow
      POSTGRES_PASSWORD: mlflow
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mlflow"]
      interval: 5s
      timeout: 5s
      retries: 5

  mlflow-server:
    build:
      context: .
      dockerfile: docker/mlflow-server.Dockerfile
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "5001:5000"
    volumes:
      - mlartifacts:/mlartifacts
    command: >
      mlflow server
      --backend-store-uri postgresql://mlflow:mlflow@postgres:5432/mlflow
      --serve-artifacts
      --artifacts-destination /mlartifacts
      --host 0.0.0.0
      --port 5000

volumes:
  pg_data:
  mlartifacts:
```

`healthcheck`와 `depends_on: condition: service_healthy`를 같이 쓴 데는 이유가 있다. postgres 컨테이너가 "떠 있다"는 것과 "접속 가능하다"는 건 다른 얘기다. `pg_isready`로 진짜 준비됐는지 확인한 다음에야 mlflow-server가 붙게 만들지 않으면, DB가 아직 초기화 중일 때 mlflow-server가 접속을 시도했다 실패하는 상황이 종종 생긴다.

`--serve-artifacts --artifacts-destination /mlartifacts`는 이번 구성에서 제일 신경 쓴 부분이다. 이 옵션 없이 로컬 파일 경로를 그냥 artifact root로 쓰면, mlflow 클라이언트(나중에 만들 `pipeline` 컨테이너)가 서버와 똑같은 파일시스템 경로를 마운트하고 있어야 한다. `--serve-artifacts`를 켜두면 클라이언트는 tracking 서버에 HTTP로 요청만 보내면 되고, 실제 파일을 어디에 어떻게 저장할지는 서버가 알아서 처리한다. 컨테이너마다 볼륨 경로를 억지로 맞출 필요가 사라지는 셈이다.

named volume(`pg_data`, `mlartifacts`)은 그냥 컨테이너를 내렸다 올려도 실험 기록과 아티팩트가 사라지지 않게 하려고 넣었다.

## 트러블슈팅: 포트 5000 충돌

`docker compose up -d mlflow-server`를 실행했더니 다짜고짜 이런 에러가 떴다.

```
Error response from daemon: ports are not available: exposing port TCP 0.0.0.0:5000 -> 127.0.0.1:0:
listen tcp 0.0.0.0:5000: bind: address already in use
```

`lsof -nP -iTCP:5000 -sTCP:LISTEN`으로 범인을 잡아보니 `ControlCenter`였다. macOS Monterey부터 AirPlay Receiver가 기본적으로 5000번 포트를 깔고 앉아있다는 걸 이번에 처음 알았다. 맥에서 mlflow나 Flask 개발 서버를 5000번 포트로 띄우다 이 문제를 겪는 사람이 꽤 있는 모양이다.

시스템 설정에서 AirPlay 수신을 끄는 대신, 그냥 호스트 쪽 포트만 바꿨다. 컨테이너 내부는 여전히 5000번을 쓰고, 호스트에서는 5001번으로 접속하도록 `ports: ["5001:5000"]`로 고쳤다.

## 지금 상태

```
$ docker compose up -d mlflow-server
$ docker compose ps -a
NAME                    STATUS                    PORTS
mlops-mlflow-server-1   Up                        0.0.0.0:5001->5000/tcp
mlops-postgres-1        Up (healthy)              5432/tcp

$ curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:5001/
HTTP 200
```

postgres와 mlflow-server 두 서비스가 붙었고, `http://localhost:5001`에서 mlflow UI가 응답한다.

## 다음 할 일

postgres에 실험 기록이 실제로 남는지, 아티팩트 프록시 업로드가 제대로 되는지는 테스트 run 하나로 금방 확인할 수 있을 것이다. 그다음이 문제인데, `.env`를 두 군데로 나눠 관리하던 게 슬슬 헷갈리기 시작했고, 무엇보다 학습 대상으로 잡아둔 Transformer를 그대로 밀고 갈 수 있을지부터 다시 따져봐야 한다 — 이 부분은 다음 편에서 정리한다.
