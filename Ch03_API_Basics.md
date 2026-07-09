# Chapter 3. API 연동 기초 (Java + REST)

[← 목차로](README.md)

> **학습 목표**
> - Foundry 프로젝트의 엔드포인트와 인증 구조를 이해한다.
> - JDK 21 기반의 Maven 프로젝트를 설정하고 필요한 의존성을 추가한다.
> - Native Azure SDK와 LangChain4j를 각각 사용하여 GPT-5 모델과 대화하는 코드를 작성한다.
> - REST API(curl)를 통해 SDK 없이 모델을 호출하는 방법을 익힌다.

> **전제 조건**
> - Chapter 2에서 GPT-5 모델 배포 완료
> - JDK 21 및 Apache Maven 설치 완료
> - 배포된 모델의 Endpoint 및 API Key 확보

---

## 1. API 경로와 라우팅 구조

Foundry 프로젝트를 생성하고 모델을 배포하면 호출 가능한 엔드포인트가 생성된다. 2026년 현재 Foundry는 더 단순하고 일관된 라우팅 체계를 제공한다.

### 1.1 주요 API 경로 (Stable Routes)

🔴 **2026 변경** : 과거에는 `api-version=2024-02-15-preview`와 같이 모든 요청에 날짜 기반의 쿼리 파라미터를 명시해야 했다. 현재는 `/openai/v1/`로 시작하는 고정된(stable) 경로를 통해 버전 관리의 복잡성을 줄였다.

- **Chat Completions**: `/openai/v1/chat/completions`
  - 가장 일반적으로 사용되는 대화형 API이다. GPT-5 계열 모델과의 텍스트 기반 상호작용에 사용된다.
- **Responses API**: `/openai/v1/responses`
  - 🔴 **2026 변경** : 기존 Assistants API(Threads, Messages 등)를 대체하는 차세대 에이전트 인터페이스이다. 멀티모달 처리 및 추론 모델(o-series) 연동 시 권장된다.
- **Embeddings**: `/openai/v1/embeddings`
  - 텍스트를 벡터로 변환하여 검색(RAG) 엔진에 입력하기 위한 API이다.

### 1.2 엔드포인트 구성
Foundry의 엔드포인트는 보통 다음과 같은 형식을 가진다.
`https://<your-foundry-resource-name>.openai.azure.com/`
이 기본 주소 뒤에 위에서 언급한 API 경로를 붙여 사용한다.

---

## 2. 두 가지 인증 방식

Foundry는 보안 수준과 운영 환경에 따라 두 가지 인증 방식을 제공한다.

### 2.1 API Key 방식
- **헤더**: `api-key: <YOUR_KEY>`
- **특징**: 로컬 개발이나 빠른 프로토타이핑에 적합하다. Foundry 포털의 'Project settings'에서 키를 확인할 수 있다.
- **주의**: 소스 코드에 키를 직접 입력하지 말고 반드시 환경변수를 사용한다.

### 2.2 Entra ID (전 Azure AD) 방식
- **헤더**: `Authorization: Bearer <TOKEN>`
- **특징**: 프로덕션 환경에서 필수적으로 사용된다. API Key 노출 위험이 없으며, Role-Based Access Control(RBAC)을 통해 세밀한 권한 제어가 가능하다.
- **구현**: Java SDK에서는 `DefaultAzureCredential` 클래스를 사용하여 로컬의 `az login` 정보나 서버의 Managed Identity로부터 자동으로 토큰을 획득한다.

⚠️ **함정** : `Bearer` 토큰은 보안상 약 60분 후에 만료된다. Azure SDK는 이를 자동으로 갱신(Refresh)해주지만, 로컬에서 개발 중인 경우 세션이 만료되면 `az login`을 다시 실행해야 할 수도 있다.

---

## 🔧 실습 1 — Maven 프로젝트 셋업

본격적인 코딩 전, JDK 21 환경에서 Maven 프로젝트를 구성한다.

### Directory 구조
```text
foundry-app/
├── pom.xml
└── src/
    └── main/
        └── java/
            └── com/
                └── example/
                    └── App.java
```

### pom.xml 설정
핵심 라이브러리 버전을 정확히 명시하는 것이 중요하다.

<!-- pom.xml -->
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>foundry-app</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <azure.ai.projects.version>2.1.0</azure.ai.projects.version>
        <azure.identity.version>1.18.4</azure.identity.version>
        <azure.ai.openai.version>1.0.0-beta.16</azure.ai.openai.version>
        <langchain4j.version>1.17.0</langchain4j.version>
    </properties>

    <dependencies>
        <!-- Foundry Project Client -->
        <dependency>
            <groupId>com.azure</groupId>
            <artifactId>azure-ai-projects</artifactId>
            <version>${azure.ai.projects.version}</version>
        </dependency>

        <!-- Azure Identity for Auth -->
        <dependency>
            <groupId>com.azure</groupId>
            <artifactId>azure-identity</artifactId>
            <version>${azure.identity.version}</version>
        </dependency>

        <!-- OpenAI SDK (Beta) -->
        <dependency>
            <groupId>com.azure</groupId>
            <artifactId>azure-ai-openai</artifactId>
            <version>${azure.ai.openai.version}</version>
        </dependency>

        <!-- LangChain4j Azure Integration -->
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-azure-open-ai</artifactId>
            <version>${langchain4j.version}</version>
        </dependency>

        <!-- Logging -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-simple</artifactId>
            <version>2.0.13</version>
        </dependency>
    </dependencies>
