---
title: n8n(nodemation) 컨테이너 빌드 기록
date: 2026-08-28 20:30:00 +0900
categories: [Data Engineer, n8n]
tags: [n8n, nodemation, docker, build-log]
---

지난 글에서 정리한 n8n(nodemation)을 실제로 기존 docker-compose 스택에 붙여봤다. 개념 정리와 실제 빌드 사이에는 항상 몇 가지 확인할 게 남기 마련이라, 이번엔 그 과정만 짧게 남겨둔다.

## docker-compose에 서비스 추가

기존에 qdrant, postgres, streamlit 세 서비스가 떠 있는 스택에 n8n 서비스 하나를 그대로 추가했다.

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

기본 이미지는 인증이 꺼져 있는 상태로 뜨기 때문에, `N8N_BASIC_AUTH_ACTIVE`를 켜고 계정 정보를 환경변수로 뺐다. 워크플로우 데이터는 `n8n_data` 볼륨에 남도록 해서 컨테이너를 내렸다 올려도 작업한 내용이 사라지지 않게 했다.

## 기동 확인

```
docker compose up -d n8n
docker ps --filter "name=qwen_n8n"
```

```
NAMES      STATUS         PORTS
qwen_n8n   Up 5 seconds   0.0.0.0:5678->5678/tcp
```

헬스체크 엔드포인트로 응답도 확인

```
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5678/healthz
200
```

여기까지는 별문제 없이 끝났다.

## 확인해야 했던 제약 하나

n8n 공식 이미지는 Node.js 베이스라 **Python이 들어있지 않다.** 개념 정리 글에서는 "Execute Command 노드로 스크립트를 직접 실행한다"고 썼는데, 지금 올린 컨테이너 안에서는 그게 안 된다는 뜻이다.

선택지는 두 가지다.

- 매칭 스크립트를 감싸는 작은 API 서버(FastAPI 등)를 따로 띄우고, n8n은 HTTP Request 노드로 그 API만 호출
- n8n 이미지 위에 Python과 필요한 의존성을 얹은 커스텀 이미지를 빌드

전자가 구조도 단순하고, 스크립트가 늘어나도 엔드포인트만 추가하면 되니 확장성도 낫다. 
