# 🐳 Vantiq Edge Docker 배포 매뉴얼

Vantiq Edge Docker Deployment & 운영 매뉴얼

> 본 문서는 https://test.vantiq.com/docs/system/vantiqedge/#docker-deployment 를 기반으로
> 
> 
> **Docker 기반 Vantiq Edge 배포, 운영, 업그레이드, 초기 설정**까지를
> 
> **실무 운영 관점에서 한글로 정리한 통합 매뉴얼**입니다.
> 

---

## 0. 기준 디렉토리 정의 (중요)

본 문서에서는 다음 디렉토리를 **기준 디렉토리(Base Directory)** 로 사용합니다.

```
/opt/vantiq-edge
├─ docker-compose.yml
├─ config/
│  ├─ license.key
│  └─ public.pem
├─ log/              (선택)
├─ upgrade/          (업그레이드 시)
```

- 이 디렉토리를 이하 **`EDGE_HOME`** 이라고 부릅니다.
- **docker-compose.yml 이 위치한 디렉토리에서만** `docker compose` 명령을 실행해야 합니다.

```bash
EDGE_HOME=/opt/vantiq-edge
```

---

## 1. Vantiq Edge 개요

**Vantiq Edge**는 데이터 소스와 가까운 Edge 환경에서 실행되는

**비클러스터형 Vantiq 서버**입니다.

### 주요 특징

- Low Latency 실시간 처리
- Vantiq Core 기능 제공
- Vantiq Cloud와 연동 가능 (선택)

### 포함되지 않는 기능

- Grafana 기반 모니터링
- OAuth Provider 연동
- 클러스터 배포

---

## 2. 배포 방식 개요

| 방식 | 설명 |
| --- | --- |
| **Docker 배포 (권장)** | AI Assistant, GenAI Flow, Qdrant 포함 |
| Executable JAR | 단일 서버, AI 기능 미지원 |

> ✅ 운영 환경에서는 Docker 배포 필수
> 

---

## 3. 사전 요구사항

---

### 3.1 라이선스 파일

Vantiq Edge 기동을 위해 아래 파일이 필요합니다.

- `license.key`
- `public.pem`

📩 발급: `support@vantiq.com`

---

### 3.2 quay.io 접근 권한 (필수)

Vantiq Edge Docker 이미지는 **Private quay.io Repository**에 존재합니다.

### 준비 절차

1. quay.io 계정 생성
2. quay.io ID를 `support@vantiq.com`으로 전달
3. 승인 후 이미지 Pull 가능

---

### 3.3 시스템 요구사항

### 하드웨어

| 항목 | 최소 사양 |
| --- | --- |
| CPU | 64-bit x86 (Docker 필수) |
| Memory | 8GB 이상 |
| Storage | 32GB 이상 |

### 소프트웨어

- Linux / macOS / Windows (64-bit)
- Docker Engine 19.03.12+
- Docker Compose
- **MongoDB 4.2 이하 (⚠️ 4.3 이상 미지원)**

---

## 4. 작업 디렉토리 준비

📍 **실행 위치:** 서버 내 아무 위치 (보통 `/opt`)

```bash
sudo mkdir -p /opt/vantiq-edge/config
cd /opt/vantiq-edge
```

---

### 4.1 라이선스 파일 복사

📍 **실행 위치:** 어디서든 가능

```bash
cp license.key public.pem /opt/vantiq-edge/config/
```

---

## 5. Docker Compose 설정

---

### 5.1 docker-compose.yml 생성

📍 **실행 위치:** `EDGE_HOME`

```bash
cd /opt/vantiq-edge
vi docker-compose.yml
```

---

### 5.2 docker-compose.yml 예시

