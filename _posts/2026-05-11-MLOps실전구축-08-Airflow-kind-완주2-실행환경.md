---
layout: post
title: "[MLOps 실전 구축] 8. Airflow + kind 완주 (2): 실행 환경"
series: "MLOps 실전 구축"
date: 2026-05-11
categories: [Data Engineer, MLops]
tags: [kubernetes, kind, docker, troubleshooting]
---

7편에서 Airflow/K8s 인증 문제를 잡은 뒤에도 두 가지가 더 있었다. 둘 다 로컬에서는 가려져 있다가 K8s로 옮기니 드러난 부류의 문제다.

## 문제 3: 컨테이너 안에 스크립트가 없다

증상. Pod는 멀쩡히 뜨는데, 컨테이너 안에 `collect.py`가 없다고 한다.

원인. `pipeline` 이미지는 세션 초반에 딱 한 번 빌드된 뒤로 재빌드된 적이 없었다. docker-compose에서는 `volumes`로 항상 최신 코드를 덮어써서(bind mount) 이 사실이 가려져 있었는데, `KubernetesPodOperator`는 그런 마운트 없이 이미지에 실제로 구워진 파일만 본다. 그래서 낡은 상태가 그대로 드러났다.

해결. 이미지를 재빌드한 뒤 `kind load docker-image`로 kind 안의 스냅샷도 갱신했다. `kind load`는 그 순간의 스냅샷이라, 호스트에서 이미지를 다시 만들어도 자동으로 반영되지 않는다. 매번 다시 로드해야 한다.

## 문제 4: 여기저기 500, 컨테이너가 이미 죽어있음

증상. `train` 태스크에서 500. 확인해보니 mlflow-server, postgres 같은 것들이 이미 죽어 있었다.

원인. 재빌드를 반복하며 쌓인 dangling 이미지로 Docker Desktop 가상 디스크가 꽉 차서, postgres가 `No space left on device`로 패닉했다. 게다가 서비스에 restart 정책이 없어서 한 번 죽으면 자동 복구도 안 됐다.

해결. 이미지와 빌드 캐시를 prune해서 공간을 확보하고 스택을 재기동했다.

## 최종 확인

```
run_id    run_name      mape
7f9a2644  sarima-step2  12.98   ← K8s Pod(train)가 만든 run, 기존 정상값과 일치
```

수집-전처리-학습이 로컬 docker-compose가 아니라 실제 K8s Pod 위에서 끝까지 돌았고, 결과 메트릭이 이전 정상값과 일치했다. 파이프라인이 오케스트레이션 위에서 완주된 것이다.

## 앞으로 손볼 것 (기록만)

- 서비스에 자동 복구 정책 추가. 컨테이너가 죽어도 되살아나게.
- 정기적인 이미지 prune 습관화, 또는 디스크 알람.
- 지금 MLflow 접근 방식(`host.docker.internal`)은 임시방편이다. 장기적으로는 kind 클러스터 안에 더 견고한 접근 경로를 둘 필요가 있다.
