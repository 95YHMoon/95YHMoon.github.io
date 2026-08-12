---
layout: post
title: "K8s 개념 정리 — kind 멀티노드, Pod/Deployment, Airflow KubernetesExecutor 연결"
date: 2026-04-29
categories: [Data Engineer,MLops]
tags: [kubernetes, kind, airflow, kubernetesexecutor]
---

이번 구간에서 다룬 K8s 핵심 개념들을 정리한다. ([전편](/notations/2026-04-28-airflow-k8s-direction)에서 방향을 정했던 것의 실습편.)

## kind로 멀티노드 클러스터 만들기

기본값(`kind create cluster`)은 노드 1개(control-plane 겸 worker)다. 여러 노드 구성은 설정 파일로 명시한다:

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

`nodes:` 목록의 항목 하나 = 컨테이너 하나 = 노드 하나. **control-plane 노드는 기본적으로 taint(오염 표식)가 걸려서 일반 워크로드(Pod)가 배치되지 않는다** — 클러스터 관리 기능이 일반 작업 부하와 자원을 다투지 않게 하기 위한 설계다. 그래서 실제 애플리케이션을 올리려면 worker가 최소 1개 이상 필요하다.

## Pod / Deployment / ReplicaSet 계층

K8s에서 워크로드를 표현하는 오브젝트들은 계층을 이룬다:

```
Deployment ("이 템플릿으로 N개 유지해라")
    ↓ 자동 생성
ReplicaSet ("지금 몇 개 떠있나 세고 부족하면 채움")
    ↓ replicas 수만큼 자동 생성
Pod × N (실제 컨테이너가 도는 단위)
```

- **Pod 단독으로 만들면** 하나 만들어지고 끝 — 죽어도 다시 안 살아난다.
- **Deployment로 만들면** Deployment/ReplicaSet/Pod 세 오브젝트가 다 생기고, Pod 이름도 `<deployment이름>-<해시>-<랜덤>` 식으로 자동 생성된다 (Deployment 이름 자체가 Pod 이름이 되는 게 아니다).
- Deployment는 Pod를 **이름이 아니라 라벨**로 추적한다 — `spec.selector.matchLabels`와 `spec.template.metadata.labels`가 같은 값이어야 서로 연결된다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deployment
spec:
  replicas: 4
  selector:
    matchLabels: {app: hello}
  template:
    metadata:
      labels: {app: hello}
    spec:
      containers:
        - name: hello
          image: busybox
          command: ["sh", "-c", "sleep 3600"]
```

`replicas: 4`로 만들면 스케줄러가 이 4개를 worker 노드들에 나눠 배치한다(`kubectl get pods -o wide`의 `NODE` 컬럼으로 분산 확인 가능). 이게 K8s가 자원을 여러 노드에 걸쳐 효율적으로 쓰는 방식의 제일 기본적인 그림이다.

## 자주 쓰는 kubectl 조회 패턴

| 명령 | 용도 |
|---|---|
| `kubectl get pods` | 현재 상태를 한 번 스냅샷으로 |
| `kubectl get pods -w` | `-w`(watch) — 상태 바뀔 때마다 계속 출력. `ContainerCreating → Running` 전이를 실시간으로 볼 때 |
| `kubectl get pods -o wide` | 기본엔 안 보이는 `NODE`/`IP` 컬럼 추가 |
| `kubectl logs <pod>` | 컨테이너 stdout. 컨테이너가 아직 안 떴으면(`ContainerCreating`) 에러남 |
| `kubectl describe pod <pod>` | 어느 노드에 배치됐는지, 이벤트 로그 등 상세정보 |
| `kubectl get deployments,replicasets,pods` | 세 계층을 한 번에 나열 |

## docker-compose ↔ kind: 서로 다른 두 네트워크 잇기

Airflow(docker-compose가 만든 `mlops_default` 네트워크)와 kind(자체 `kind` 네트워크)는 기본적으로 서로 존재를 모르는 별개의 도커 네트워크다. 연결하는 방법은 두 갈래:

| 방법 | 방식 |
|---|---|
| `host.docker.internal` | 호스트가 노출한 포트(예: kind API의 `127.0.0.1:<port>`)를 컨테이너 안에서 특수 호스트명으로 우회 접근 |
| **같은 네트워크에 합류** (채택) | 컨테이너가 네트워크 여러 개에 동시에 속할 수 있다는 점을 이용, Airflow 컨테이너를 kind의 네트워크에도 연결. 포트 매핑 없이 컨테이너 이름으로 직접 통신 |

두 번째 방법을 쓰려면, docker-compose 입장에서 kind가 만든 네트워크는 "내가 만든 게 아니라 이미 있는 것"이므로 `external: true`로 선언해야 한다:

```yaml
services:
  airflow-scheduler:
    ...
    networks:
      - default   # 기존 mlflow-server 등과 통신
      - kind      # kind 컨트롤플레인과 통신

networks:
  default: {}
  kind:
    external: true
```

연결되면, 컨테이너 안에서 kind의 컨트롤플레인 컨테이너 이름(`mlops-learn-control-plane`)으로 K8s API에 직접 도달 가능해진다 (`https://mlops-learn-control-plane:6443`). 호스트가 매핑해준 포트(`127.0.0.1:63780` 같은)는 호스트 자신을 위한 것이라, 다른 컨테이너 안에서는 의미가 없다는 점이 이 문제의 핵심.

## Airflow에서 KubernetesExecutor를 쓰기 위한 전제조건

1. **패키지**: `apache-airflow-providers-cncf-kubernetes` (K8s Python 클라이언트 + `KubernetesExecutor` 클래스 제공). DockerOperator용 `apache-airflow-providers-docker`와는 별개 패키지.
2. **네트워크 연결**: 위 항목.
3. **kubeconfig(인증)**: 아직 미완료 — 다음 단계.
4. **Pod Template**: 아직 미완료 — 태스크마다 만들 Pod의 스펙(이미지 등) 정의.
5. **실행기 전환**: `AIRFLOW__CORE__EXECUTOR: LocalExecutor` → `KubernetesExecutor`.

## 진행 방식 메모

파일 수정(Dockerfile, docker-compose.yml, DAG 등)도 명령 실행과 동일하게 취급 — 개념을 설명받았다고 곧바로 대신 고쳐주는 게 아니라, 직접 고치거나 사전 협의 후 진행하는 흐름으로 확정.
