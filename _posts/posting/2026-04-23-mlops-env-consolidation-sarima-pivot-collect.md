---
layout: post
title: "로컬 MLflow로 MLOps 파이프라인 만들기 (2) — .env 통합, SARIMA로 피벗, 수집 단계"
date: 2026-04-23
categories: [Data Engineer,MLops]
tags: [mlflow, docker-compose, dotenv, sarima, pmdarima, statsmodels, ecount]
---

[1편](/notations/2026-04-23-mlops-docker-compose-mlflow-setup)에서 postgres + mlflow-server docker-compose 환경을 띄우는 데까지 갔다. 이번 편은 그 뒤로 있었던 세 가지 — **.env 정리**, **학습 모델을 Transformer에서 SARIMA로 바꾼 결정**, 그리고 **수집 단계 스크립트** — 를 기록한다.

## 1. .env를 하나로 통합

원래는 관심사별로 `.env`를 분리했었다.

- `mlops/.env` — mlflow docker-compose 스택의 postgres 계정
- `source/.env` — Ecount ERP API 인증정보 (`ZONE`, `COM_CODE`, `USER_ID`, `API_CERT_KEY`)

이렇게 나눈 이유는 있었다. Ecount API 키는 외부 시스템에 대한 장기 유효 비밀값이고, postgres 계정은 로컬 컨테이너용 일회성 개발 비밀값이라 성격이 다르다는 논리였다. 근데 실제로 관리하는 입장에서는 "어디에 뭐가 있더라"를 매번 떠올려야 해서 오히려 헷갈린다는 지적을 받았고, 맞는 말이라 하나로 합쳤다.

- 실제 값은 프로젝트 루트의 `Ecount/.env` 하나에만 존재 (`ECOUNT_*`, `POSTGRES_*` 변수를 접두어로 구분)
- `mlops/.env`는 `../.env`를 가리키는 **심볼릭 링크**로 바꿨다 — docker compose가 기본적으로 자신이 실행되는 디렉토리에서 `.env`를 찾기 때문에, 파일을 옮기지 않고 링크만 걸어서 "실제로는 파일 하나"를 유지하면서 docker compose의 기본 동작도 그대로 살렸다.
- python 쪽(`SalesUpload.py`, `testApi.ipynb`, `mkorder.ipynb`)은 원래 `load_dotenv()`를 호출하는데, 이 함수가 내부적으로 `find_dotenv()`로 현재 위치에서 상위 디렉토리까지 훑어서 `.env`를 찾기 때문에 `source/` 안에 `.env`가 없어도 루트의 `.env`를 알아서 찾아준다. 코드 수정 없이 파일 배치만 바꿔서 해결됐다.

## 2. Transformer → SARIMA로 학습 대상 피벗

1편에서는 `06_transfomer_매출데이터v2.ipynb`의 Transformer(멀티헤드 어텐션 + month embedding)를 학습 단계의 대상으로 잡았었다. 그런데 실제로는 **Transformer 결과가 잘 안 나와서 이미 SARIMA를 베이스 모델로 선택**해뒀었다는 걸 뒤늦게 확인했다.

기존 저장소를 다시 뒤져보니 SARIMA 관련 작업이 꽤 있었다.

- `data/SARIMA_DD_20250507.R` — 지금 이 파이프라인이 다루는 것과 **동일한 데이터**(`{project}_sales_resample_v2.xlsx`)로 스텝별 SARIMA 실험. 로그변환 → KPSS/ADF 정상성 검정 → 차분 → ACF/PACF → 여러 `(p,d,q)(P,D,Q)[12]` 후보를 AIC로 비교 → train/test 평가까지 되어있는 완성도 있는 R 스크립트.
- `data/analysis/2025_모코_*세_예측(SARIMAX).R` 등 — MOCO 브랜드 라인용 SARIMA/SARIMAX 실험들.
- `source/predict/v2/ts_benchmark.py` — Python 쪽에도 이미 `SARIMAModel` 클래스가 있었다. `pmdarima.auto_arima`로 자동 차수 탐색(R의 `auto.arima`와 가장 유사)하는 모드와, 수동으로 `(p,d,q)(P,D,Q,m)`를 지정하는 모드 둘 다 지원하는, 꽤 잘 만들어진 래퍼였다.

그래서 계획을 수정: 학습 단계는 Transformer를 포팅하는 대신, **이미 있는 `SARIMAModel`을 재사용**하기로 했다. `ts_benchmark.py` 전체를 그대로 import하면 `torch`, `prophet` 같은 무거운 의존성(LSTM/Transformer/Prophet 비교용)까지 같이 끌려오기 때문에, `SARIMAModel` 클래스만 뽑아서 `mlops/src/sarima_model.py`로 분리했다. auto_arima 모드로 빠르게 스모크 테스트를 해봤다:

