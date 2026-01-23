# Vantiq CLI 연결 설정 가이드

---

## 1. 목적

Vantiq CLI(Command Line Interface)를 사용해

- 여러 Vantiq 서버(dev / customer / on-prem 등)에 연결하고
- 프로젝트, 타입, 룰, 소스 등 리소스를 CLI로 관리한다.

---

## 2. 사전 준비사항 (Prerequisites)

- **Java 11 이상 설치**
    
    ```bash
    java -version
    ```
    
- Vantiq CLI ZIP 파일 다운로드 후 압축 해제
- CLI 실행 파일이 PATH에 포함되어 있으면 편리함

---

## 3. Vantiq CLI 기본 구조

- CLI 실행 명령
    - Mac / Linux: `vantiq`
    - Windows: `vantiq.bat`
- 서버 연결 정보는 **profile 파일**로 관리
- 하나의 profile 파일에 **여러 서버 프로파일 정의 가능**

---

## 4. Profile 파일 위치

| OS | 위치 |
| --- | --- |
| Mac / Linux | `~/.vantiq/profile` |
| Windows | `%UserProfile%\.vantiq\profile` |

> ⚠️ 서버별로 파일을 나누지 않고
> 
> 
> **하나의 profile 파일에 여러 프로파일 블록을 정의하는 것이 정석**
> 

---

## 5. Profile 구성 방식

### 5.1 기본 개념

- **Vantiq 서버 1개 = 프로파일 1개**
- 프로파일에는 다음 정보 포함
    - 서버 URL
    - 인증 정보 (token 또는 username/password)

---

### 5.2 Profile 예시 (멀티 서버)

```
dev
{
    url = 'https://dev.vantiq.com'
    token = 'DEV_TOKEN'
}

customer
{
    url = 'https://vantiq.customer.com'
    token = 'CUSTOMER_TOKEN'
}

onprem
{
    url = 'https://vantiq.internal.local'
    username = 'admin'
    password = 'admin123'
}

```

- 프로파일 이름: `dev`, `customer`, `onprem` (짧고 명확하게)
- 인증 방식 혼합 가능 (token / 계정)

---

## 6. CLI 연결 확인 방법

### 6.1 기본 연결 테스트

```bash
vantiq list projects

```

- 성공 → 인증 및 통신 정상
- 실패 → profile 또는 인증 정보 점검 필요

---

### 6.2 특정 프로파일 지정

```bash
vantiq -s dev list projects
vantiq -s customer list types

```

- `s` 옵션 = 사용할 프로파일 선택

---

## 7. 연결 성공 판단 기준

아래 중 하나라도 정상 출력되면 **CLI 연결 완료**

```bash
vantiq list projects
vantiq list types
vantiq list rules

```

---

## 8. 실무 권장 프로파일 구성 패턴

| 프로파일명 | 용도 |
| --- | --- |
| dev | dev.vantiq.com |
| staging | 사내 테스트 서버 |
| prod | 고객 운영 서버 |
| onprem | 폐쇄망 / 내부망 |

---

## 9. 인증 방식 선택 가이드

| 상황 | 권장 |
| --- | --- |
| 개인 개발 / PoC | OAuth Token |
| 고객사 운영 | Token |
| 폐쇄망 / 내부 | username/password 또는 내부 토큰 |

> ✅ Token 방식 권장
> 
> - 비밀번호 변경 영향 없음
> - 자동화 / Git 연계에 안정적

---

## 10. 자주 발생하는 연결 이슈

| 증상 | 원인 |
| --- | --- |
| 401 Unauthorized | 토큰 만료 |
| profile not found | 파일 위치 또는 이름 오류 |
| list는 되나 export 실패 | 권한 부족 |
| on-prem 접속 불가 | DNS / TLS / 방화벽 |

---

## 11. 운영 팁 (Best Practice)

- 서버별 프로파일 명칭 규칙 통일
- Token 만료 시 주기적으로 갱신
- CLI + Git 연동 시 **프로파일 이름 고정**
- 운영(prod) 서버는 읽기/쓰기 권한 분리 권장

---
