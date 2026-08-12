---
layout: post
title: "MLflow 개념 정리 — 시계열 수요예측 파이프라인으로 익힌 것들"
date: 2026-04-23
categories: [Data Science,MLops]
tags: [mlflow, concepts]
---

MLflow를 처음 만져보면서, `mlops/` 파이프라인(시계열 수요예측 SARIMA 모델)을 예시 삼아 개념을 하나씩 확인해나갔다. 그 과정을 정리해둔다. **웹 UI에서 확인한 부분**과 **소스 코드에서 실제로 호출한 부분**을 나눠서 적었다 — "화면에서 뭘 보고 있는지"와 "코드가 뭘 하고 있는지"는 따로 익혀야 나중에 헷갈리지 않는다.

### 목차

- MLflow가 하는 일 한 줄 요약
- Part 1. 웹 UI
- Part 2. 소스 코드에서 쓰는 MLflow API
  - 2-1. 실험/Run 생명주기
  - 2-2. 파라미터 기록 — `log_param`
  - 2-3. 메트릭 기록 — `log_metric` / `log_metrics`
  - 2-4. 모델 저장 + Registry 등록 — `mlflow.pyfunc.log_model`
  - 2-5. 모델 불러오기 / 추론 — `load_model`
  - 2-6. Registry/Run 조회·관리 — `MlflowClient`
  - 2-7. 서빙 — `mlflow models serve` (CLI)
- 아직 안 써본 것들

## MLflow가 하는 일 한 줄 요약

모델을 학습시킬 때마다 생기는 "이번엔 뭘 넣었고 결과가 어땠는지"를 자동으로 기록해주고(Tracking), 마음에 드는 결과물을 "이게 진짜 버전이다"라고 등록해서 나중에 꺼내 쓰거나 API로 서빙할 수 있게 해주는(Model Registry) 도구다.

---

# Part 1. 웹 UI (`http://localhost:5001`)

상단 네비게이션부터 계층적으로 내려가면서, 각 화면에서 뭘 확인할 수 있는지 정리했다.

```
localhost:5001
├── 상단 네비게이션 바
│   ├── Experiments  ← 기본 진입 화면
│   └── Models       ← Model Registry 화면
│
├── [Experiments]
│   │
│   ├── 좌측 사이드바 — Experiment 목록
│   │   └── {project}-sales-forecast   (지금까지 만든 유일한 experiment)
│   │       train.py의 mlflow.set_experiment(...)로 생성됨
│   │
│   └── Experiment를 클릭하면 메인 화면 — Runs 목록
│       │
│       ├── 상단 뷰 전환 탭: Table / Chart
│       │   (Chart 뷰는 run들의 메트릭 변화를 그래프로 보여줌)
│       │
│       ├── Runs 테이블
│       │   ├── 기본 컬럼: 체크박스 / Run Name / Created(생성 시각) / Duration
│       │   ├── "Columns" 버튼 → 표시할 Parameter/Metric 컬럼을 직접 추가 가능
│       │   │   (step, sarima_order, mape, valid_mape 컬럼을 켜니
│       │   │    테이블만 보고도 run들이 구분됐다 — Run Name이 전부
│       │   │    "sarima-step2"로 똑같아서 헷갈렸던 문제가 이렇게 풀렸다)
│       │   └── 체크박스로 여러 run 선택 → 상단 "Compare" 버튼
│       │       └── Compare 화면
│       │           ├── Parallel Coordinates Plot (파라미터-메트릭 관계 시각화)
│       │           ├── Scatter Plot / Box Plot
│       │           └── 파라미터·메트릭 비교 표 (run을 열로, 값을 행으로)
│       │
│       └── 개별 Run Name 클릭 → Run 상세 페이지
│           ├── Overview 탭
│           │   ├── Parameters 표 — mlflow.log_param()으로 기록한 것들
│           │   │   (step, test_size, valid_size, n_obs, sarima_order,
│           │   │    selected_d, selected_D, model_trained_through, ...)
│           │   ├── Metrics 표 — mlflow.log_metric(s)로 기록한 것들
│           │   │   (mae, rmse, mape, valid_mape)
│           │   └── Tags (mlflow.runName 등 시스템 태그 + 따로 안 넣은
│           │       사용자 태그도 여기 표시됨)
│           └── Artifacts 탭
│               └── model/  ← mlflow.pyfunc.log_model()이 저장한 파일들
│                   ├── MLmodel        (이 모델의 flavor/시그니처 메타정보)
│                   ├── python_model.pkl (피클된 SarimaForecastModel 인스턴스)
│                   ├── code/          (code_paths로 넘긴 sarima_model.py, serving_model.py)
│                   └── requirements.txt / conda.yaml (재현용 의존성 목록)
│
└── [Models]  (Model Registry — Run과는 별개 화면)
    │
    ├── 등록된 모델 목록
    │   └── {project}-sales-forecast
    │       (train.py의 log_model(..., registered_model_name=...) 호출마다
    │        여기 새 버전이 하나씩 쌓인다)
    │
    └── 모델 이름 클릭 → 버전 목록 테이블
        ├── 컬럼: Version / Registered at / Source Run(클릭하면 그 run으로 이동)
        │         / Stage 또는 Alias
        │   (버전 1~5까지 있는데, v1·v2는 같은 run에서 나온 것이었다 — 등록
        │    중 아티팩트 업로드가 재시도되며 중복 등록된 것으로 보인다)
        └── 버전 클릭 → 버전 상세 페이지
            ├── Source Run 링크 — 이 버전이 어느 run의 결과물인지
            ├── Schema — infer_signature()로 넣어둔 입력(horizon: long)/
            │            출력(forecast: double) 스키마
            ├── Description / Tags
            └── Alias 설정 UI (예: "champion"을 이 버전에 붙이면
                models:/{project}-sales-forecast@champion 로 참조 가능해짐 —
                버전 번호를 안 외워도 되는 방식. 아직 붙여보진 않았다)
```