```python
m = SARIMAModel()  # order=None → auto_arima 모드
m.fit(series)
print(m.order_str)      # ARIMA(0,1,1)(2,0,1)[12] [auto]
print(m.predict(6))     # 6개월 예측
```

이 피벗 덕분에 `pipeline` Docker 이미지도 훨씬 가벼워졌다 — TensorFlow/tensorflow-addons 없이 `statsmodels` + `pmdarima` + `pandas`/`scikit-learn`/`openpyxl`/`mlflow`만 있으면 된다.

```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN pip install --no-cache-dir \
    mlflow==2.14.1 pandas==2.2.2 numpy==1.26.4 \
    statsmodels==0.14.6 pmdarima==2.0.4 \
    scikit-learn==1.5.0 openpyxl==3.1.2
COPY src/ /app/src/
ENV GIT_PYTHON_REFRESH=quiet
```

(`statsmodels==0.14.2`로 처음 고정했다가 최신 `scipy`와 맞물려서 `ImportError: cannot import name '_lazywhere' from 'scipy._lib._util'`가 나서 `0.14.6`으로 올렸다 — 로컬 conda 환경에서 이미 검증된 조합을 그대로 가져온 것.)

이 이미지로 mlflow-server에 실제로 run을 찍어서 **postgres에 파라미터/메트릭이 저장되는지**, **`--serve-artifacts` 프록시로 아티팩트 업로드가 되는지**까지 다 확인했다.

## 3. 수집 단계 (`collect.py`)

Ecount에는 매출 이력을 가져오는 GET API가 없다는 걸 이미 확인했었다(1편 참고). 지금은 Ecount 웹 화면에서 수동 export한 `data/김홍은_나은교육_출고수량.xlsx`가 원본이고, 이건 나중에 Airflow가 갱신할 예정이지만 최종 형태(엑셀 그대로 갱신 vs CSV vs DB 적재)는 아직 미정이다.

그래서 "수집" 단계는 **원본이 뭐든 검증하고 스냅샷 뜨는 스크립트**로 정의했고, `--source` 인자로 원본 경로를 받게 만들어서 나중에 Airflow 산출물 형식이 바뀌어도 스크립트 자체는 안 바뀌게 했다.

검증 로직은 다음 단계인 `05_매출데이터_김홍은_전처리.ipynb`가 실제로 기대하는 구조를 그대로 근거로 삼았다 — 연도별 시트(`2017년도`~`2024년도`), 시트마다 다른 위치(`colno`)에서 시작하는 `항목`/`수량` 컬럼, 타깃 품목 존재 여부까지 확인한다.

### 트러블슈팅: VirtioFS 바인드 마운트에서 원본 파일 읽기 실패

`collect.py`를 처음 돌렸을 때 원본 파일을 열자마자 이런 에러가 났다.

```
OSError: [Errno 35] Resource deadlock avoided
```

`pandas.ExcelFile()`뿐 아니라 순수 `open(path, 'rb').read()`로도 재현됐고, 다른 `.xlsx` 파일들에서도 동일하게 발생했다. 반면 컨테이너 안에서 새로 만든 텍스트 파일이나, 호스트에서 `cp`로 복사한 사본은 같은 마운트 경로에서도 문제없이 읽혔다. 원인은 macOS Docker Desktop의 **VirtioFS 바인드 마운트가 특정 파일을 통째로 read()할 때 생기는 알려진 버그**로 보인다 (파일 자체나 내용 문제가 아니라, 마운트된 원본 inode를 큰 덩어리로 직접 읽을 때만 재현됨).

해결은 간단했다 — `collect.py`의 순서를 "원본을 직접 검증"에서 "**먼저 스냅샷을 복사하고, 그 스냅샷을 검증**"으로 바꿨다. `shutil.copy2`로 한 번 복사한 파일은 문제없이 읽혔다.

```python
def main() -> None:
    ...
    dest = snapshot(args.source, args.raw_dir)  # 먼저 복사
    validate(dest)                               # 복사본을 검증
```

결과:

```
$ docker compose run --rm pipeline python src/collect.py
[collect] 스냅샷 저장: /data/raw/sales_20260723_023853.xlsx
[collect] 검증 통과: 8개 연도 시트 확인 완료 (/data/raw/sales_20260723_023853.xlsx)
```

## 다음 할 일

- 전처리 단계: `05_매출데이터_김홍은_전처리.ipynb`의 로직(품목 필터 → 스텝/월 파싱 → 월별 리샘플)을 `preprocess.py`로 포팅
- 학습 단계: `SARIMAModel` + mlflow tracking/registry 연동
- 서빙 단계: pyfunc 래퍼 + `mlflow models serve`
