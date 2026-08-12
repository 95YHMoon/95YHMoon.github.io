---
layout: post
title: "한 Experiment 안에서 여러 모델(스텝) 비교하기"
date: 2026-04-28
categories: [Data Engineer,MLops]
tags: [mlflow, experiment-tracking, sarima]
---

"특정 변수를 바꿀 때마다 나오는 모델들을 한 experiment에서 관리하고 싶다"는 질문에서
시작한 실습. 결론부터 말하면 **이미 지금 구조가 그렇게 되어 있다** — `train.py`가
`mlflow.set_experiment("{project}-sales-forecast")`로 고정해두고, 바뀌는 값(`step`)을
`mlflow.log_param("step", args.step)`으로 기록해두기만 하면, 실행할 때마다 같은
Experiment 안에 새 Run으로 쌓인다. 지금까지 스텝 2만 돌려봤어서, 스텝 0/1/3까지
마저 돌려서 실제로 확인해봤다.

## 실행

```bash
cd mlops
for s in 0 1 3; do
  docker compose run --rm pipeline python src/train.py --step $s
done
```

각 실행마다 Registry에 새 버전이 하나씩 등록되고(v6, v7, v8), 전부 같은 Experiment
`{project}-sales-forecast` 아래에 Run으로 쌓였다.

## 결과 비교

`MlflowClient.search_runs()`로 Experiment 안의 모든 Run을 코드로 직접 조회해서
표로 정리했다 (UI의 Compare 기능으로도 똑같이 볼 수 있음):

```python
client = mlflow.tracking.MlflowClient()
exp = client.get_experiment_by_name("{project}-sales-forecast")
runs = client.search_runs([exp.experiment_id], order_by=["attributes.start_time ASC"])
```

| step | 선택된 order | d, D | valid_mape | **test mape** |
|---|---|---|---|---|
| 0 | ARIMA(2,1,0)(0,1,0)[12] | 1, 1 | 2.91 | **7.06** |
| 1 | ARIMA(2,1,0)(2,0,0)[12] | 1, 0 | 4.90 | **6.58** |
| 2 | ARIMA(2,1,0)(0,1,0)[12] | 1, 1 | 1.76 | **12.98** |
| 3 | ARIMA(2,1,0)(2,0,0)[12] | 1, 0 | 2.87 | **35.53** |

(스텝 2 행은 지난 디버깅 과정에서 나온 run들 중 최종본만 추림 — 실제 Experiment
안에는 스텝 2에 대한 run이 4개 더 있다, 42.17/40.83/97.76/12.98 순으로 남아있는
디버깅 히스토리.)

## 발견: 스텝별로 정확도 편차가 크다

스텝 0, 1은 6~7%대로 꽤 준수한데, **스텝 3만 유독 35.5%로 나쁘다.** `test_pred`를
보면 실제값(3152~3254 근처)보다 매번 1000 가까이 낮게 예측한다 — 지난번 스텝 2에서
"validation에서는 좋았는데 실제로는 추세를 못 따라가는" 패턴과 비슷해 보인다.
validation_mape(2.87)는 오히려 스텝 0/1보다도 좋았는데 test에서 크게 틀렸다는 게,
d/D 선택 자체의 문제라기보다 **스텝 3 데이터가 다른 스텝보다 표본 특성상 더
불안정할 가능성**을 시사한다. 원인 분석은 다음 과제로 남겨둔다 — 지금은 "여러
모델을 한 experiment에서 비교"하는 목적 자체는 충분히 달성됐다.

## 참고: nested run (안 써봤지만 있는 기능)

지금은 4개 스텝을 셸에서 for문으로 따로따로 돌렸는데, 조합이 많아지면(`select_best_order`의
d×D 6개 조합처럼) `mlflow.start_run(nested=True)`로 "부모
run 하나 아래 자식 run들"을 묶어서 UI에 트리로 보이게 할 수도 있다. 지금은 print로만
남기고 있는 d/D 탐색 과정 자체를 mlflow run으로도 남기고 싶다면 다음에 시도해볼 것.
