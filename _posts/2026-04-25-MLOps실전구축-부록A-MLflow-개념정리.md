---
layout: post
title: "[MLOps 실전 구축] 부록 A. MLflow 개념 정리"
series: "MLOps 실전 구축"
date: 2026-04-25
categories: [Data Science, MLops]
tags: [mlflow, concepts, reference]
---

> 본문 흐름에서 떼어낸 레퍼런스다. 파이프라인을 만들며 익힌 MLflow 개념을, "웹 UI에서 뭘 보는가"와 "코드가 뭘 호출하는가"로 나눠 정리했다. 이 둘은 따로 익혀야 나중에 안 헷갈린다.

* 목차
{:toc}

## 한 줄 요약

MLflow는 학습할 때마다 생기는 "이번엔 뭘 넣었고 결과가 어땠는지"를 자동으로 기록해주고(Tracking), 마음에 드는 결과물을 "이게 진짜 버전이다"라고 등록해서 나중에 꺼내 쓰거나 API로 서빙하게 해주는(Model Registry) 도구다.

---

## Part 1. 웹 UI

상단 네비게이션은 크게 둘이다. Experiments(기본 진입)와 Models(Registry).

**Experiments** — 좌측에 experiment 목록이 있고, 하나를 클릭하면 그 안의 Run들이 테이블로 뜬다.

- 기본 컬럼(이름·생성 시각·소요 시간) 외에, Columns 버튼으로 원하는 파라미터·메트릭 컬럼을 직접 추가할 수 있다. Run 이름이 죄다 비슷할 때 `step`, `mape` 같은 컬럼을 켜면 테이블만 보고도 run이 구분된다. 이름이 전부 "sarima-step2"라 헷갈리던 게 이렇게 풀렸다.
- 여러 run을 체크하고 Compare를 누르면 Parallel Coordinates, Scatter, Box Plot과 비교 표로 메트릭 변화가 한눈에 들어온다.
- 개별 Run을 클릭하면 Overview(Parameters·Metrics·Tags 표)와 Artifacts(`log_model`이 저장한 모델 파일들 — 메타정보, 피클, 의존성 목록 등)를 볼 수 있다.

**Models (Registry)** — Run과는 별개 화면이다. 학습 때 `registered_model_name`으로 등록한 모델이 여기 쌓이고, 이름을 클릭하면 버전 목록이 나온다. 버전 상세에서는 어느 Run에서 나왔는지(Source Run), 입출력 스키마(Schema), 그리고 Alias를 확인·설정한다. Alias는 예를 들어 `champion`을 어떤 버전에 붙여두면 버전 번호 대신 이름으로 참조할 수 있게 해주는 것이다.

---

## Part 2. 소스 코드에서 쓰는 API

### 실험 / Run 생명주기

```python
mlflow.set_experiment("sales-forecast")   # 없으면 생성
with mlflow.start_run(run_name=f"sarima-step{step}"):
    ...   # 이 블록 안의 log_* 호출이 전부 이 run에 귀속
    # 블록이 끝나면 정상이든 예외든 run이 자동으로 마무리됨
```

`with` 블록을 쓰면 `end_run()`을 따로 부르지 않아도 컨텍스트 매니저가 알아서 닫아준다.

### 파라미터 · 메트릭 기록

파라미터는 실행 전에 정해지는 설정값, 메트릭은 실행 결과로 나오는 숫자다.

```python
mlflow.log_param("sarima_order", model.order_str)   # 여러 개는 log_params(dict)
mlflow.log_metrics({"mae": mae, "rmse": rmse, "mape": mape})
```

같은 key로 여러 번 기록하면 step별 시계열(예: epoch별 loss)로도 쌓이지만, 여기서는 run당 한 번씩 단일 값으로만 썼다.

### 모델 저장 + Registry 등록 — `mlflow.pyfunc.log_model`

가장 복잡한데 핵심이기도 하다. SARIMA는 순수 통계 모델이라 기본 제공 flavor(sklearn, tensorflow 등)에 안 맞는다. MLflow는 이런 경우를 위해 `pyfunc.PythonModel` 인터페이스를 열어둔다. `predict(self, context, model_input)` 하나만 구현하면 그 안에 뭐가 들었든 동일하게 저장·서빙된다.

```python
wrapped = SarimaForecastModel(model)        # pyfunc.PythonModel 구현 래퍼
signature = infer_signature(input_example, output_example)

mlflow.pyfunc.log_model(
    artifact_path="model",
    python_model=wrapped,
    code_paths=[...],                        # 래퍼가 의존하는 코드도 함께 스냅샷
    input_example=input_example,
    signature=signature,
    registered_model_name="sales-forecast",  # 이 인자를 주면 Registry에도 자동 등록
)
```

`registered_model_name`을 빼면 이 run의 아티팩트로만 저장되고 Registry에는 안 올라간다.

### 불러오기 · 조회

```python
model = mlflow.pyfunc.load_model("models:/sales-forecast/5")  # Registry에서 로드
model.predict(pd.DataFrame({"horizon": [6]}))

client = mlflow.tracking.MlflowClient()      # 특정 run/experiment/model 조회·관리
runs = client.search_runs([exp_id])
```

### 서빙 (CLI)

```bash
mlflow models serve -m "models:/sales-forecast/5" \
  --host 0.0.0.0 --port 5002 --env-manager local
```

떠 있는 서버는 `POST /invocations`로 호출하고, 요청 형식은 `log_model`에 넘긴 `signature`로 검증된다.

---

## 아직 안 써본 것

버전 번호 대신 참조하게 해주는 Alias, "지금까지 run 중 mape가 제일 낮은 것 찾기" 같은 `search_runs` 자동화는 아직 안 써봤다. 스텝을 여러 개 돌리기 시작하면 이런 조회 자동화가 아쉬워진다(→ 5장).