이 화면에서 실제로 확인해본 것: `{project}-sales-forecast` experiment의 Runs 테이블에서 Columns로 `mape` 컬럼을 켜니 4개 run의 값(42.17 → 40.82 → 97.75 → 12.97)이 실제로 다르다는 게 한눈에 보였다. 그 4개를 체크해서 Compare를 열어보면 메트릭이 어떻게 바뀌어왔는지 그래프로 나오고, Models 탭에서 버전 5(가장 최근, mape 12.97) 상세 페이지를 열면 Schema와 Source Run이 같이 뜬다.

---

# Part 2. 소스 코드에서 쓰는 MLflow API

실제로 호출한 것만 정리했다. `mlops/src/train.py`와 `serving_model.py`에 들어있는 것, 그리고 검증 삼아 터미널에서 직접 호출해본 것을 구분해서 적는다.

### 2-1. 실험/Run 생명주기

```python
mlflow.set_experiment("{project}-sales-forecast")   # 없으면 생성, 있으면 그걸 사용
with mlflow.start_run(run_name=f"sarima-step{args.step}"):
    ...   # 이 블록 안에서 일어나는 log_* 호출이 전부 이 run에 귀속됨
    # 블록이 끝나면(정상 종료든 예외든) run이 자동으로 FINISHED/FAILED 처리됨
```
(`train.py:133-134`) `with` 블록을 쓰는 이유는 간단하다. run을 명시적으로 `end_run()` 하지 않아도 컨텍스트 매니저가 알아서 마무리해준다.

### 2-2. 파라미터 기록 — `log_param`

실행 *전에* 정해지는 설정값을 하나씩 기록한다. 문자열이든 숫자든 상관없고, Run 상세 페이지의 Parameters 표에 그대로 뜬다.

```python
mlflow.log_param("step", args.step)
mlflow.log_param("sarima_order", model.order_str)
mlflow.log_param("selected_d", best_d)
mlflow.log_param(
    "model_trained_through",
    str(series.index[len(selection_train_log) - 1].date()),
)
```
(`train.py:135-149`) 여러 개를 한 번에 넣고 싶으면 `mlflow.log_params({...})`(dict)도 있는데, 이번엔 안 썼다 — 값이 계산되는 시점이 서로 달라서 하나씩 넣는 게 더 자연스러웠다.

### 2-3. 메트릭 기록 — `log_metric` / `log_metrics`

실행 *결과로* 나오는 숫자다. 같은 key로 여러 번 기록하면 step(스텝 번호)별로 시계열처럼 쌓이기도 하는데(예: epoch별 loss), 여기서는 run당 한 번씩만 기록해서 단일 값으로만 썼다.

```python
mlflow.log_metric("valid_mape", valid_mape)          # 값 하나
mlflow.log_metrics(metrics)                          # {"mae":..,"rmse":..,"mape":..} dict 한 번에
```
(`train.py:148, 164`)

### 2-4. 모델 저장 + Registry 등록 — `mlflow.pyfunc.log_model`