</project>
```

⚠️ **함정** : `azure-ai-openai` 라이브러리는 현재 BETA 상태이다. Maven dependency 설정 시 반드시 `<version>1.0.0-beta.16</version>`을 명시해야 한다. `LATEST`나 `RELEASE` 태그를 사용하면 호환되지 않는 상위 버전이 로드되어 런타임 에러가 발생할 가능성이 높다.

### 환경변수 설정
터미널(또는 시스템 환경변수)에서 다음 값을 설정한다.

```powershell
# Windows PowerShell
$env:AZURE_FOUNDRY_ENDPOINT = "https://your-foundry.openai.azure.com/"
$env:AZURE_FOUNDRY_KEY = "your-api-key-here"
$env:AZURE_FOUNDRY_DEPLOYMENT = "gpt-5-mini" # 본인이 배포한 이름
```

---

## 🔧 실습 2 — Native Azure SDK로 Hello Foundry

가장 기본적인 방식인 Azure AI SDK를 사용하여 GPT-5와 첫 인사를 나눈다. `com.azure.ai.projects` 라이브러리는 Foundry 프로젝트 전체를 관리하며, 내부적으로 `OpenAIClient`를 생성해 대화를 수행한다.

```java
// App.java
package com.example;

import com.azure.ai.projects.AIProjectClient;
import com.azure.ai.projects.AIProjectClientBuilder;
import com.azure.ai.openai.models.ChatCompletions;
import com.azure.ai.openai.models.ChatCompletionsOptions;
import com.azure.ai.openai.models.ChatRequestMessage;
import com.azure.ai.openai.models.ChatRequestUserMessage;
import com.azure.ai.openai.OpenAIClient;
import com.azure.core.exception.HttpResponseException;
import com.azure.core.credential.AzureKeyCredential;

import java.util.ArrayList;
import java.util.List;

