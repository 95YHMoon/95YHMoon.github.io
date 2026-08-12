---
layout: post
title: "[MLOps 실전 구축] 5. 한 Experiment 안에서 여러 모델 비교"
series: "MLOps 실전 구축"
date: 2026-04-28
categories: [Data Engineer, MLops]
tags: [mlflow, experiment-tracking, sarima]
---

"변수를 바꿀 때마다 나오는 모델들을 한 experiment에서 관리하고 싶다"는 생각에서 시작한 실습이다. 결론부터 말하면, 지금 구조가 이미 그렇게 되어 있다. `train.py`가 experiment 이름을 고정해두고 바뀌는 값(`step`)을 파라미터로 기록하기만 하면, 실행할 때마다 같은 Experiment 안에 새 Run으로 쌓인다. 지금까지 스텝 2만 돌려봤으니 나머지(0/1/3)를 마저 돌려서 실제로 확인했다.

## 실행

```bash
for s in 0 1 3; do
  docker compose run --rm pipeline python src/train.py --step $s
done
```

각 실행마다 Registry에 새 버전이 하나씩 등록되고, 전부 같은 Experiment 아래 Run으로 쌓였다.

## 결과 비교

`MlflowClient.search_runs()`로 Experiment 안의 모든 Run을 코드로 조회해서 표로 정리했다. UI의 Compare로도 똑같이 볼 수 있다.

| step | 선택된 order | valid_mape | **test mape** |
|---|---|---|---|
| 0 | ARIMA(2,1,0)(0,1,0)[12] | 2.91 | **7.06** |
| 1 | ARIMA(2,1,0)(2,0,0)[12] | 4.90 | **6.58** |
| 2 | ARIMA(2,1,0)(0,1,0)[12] | 1.76 | **12.98** |
| 3 | ARIMA(2,1,0)(2,0,0)[12] | 2.87 | **35.53** |

## 발견: 스텝별 정확도 편차가 크다

스텝 0, 1은 6~7%대로 준수한데, 스텝 3만 유독 35.5%로 나쁘다. 예측값을 보면 실제값보다 매번 1000 가까이 낮게 나온다. 재미있는 건 validation MAPE(2.87)는 오히려 스텝 0/1보다 좋았는데 test에서 크게 틀렸다는 점이다. 이건 d/D 선택 자체의 문제라기보다 스텝 3 데이터가 표본 특성상 더 불안정할 가능성을 시사한다. 원인 분석은 다음 과제로 남긴다. 지금은 "여러 모델을 한 experiment에서 비교"라는 목적 자체는 충분히 이뤘다.

## 참고: nested run

지금은 스텝들을 셸 for문으로 따로 돌렸는데, 조합이 많아지면(d×D 탐색처럼) `start_run(nested=True)`로 "부모 run 하나 아래 자식 run들"을 묶어서 UI에 트리로 보이게 할 수도 있다. 지금은 print로만 남기는 d/D 탐색 과정 자체를 run으로 남기고 싶어지면 그때 시도해볼 생각이다.
