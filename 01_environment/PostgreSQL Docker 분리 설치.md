# PostgreSQL Docker 분리 설치

# 📁 1단계. PostgreSQL 전용 폴더 생성

```bash
cd ~
mkdir postgres
cd postgres
```

데이터 저장용 폴더 생성:

```bash
mkdir data
```

현재 구조:

```
postgres/
├─ data/
```

---

# 📝 2단계. docker-compose.yml 생성

```bash
nano docker-compose.yml
```

아래 내용을 **그대로 붙여넣기** 👇

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:15
    container_name: postgres-biz
    restart: unless-stopped
    environment:
      POSTGRES_DB: bizdb
      POSTGRES_USER: bizuser
      POSTGRES_PASSWORD: bizpass
    volumes:
      - ./data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
```

저장:

- `Ctrl + O` → Enter
- `Ctrl + X`

---

# 🚀 3단계. PostgreSQL 컨테이너 실행

```bash
docker compose up -d
```

---

# 🔍 4단계. 정상 실행 확인

### 4-1. 컨테이너 상태

```bash
docker ps
```

정상 예:

```
postgres-biz   postgres:15   Up ...
```

---

### 4-2. 로그 확인

```bash
docker compose logs -f
```

아래 메시지가 보이면 성공:

```
database system is ready to accept connections
```

`Ctrl + C`로 종료

---

# 🔐 5단계. DB 접속 테스트 (중요)

```bash
docker exec -it postgres-biz psql -U bizuser -d bizd
```

프롬프트가 나오면 성공:

```sql
SELECT version();
\q
```

---

# 📦 6단계. 최종 상태 정리

지금 상태는 이거야:

```
/home/ubuntu/postgres        ✅ PostgreSQL 전용 폴더
└─ data/                     ✅ DB 데이터 영구 저장
```

컨테이너:

```
postgres-biz (PostgreSQL)    ✅ 실행 중
```

---

# 🔗 7단계. (미리) Vantiq / JDBC Source 연결 정보

나중에 쓸 정보:

### 외부 / 다른 컨테이너에서

```
Host: <서버 IP>
Port: 5432
DB: bizdb
User: bizuser
Password: bizpass
```

### 같은 Docker 네트워크일 경우

```
jdbc:postgresql://postgres-biz:5432/bizdb
```

---

# ⚠️ 문제 생길 때 바로 확인할 것

### 🔸 포트 충돌

```
Error: bind: address already in use
```

➡ 포트 변경:

```yaml
ports:
  - "5433:5432"
```

---

### 🔸 권한 오류 (Linux)

```bash
sudo chmod -R 777 data
```

---

---

# 🧩 1단계. PostgreSQL 컨테이너 접속

```bash
docker exec -it postgres-biz psql -U bizuser -d bizdb
```

정상 접속되면 이런 프롬프트가 나와:

```
bizdb=#
```

---

# 🧱 2단계. 테스트 테이블 생성

아래 SQL을 그대로 실행 👇

```sql
CREATE TABLE test_device (
  id SERIAL PRIMARY KEY,
  device_id VARCHAR(50) NOT NULL,
  temperature NUMERIC(5,2),
  status VARCHAR(20),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

성공하면:

```
CREATE TABLE
```

---

# 🔍 3단계. 테이블 생성 확인

```sql
\d test_device
```

또는

```sql
SELECT * FROM information_schema.tables
WHERE table_name = 'test_device';
```

---

# ✏️ 4단계. 테스트 데이터 입력

```sql
INSERT INTO test_device (device_id, temperature, status)
VALUES
  ('sensor-001', 23.5, 'NORMAL'),
  ('sensor-002', 31.2, 'WARNING'),
  ('sensor-003', 45.8, 'CRITICAL');
```

결과:

```
INSERT 0 3
```

---

# 📊 5단계. 데이터 조회

```sql
SELECT * FROM test_device;
```

예시 결과:

```
 id | device_id  | temperature |  status   |        created_at
----+------------+-------------+-----------+----------------------------
  1 | sensor-001 |       23.50 | NORMAL    | 2026-01-06 07:15:23
  2 | sensor-002 |       31.20 | WARNING   | 2026-01-06 07:15:23
  3 | sensor-003 |       45.80 | CRITICAL  | 2026-01-06 07:15:23
```

---

# 🧪 6단계. Vantiq 연동 테스트용 쿼리 (미리)

나중에 JDBC Source에서 바로 쓰기 좋은 쿼리 예시야.

### 🔹 전체 조회

```sql
SELECT * FROM test_device;
```

### 🔹 상태 조건 조회

```sql
SELECT * FROM test_device WHERE status = 'CRITICAL';
```

### 🔹 최신 데이터

```sql
SELECT * FROM test_device ORDER BY created_at DESC LIMIT 1;
```

---

# 🚪 7단계. PostgreSQL 종료