가장 복잡하지만 핵심인 부분이었다. SARIMA 모델은 순수 통계 모델 객체라 mlflow가 기본 제공하는 flavor(sklearn, tensorflow 등)에 안 맞는다. mlflow는 이런 경우를 위해 `mlflow.pyfunc.PythonModel`이라는 범용 인터페이스를 열어두는데, `predict(self, context, model_input)` 메서드 하나만 구현하면 그 안에 뭐가 들어있든 mlflow가 동일하게 저장·서빙해준다. 이 인터페이스에 맞춰 만든 래퍼 클래스를 학습 직후 `log_model`로 감싸서 저장+등록한다.

```python
wrapped = SarimaForecastModel(model)  # mlflow.pyfunc.PythonModel을 구현한 래퍼
signature = infer_signature(input_example, output_example)  # 입출력 스키마 추론

mlflow.pyfunc.log_model(
    artifact_path="model",              # 이 run 안에서 아티팩트가 저장될 경로 이름
    python_model=wrapped,               # 위에서 만든 래퍼 인스턴스 (cloudpickle로 저장됨)
    code_paths=[".../sarima_model.py",
                ".../serving_model.py"], # 래퍼가 의존하는 코드도 같이 스냅샷
    input_example=input_example,        # 스키마 추론 + UI에 예시로 표시
    signature=signature,
    registered_model_name="{project}-sales-forecast",  # 이 인자를 주면 Registry에도 자동 등록
)
```
(`train.py:169-179`, `serving_model.py:24-31`) `registered_model_name`을 안 넘기면 이 run의 아티팩트로만 저장되고 Registry에는 안 올라간다. 학습 즉시 등록까지 원해서 매번 같이 넘겼다.

### 2-5. 모델 불러오기 / 추론 — `load_model`

Registry나 특정 run의 아티팩트에서 모델을 다시 불러와 바로 예측에 쓸 수 있다. 파이프라인 코드에는 안 넣었고, 등록이 잘 됐는지 검증할 때 터미널에서 직접 호출해봤다.

```python
model = mlflow.pyfunc.load_model("models:/{project}-sales-forecast/5")
model.predict(pd.DataFrame({"horizon": [6]}))
```

### 2-6. Registry/Run 조회·관리 — `MlflowClient`

`mlflow.log_*`/`mlflow.pyfunc.*`가 "지금 이 run에" 뭔가 하는 함수라면, `MlflowClient`는 특정 run/experiment/model을 콕 집어 조회·관리하는 API다. 디버깅하면서 실제로 이렇게 썼다.

```python
client = mlflow.tracking.MlflowClient()

exp = client.get_experiment_by_name("{project}-sales-forecast")
runs = client.search_runs([exp.experiment_id])          # 이 실험의 모든 run 조회

versions = client.search_model_versions(
    "name='{project}-sales-forecast'"
)                                                        # 등록된 모든 버전 조회

client.delete_experiment(exp.experiment_id)              # (smoke-test 실험 정리할 때 사용)
```

### 2-7. 서빙 — `mlflow models serve` (CLI)

지금까지는 다 파이썬 코드였는데, 서빙은 CLI 명령으로 REST 서버를 띄운다.

```bash
mlflow models serve \
  -m "models:/{project}-sales-forecast/5" \
  --host 0.0.0.0 --port 5002 \
  --env-manager local     # 이미 컨테이너에 맞는 의존성이 있으니 별도 가상환경 생성 생략
```

떠 있는 서버는 `POST /invocations`로 호출한다. 요청 형식은 `log_model`에 넘겼던 `signature`를 기준으로 검증된다.

```bash
curl -X POST http://localhost:5002/invocations \
  -H "Content-Type: application/json" \
  -d '{"dataframe_split": {"columns": ["horizon"], "data": [[6]]}}'
```

---

## 아직 안 써본 것들

alias(`client.set_registered_model_alias(...)`)는 화면에서 보긴 했지만 실제로 붙여본 적은 없다. 버전 번호 대신 alias로 참조하게 해두면 새 버전을 등록해도 서빙 쪽 코드를 안 건드려도 된다는 건 알겠는데, 지금은 버전이 몇 개 안 돼서 아직 필요를 못 느꼈다. `mlflow.search_runs()`로 "지금까지 run 중 mape가 제일 낮은 걸 코드로 찾기" 같은 것도 해보면 좋을 텐데 아직이다. 스텝을 여러 개 돌리기 시작하면 — 실제로 다음에 스텝 0/1/3을 마저 돌리면서 한 Experiment 안에 여러 모델을 모아두는 걸 시도했는데, 그때 이런 조회 자동화가 아쉬웠다.
