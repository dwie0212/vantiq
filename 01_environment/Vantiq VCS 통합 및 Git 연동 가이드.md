# 🚀 Vantiq VCS 통합 및 Git 연동 가이드

## 📌 목차

- Vantiq CLI 스크립트 상세 분석
- VCSSERVER: IDE와 로컬의 가교
- 멀티 환경을 위한 프로파일 설정
- GitLab/GitHub 연동 워크플로우
- 인증 방식 상세: SSH vs HTTPS
- Git Remote 및 충돌 관리

---

## ## Vantiq CLI 스크립트 상세 분석

### **1. 파일 위치 및 개요**

- **경로**: `/Vantiq/dev/vantiq-1.43.4/bin/vantiq`
- **성격**: Gradle이 생성한 POSIX 호환 셸 스크립트로, Apache License 2.0을 따릅니다.

### **2. 스크립트 주요 구성 요소**

- **APP_HOME 설정 (67-89줄)**: 스크립트의 실제 위치를 추적하여 `APP_HOME` 경로를 결정하며, 심볼릭 링크 처리 로직을 포함합니다.
- **유틸리티 함수 (94-103줄)**:
    - `warn()`: 사용자에게 경고 메시지 출력.
    - `die()`: 치명적 에러 발생 시 메시지 출력 후 스크립트 종료.
- **OS 감지 및 최적화 (105-115줄)**: `darwin`(macOS), `cygwin`, `msys` 등을 감지하여 OS별 경로 변환 및 플래그를 설정합니다.
- **CLASSPATH 설정 (117줄)**: Vantiq CLI 실행에 필요한 모든 JAR 라이브러리를 클래스패스에 추가합니다.
- **인자 전달 및 실행 (213-216줄, 250줄)**:
    - `set --`: 모든 명령줄 인자를 Java 메인 클래스(`io.vantiq.cli.Vantiq`)로 전달하도록 재구성합니다.
    - `exec "$JAVACMD"`: 최종적으로 JVM을 실행하여 Vantiq 명령어를 수행합니다.
- **JVM 옵션 관리 (243-248줄)**: `JAVA_OPTS`나 `VANTIQ_OPTS` 환경변수를 읽어 JVM 실행 옵션을 병합합니다.

---

## ## VCSSERVER: IDE와 로컬의 가교

### **1. 개요**

`vantiq VCSSERVER`는 브라우저 기반의 Vantiq IDE(Modelo)가 로컬 파일 시스템에 접근할 수 있도록 돕는 **로컬 서버**를 구동합니다.

### **2. 실행 및 동작**

- **명령어**: `./bin/vantiq VCSSERVER`.
- **동작 프로세스**:
    1. **서버 시작**: `localhost:4347` 포트 점유.
    2. **활성화 확인**: `Server active (1.43.4)` 메시지 출력 확인.
    3. **브릿지 역할**: IDE의 동기화 요청을 받아 로컬 디렉토리의 파일을 쓰고 읽습니다.

### **3. 주요 IDE 기능**

- **Sync Project to VCS**: 현재 IDE 프로젝트 상태를 로컬 디렉토리로 내보냅니다.
- **Sync Project from VCS**: 로컬 디렉토리 파일로 IDE 프로젝트를 덮어씁니다 (작업 되돌리기).
- **Combine Project from VCS**: 다른 경로의 프로젝트 리소스를 현재 프로젝트에 병합합니다.

---

## ## 멀티 환경을 위한 프로파일 설정

### **1. 프로파일 정의**

여러 서버 환경(개발, 운영, 개인용)의 URL과 접속 토큰을 편리하게 전환하며 사용하기 위한 설정입니다.

### **2. 설정 파일 위치**

- **Mac/Linux**: `~/.vantiq/profile`
- **Windows**: `%UserProfile%\.vantiq\profile`

### **3. 설정 예시 (Groovy 형식)**

```jsx
base {
    url = 'https://dev.vantiq.com'
    token = 'your-token-here'
}
personal {
    url = 'https://personal.vantiq.com'
    token = 'personal-token'
}
```

### **4. 프로파일 사용법**

명령어 실행 시 `-s` 옵션으로 사용할 프로파일을 지정합니다. 스크립트 내 `"$@"` 인자 전달 로직 덕분에 가능합니다.

- `vantiq -s personal VCSSERVER`

---

## ## GitLab/GitHub 연동 워크플로우

Vantiq은 파일 동기화만 담당하므로, 실제 버전 기록 및 원격 저장은 Git을 통해 별도로 수행해야 합니다.

1. **서버 실행**: `vantiq VCSSERVER` 실행.
2. **IDE 동기화**: `Sync Project to VCS` 클릭하여 로컬 폴더로 파일 내보내기.
3. **Git 초기화**: 해당 폴더에서 `git init` 수행.
4. **원격 연결**: GitLab/GitHub 저장소 URL 연결 
    - `git remote add origin git@gitlab.com:your-username/your-project.git`).
5. **버전 기록 및 업로드**:
    - `git add .`
    - `git commit -m "Commit Message"`
    - `git push -u origin main`

---

## ## 인증 방식 상세: SSH vs HTTPS

### **1. SSH 방식 (권장)**

- **특징**: 키 기반 인증으로 보안성이 높고 자동 로그인 가능.
- **설정 단계**:
    1. 키 생성: `ssh-keygen -t ed25519 -C "email@example.com"`
    2. 키 복사: `pbcopy < ~/.ssh/id_ed25519.pub`
    3. GitLab 등록: Settings > SSH Keys에 붙여넣기.
    4. 연결 테스트: `ssh -T git@gitlab.com`

### **2. HTTPS 방식**

- **특징**: 설정은 쉬우나 매번 인증 필요. GitLab은 **Personal Access Token** 사용 필수.
- **설정 단계**:
    1. 토큰 생성: GitLab > Access Tokens에서 `write_repository` 권한으로 생성.
    2. 푸시 인증: 비밀번호 입력란에 **생성한 토큰**을 입력.

### **3. 방식 비교 및 전환**

| **항목** | **SSH** | **HTTPS** |
| --- | --- | --- |
| 초기 설정 | 복잡 (키 등록 필요) | 간단 (URL 입력) |
| 사용 편의성 | 매우 편리 (인증 자동) | 매번 토큰 입력 필요 |
| **방식 변경** | `git remote set-url origin git@...` | `git remote set-url origin https://...` |

---

## ## Git Remote 개념 및 충돌 관리

### **1. Remote 저장소의 독립성**

- 각 로컬 디렉토리(`.git` 폴더 기준)는 **개별적인 원격 저장소 설정**을 가집니다.
- Vantiq 프로젝트 A와 프로젝트 B를 서로 다른 GitLab 레포지토리에 연결해도 아무런 간섭이 발생하지 않습니다.

### **2. 충돌 방지 및 해결**

- **상황**: 원격 저장소에 이미 파일이 있는 경우.
- **해결**:
    1. `git fetch origin`으로 변경 사항 확인.
    2. `git pull origin main`으로 병합 후 로컬에서 충돌 해결.
    3. Vantiq IDE에서 `Sync Project from VCS`를 눌러 병합된 최종 상태를 IDE에 반영.
