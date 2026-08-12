---
layout: post
title: "로컬 MLflow로 MLOps 파이프라인 만들기 (4) — MAPE 42% → 13%, 디버깅 기록"
date: 2026-04-23
categories: [Data Engineer,MLops]
tags: [mlflow, sarima, pmdarima, evaluation, model-selection]
---

[3편](/notations/2026-04-23-mlops-preprocess-train-serve)에서 파이프라인을 한 바퀴 돌렸을 때 MAPE가 42%로 나왔다. 예전에 이 데이터로 SARIMA를 돌렸을 때는 17% 정도였던 걸로 기억하는데, 어디서 꼬였는지 추적한 기록.

## 문제 1: 테스트셋이 사실상 1개 데이터포인트였다

`--test-size 6`으로 마지막 6개월을 테스트셋으로 잡았는데, 스텝 2 기준 실제 값이 `[3428, 0, 0, 0, 0, 0]`이었다. 원본 export 파일이 2024-07까지만 실측이 있고 그 뒤(8~12월)는 "아직 기록 안 됨"이 0으로 채워져 들어온 거였다(2편에서 이미 확인했던 문제). `evaluate()`는 실제값이 0인 달을 나눗셈 문제로 마스킹해서 제외하는데, 그러면 남는 건 2024-07 딱 한 달치 오차뿐이다. "6개월 평가"라고 부르고 있었지만 사실상 1개 포인트짜리 지표였던 셈.

고침: 시계열 끝에서부터 연속된 0(미기록)을 자동으로 잘라내고, 그 앞의 "진짜 마지막 실측 시점" 기준으로 테스트 구간을 잡도록 `trim_trailing_zero_tail()`을 추가했다.

```python
def trim_trailing_zero_tail(series: pd.Series) -> pd.Series:
    values = series.values
    end = len(values)
    while end > 0 and values[end - 1] == 0:
        end -= 1
    return series.iloc[:end]
```

## 문제 2: auto_arima가 추세를 놓친다

테스트셋을 고쳤는데도 MAPE가 40%대에 머물렀다. 확인해보니 `auto_arima`가 `ARIMA(0,0,0)(2,0,0)[12]`, 즉 **차분을 전혀 안 한 모델**을 골랐다. 이 시계열은 몇 년에 걸쳐 뚜렷하게 우상향하는데(2000대 → 3500대), 차분이 없으면 모델이 그냥 과거 평균 근처로 예측해버린다.

이상한 건, `pmdarima.arima.ndiffs`/`nsdiffs`로 KPSS/ADF/PP/OCSB 단위근 검정을 직접 돌려봐도 전부 "d=0, D=0으로 충분하다"고 나왔다는 것. 통계적으로 "단위근이 없다"는 것과 "추세가 없다"는 건 다른 이야기라, 이 시리즈처럼 노이즈 대비 추세가 두드러지는 경우 단위근 검정만으로는 차분 필요성을 못 잡아내는 걸 실제로 겪었다.

### 해결: d/D를 검정이 아니라 validation 성능으로 고른다

수동으로 order를 고정하는 대신, **d∈{0,1,2} × D∈{0,1} 조합을 실제로 다 학습시켜서 validation 구간 예측 오차(MAPE)로 최적 조합을 고르는** 반자동 방식으로 바꿨다. p,q,P,Q는 각 d/D 조합 안에서 여전히 `auto_arima`가 AIC 기준으로 알아서 찾는다.

```
d=0 D=0 -> ARIMA(0,0,0)(2,0,0)[12]  valid_mape=35.17
d=0 D=1 -> ARIMA(0,0,0)(0,1,0)[12]  valid_mape=23.79
d=1 D=0 -> ARIMA(2,1,0)(2,0,0)[12]  valid_mape=3.47
d=1 D=1 -> ARIMA(2,1,0)(0,1,0)[12]  valid_mape=1.76   ← 선택
d=2 D=0 -> ARIMA(3,2,0)(2,0,0)[12]  valid_mape=18.24
d=2 D=1 -> ARIMA(3,2,0)(0,1,0)[12]  valid_mape=24.12
```

## 문제 3: 골라놓고 refit 했더니 예측이 0으로 붕괴

`d=1, D=1`이 validation에서 제일 좋았길래, 이 설정으로 (validation 구간까지 포함한) 더 많은 데이터로 재학습(refit)해서 최종 테스트에 썼다. 그랬더니 예측값이 전부 0 근처로 붕괴해버렸다 — MAPE 97%.

원인은 계절차분(D=1, 주기 12)까지 겹치면 사실상 1년치 데이터를 통째로 잃는 건데, 월별 데이터가 80~90개 정도뿐이라 표본이 조금만 바뀌어도(6개월 추가) 계절 성분 추정이 완전히 달라지는 걸로 보인다. 같은 order(`ARIMA(2,1,0)(0,1,0)[12]`)인데 학습 데이터만 6개월 늘었을 뿐인데 전혀 다른 모델처럼 행동한 것.

해결: **refit하지 않는다.** validation에 썼던 모델(선택 당시 학습 데이터까지만 학습된 상태)을 그대로 재사용해서, 예측 지평선만 `valid_size + test_size`로 늘려 뒤쪽 `test_size`개월만 최종 평가/서빙에 쓴다.

```python
full_pred_log = model.predict(args.valid_size + args.test_size)
pred = np.clip(np.expm1(full_pred_log), a_min=0, a_max=None)[args.valid_size:]
```

결과:

```
[train] test_actual=[   0 3627 3474 3523 3519 3428]
[train] test_pred   =[   0.  3081.7 4016.3 3149.8 3190.2 2937.3]
[train] metrics={'mae': 380.04, 'rmse': 424.17, 'mape': 12.98}
```

**MAPE 42% → 12.98%.** 예전에 봤던 17%와 비슷하거나 더 낫다.

## 알아둘 점 (트레이드오프)

지금 등록/서빙되는 모델은 refit을 안 했기 때문에, "train+valid+test를 다 뺀" 시점까지의 데이터로만 학습돼 있다 — 즉 실제 운영에서 최신 데이터를 다 반영한 모델은 아니다. 이건 mlflow에 `model_trained_through` 파라미터로 남겨서 투명하게 확인 가능하게 해뒀다. 근본적으로는, 이 파이프라인 자체가 Airflow로 데이터가 최신화될 때마다 재학습되는 걸 전제로 하고 있어서, 매번 재학습 시점의 최신 데이터로 새로 이 d/D 탐색 과정을 다시 거치면 자연히 해소되는 구조다.

## 다음 할 일

- 나머지 스텝(0,1,3)에도 같은 파이프라인 적용해서 d/D 선택이 스텝마다 다르게 나오는지 확인
- Airflow 연동 후 실데이터가 최신화되면 이 run을 다시 돌려서 재검증
