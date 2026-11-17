---
title: "[데이터 파이프라인 #0] Airflow와 Iceberg(Polaris) 로컬 환경 연동 기록"
date: 2026-10-13 17:00:00 +0900
categories: [Data Engineer, Infrastructure]
tags: [airflow, pyiceberg, iceberg, docker-compose, data-engineering]
description: 기존 Airflow 환경을 Apache Iceberg 저장층(Polaris + SeaweedFS)에 맞춰 재설계하고, 제한된 리소스 환경에 맞춘 컨테이너 최적화 및 보안 강화 기법을 다룬다.
---

#### 개요

Apache Airflow는 데이터 오케스트레이션 계층에서 작업 의존성 관리와 스케줄링을 담당한다. 본 플랫폼 구축 프로젝트에서 정의한 Airflow의 주 임무는 **"매시간 주문 API 데이터를 수집하여 Iceberg `raw.orders` 테이블에 적재하는 파이프라인 구동"**이다.

본 문서에서는 일반적인 설치 절차를 생략하고, **Iceberg 저장층 연동 구조**와 **로컬 개발 환경(MacBook Air)의 제약 극복을 위한 컨테이너 아키텍처 재설계 포인트**를 중심으로 기술한다.

---

#### 1. Airflow와 PyIceberg 연동 아키텍처

파이프라인 코드 내부에 데이터베이스 접속 정보나 인증 정보를 하드코딩하는 안티패턴을 배제하기 위해, 모든 Catalog 연결 설정은 **환경 변수 패턴**으로 추상화하여 주입했다.

```yaml
# docker-compose 환경 변수 설정 예시
PYICEBERG_CATALOG__LAKEHOUSE__TYPE: rest
PYICEBERG_CATALOG__LAKEHOUSE__URI: http://polaris:8181/api/catalog
PYICEBERG_CATALOG__LAKEHOUSE__WAREHOUSE: lakehouse
PYICEBERG_CATALOG__LAKEHOUSE__CREDENTIAL: airflow:******
PYICEBERG_CATALOG__LAKEHOUSE__SCOPE: PRINCIPAL_ROLE:ALL
PYICEBERG_CATALOG__LAKEHOUSE__S3__ENDPOINT: http://seaweedfs:8333
PYICEBERG_CATALOG__LAKEHOUSE__S3__ACCESS_KEY_ID: S3_KEY
PYICEBERG_CATALOG__LAKEHOUSE__S3__SECRET_ACCESS_KEY: S3_SECRET
```

PyIceberg 라이브러리는 다음의 명명 규칙에 따라 환경 변수를 파싱한다.
- `PYICEBERG_CATALOG__<CATALOG_NAME>__<PROPERTY>` 구문을 통해 해당 카탈로그 인스턴스 옵션으로 변환
- 이중 언더스코어(`__`)는 점(`.`)으로, 단일 언더스코어(`_`)는 하이픈(`-`)으로 매핑 (예: `S3__ACCESS_KEY_ID` → `s3.access-key-id`)

이를 통해 DAG 코드상에서는 단 한 줄의 선언으로 카탈로그 연결을 안전하게 로드할 수 있다.

```python
from pyiceberg.catalog import load_catalog

catalog = load_catalog("lakehouse")
```

##### Trino와 Airflow의 카탈로그 설정 비교

| 구성 항목 | Trino Query Engine | Airflow Orchestrator (PyIceberg) |
| :--- | :--- | :--- |
| **설정 위치** | `catalog/iceberg.properties` | 환경 변수 (`PYICEBERG_CATALOG__...`) |
| **Polaris 계정** | `trino` (Principal) | `airflow` (Principal) |
| **객체 스토리지 접근** | 동일 S3 Key | 동일 S3 Key |
| **Target Catalog** | `lakehouse` | `lakehouse` |

두 분산 구성 요소가 동일한 Polaris REST Catalog를 참조하므로, Airflow에서 적재 후 **트랜잭션이 커밋(Commit)되는 즉시** Trino에서 최신 스냅샷을 쿼리할 수 있다. Polaris의 메타데이터 포인터가 원자적으로 갱신되기 전까지는 쓰기 작업 중인 파티션이 읽기 작업에 노출되지 않아 데이터 일관성이 보장된다.

---

#### 2. 컨테이너 토폴로지 및 리소스 제어

로컬 환경의 메모리 상한을 고려하여 컨테이너 실행 흐름과 리소스 할당을 경량화했다.

```
db-setup ─→ airflow-init ─→ airflow-apiserver ─→ airflow-scheduler
                         └→ airflow-dag-processor      ↑
                                       polaris-setup ──┘
```