public class App {
    public static void main(String[] args) {
        // 환경 변수 로드
        String endpoint = System.getenv("AZURE_FOUNDRY_ENDPOINT");
        String key = System.getenv("AZURE_FOUNDRY_KEY");
        String deployment = System.getenv("AZURE_FOUNDRY_DEPLOYMENT");

        if (endpoint == null || key == null || deployment == null) {
            System.err.println("환경변수가 설정되지 않았음.");
            return;
        }

        try {
            // 1. 클라이언트 빌드 (API Key 방식)
            // 실제 운영 환경이라면 .credential(new DefaultAzureCredentialBuilder().build()) 권장
            OpenAIClient client = new AIProjectClientBuilder()
                .endpoint(endpoint)
                .credential(new AzureKeyCredential(key))
                .buildOpenAIClient();

            // 2. 대화 메시지 구성
            List<ChatRequestMessage> messages = new ArrayList<>();
            messages.add(new ChatRequestUserMessage("안녕 Foundry! Java로 보내는 첫 메시지야."));

            // 3. 요청 옵션 설정
            ChatCompletionsOptions options = new ChatCompletionsOptions(messages);
            
            // 🔴 2026 변경: GPT-5 계열은 maxTokens 대신 maxCompletionTokens 사용 권장
            options.setMaxCompletionTokens(500);
            
            // 🔴 2026 변경: GPT-5 모델은 temperature(1.0) 고정을 기본으로 함
            options.setTemperature(1.0);

            // 4. API 호출
            System.out.println("응답 대기 중...");
            ChatCompletions completions = client.getChatCompletions(deployment, options);

            // 5. 결과 출력
            String reply = completions.getChoices().get(0).getMessage().getContent();
            System.out.println("Foundry 응답: " + reply);

        } catch (HttpResponseException e) {
            System.err.println("HTTP 에러 발생: " + e.getResponse().getStatusCode());
            System.err.println("메시지: " + e.getMessage());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

⚠️ **함정** : `AzureKeyCredential` 인스턴스를 생성할 때 인자로 넣어야 하는 것은 API Key 문자열이다. 엔드포인트 URL을 넣지 않도록 주의한다. 또한, 배포 이름(`deployment`)과 모델 이름(`model`)을 혼동하지 말아야 한다. 코드에서 호출 시 사용하는 것은 Foundry 포털에서 설정한 'Deployment name'이다.

⚠️ **함정** : GPT-5 계열 모델은 응답 길이를 제한할 때 `maxTokens()` 대신 `maxCompletionTokens()`라는 새로운 메서드 이름을 사용한다. 기존 라이브러리 문법을 그대로 사용하면 의도한 대로 토큰 제한이 걸리지 않을 수 있다.

---

## 🔧 실습 3 — LangChain4j로 Hello Foundry

복잡한 애플리케이션을 개발할 때는 Native SDK보다 추상화 수준이 높은 **LangChain4j**를 사용하는 것이 생산성 면에서 유리하다. LangChain4j는 체인 구성, 메모리 관리, 도구 호출 등을 표준화된 인터페이스로 제공한다.

```java
// App.java (LangChain4j version)
package com.example;

import dev.langchain4j.model.azure.AzureOpenAiChatModel;
import dev.langchain4j.model.chat.ChatLanguageModel;

public class App {
    public static void main(String[] args) {
        String endpoint = System.getenv("AZURE_FOUNDRY_ENDPOINT");
        String key = System.getenv("AZURE_FOUNDRY_KEY");
        String deployment = System.getenv("AZURE_FOUNDRY_DEPLOYMENT");

        // LangChain4j는 빌더 패턴으로 매우 직관적인 설정 제공
        ChatLanguageModel model = AzureOpenAiChatModel.builder()
            .endpoint(endpoint)
            .apiKey(key)
            .deploymentName(deployment)
            .maxRetries(3) // 자동 재시도 내장
            .logRequests(true) // 요청 로그 출력
            .logResponses(true) // 응답 로그 출력
            .build();

        System.out.println("LangChain4j로 질문 전송...");
        String answer = model.generate("Foundry와 LangChain4j의 조합은 어때?");
        
        System.out.println("응답: " + answer);
    }
}
```

💡 **팁** : 왜 LangChain4j인가? 
Native SDK는 Azure 전용 기능(예: Content Safety 상세 설정)을 제어할 때 강력하지만, 코드가 다소 장황하다. LangChain4j는 "질문을 던지고 문자열을 받는다"는 행위를 단 한 줄의 `generate()` 메서드로 압축해준다. 또한 향후 모델을 Anthropic이나 로컬 LLM으로 교체해야 할 때 인터페이스 호환성이 뛰어나다.

---

## REST curl 등가

SDK가 없는 환경이나 디버깅 시에는 직접 HTTP 요청을 보낼 수 있다.

### curl (Bash/Linux/macOS)
```bash
curl https://${AZURE_FOUNDRY_ENDPOINT}/openai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "api-key: ${AZURE_FOUNDRY_KEY}" \
  -d '{
    "model": "gpt-5-mini",
    "messages": [{"role": "user", "content": "REST API 테스트"}]
  }'
```

### Windows PowerShell (Invoke-RestMethod)
```powershell
$headers = @{
    "api-key" = $env:AZURE_FOUNDRY_KEY
    "Content-Type" = "application/json"
}

$body = @{
    messages = @(
        @{ role = "user"; content = "PowerShell 테스트" }
    )
} | ConvertTo-Json

Invoke-RestMethod -Uri "${env:AZURE_FOUNDRY_ENDPOINT}openai/v1/chat/completions" `
    -Method Post `
    -Headers $headers `
    -Body $body
```

---

## 에러 처리와 재시도 정책

LLM 호출은 네트워크 지연이나 할당량 제한(Rate Limit)으로 인해 실패할 가능성이 상존한다.

- **HTTP 429 (Too Many Requests)**: 설정된 TPM(Tokens Per Minute) 또는 RPM(Requests Per Minute)을 초과했을 때 발생한다. 이때는 즉시 재시도하지 말고 지수 백오프(Exponential Backoff)를 적용해야 한다.
- **HTTP 500/503**: 서버 내부 오류이다. 일시적일 확률이 높으므로 짧은 대기 후 재시도한다.

Azure Java SDK는 기본적으로 재시도 정책을 내장하고 있다. `AIProjectClientBuilder` 수준에서 커스텀 `RetryPolicy`를 주입할 수 있지만, 기본값만으로도 대부분의 일시적 오류는 해결된다.

---

## VS Code / Copilot Chat 연동

개발 중 코드를 직접 짜지 않고 Copilot Chat을 통해 API 호출 코드를 생성하거나 테스트할 때의 팁이다.

사용자는 앞선 대화에서 Azure 연동 옵션에 대해 질문한 바 있다. VS Code의 Azure AI Foundry 확장 프로그램을 설치하면 엔드포인트와 키를 직접 복사할 필요 없이 프로젝트 리스트에서 바로 선택하여 환경변수 파일(.env)을 생성할 수 있다.

💡 **팁** : Copilot Chat에서 Azure OpenAI 설정을 물으면 `Custom Endpoint` 필드를 비워두는 것이 일반적이다. Azure 전용 프리셋이 이미 인증 로직과 API 버전을 올바르게 처리하기 때문이다. 만약 독자적인 Gateway를 거쳐야 하는 특수한 상황이 아니라면 Azure 기본 경로를 그대로 사용하는 것이 가장 안전하다.

---

## 로깅과 디버깅

API 호출이 왜 실패하는지, 어떤 JSON이 오가는지 확인하려면 로깅 설정이 필수적이다.

1. **SLF4J + slf4j-simple**: 앞선 `pom.xml`에 추가한 의존성이다.
2. **환경변수로 로그 레벨 조절**:
   - `AZURE_LOG_LEVEL=DEBUG`를 설정하면 HTTP 요청의 헤더와 바디 전문이 콘솔에 출력된다. 이는 네트워크 구간의 문제를 파악하는 데 결정적인 도움을 준다.

## 3. Java SDK 심층 분석: AIProjectClient vs OpenAIClient

Foundry 개발을 시작할 때 가장 먼저 마주하는 질문은 "어떤 클라이언트를 써야 하는가?"이다.

### 3.1 AIProjectClient (The Manager)
`azure-ai-projects` 패키지에 포함된 `AIProjectClient`는 이름 그대로 '프로젝트 전체'를 관리하는 통합 컨트롤러다.
- **역할**: 프로젝트 설정 로드, 커넥션(Connection) 관리, 에이전트 서비스(Agent Service) 호출, 추론 클라이언트 생성.
- **장점**: 프로젝트 ID나 연결된 리소스 정보를 수동으로 입력할 필요 없이, Foundry 포털의 설정을 코드로 그대로 가져올 수 있다.
- **코드 예시**:
  ```java
  AIProjectClient projectClient = new AIProjectClientBuilder()
      .connectionString("eastus.api.azure.com;00000000-0000-0000-0000-000000000000;rg-name;project-name")
      .credential(new DefaultAzureCredentialBuilder().build())
      .buildClient();
  ```

### 3.2 OpenAIClient (The Worker)
`azure-ai-openai` 패키지의 `OpenAIClient`는 실제 '모델 추론'만을 담당하는 전용 클라이언트다.
- **역할**: Chat Completions, Embeddings, Image Generation API 호출.
- **장점**: 가볍고 빠르며, OpenAI 표준 API 규격을 완벽히 따른다.
- **연동**: `AIProjectClient`로부터 `OpenAIClient`를 생성하는 것이 2026년형 표준 워크플로다. 이렇게 하면 프로젝트의 보안 설정과 엔드포인트를 클라이언트가 자동으로 상속받는다.

---

## 4. 고급 HTTP 클라이언트 설정 (Resilience & Performance)

엔터프라이즈 급 앱에서는 단순히 API를 호출하는 것을 넘어, 네트워크 환경의 불안정성에 대비해야 한다. Azure SDK는 내부적으로 `Azure-Core` HTTP 파이프라인을 사용하며, 이를 통해 세밀한 튜닝이 가능하다.

### 4.1 타임아웃 및 프록시 설정
기본 타임아웃(보통 60초)은 GPT-5의 긴 추론 시간이나 대량의 스트리밍 응답 시 짧을 수 있다.
```java
HttpClient httpClient = new NettyAsyncHttpClientBuilder()
    .readTimeout(Duration.ofSeconds(120)) // 2분으로 확장
    .responseTimeout(Duration.ofSeconds(120))
    .proxy(new ProxyOptions(ProxyOptions.Type.HTTP, new InetSocketAddress("proxy.company.com", 8080)))
    .build();

OpenAIClient client = new AIProjectClientBuilder()
    .endpoint(endpoint)
    .credential(new AzureKeyCredential(key))
    .httpClient(httpClient) // 커스텀 클라이언트 주입
    .buildOpenAIClient();
```

### 4.2 로깅 정책 (Logging Policy)
디버깅을 위해 HTTP 요청/응답 전문을 보고 싶다면 로깅 정책을 추가하라.
```java
OpenAIClient client = new AIProjectClientBuilder()
    .endpoint(endpoint)
    .credential(new AzureKeyCredential(key))
    .httpLogOptions(new HttpLogOptions().setLogLevel(HttpLogDetailLevel.BODY_AND_HEADERS))
    .buildOpenAIClient();
```
*주의: 프로덕션 환경에서는 민감 정보가 로그에 남지 않도록 반드시 `NONE`으로 설정하거나 특정 헤더만 남기도록 필터링해야 한다.*

---

## 5. 스트리밍 응답 (Server-Sent Events) 구현

사용자 경험(UX)을 극대화하려면 모델의 답변이 생성되는 즉시 화면에 뿌려주는 스트리밍 방식이 필수적이다.

### 5.1 Java 기반 스트리밍 코드
Azure SDK는 `IterableStream`을 통해 스트리밍을 지원한다.
```java
List<ChatRequestMessage> messages = List.of(new ChatRequestUserMessage("우주에 대해 길게 설명해줘."));
ChatCompletionsOptions options = new ChatCompletionsOptions(messages);

// 스트리밍 호출
IterableStream<ChatCompletions> streamingCompletions = 
    client.getChatCompletionsStream(deployment, options);

streamingCompletions.forEach(chatChunk -> {
    if (!chatChunk.getChoices().isEmpty()) {
        String content = chatChunk.getChoices().get(0).getDelta().getContent();
        if (content != null) {
            System.out.print(content); // 토큰이 생성될 때마다 즉시 출력
            System.out.flush();
        }
    }
});
```

---

## 6. Native SDK vs LangChain4j: 심층 비교 (Decision Matrix)

어떤 도구를 선택할지는 프로젝트의 성격에 달렸다.

| 비교 항목 | Native Azure SDK (Java) | LangChain4j |
| :--- | :--- | :--- |
| **추상화 수준** | 저수준 (Low-level) | 고수준 (High-level) |
| **제어력** | API의 모든 파라미터 미세 조정 가능 | 표준화된 인터페이스로 제어력 제한적 |
| **러닝 커브** | Azure 에코시스템 이해 필요 | LLM 추상화 개념 이해 필요 |
| **확장성** | Azure 서비스 간 연동 최적화 | 타사 모델(Claude, Gemini) 전환 용이 |
| **기능 범위** | 추론, 관리, 보안 통합 | 프롬프트 관리, RAG, 메모리 중심 |

### 6.1 추천 시나리오
- **Native SDK 추천**: 특정 리전의 특수한 보안 정책을 다루거나, Foundry의 신기능(예: Responses API v2의 저수준 이벤트)을 가장 먼저 적용해야 할 때.
- **LangChain4j 추천**: RAG 시스템을 구축하거나, 여러 모델을 섞어서 쓰는 에이전트를 만들 때. 특히 Spring Boot와 연동하여 '선언적 AI 서비스'를 구축하고 싶을 때 최강의 생산성을 보여준다.

---

## 7. 토큰 관리와 비용 통제 (Token Management)

자바 앱에서 토큰 사용량을 추적하고 비용을 예측하는 것은 운영팀의 핵심 요구사항이다.

### 7.1 Tiktoken을 활용한 로컬 토큰 카운팅
API를 호출하기 전, 프롬프트의 길이를 미리 파악하여 쿼터 초과를 방지할 수 있다. `knuddels:jtokkit` 라이브러리를 사용한다.
```java
// pom.xml에 추가: com.knuddels:jtokkit:1.1.0
EncodingRegistry registry = Encodings.newDefaultEncodingRegistry();
Encoding enc = registry.getEncodingForModel(ModelType.GPT_4O); // GPT-5도 동일 인코딩 사용

int tokenCount = enc.encode("안녕하세요, 토큰을 계산해봅시다.").size();
System.out.println("예상 토큰 수: " + tokenCount);
```

### 7.2 응답에서 실제 사용량 추출
모델의 응답 객체에는 항상 `Usage` 정보가 포함되어 있다. 이를 로그에 기록하여 실제 과금 데이터를 수집하라.
```java
ChatCompletions completions = client.getChatCompletions(deployment, options);
CompletionsUsage usage = completions.getUsage();
System.out.printf("Input: %d, Output: %d, Total: %d%n", 
    usage.getPromptTokens(), usage.getCompletionTokens(), usage.getTotalTokens());
```

---

## 8. Entra ID(Managed Identity) 기반 무키(Keyless) 인증 구축

보안 사고의 90%는 API 키 유출에서 시작된다. 2026년형 기업용 앱은 무조건 Managed Identity를 써야 한다.

### 8.1 개발 환경 설정 (`az login`)
로컬 개발 시에는 개발자의 Azure 계정 권한을 빌려 쓴다.
1. 터미널에서 `az login` 실행.
2. `az account set --subscription <subscription-id>`로 구독 확인.
3. 코드에서 `DefaultAzureCredential` 사용.

### 8.2 프로덕션 환경 (App Service / AKS)
Azure 리소스(VM, Web App 등)에 'System Assigned Identity'를 활성화하고, 해당 ID에 AI 리소스에 대한 **'Cognitive Services OpenAI User'** 역할을 부여한다.

```java
// Managed Identity 인증 코드
OpenAIClient client = new AIProjectClientBuilder()
    .endpoint(endpoint)
    .credential(new DefaultAzureCredentialBuilder().build()) // 키 없이 자동으로 인증
    .buildOpenAIClient();
```
이 방식의 장점은 로컬(`az login` 토큰 사용)과 클라우드(Managed Identity 사용)에서 **코드를 단 한 줄도 바꿀 필요가 없다**는 점이다.

---

## 9. 에러 처리 전략 (Resilience Patterns)

LLM은 일반적인 DB와 달리 응답 시간이 길고 실패 확률이 상대적으로 높다. 따라서 다음과 같은 탄력성 패턴이 필요하다.

### 9.1 지수 백오프 (Exponential Backoff)
429(Rate Limit) 에러 발생 시 즉시 재시도하면 상황이 악화된다. 대기 시간을 2배씩 늘리며 재시도하라. Azure SDK는 이를 기본 제공하지만, 설정을 통해 최적화할 수 있다.
```java
OpenAIClient client = new AIProjectClientBuilder()
    .retryOptions(new RetryOptions(new ExponentialBackoffOptions()
        .setMaxRetries(5)
        .setBaseDelay(Duration.ofSeconds(2))))
    .buildOpenAIClient();
```

### 3.2 서킷 브레이커 (Circuit Breaker)
모델이 지속적으로 에러를 내거나 응답이 너무 늦다면, 잠시 연결을 끊고 폴백(Fallback) 로직(예: 캐시된 응답 반환 혹은 가벼운 모델로 우회)을 작동시켜야 한다. 이는 `Resilience4j` 같은 별도 라이브러리와 조합하여 구현한다.

---

## 10. 비동기 호출 (Async Programming)

동시 접속자가 많은 웹 애플리케이션에서는 동기(Blocking) 호출이 스레드 풀을 고갈시킬 수 있다. `OpenAIAsyncClient`를 사용하여 리소스를 아껴야 한다.

```java
OpenAIAsyncClient asyncClient = new AIProjectClientBuilder()
    .endpoint(endpoint)
    .credential(new AzureKeyCredential(key))
    .buildOpenAIAsyncClient();

asyncClient.getChatCompletions(deployment, options)
    .subscribe(
        result -> System.out.println("응답: " + result.getChoices().get(0).getMessage().getContent()),
        error -> System.err.println("에러: " + error.getMessage()),
        () -> System.out.println("완료!")
    );
```
Reactor(`Mono`, `Flux`) 기반의 비동기 코딩은 고성능 AI API 서버 구축의 핵심이다.

---

## 11. 텍스트 인코딩과 CJK 처리 (Character Encoding)

한국어(CJK) 처리 시 발생할 수 있는 인코딩 이슈를 예방해야 한다.

- **UTF-8 보장**: 모든 문자열 처리는 UTF-8을 기본으로 한다. JDK 17 이상부터는 `StandardCharsets.UTF_8`을 명시적으로 사용하는 습관을 들이자.
- **Normalize**: 사용자 입력에 포함된 특수 문자나 유니코드 조합 자모를 `Normalizer` 클래스를 통해 정규화한 뒤 모델에 보내면, 토큰 소모를 미세하게 줄이고 인식 정확도를 높일 수 있다.

---

## 13. Spring AI와 Foundry 연동 (Modern Spring Boot)

2026년 현재, 자바 백엔드 개발의 표준인 Spring Boot 환경에서는 **Spring AI** 라이브러리가 Foundry 연동의 핵심 도구로 자리 잡았다.

### 13.1 의존성 추가 (Maven)
Spring AI는 Azure OpenAI 전용 스타터를 제공한다.
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-azure-openai-spring-boot-starter</artifactId>
    <version>2.0.0</version>
</dependency>
```

### 13.2 설정 (application.yml)
프로퍼티 설정을 통해 코드 수정 없이 환경을 전환할 수 있다.
```yaml
spring:
  ai:
    azure:
      openai:
        endpoint: ${AZURE_FOUNDRY_ENDPOINT}
        api-key: ${AZURE_FOUNDRY_KEY}
        chat:
          options:
            deployment-name: gpt-5-mini
            temperature: 0.7
```

### 13.3 선언적 서비스 구현
`ChatClient` 인터페이스를 통해 비즈니스 로직에만 집중할 수 있다.
```java
@Service
public class AiService {
    private final ChatClient chatClient;

    public AiService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public String askSomething(String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .call()
            .content();
    }
}
```

---

## 14. 구조화된 출력 (Structured Output) 처리

AI의 답변을 자바 객체(POJO)로 즉시 변환하는 것은 데이터 처리 파이프라인의 필수 단계다.

### 14.1 Native SDK에서의 JSON Mode
모델이 유효한 JSON만 내놓도록 강제하고, 이를 Jackson 라이브러리로 파싱한다.
```java
ChatCompletionsOptions options = new ChatCompletionsOptions(messages)
    .setResponseFormat(new ChatCompletionsJsonResponseFormat()); // JSON 모드 활성화

ChatCompletions completions = client.getChatCompletions(deployment, options);
String jsonResponse = completions.getChoices().get(0).getMessage().getContent();

// Jackson ObjectMapper로 DTO 변환
ObjectMapper mapper = new ObjectMapper();
UserDto user = mapper.readValue(jsonResponse, UserDto.class);
```

### 14.2 LangChain4j의 AiServices 활용 (추천)
인터페이스 정의만으로 JSON 파싱을 자동화할 수 있다.
```java
interface CustomerAssistant {
    @UserMessage("추출해줘: {{it}}")
    CustomerInfo extractInfo(String text);
}

CustomerAssistant assistant = AiServices.create(CustomerAssistant.class, model);
CustomerInfo info = assistant.extractInfo("서울에 사는 30세 홍길동입니다.");
// info.getName() -> "홍길동" (자동 파싱 완료)
```

---

## 15. 멀티모달(Multi-modal) 데이터 처리

GPT-5.5의 강력한 이미지 분석 능력을 자바에서 활용하는 방법이다.

### 15.1 이미지 분석 요청 (Java SDK)
로컬 이미지 파일이나 URL을 모델에 전달한다.
```java
List<ChatRequestMessage> messages = new ArrayList<>();
List<ChatMessageContentItem> contentItems = new ArrayList<>();

// 텍스트 지시
contentItems.add(new ChatMessageTextContentItem("이 이미지에 무엇이 보이나요?"));

// 이미지 데이터 (Base64)
byte[] imageBytes = Files.readAllBytes(Paths.get("image.jpg"));
String base64Image = Base64.getEncoder().encodeToString(imageBytes);
contentItems.add(new ChatMessageImageContentItem(
    new ChatMessageImageUrl("data:image/jpeg;base64," + base64Image)
));

messages.add(new ChatRequestUserMessage(contentItems));
```

---

## 16. 글로벌 배치(Global Batch) API 활용

수만 건의 데이터를 비실시간으로 처리할 때 비용을 50% 절감하는 방법이다.

### 16.1 배치 작업 생성
학습 데이터와 동일한 JSONL 파일을 입력으로 보낸다.
```java
// 1. 입력 파일 업로드
FileClient fileClient = projectClient.getFileClient();
AIFile inputFile = fileClient.uploadFile(Paths.get("batch_input.jsonl"), "batch_input");

// 2. 배치 작업 생성
BatchClient batchClient = projectClient.getBatchClient();
BatchJob job = batchClient.createJob(inputFile.getId(), "/openai/v1/chat/completions", "gpt-5-mini");

System.out.println("배치 작업 ID: " + job.getId());
```

### 16.2 상태 모니터링
작업이 완료될 때까지 주기적으로 상태를 체크한다.
```java
while (true) {
    job = batchClient.getJob(job.getId());
    System.out.println("현재 상태: " + job.getStatus());
    if (job.getStatus().equals("completed")) break;
    Thread.sleep(60000); // 1분 대기
}

// 3. 결과 다운로드
AIFile outputFile = fileClient.getFile(job.getOutputFileId());
fileClient.downloadFile(outputFile.getId(), Paths.get("batch_output.jsonl"));
```

## 17. 실무형 에러 핸들링: Resilience4j 서킷 브레이커 구현

단순한 재시도를 넘어, 모델 응답이 지속적으로 실패할 때 시스템 전체의 마비(Thread Starvation)를 막기 위한 서킷 브레이커 패턴을 적용한다.

### 17.1 Resilience4j 설정 (Java)
```java
// 서킷 브레이커 설정
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
    .failureRateThreshold(50) // 실패율 50% 이상 시 차단
    .waitDurationInOpenState(Duration.ofMillis(10000)) // 10초 후 반개방 상태로 전환
    .permittedNumberOfCallsInHalfOpenState(3)
    .slidingWindowSize(10)
    .build();

CircuitBreakerRegistry registry = CircuitBreakerRegistry.of(config);
CircuitBreaker circuitBreaker = registry.circuitBreaker("foundry-ai-service");

// 사용 예시
String response = circuitBreaker.executeSupplier(() -> {
    return model.generate("중요한 비즈니스 로직 질문");
});
```

### 17.2 폴백(Fallback) 전략 수립
서킷이 열렸을 때 사용자에게 보여줄 대안을 마련하라.
- **정적 응답**: "현재 시스템 점검 중입니다. 잠시 후 다시 시도해주세요."
- **모델 우회**: GPT-5 실패 시 즉시 로컬 SLM(Phi-4)이나 가벼운 모델로 전환하여 최소한의 기능 유지.
- **캐시 활용**: 동일한 질문에 대해 최근에 성공했던 응답이 있다면 이를 반환.

---

## 18. 관측 가능성(Observability) 및 트레이싱 통합

자바 앱에서 보낸 요청이 Foundry 내부에서 어떻게 처리되는지 추적하기 위해 OpenTelemetry를 통합한다.

### 18.1 OpenTelemetry 자바 에이전트 활용
별도의 코드 수정 없이 JVM 옵션만으로 모든 HTTP 호출을 가시화할 수 있다.
```bash
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=foundry-backend-app \
     -Dotel.exporter.otlp.endpoint=http://localhost:4317 \
     -jar app.jar
```

### 18.2 SDK 내장 트레이싱 활성화
Azure SDK는 `azure-core-tracing-opentelemetry`를 통해 프로그래밍 방식의 트레이싱을 지원한다.
```java
// pom.xml에 추가: com.azure:azure-core-tracing-opentelemetry
OpenAIClient client = new AIProjectClientBuilder()
    .endpoint(endpoint)
    .credential(new AzureKeyCredential(key))
    .addPolicy(new OpenTelemetryTracingPolicy()) // 트레이싱 정책 추가
    .buildOpenAIClient();
```
이를 통해 Foundry 포털의 **Traces** 메뉴에서 자바 앱의 요청 타임라인과 내부 병목 지점을 시각적으로 확인할 수 있다.

---

## 19. 보안 심화: 가상 네트워크(VNet)와 IP 제한

엔드포인트가 공개되어 있다면 API 키가 있더라도 DDoS 공격이나 무단 접근의 위협이 있다.

### 19.1 인바운드 IP 제한
사내 오피스나 데이터 센터의 공인 IP 대역만 허용하도록 Foundry 리소스의 'Networking' 설정을 강화하라.
- **코드 레벨 대응**: SDK 호출 시 요청자의 IP 주소를 `X-Forwarded-For` 헤더에 담아 전달하면, Foundry 가드레일이 이를 분석하여 비정상적인 접근을 차단하는 데 도움을 준다.

### 19.2 Private Link를 통한 망 분리
자바 앱이 실행되는 Azure 가상 네트워크와 Foundry 리소스를 Private Link로 연결한다. 이 경우 엔드포인트는 `*.privatelink.openai.azure.com` 주소를 가지게 되며, 외부 인터넷에서는 해당 도메인 자체가 조회되지 않아 완벽한 논리적 격리가 완성된다.

### 14.3 JSON 스키마를 활용한 엄격한 검증 (Strict JSON Schema)

2026년형 Foundry는 JSON 모드에서 단순한 'JSON 형태'를 넘어, 사전에 정의된 **JSON Schema**를 엄격히 따르도록 강제하는 기능을 지원한다.

```java
// JSON Schema 정의 (Jackson 사용 시)
String jsonSchema = """
    {
      "name": "CustomerExtraction",
      "strict": true,
      "schema": {
        "type": "object",
        "properties": {
          "name": { "type": "string" },
          "age": { "type": "integer" },
          "orders": {
            "type": "array",
            "items": { "type": "string" }
          }
        },
        "required": ["name", "age", "orders"],
        "additionalProperties": false
      }
    }
    """;

ChatCompletionsOptions options = new ChatCompletionsOptions(messages)
    .setResponseFormat(new ChatCompletionsJsonResponseFormat(jsonSchema));
```
`strict: true` 옵션을 사용하면 모델은 무조건 스키마를 100% 준수하는 답변만 생성하며, 이를 통해 자바 앱에서의 역직렬화(Deserialization) 에러 발생 확률을 0%에 수렴하게 만든다.

---

## 21. Rate Limit 헤더 분석 및 동적 스로틀링 (Rate Limit Analysis)

Foundry의 응답 헤더에는 현재 할당량 상태를 파악할 수 있는 핵심 지표들이 포함되어 있다. 이를 분석하여 앱 수준에서 선제적으로 요청 속도를 조절할 수 있다.

### 21.1 주요 헤더 정보
- `x-ratelimit-remaining-requests`: 현재 분(Minute) 내에 남은 요청 횟수.
- `x-ratelimit-remaining-tokens`: 현재 분 내에 남은 토큰 양.
- `x-ratelimit-reset-requests`: 요청 횟수 쿼터가 초기화되기까지 남은 시간.
- `x-ratelimit-reset-tokens`: 토큰 쿼터가 초기화되기까지 남은 시간.

### 21.2 자바에서 헤더 추출 및 활용
```java
ChatCompletions completions = client.getChatCompletions(deployment, options);
completions.getRawResponse().getHeaders().toMap().forEach((name, value) -> {
    if (name.startsWith("x-ratelimit")) {
        System.out.println(name + ": " + value);
    }
});
```
이 데이터를 Redis 등에 공유하여 클러스터 환경의 모든 서버가 동일한 쿼터 상태를 공유하고, 쿼터 임계치 도달 시 스스로 요청을 지연시키는 'Smart Throttling'을 구현할 수 있다.

---

## 22. AI 서비스 단위 테스트 및 모킹 (Unit Testing & Mocking)

매번 실제 API를 호출하며 테스트하는 것은 비용과 시간 측면에서 비효율적이다.

### 22.1 Mockito를 사용한 SDK 모킹
비즈니스 로직 검증 시 SDK의 응답을 가상으로 생성한다.
```java
@Test
void testAiLogic() {
    OpenAIClient mockClient = mock(OpenAIClient.class);
    ChatCompletions mockCompletions = mock(ChatCompletions.class);
    ChatChoice mockChoice = mock(ChatChoice.class);
    ChatMessage mockMessage = new ChatMessage(ChatRole.ASSISTANT, "가짜 응답입니다.");

    when(mockChoice.getMessage()).thenReturn(mockMessage);
    when(mockCompletions.getChoices()).thenReturn(List.of(mockChoice));
    when(mockClient.getChatCompletions(anyString(), any())).thenReturn(mockCompletions);

    // AI 서비스를 호출하는 실제 로직 테스트...
}
```

### 22.2 WireMock을 사용한 REST 레벨 모킹
네트워크 타임아웃이나 429 에러 상황을 재현하여 에러 처리 로직을 검증할 때 유용하다.
```java
stubFor(post(urlEqualTo("/openai/v1/chat/completions"))
    .willReturn(aResponse()
        .withStatus(429)
        .withHeader("Retry-After", "5")
        .withBody("{\"error\": {\"message\": \"Rate limit exceeded\"}}")));
```

---

## 23. 고성능 파이프라인 최적화: HTTP/2 및 Connection Pooling

대규모 동시 요청이 발생하는 서비스에서는 HTTP 커넥션 관리 능력이 전체 시스템의 처리량(Throughput)을 결정한다.

### 23.1 HTTP/2 활성화
Azure SDK for Java는 Netty나 OkHttp를 통해 HTTP/2를 지원한다. HTTP/2의 멀티플렉싱 기능을 통해 단일 커넥션으로 여러 AI 요청을 병렬로 처리할 수 있다.
```java
HttpClient httpClient = new NettyAsyncHttpClientBuilder()
    .protocol(HttpLogDetailLevel.BASIC) // HTTP/2 우선 협상
    .connectionKeepAlive(true)
    .build();
```

### 23.2 커넥션 풀링(Connection Pooling) 전략
매 요청마다 TCP 핸드셰이크를 수행하는 것은 낭비다. 커넥션 풀을 적절히 유지하여 지연 시간을 줄여야 한다.
- **Max Connections**: 서비스의 동시 접속자 수에 맞춰 500~1000개 수준으로 확장.
- **Idle Timeout**: 사용되지 않는 커넥션을 너무 빨리 닫지 않도록 30~60초 정도로 설정.

### 23.3 리소스 해제와 종료 (Resource Cleanup)

자바 애플리케이션이 종료될 때 열려 있는 HTTP 커넥션이나 비동기 스레드 풀을 안전하게 닫아주어야 한다.

```java
// Spring Boot 환경이라면 @PreDestroy 사용
@PreDestroy
public void cleanup() {
    // 비동기 클라이언트의 경우 내부적으로 사용하는 리소스가 GC에 의해 관리되지만, 
    // 명시적인 클로징이 필요한 경우 처리 로직을 넣는다.
}
```

---

## 24. API 버저닝 전략과 하위 호환성 (API Versioning)

Azure Foundry는 빠른 속도로 발전하며 매달 새로운 API 버전을 출시한다.

-   **Stable vs Preview**: 프로덕션 환경에서는 가급적 `preview`가 붙지 않은 버전을 사용하라. `preview` 버전은 새로운 기능(예: GPT-5의 최신 멀티모달 기능)을 가장 먼저 써볼 수 있지만, API 규격이 예고 없이 변경될 수 있다.
-   **하위 호환성 유지**: 특정 기능을 위해 이전 버전의 API를 사용해야 하는 경우(예: 특정 리전의 레거시 모델), `AIProjectClient` 수준에서 API 버전을 명시적으로 지정하여 호출할 수 있다. 이는 복잡한 대규모 시스템에서 모델별로 최적의 API 경로를 선택할 수 있게 해준다.

### 24.1 챕터 요약 및 핵심 정리

본 장에서는 자바 개발자가 Microsoft Foundry의 강력한 기능을 활용하기 위해 반드시 알아야 할 API 연동의 모든 것을 다루었다.

- **통합 클라이언트 아키텍처**: `AIProjectClient`로 전체 설정을 관리하고 `OpenAIClient`로 실제 추론을 수행하는 이중 구조를 이해하라.
- **보안의 현대화**: API 키 중심의 과거 방식에서 탈퇴하여 **Entra ID(Managed Identity)** 기반의 무키(Keyless) 인증을 기본으로 채택하라.
- **추상화 도구 활용**: 저수준 제어가 필요할 때는 Native SDK를, 생산성이 중요할 때는 **LangChain4j**나 **Spring AI**를 선택하라.
- **탄력성 확보**: 네트워크 오류와 할당량 제한에 대비하여 **지수 백오프**와 **서킷 브레이커** 패턴을 반드시 적용하라.
- **성능 최적화**: 사용자 경험을 위한 **스트리밍** 응답과 대량 처리를 위한 **Batch API**를 상황에 맞게 혼용하라.

이제 여러분은 단순한 API 호출자가 아닌, 견고하고 안전한 엔터프라이즈 AI 애플리케이션을 구축할 수 있는 실무 엔지니어의 기초를 갖췄다.

---

## 25. 결론 및 실전 체크리스트

3장에서는 자바와 REST API를 통해 Microsoft Foundry의 강력한 기능을 실제 코드로 구현하는 기초를 닦았다.

### 🚀 실무 적용 체크리스트
- [ ] **DefaultAzureCredential**을 사용하여 로컬과 클라우드 인증을 통일했는가?
- [ ] GPT-5 계열 호출 시 **maxCompletionTokens**와 **temperature=1.0** 설정을 확인했는가?
- [ ] 대용량 데이터 처리에 **Global Batch API** 도입을 검토했는가?
- [ ] 사용자 경험 개선을 위해 **Streaming** 응답을 구현했는가?
- [ ] API 키 유출 방지를 위해 모든 키를 **환경변수**나 **Key Vault**로 관리하고 있는가?
- [ ] 장애 복구력을 위해 **지수 백오프**와 **서킷 브레이커**를 적용했는가?

이제 여러분은 모델을 자유자재로 호출할 수 있는 능력을 갖췄다. 하지만 똑똑한 모델도 질문이 모호하면 엉뚱한 답을 내놓는다. 다음 장에서는 모델의 잠재력을 100% 끌어올리기 위한 **프롬프트 엔지니어링**과 **구조화된 출력**의 정수를 배울 것이다.

---

## 📚 더 읽기

- [Azure AI Foundry Java SDK Hub](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/develop/java)
- [Azure AI Projects Client Library for Java (README)](https://github.com/Azure/azure-sdk-for-java/tree/main/sdk/ai/azure-ai-projects)
- [LangChain4j Azure OpenAI Documentation](https://docs.langchain4j.dev/integrations/language-models/azure-openai)
- [Azure OpenAI Service REST API Reference](https://learn.microsoft.com/en-us/azure/ai-services/openai/reference)
- [Entra ID Authentication for Azure AI Services](https://learn.microsoft.com/en-us/azure/ai-services/authentication)

---

[← Ch.2 첫 모델 배포](Ch02_First_Deployment.md) | [Ch.4 Prompt Engineering →](Ch04_Prompt_Engineering.md)
