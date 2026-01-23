# 🔗 Vantiq - Ollama(Llama 3.1) 로컬 연동 가이드 (최종본)

이 문서는 로컬 Docker 환경의 **Vantiq** 서버와 호스트 PC에서 실행 중인 **Ollama**를 연결하는 표준 절차를 안내합니다. 이 설정은 Vantiq의 OpenAI Connector 표준을 따릅니다.

---

## 1. Ollama 사전 설정 (네트워크 개방)

Ollama는 기본적으로 로컬 접속만 허용하므로, Docker 컨테이너인 Vantiq이 접근할 수 있도록 네트워크 설정을 변경해야 합니다.

1. **Ollama 앱 종료:** 메뉴바에서 Ollama를 완전히 종료합니다.
2. **환경 변수 설정 후 재실행:** 터미널에서 아래 명령어를 입력합니다.Bash
    
    `export OLLAMA_HOST=0.0.0.0
    ollama serve`
    
    - `0.0.0.0`으로 설정해야 `host.docker.internal` 또는 외부 IP를 통한 접속이 허용됩니다.

---

## 2. Vantiq LLM 리소스 설정

Vantiq IDE에서 새로운 **LLM 리소스**를 생성하고 다음과 같이 구성합니다. 공식 문서의 OpenAI 설정 방식을 차용합니다.

### 주요 설정 값 (Configuration)

| **항목** | **설정값** | **설명** |
| --- | --- | --- |
| **LLM Name** | `MyLocalLlama` | 서비스에서 호출할 고유 이름 |
| **LLM Type** | `Generative` | 생성형 AI 모델 타입 |
| **Model Name** | `llama3.1:latest` | `ollama list` 명령어로 확인되는 정확한 모델명 |
| **Configuration** | `{"openai_api_base":"http://host.docker.internal:11434/v1"}` | 호스트 PC의 Ollama API 엔드포인트 |
| **API Key Secret** | *(더미 데이터가 포함된 Secret)* | 필수값이므로 `ollama` 등의 값을 가진 Secret 선택 |

> [!NOTE]
> 
> 
> **왜 `host.docker.internal`인가요?**
> 
> Vantiq이 Docker 컨테이너 내에서 실행될 때, `localhost`는 컨테이너 자신을 가리킵니다. 호스트 PC(맥북)의 자원에 접근하려면 Docker에서 제공하는 특수 도메인인 `host.docker.internal`을 사용하는 것이 가장 안정적입니다.
> 

---

## 3. VAIL 코드를 이용한 검증

연결이 정상적인지 확인하기 위해 아래 코드를 Procedure에서 실행해 봅니다.

```jsx
PROCEDURE testOllamaConnection()
    var prompt = "Vantiq과 Ollama 연동 성공 메시지를 한 문장으로 작성해줘."
    
    try {
        var response = LLM.generate("MyLocalLlama", {
            messages: [
                { "role": "user", "content": prompt }
            ]
        })
        log.info("Ollama 응답: " + response.choices[0].message.content)
    } catch(e) {
        log.error("연결 실패: " + e)
    }
```

---

## 4. 트러블슈팅 및 참고 사항

- **404 에러:** `http://localhost:11434/v1` 접속 시 404가 뜨는 것은 정상입니다. 해당 경로는 API 전용이므로 `.../v1/models`로 접속하여 실제 모델 목록이 나오는지 확인해야 합니다.
- **API Key 필수:** Vantiq 내부 커넥터 규격상 API Key가 없으면 실행 시 에러가 발생합니다. Ollama가 키를 검사하지 않더라도 Vantiq Secret에는 아무 값이나 채워져 있어야 합니다.
- **공식 문서 참조:** 더 자세한 파라미터 설정은 [Vantiq 공식 LLM 문서(OpenAI)](https://dev.vantiq.com/docs/system/llms/#openai)를 확인하세요.