| 서비스 컨테이너 | 담당 기능 | 메모리 Limit / 실시간 사용량 |
| :--- | :--- | :--- |
| `airflow-init` | DB 마이그레이션 수행 (완료 후 정지) | Exited (0) |
| `airflow-apiserver` | Web UI 제공 및 상태 보고 엔드포인트 | 768 MiB / 293 MiB |
| `airflow-scheduler` | 스케줄링 및 **LocalExecutor 기반 작업 수행** | 1 GiB / 436 MiB |
| `airflow-dag-processor` | `dags/` 디렉터리 파싱 및 DAG 메타데이터 갱신 | 384 MiB / 173 MiB |

> **LocalExecutor 채택 사유**: 별도의 Celery Worker 컨테이너를 구동하지 않고 Scheduler 프로세스 내에서 작업을 직접 실행함으로써 컨테이너 오버헤드를 최소화했다.

---

#### 3. 클라우드 구성 대비 최적화 포인트

| 구분 | 이전 (Cloud/BigQuery 빌드) | 현재 (Local Iceberg 빌드) | 변경 사유 및 기술적 효과 |
| :--- | :--- | :--- | :--- |
| **Triggerer** | 활성화 | **제거** | 비동기 대기 작업 미사용에 따른 메모리 점유 최소화 |
| **Memory Limit** | Unset (무제한) | 컨테이너별 strict 지정 | 로컬 host OOM(Out of Memory) 방지 |
| **Web Worker** | 4개 프로세스 | 1개 프로세스 | 리소스 절감을 위한 설정 조정 |
| **DAG Auto-start** | 즉시 등록 및 실행 | **일시정지 상태로 신규 등록** (`dags_are_paused_at_creation=True`) | 코드 수정 시 외부 API의 예기치 않은 호출 방지 |
| **Log Storage** | 로컬 디렉터리 Mount | Named Volume 분리 | 민감 자격 증명 로그의 Git 레포지토리 유출 차단 |
| **Web Port** | `8080` | `8081` | Trino 코디네이터(`8080`)와의 포트 충돌 회피 |

---

#### 4. 이미지 커스텀 및 의존성 관리

공식 Airflow 파이썬 이미지 기반으로 최소한의 필수 패키지만 고정 버전으로 설치했다.

```dockerfile
FROM apache/airflow:3.3.2-python3.12
RUN pip install --no-cache-dir pyiceberg[pyarrow]==0.12.0
```

- PyIceberg 0.12.0 버전을 적용하여 PyArrow와의 호환성을 확보했다.

---

#### 5. 보안 인증 정보 관리 체계

| 항목 | 역할 및 성격 | 변경 시 주의사항 |
| :--- | :--- | :--- |
| Admin Password | 웹 로그인 검증용 | 컨테이너 재시작 시 덮어쓰기 가능 |
| Fernet Key | DB 저장 데이터 암호화 키 | **변경 시 기존 데이터 복호화 불가** |

> **Warning**: Fernet Key는 메타데이터베이스 내 Connection, Variable 등의 민감 필드를 복호화하는 대칭키이다. 사전 조치 없이 키를 변경할 경우 기존 접속 정보 파싱 과정에서 에러가 발생한다. 키 로테이션 시 새 키와 구 키를 쉼표로 나열한 후 `airflow rotate-fernet-key` 명령을 수행해야 한다.

---

#### 6. 인프라 정상 작동 검증 (`_infra_smoke` DAG)

인프라 구성 완료 후 연동 상태 모니터링을 위해 Smoke Test DAG를 수행했다.

| 작업 단계 (Task) | 검증 요소 | 최종 결과 |
| :--- | :--- | :--- |
| `polaris_namespaces` | `load_catalog("lakehouse")` 호출 및 `raw` 네임스페이스 존재 여부 확인 | `['raw']` 수신 성공 |
| `seaweedfs_bucket` | PyArrow S3 파일시스템 인터페이스를 통한 `lake` 버킷 접근성 확인 | `['raw']` 버킷 엔트리 확인 |

---

#### 회고 및 시사점

동일한 오케스트레이션 도구라도 운영 환경(클라우드 분산 환경 vs 단일 노드 로컬 환경)에 따라 리소스 할당 전략과 토폴로지가 상이해야 한다. 특히 외부 API 연동이 포함된 파이프라인 개발 시 `dags_are_paused_at_creation=True` 설정과 시크릿 관리 체계를 엄격히 적용함으로써 안정성을 확보할 수 있었다.

---