```yaml
services:
  vantiq_edge:
    container_name: vantiq_edge_server
    image: quay.io/vantiq/vantiq-edge:1.43
    ports:
      - 8080:8080
    depends_on:
      - vantiq_edge_mongo
      - vantiq_edge_qdrant
    restart: unless-stopped
    volumes:
      - ./config/license.key:/opt/vantiq/config/license.key
      - ./config/public.pem:/opt/vantiq/config/public.pem
    networks:
      - vantiq_edge

  vantiq_edge_mongo:
    container_name: vantiq_edge_mongo
    image: bitnamilegacy/mongodb:4.2.21
    restart: unless-stopped
    environment:
      - MONGODB_USERNAME=ars
      - MONGODB_PASSWORD=ars
      - MONGODB_DATABASE=ars02
      - MONGODB_ROOT_USER=root
      - MONGODB_ROOT_PASSWORD=ars
    volumes:
      - vantiq_edge_data:/bitnami:rw
    networks:
      vantiq_edge:
        aliases: [edge-mongo]

  vantiq_ai_assistant:
    container_name: vantiq_ai_assistant
    image: quay.io/vantiq/ai-assistant:1.43
    restart: unless-stopped
    network_mode: "service:vantiq_edge"

  vantiq_genai_flow_service:
    container_name: vantiq_genai_flow_service
    image: quay.io/vantiq/genaiflowservice:1.43
    restart: unless-stopped
    command: ["uvicorn", "app.genaiflow_service:app", "--host", "0.0.0.0", "--port", "8889"]
    network_mode: "service:vantiq_edge"

  vantiq_edge_qdrant:
    container_name: vantiq_edge_qdrant
    image: qdrant/qdrant:v1.13.4
    restart: unless-stopped
    volumes:
      - qdrantData:/qdrant/storage
    networks:
      vantiq_edge:
        aliases: [edge-qdrant]

  vantiq_unstructured_api:
    container_name: vantiq_unstructured_api
    image: quay.io/vantiq/unstructured-api:0.0.82
    restart: unless-stopped
    environment:
      - PORT=18000
      - UNSTRUCTURED_PARALLEL_MODE_ENABLED=true
      - UNSTRUCTURED_PARALLEL_MODE_URL=http://localhost:18000/general/v0/general
      - UNSTRUCTURED_PARALLEL_MODE_SPLIT_SIZE=20
      - UNSTRUCTURED_PARALLEL_MODE_THREADS=4
      - UNSTRUCTURED_DOWNLOAD_THREADS=4
    network_mode: "service:vantiq_edge"

networks:
  vantiq_edge:
    ipam:
      config: []
volumes:
  vantiq_edge_data: {}
  qdrantData: {}
```

⚠️ **주의**

- MongoDB 계정/DB 이름 변경 금지
- SNAPSHOT 태그 사용 금지
- AI Assistant / GenAI Flow 버전은 Edge와 **Major.Minor 동일**

---

## 6. Docker 로그인

📍 **실행 위치:** 어디서든 가능

```bash
docker login quay.io
```

---

## 7. Vantiq Edge 기동

📍 **실행 위치:** `EDGE_HOME`

```bash
cd /opt/vantiq-edge
docker compose up -d
```

---

### 7.1 상태 확인

```bash
docker compose ps
```

---

### 7.2 로그 확인

```bash
docker compose logs -f vantiq_edge
```

전체 로그:

```bash
docker compose logs -f
```

---

## 8. 운영 명령어 정리

📍 **모두 `EDGE_HOME` 에서 실행**

### Edge 서버 재시작

```bash
docker compose restart vantiq_edg
```

### Edge 서버만 중지

```bash
docker compose stop vantiq_edge

```

### 전체 서비스 중지

```bash
docker compose down
```

> ⚠️ docker run 직접 사용 금지
> 
> 
> → 반드시 `docker compose` 사용
> 

---

## 9. 이미지 업데이트 (Patch)

📍 **실행 위치:** `EDGE_HOME`

```bash
docker compose pull
docker compose up -d
```

---

## 10. 업그레이드 (Qdrant Migration)

> 본 섹션은 Vantiq Edge 1.42 → 1.43 업그레이드를 기준으로 작성되었습니다.
> 
> 
> 해당 업그레이드는 **Qdrant 벡터 DB 마이그레이션이 필수**이며, **MongoDB 데이터는 유지**됩니다.
> 

---

### 10.1 업그레이드 개요 (반드시 읽기)

