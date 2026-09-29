---
title: n8n 컨테이너 빌드 및 스택 구성 기록
date: 2026-08-28 20:30:00 +0900
categories: [Data Engineer, n8n]
tags: [n8n, nodemation, docker, build-log]
---

기존 Docker Compose 기반 스택(Qdrant, PostgreSQL, Streamlit)에 n8n 서비스를 추가한 과정 기록.

## docker-compose 서비스 추가

`docker-compose.yml` 파일에 n8n 서비스를 다음과 같이 구성했다.

```yaml
n8n:
  image: n8nio/n8n:latest
  container_name: qwen_n8n
  restart: unless-stopped
  ports:
    - "${N8N_PORT:-5678}:5678"
  environment:
    N8N_BASIC_AUTH_ACTIVE:   "true"
    N8N_BASIC_AUTH_USER:     ${N8N_BASIC_AUTH_USER:-admin}
    N8N_BASIC_AUTH_PASSWORD: ${N8N_BASIC_AUTH_PASSWORD:-changeme}
    N8N_HOST:                ${N8N_HOST:-localhost}
    N8N_PORT:                5678
    N8N_PROTOCOL:            http
    GENERIC_TIMEZONE:        Asia/Seoul
    TZ:                      Asia/Seoul
  volumes:
    - n8n_data:/home/node/.n8n
  networks:
    - qwen_net
```

- 보안을 위해 N8N_BASIC_AUTH_ACTIVE를 활성화하고 인증 정보를 환경변수로 분리
- 워크플로우 및 노드 설정 유지를 위해 n8n_data 볼륨을 마운트

```bash
docker compose up -d n8n
docker ps --filter "name=qwen_n8n"
```

헬스체크
```
NAMES      STATUS         PORTS
qwen_n8n   Up 5 seconds   0.0.0.0:5678->5678/tcp
```

## 런타임 제약사항 및 대안
n8n 공식 Docker 이미지는 Node.js 환경 기반이므로 기본적으로 Python 런타임이 포함되어 있지 않다. 따라서 Execute Command 노드를 통해 컨테이너 내부에서 Python 스크립트를 직접 호출할 수 없다.

이를 해결하기 위한 구조적 선택지는 두 가지다.

1. 외부 API 서버 분리 : Python 스크립트 연산부를 별도 API(FastAPI 등) 컨테이너로 띄우고, n8n에서는 HTTP Request 노드로 호출
2. 커스텀 이미지 빌드: n8n 베이스 이미지에 Python 및 관련 의존 패키지를 추가 설치한 Dockerfile을 직접 빌드

연산 로직과 오케스트레이션의 결합도를 낮추고 모듈 확장성을 확보하기 위해 API 분리 방식으로 파이프라인을 구성