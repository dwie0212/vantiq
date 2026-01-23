# 📘 JDBC Source Extension 완전 가이드

---

## 목차

1. 사전 요구사항
2. 저장소 클론
3. JDBC 드라이버 준비 (중요 ⭐)
4. 프로젝트 빌드
5. 로컬에서 직접 실행
6. Docker 이미지 빌드
7. Docker / Docker Compose 실행
8. Vantiq IDE 설정 (반드시 필요)
9. 자주 발생하는 오류 & 해결
10. 전체 흐름 요약

---

## 0. 전체 아키텍처 개요

```
┌──────────────┐        WebSocket        ┌────────────────┐
│ Vantiq Edge  │ <────────────────────> │ JDBCSource    │
│ (Docker)     │                         │ (Docker)       │
└──────────────┘                         └────────────────┘
                                                  │
                                                  │ JDBC
                                                  ▼
                                         ┌────────────────┐
                                         │ PostgreSQL     │
                                         │ (Docker)       │
                                         └────────────────┘
```

- **Vantiq Edge**: Source 관리 및 설정
- **JDBC Source**: Vantiq Extension Source (Java 기반)
- **PostgreSQL**: 대상 DB
- 세 컨테이너는 **동일 Docker network**에 존재해야 함

---

## 1. 사전 요구사항

### 필수

- **Java 11 이상** (빌드용)
- **Docker**
- **Docker Compose**
- **Git**

### DB 사용 시

- PostgreSQL / MySQL 등 실제 DB
- **해당 DB의 JDBC Driver JAR**

> ⚠️ JDBC Source는 JDBC Driver를 자동으로 포함하지 않는다
> 
> 
> → 반드시 **직접 다운로드 + 빌드 시 포함**해야 함
> 

---

## 2. 저장소 클론

```bash
git clone https://github.com/Vantiq/vantiq-extension-sources.git
cd vantiq-extension-sources
```

구조 확인:

```
vantiq-extension-sources
 ├─ jdbcSource
 ├─ extjsdk
 └─ ...
```

---

## 3. JDBC 드라이버 준비 ⭐ (가장 중요)

### 3.1 PostgreSQL JDBC Driver 다운로드

```bash
cd jdbcSource
mkdir -p drivers
cd drivers
wget https://jdbc.postgresql.org/download/postgresql-42.7.3.jar
```

결과:

```
jdbcSource/drivers/postgresql-42.7.3.jar
```

---

### 3.2 환경 변수 설정 (빌드에 필수)

> 이 단계가 빠지면 No suitable driver 에러 발생
> 

```bash
export JDBC_DRIVER_LOC=$(pwd)/postgresql-42.7.3.jar
```

확인:

```bash
echo $JDBC_DRIVER_LOC
```

---

## 4. 프로젝트 빌드

### 4.1 Gradle 빌드

```bash
cd ../../   # vantiq-extension-sources 루트
./gradlew jdbcSource:assemble
```

---

### 4.2 빌드 결과 확인

```bash
ls jdbcSource/build/distributions
```

출력 예:

```
jdbcSource.tar
jdbcSource.zip
```

드라이버 포함 여부 확인:

```bash
tar -tf jdbcSource/build/distributions/jdbcSource.tar | grep postgresql
```

---

## 5. 로컬에서 직접 실행 (Docker 없이)

### 5.1 server.config 생성

```bash
cd jdbcSource

cat > server.config << 'EOF'
targetServer=http://vantiq_edge_server:8080
authToken=YOUR_VANTIQ_AUTH_TOKEN
sources=PostgresSource
sendPings=true
EOF
```

---

### 5.2 실행 파일 준비

```bash
cd build/distributions
tar -xf jdbcSource.tar
cd jdbcSource
```

### 5.3 실행

```bash
./bin/jdbcSource
```

또는 명시적으로:

```bash
./bin/jdbcSource ../../server.config
```

---

## 6. Docker 이미지 빌드

### 6.1 tar 파일을 Docker 빌드 컨텍스트로 복사

```bash
cd jdbcSource
cp build/distributions/jdbcSource.tar .
```

---

### 6.2 Docker 이미지 빌드

```bash
docker build \
  -t vantiq-jdbc-source:1.1 \
  -f src/main/docker/Dockerfile .
```

확인:

```bash
docker images | grep vantiq-jdbc-source
```

---

## 7. Docker / Docker Compose 실행

### 7.1 Docker Network 준비

> Vantiq 서버, JDBC Source, DB는 같은 네트워크여야 함
> 

```bash
docker network create vantiq-net
```

---

### 7.2 Docker로 단독 실행

```bash
docker run -d \
  --name vantiq_jdbc_source \
  --network vantiq-net \
  -v $(pwd)/server.config:/app/server.config:ro \
  vantiq-jdbc-source:1.1
```

---

### 7.3 Docker Compose 실행 (권장)

### docker-compose.yml

```yaml
services:
  jdbc-source:
    image: vantiq-jdbc-source:1.1
    container_name: vantiq_jdbc_source
    restart: unless-stopped
    volumes:
      - ./server.config:/app/server.config:ro
    networks:
      - vantiq-net

networks:
  vantiq-net:
    external: true
```

실행:

```bash
docker compose up -d
docker compose logs -f
```

---

## 8. Vantiq IDE 설정 (필수)

> ❗ JDBC Source 컨테이너만 실행하면 절대 연결되지 않는다
> 
> 
> → **IDE에서 Source Type + Source 생성이 먼저**
> 

---

### 8.1 Source Type 등록

```bash
vantiq -s <profile> load sourceimpls \
  jdbcSource/src/test/resources/jdbcImpl.json
```

IDE에서 **Type = JDBC** 확인

---

### 8.2 Source 생성

- Name: `PostgresSource`
- Type: `JDBC`

Configuration 예시:

```json
{
  "vantiq": {
    "packageRows": true
  },
  "jdbcConfig": {
    "general": {
      "username": "bizuser",
      "password": "bizpass",
      "dbURL": "jdbc:postgresql://postgres-biz:5432/bizdb",
      "asynchronousProcessing": true,
      "maxActiveTasks": 10,
      "maxQueuedTasks": 20
    }
  }
}
```

> ⚠️ postgres-biz는 Docker 컨테이너 이름
> 

---

## 9. 자주 터지는 오류 & 해결

### ❌ No suitable driver

- `JDBC_DRIVER_LOC` 미설정
- 빌드 다시 필요

```bash
export JDBC_DRIVER_LOC=...
./gradlew jdbcSource:clean jdbcSource:assemble
```

---

### ❌ sourceNotFound

- IDE에서 Source 미생성
- 이름 불일치 (`sources=PostgresSource`)

---

### ❌ DB 접속 안 됨

- Docker 네트워크 확인
- 컨테이너 이름으로 접속해야 함

```bash
nc -zv postgres-biz 5432
```

---

## 10. 전체 흐름 요약

```
GitHub Clone
  ↓
JDBC Driver 준비 (⭐)
  ↓
Gradle 빌드
  ↓
(server.config)
  ↓
Docker 이미지 생성
  ↓
Docker / Compose 실행
  ↓
Vantiq IDE Source Type 등록
  ↓
Vantiq IDE Source 생성
  ↓
연결 성공 🎉
```

---

## 참고 자료

- https://github.com/Vantiq/vantiq-extension-sources
- https://github.com/Vantiq/vantiq-extension-sources/blob/master/extjsdk/README.md
