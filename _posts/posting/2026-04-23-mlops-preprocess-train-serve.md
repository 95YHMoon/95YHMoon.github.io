---
layout: post
title: "로컬 MLflow로 MLOps 파이프라인 만들기 (3) — 전처리, 학습, 서빙까지 완주"
date: 2026-04-23
categories: [Data Engineer,MLops]
tags: [mlflow, sarima, pmdarima, pyfunc, model-registry, rest-api]
---

[2편](/notations/2026-04-23-mlops-env-consolidation-sarima-pivot-collect)에서 수집 단계까지 끝냈다. 이번 편에서 전처리 → 학습 → 서빙까지 붙여서 **수집-전처리-학습-서빙 4단계를 실제로 한 바퀴** 돌렸다.

## 전처리 (`preprocess.py`)

`05_매출데이터_김홍은_전처리.ipynb`의 로직 — 연도별 시트에서 타깃 품목만 골라 `항목` 문자열에서 정규식으로 스텝/월을 파싱하고, 스텝별로 월별 리샘플링 — 을 그대로 함수화했다. 딱 하나만 바꿨다: 원본 노트북은 리샘플 범위의 끝을 `end='2025-02-02'`로 하드코딩해놨는데, 이러면 나중에 데이터가 더 들어와도 파이프라인이 조용히 2025년 2월에서 멈춘다. 실제 데이터의 최신 날짜를 기준으로 동적으로 계산하게 고쳤다.

### 삽질: "1년 밀림" 오해

처음에 결과를 기존 `data/{project}_sales_resample_v2.xlsx`(레거시)와 비교했더니 특정 스텝에서 값이 정확히 1년씩 밀려서 나왔다. "년도 시트의 1월호가 사실은 다음 해 1월인데 롤오버 처리가 안 된 버그"라고 판단하고 고치려던 참이었는데 — 알고 보니 **의도된 정규화**였다. 1월호 데이터를 그 시트의 연도 그대로 남겨두는 게 맞는 설계였고, 오히려 레거시 파일 쪽이 예전 버전 로직으로 만들어진 stale 파일이었다. 남의 코드(과거의 나 포함)를 고치기 전에 "왜 이렇게 짰지"부터 확인해야 한다는 걸 다시 배웠다.

## 학습 (`train.py`)

1편~2편에서 정리했듯, 실제로 쓰기로 한 모델은 Transformer가 아니라 **SARIMA**(`mlops/src/sarima_model.py`, `pmdarima.auto_arima` 모드)다. 학습 스크립트는:

1. 전처리 산출물에서 지정한 `--step`의 시계열만 골라서
2. `log1p` 변환 후 마지막 N개월(기본 6개월)을 테스트셋으로 분리
3. `SARIMAModel()`로 auto_arima 학습
4. MAE/RMSE/MAPE를 계산해서 `mlflow.log_metrics()`
5. 학습된 모델을 `mlflow.pyfunc.PythonModel`로 감싸서 **Model Registry에 바로 등록**

```python
mlflow.pyfunc.log_model(
    artifact_path="model",
    python_model=wrapped,
    code_paths=[...],
    input_example=input_example,
    signature=signature,
    registered_model_name="{project}-sales-forecast",
)
```

Transformer였다면 `(timeseries_input, month_input)` 두 개 입력 + 별도 스케일러 두 개를 다 같이 아티팩트로 묶어야 해서 서빙용 래퍼를 학습과 분리된 단계로 나중에 따로 만들어야 했을 텐데, SARIMA는 **학습이 끝난 시점에 이미 예측에 필요한 모든 게 모델 객체 하나에 들어있어서** (`predict(horizon)`만 호출하면 끝) 학습 단계에서 바로 pyfunc로 감싸 등록해버릴 수 있었다. 모델을 바꾼 덕에 서빙 설계도 훨씬 단순해진 셈.

```
[train] step=2 order=ARIMA(0,0,0)(2,0,0)[12] [auto]
[train] metrics={'mae': 1617.45, 'rmse': 1785.45, 'mape': 42.17}
Successfully registered model '{project}-sales-forecast'. version 1
```

MAPE가 42%로 높게 나온 이유가 있다 — 테스트 구간(2024-08~12)이 실제로는 "아직 매출 기록이 없는" 달인데 전처리 단계에서 0으로 채워졌기 때문이다(뒤에 트러블슈팅 참고). 실제 매출이 0으로 폭락한 게 아니라 데이터 미기록 구간이라 지금 메트릭은 액면 그대로 믿을 수 없다 — 이 부분은 일단 그대로 두기로 하고(원본 파일이 Airflow로 최신화되면 자연히 해소될 문제), 인지하고 있다는 것만 기록해둔다.

## 서빙 (`mlflow models serve`)

레지스트리에 등록된 모델을 바로 REST로 띄웠다.

```bash
docker compose run --rm -d --publish 5002:5002 --name mlops-serve-test pipeline \
  mlflow models serve -m "models:/{project}-sales-forecast/1" \
  --host 0.0.0.0 --port 5002 --env-manager local
```

`--env-manager local`이 핵심이다 — 기본값은 모델 학습 당시 환경을 conda로 새로 만들어서 격리하는 건데, `pipeline` 이미지에 이미 학습과 똑같은 의존성이 들어있으니 그럴 필요가 없다. conda 없이 지금 파이썬 환경을 그대로 쓰게 했다.

```bash
curl -X POST http://localhost:5002/invocations \
  -H "Content-Type: application/json" \
  -d '{"dataframe_split": {"columns": ["horizon"], "data": [[6]]}}'
```

```json
{"predictions": [
  {"forecast": 1982.28}, {"forecast": 9.25}, {"forecast": 2132.17},
  {"forecast": 2033.45}, {"forecast": 2131.01}, {"forecast": 1953.07}
]}
```

학습 때 확인한 예측값과 정확히 동일하게 나왔다 — Registry에 저장된 모델을 REST로 불러온 게 검증됨.

## 지금까지 한 바퀴 요약

```
collect.py  → data/raw/sales_<timestamp>.xlsx
preprocess.py → data/processed/{project}_sales_resample_v2.xlsx
train.py    → mlflow experiment run + Model Registry v1
serve       → mlflow models serve → REST /invocations
```

수집-전처리-학습-서빙 네 단계가 로컬 docker-compose(postgres + mlflow-server + pipeline) 위에서 전부 연결됐다.

## 남은 것들

- 원본 데이터의 미기록 꼬리 구간(2024-08~) 때문에 지금 메트릭이 왜곡돼 있다는 점 — Airflow 쪽에서 최신 데이터가 붙으면 자연히 해소될 예정
- 지금은 스텝 2 하나만 검증했다 — 나머지 스텝(0,1,3) 확장은 다음 과제
- Ecount 원본 export를 Airflow가 갱신하는 방식(CSV vs DB 적재)은 아직 미정 — 정해지면 `collect.py --source`만 바꾸면 됨