### 업그레이드 시 핵심 사항

- ✅ **Qdrant Migration은 필수**
- ✅ Migration은 **1회만 수행**
- ❌ MongoDB 데이터 삭제 없음
- ❌ `upgrade.sh` 실행 중 Edge 서버는 반드시 중지 상태
- ⚠️ `ALREADY_EXISTS` 메시지는 정상 동작

---

### 업그레이드 경로 제약

| 현재 버전 | 업그레이드 가능 경로 |
| --- | --- |
| 1.41 | ❌ 직접 1.43 불가 |
| 1.41 | ✅ 1.42 → 1.43 순차 업그레이드 |
| 1.42 | ✅ 1.43 가능 |

> ⚠️ 1.41 → 1.42 업그레이드 시에는
> 
> 
> `qdrant-migration:1.42.x` 이미지를 사용해야 합니다.
> 

---

### 10.2 업그레이드 전 체크리스트 (권장)

업그레이드 전에 반드시 확인하세요.

- [ ]  현재 Edge 서비스 정상 동작
- [ ]  MongoDB 백업 완료
- [ ]  Qdrant 데이터 중요 여부 확인
- [ ]  docker-compose.yml 백업
- [ ]  현재 이미지 버전 확인

```bash
cd /opt/vantiq-edge
docker compose ps
```

---

### 10.3 upgrade 디렉토리 생성

📍 **실행 위치:** `EDGE_HOME`

```bash
cd /opt/vantiq-edge
mkdir upgrade
cd upgrade
```

결과 구조:

```
/opt/vantiq-edge/upgrade
├─ mongoDbService.json
├─ vectorDbService.json
├─ io.vantiq.aimanager.AiManager.json
├─ upgrade.sh
```

---

### 10.4 Migration 설정 파일 생성

> ⚠️ 이 파일들은 Migration 컨테이너가 MongoDB / Qdrant에 접근하기 위해 필수입니다.
> 

---

### 10.4.1 MongoDB 설정

📄 `mongoDbService.json`

```json
{
  "hosts": [
    { "host": "vantiq_edge_mongo" }
  ]
}
```

- `host` 값은 **docker-compose.yml의 MongoDB 서비스명**
- 네트워크 alias 사용 가능

---

### 10.4.2 Qdrant Vector DB 설정

📄 `vectorDbService.json`

```json
{
  "service": {
    "hostname": "edge-qdrant"
  }
}
```

- `edge-qdrant`는 Qdrant 서비스의 network alias

---

### 10.4.3 AI Manager 설정

📄 `io.vantiq.aimanager.AiManager.json`

```json
{
  "config": {
    "semanticIndexService": {
      "vectorDB": {
        "host": "edge-qdrant"
      }
    }
  }
}
```

- Semantic Index가 사용하는 Vector DB 위치 정의
- Migration 시 **기존 인덱스 → 신규 컬렉션 변환에 사용**

---

### 10.5 Edge 서버 중지 (필수)

📍 **실행 위치:** `EDGE_HOME`

```bash
cd /opt/vantiq-edge
docker compose stop vantiq_edge

```

> ⚠️ 전체 스택을 내리지 말고
> 
> 
> ❗ **반드시 vantiq_edge 서비스만 중지**
> 

---

### 10.6 Migration 스크립트 생성

📄 `upgrade.sh`

📍 **실행 위치:** `/opt/vantiq-edge/upgrade`

```bash
vi upgrade.sh
```

```bash
#!/bin/bash

network_name=$(docker networkls --format'{{.Name}}' | grep'vantiq_edge')

docker run --rm \
  --name qdrant_migration_143 \
  --network$network_name \
  -v ./mongoDbService.json:/opt/vantiq/config/mongoDbService.json \
  -v ./vectorDbService.json:/opt/vantiq/config/vectorDbService.json \
  -v ./io.vantiq.aimanager.AiManager.json:/opt/vantiq/config/io.vantiq.aimanager.AiManager.json \
  quay.io/vantiq/qdrant-migration:1.43
```

---

### 실행 권한 부여

```bash
chmod +x upgrade.sh
```

---

### 10.7 Migration 실행

📍 **실행 위치:** `/opt/vantiq-edge/upgrade`

```bash
./upgrade.sh
```

### 실행 중 로그 예시 (정상)

```
ALREADY_EXISTS: Collection semantic_index already exists
```

> ✅ 동일 컬렉션이 여러 Semantic Index에서 사용될 경우 발생
> 
> 
> ❌ 에러 아님 / 무시 가능
> 

---

### 10.8 업그레이드 완료 후 서비스 재기동

📍 **실행 위치:** `EDGE_HOME`

```bash
cd /opt/vantiq-edge
docker compose down
```

---

### docker-compose.yml 이미지 버전 변경

```yaml
image:quay.io/vantiq/vantiq-edge:1.43
image:quay.io/vantiq/ai-assistant:1.43
image:quay.io/vantiq/genaiflowservice:1.43
```

> Major.Minor 반드시 동일
> 

---

### 서비스 재기동

```bash
docker compose up -d
```

---

### 10.9 업그레이드 후 검증

### 서비스 상태 확인

```bash
docker compose ps
```

---

### Edge 로그 확인

```bash
docker compose logs -f vantiq_edge
```

---

### IDE 접속 확인

```
http://<EDGE_HOST>:8080/ui/ide/index.html
```

- 기존 Namespace / Organization 유지 여부 확인
- Semantic Index 정상 조회 확인

---

### 10.10 업그레이드 후 정리

> Migration은 1회성 작업
> 

```bash
cd /opt/vantiq-edge
rm -rf upgrade
```

---

### 10.11 업그레이드 트러블슈팅

### Edge 기동 실패 시

- MongoDB / Qdrant 컨테이너 상태 확인
- 이미지 버전 불일치 여부 확인
- `docker compose logs -f` 전체 로그 확인

---

### Migration 컨테이너 실패 시

- 네트워크 이름 확인
- MongoDB / Qdrant 서비스명 불일치 여부
- 설정 JSON 파일 경로 확인

---

### 10.12 업그레이드 핵심 요약 (Notion Callout 추천)

> ✅ Qdrant Migration은 Edge 중지 상태에서 1회만 실행
> 
> 
> ✅ **ALREADY_EXISTS 로그는 정상**
> 
> ❌ **1.41 → 1.43 직접 업그레이드 불가**
> 

---

## 11. 로그 설정 (선택)

📍 **실행 위치:** `EDGE_HOME`

```bash
mkdir log
```

docker-compose.yml 에 추가:

```yaml
- ./config/logback.xml:/opt/vantiq/config/logback.xml
- ./log:/var/log/vantiq
```

적용:

```bash
docker compose restart vantiq_edge
```

---

## 12. IDE 접속 및 초기 설정

### IDE 접속

```
http://<EDGE_HOST>:8080/ui/ide/index.html
```

### 초기 계정

- ID: `system`
- PW: `fxtrt$1492`

---

### 필수 초기 설정 체크리스트

- [ ]  system 비밀번호 변경
- [ ]  LLM API Key 설정 (OPENAI_API_KEY 등)
- [ ]  Organization / Namespace 생성
- [ ]  GenAIFlowService Connector 생성 (Port 8889)
- [ ]  (선택) VideoAssistant Connector 생성 (Port 8890)

---

## 13. 백업 및 운영 관리

### MongoDB

- Edge 서버 중지 후 백업
- 외부 스토리지에 저장
- 복구 시 빈 `ars02` DB 필요

### Qdrant / Semantic Index

- Vantiq CLI `dump / load` 사용

---

## 14. 핵심 운영 원칙 (요약)

> ✅ 모든 docker compose 명령은 docker-compose.yml 이 있는 디렉토리에서 실행한다
> 

```bash
cd /opt/vantiq-edge
```

---

## 15. 지원 및 참고

- 📩 Vantiq Support: [support@vantiq.com](mailto:support@vantiq.com)
- 📚 Vantiq Docs: https://test.vantiq.com/docs
