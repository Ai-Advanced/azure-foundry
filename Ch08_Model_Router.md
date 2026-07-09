# Chapter 8. Model Router & 배포 전략

[← 목차로](README.md)

> **학습 목표**
> - 단일 모델 종속성을 탈피하기 위한 Model Router의 필요성과 작동 원리를 이해한다.
> - Responses API v2 기반의 모델 라우팅 아키텍처를 파악한다.
> - Azure Policy를 통한 모델 거버넌스 및 허용 리스트 관리 방법을 익힌다.
> - 10가지 배포 유형 중 비즈니스 요구사항에 맞는 최적의 SKU를 선택한다.
> - PTU 예약 및 Global Batch API를 활용하여 비용과 성능을 최적화한다.
> - Java SDK를 사용해 멀티 리전 페일오버 및 대량 배치 작업을 구현한다.

> **전제 조건**
> - Chapter 3. API 연동 기초 학습 완료
> - Chapter 7. Agent Service (Responses API v2) 개념 이해
> - Azure 구독 및 Foundry 프로젝트 관리자 권한

---

## 1. Model Router: 왜 필요한가

LLM 애플리케이션을 개발할 때 흔히 저지르는 실수는 특정 모델(예: gpt-5)의 특정 배포 엔드포인트에 코드를 고정(Hard-coding)하는 것이다. 이는 단기적으로는 간편하지만, 프로덕션 환경에서는 여러 문제를 야기한다.

첫째, **모델 잠금-in(Vendor/Model Lock-in)** 문제다. 특정 모델에 의존성이 생기면 더 성능이 좋거나 저렴한 신규 모델이 출시되었을 때 코드를 일일이 수정해야 한다. 2026년 현재 Foundry 모델 카탈로그에는 1900개가 넘는 모델이 존재하며, 모델 교체 주기는 점점 빨라지고 있다.

둘째, **Cost/Latency/Capability의 트레이드오프**다. 단순한 인사말에는 저렴한 `gpt-5-nano`가 적합하고, 복잡한 논리 추론에는 `gpt-5.5`가 필요하다. 이를 매번 개발자가 코드 수준에서 조건문으로 분기하는 것은 관리 부담을 가중시킨다.

셋째, **쿼터(Quota) 부족 및 가용성** 이슈다. 특정 리전의 특정 모델에 트래픽이 몰려 HTTP 429(Too Many Requests) 에러가 발생할 때, 자동으로 다른 리전이나 대체 모델로 요청을 돌려주는 메커니즘이 필요하다.

**Model Router**는 "모델 선택을 코드가 아닌 서비스 레벨에서 결정한다"는 원칙을 실현한다. 개발자는 추상화된 라우터 엔드포인트에 요청을 던지고, 실제 어떤 모델이 이 요청을 처리할지는 관리자가 설정한 정책(Policy)이나 시스템의 자동 선택(`auto`)에 맡긴다.

---

## 2. Model Router 아키텍처 (2026 GA)

2026년 정식 출시된 Model Router는 **Responses API v2** 아키텍처의 핵심 컴포넌트로 통합되었다.

### 2.1 라우팅 메커니즘
Model Router는 기존의 단일 배포 엔드포인트와 달리, 프로젝트 레벨의 가상 엔드포인트를 사용한다. 요청 시 `model` 파라미터에 구체적인 모델 이름을 넣는 대신 라우팅 규칙을 적용할 수 있다.

- **Endpoint**: `/openai/v1/responses` (Responses API v2 기반)
- **Routing Mode**:
  - **Auto-select (`model: "auto"`)**: Foundry가 요청의 복잡도, 현재 가용 쿼터, 비용 목표를 분석하여 최적의 모델을 실시간으로 선택한다.
  - **Explicit Selection**: `gpt-5-pro`, `gpt-5-nano` 등 모델 명칭을 명시하되, 실제 물리적인 배포 위치(Region)는 라우터가 결정하게 한다.
  - **Policy-based**: 기업의 거버넌스 정책에 따라 허용된 모델 리스트 내에서만 라우팅이 발생한다.

### 2.2 Responses API 통한 통합
Chapter 7에서 다룬 Responses API v2는 모델 호출과 에이전트 실행을 동일한 인터페이스로 처리한다. Model Router 역시 이 인터페이스를 따르므로, 에이전트가 사용하는 두뇌(LLM)를 라우터로 지정하면 에이전트의 성능과 비용을 동적으로 제어할 수 있다.

### 2.3 GlobalProvisionedManaged (GPM) SKU 심층 분석

2026년 새롭게 도입된 **GlobalProvisionedManaged (GPM)** SKU는 기존 PTU의 성능 보장과 Standard의 유연성을 결합한 하이브리드 배포 방식이다.

- **예약형 유연성**: 특정 기간(예: 1시간 단위) 동안만 PTU를 동적으로 예약하고, 사용이 끝나면 즉시 반납할 수 있다.
- **자동 스케일링**: 트래픽이 급증할 때 사전에 설정한 'Max GPM' 한도까지 자동으로 처리량을 늘려 지연 시간을 방지한다.
- **Java SDK 연동**: `AzureCreateResponseOptions`에서 `skuOverride("GPM")`를 설정하여 요청 단위로 우선순위를 부여할 수 있다.

```java
// GpmUsageExample.java
AzureCreateResponseOptions options = new AzureCreateResponseOptions()
    .setSkuOverride("GPM") // 해당 요청에 대해 예약된 대역폭(GPM) 우선 사용
    .setPriority(ResponsePriority.HIGH); // 큐 대기열에서 우선순위 상향
```

### 2.4 Model Router 내부 로드 밸런싱 및 가용성 메커니즘

Model Router는 단순한 프록시가 아니라, Azure의 거대한 인프라 네트워크 위에서 동작하는 지능형 트래픽 오케스트레이터이다. 내부적으로 다음과 같은 알고리즘을 통해 요청을 분산한다.

1.  **Latency-Aware Routing**: 전 세계 리전에 배포된 모델들의 실시간 응답 지연 시간(TTFT: Time To First Token)을 측정한다. 네트워크 홉(Hop)이 가장 적고 현재 부하가 낮은 리전으로 요청을 우선 배당한다.
2.  **Quota-Based Shedding**: 각 프로젝트에 할당된 TPM(Tokens Per Minute) 소모량을 실시간 추적한다. 특정 리전의 쿼터가 80% 이상 소진되면, 여유 쿼터가 있는 다른 지역의 배포 모델로 트래픽을 점진적으로 전환(Gradual Shift)한다.
3.  **Tiered Reliability**: `auto` 모드에서는 가용성이 높은 리전(예: Availability Zone이 3개 이상인 리전)을 우선 순위에 둔다. 특정 데이터 센터에 하드웨어 장애가 감지되면 라우터는 1초 이내에 해당 엔드포인트를 호출 대상에서 제외시킨다.

### 2.5 Model Router와 Service Mesh 통합 (Preview)

2026-06 프리뷰로 공개된 기능으로, Istio나 Linkerd와 같은 Service Mesh 환경에서 Model Router를 사이드카(Sidecar) 형태로 통합할 수 있다. 이를 통해 어플리케이션 코드 외부에서 mTLS 인증, 서킷 브레이커, 세밀한 트래픽 쉐이핑(Traffic Shaping)을 수행할 수 있으며, AI 호출에 대한 분산 트레이싱을 마이크로서비스 전체 맥락에서 파악할 수 있다.

---

## 3. Routing 정책 및 거버넌스 심층 분석

모델 라우팅은 단순히 "남는 자원으로 보내는 것" 이상의 지능적인 정책 결정 과정을 포함한다.

### 3.1 자동 라우팅 기준
`model: "auto"` 모드를 사용할 때 Foundry는 다음과 같은 지표를 참조한다.
- **Task Type**: 요약, 코드 생성, 대화 등 입력된 프롬프트의 성격을 파악한다.
- **Cost Target**: 가급적 저렴한 모델을 우선하되, 작업의 난이도가 높다고 판단되면 고성능 모델로 승격(Escalation)시킨다.
- **Latency Budget**: 실시간 응답이 중요한 인터랙티브 세션인지, 백그라운드 작업인지에 따라 모델을 선택한다.

### 3.2 Model Router Policy (Azure Policy 거버넌스)
기업 환경에서는 개발자가 임의로 비싼 모델을 사용하여 비용을 폭증시키는 것을 방지해야 한다. Foundry는 Azure Policy와 통합되어 강력한 모델 제어 기능을 제공한다.

- **Approved Registry Models**: IT 관리자는 "Cognitive Services Deployments should only use approved Registry Models"라는 Built-in 정책을 적용하여, 프로젝트 내에서 사용할 수 있는 모델 리스트를 제한할 수 있다.
- **Enforcement 시점**: 정책은 모델 배포(Deployment) 시점뿐만 아니라, 라우터를 통한 런타임 호출 시점에도 검증된다. 정책에 어긋나는 모델로 라우팅을 시도하면 API는 즉시 에러를 반환한다.
- **Portal UX**: 정책이 적용된 프로젝트에서는 포털 UI의 모델 선택 체크박스에서 승인되지 않은 모델들이 비활성화(Disabled) 처리된다.

⚠️ **함정**: Model Router Policy를 적용하거나 수정한 후 실제 API 호출에 반영되기까지는 약 15분의 전파(Propagation) 시간이 필요하다. 정책 수정 직후 테스트를 수행하면 이전 정책이 적용된 결과가 나올 수 있으므로 주의해야 한다.

---

## 🔧 실습: Java에서 Model Router 호출

`azure-ai-agents 2.1.0` SDK를 사용하여 세 가지 방식으로 모델을 호출하고 그 응답을 비교한다.

### pom.xml 설정
```xml
<!-- pom.xml -->
<dependency>
  <groupId>com.azure</groupId>
  <artifactId>azure-ai-agents</artifactId>
  <version>2.1.0</version>
</dependency>
<dependency>
  <groupId>com.azure</groupId>
  <artifactId>azure-identity</artifactId>
  <version>1.18.4</version>
</dependency>
```

### Java 예제: Router Comparison
이 코드는 동일한 질문을 저비용 모드, 최고성능 모드, 그리고 자동 라우팅 모드로 각각 던져 응답의 품질과 사용된 모델 정보를 확인한다.

```java
// RouterApp.java
package com.example;

import com.azure.ai.agents.AgentsClientBuilder;
import com.azure.ai.agents.ResponsesClient;
import com.azure.ai.agents.models.AzureCreateResponseOptions;
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.openai.models.conversations.Conversation;
import com.openai.models.conversations.items.ItemCreateParams;
import com.openai.models.conversations.items.EasyInputMessage;
import com.openai.models.responses.Response;
import com.openai.models.responses.ResponseCreateParams;

import java.util.List;

public class RouterApp {
    public static void main(String[] args) {
        String endpoint = System.getenv("AZURE_FOUNDRY_ENDPOINT");
        
        var builder = new AgentsClientBuilder()
            .endpoint(endpoint)
            .credential(new DefaultAzureCredentialBuilder().build());

        var responsesClient = builder.buildResponsesClient();
        var openAIClient = builder.buildOpenAIClient();

        try {
            // 1. 대화 생성
            Conversation conv = openAIClient.conversations().create();
            String convId = conv.id();

            // 2. 메시지 추가
            openAIClient.conversations().items().create(
                ItemCreateParams.builder()
                    .conversationId(convId)
                    .addItem(EasyInputMessage.builder()
                        .role(EasyInputMessage.Role.USER)
                        .content("양자 역학의 기본 원리를 유치원생도 이해할 수 있게 비유를 들어 설명해줘.")
                        .build())
                    .build()
            );

            // 3. 세 가지 라우팅 방식 호출
            callWithModel(responsesClient, convId, "gpt-5-nano"); // 저비용
            callWithModel(responsesClient, convId, "gpt-5.5");    // 최고성능
            callWithModel(responsesClient, convId, "auto");       // 자동 라우팅

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static void callWithModel(ResponsesClient client, String convId, String modelName) throws Exception {
        System.out.println("\n--- 호출 모드: " + modelName + " ---");
        
        Response response = client.createAzureResponse(
            new AzureCreateResponseOptions(),
            ResponseCreateParams.builder()
                .conversation(convId)
                .model(modelName) // 여기서 라우팅 모델 지정
                .build()
        );

        // Polling (단순화를 위해 동기식 대기)
        String status = response.status().toString();
        while ("in_progress".equals(status) || "queued".equals(status)) {
            Thread.sleep(1000);
            response = client.getAzureResponse(response.id());
            status = response.status().toString();
        }

        // 실제 어떤 모델이 사용되었는지 metadata 확인
        System.out.println("결과 상태: " + status);
        System.out.println("실제 처리 모델: " + response.model()); 
    }
}
```

💡 **팁**: 개발 초기나 스테이징 환경에서는 `model: "auto"`를 사용하여 Foundry가 추천하는 모델 조합을 탐색하고, 안정성이 중요한 프로덕션 환경에서는 성능이 검증된 특정 모델 명칭을 명시하여 라우팅하는 것이 일반적인 전략이다.

### 3.3 Azure Policy 실전 예제 (JSON)

IT 관리자가 특정 프로젝트 그룹에 대해 '승인된 모델 리스트'를 강제하는 정책 정의 예시이다. 이 정책은 `RegistryModel` 속성을 검사하여 허용되지 않은 모델의 배포나 호출을 차단한다.

```json
{
  "properties": {
    "displayName": "Foundry: 허용된 모델 레지스트리만 사용",
    "policyType": "BuiltIn",
    "mode": "Indexed",
    "parameters": {
      "allowedModels": {
        "type": "Array",
        "metadata": {
          "displayName": "허용 모델 리스트",
          "description": "배포 및 라우팅이 허용되는 모델 이름 (예: gpt-5, phi-4)"
        }
      }
    },
    "policyRule": {
      "if": {
        "allOf": [
          { "field": "type", "equals": "Microsoft.CognitiveServices/accounts/deployments" },
          { "field": "Microsoft.CognitiveServices/accounts/deployments/model.name", "notIn": "[parameters('allowedModels')]" }
        ]
      },
      "then": { "effect": "deny" }
    }
  }
}
```

이 정책을 구독(Subscription)이나 리소스 그룹 레벨에 할당하면, 개발자가 `allowedModels`에 포함되지 않은 모델로 라우터를 설정하려 할 때 "Policy violation" 에러와 함께 차단된다.

---

## 4. 배포 유형(Deployment Types) 결정 매트릭스

Foundry는 2026년 기준 10가지에 달하는 배포 유형(SKU)을 제공한다. 이를 비용, 처리량 예측가능성, 데이터 컴플라이언스(Zone)라는 세 가지 축으로 분류한 결정 트리는 다음과 같다.

### 4.1 배포 유형 분류
1.  **Global Standard (Pay-per-token)**
    - **특징**: 가장 일반적인 방식. 쓴 만큼만 낸다. 전 세계 Azure 인프라를 활용하여 가장 높은 쿼터(TPM)를 제공한다.
    - **용도**: 트래픽 변동이 심한 일반 웹 서비스.
2.  **Global Provisioned (PTU)**
    - **특징**: 일정한 처리량(Throughput)을 예약하여 점유한다. 응답 지연 시간(Latency)이 일정하게 보장된다.
    - **용도**: SLA가 중요한 엔터프라이즈 핵심 시스템.
3.  **Data Zone (Standard/Provisioned/Batch)**
    - **특징**: 데이터가 특정 구역(US 또는 EU) 밖으로 나가지 않음을 보장한다.
    - **용도**: GDPR 등 데이터 주권 준수가 필요한 규제 산업.
4.  **Standard (Regional)**
    - **특징**: 단일 리전(예: Korea Central)에 고정된다. 쿼터가 적을 수 있다.
    - **용도**: 지연 시간을 극도로 줄여야 하거나 로컬 데이터 규정이 엄격한 경우.
5.  **Global Batch**
    - **특징**: 실시간 응답이 필요 없는 대량 작업을 24시간 내에 처리한다. 비용이 50% 저렴하다.
    - **용도**: 대규모 문서 요약, 데이터셋 라벨링.

### 4.2 결정 매트릭스 (Decision Tree)
- **질문 1: 데이터가 특정 국가/대륙 내에 머물러야 하는가?**
  - YES → Regional Standard 또는 Data Zone SKU 선택.
  - NO → Global SKU 선택 (더 높은 쿼터와 최신 모델 가용성 확보).
- **질문 2: 초당 처리량(Throughput) 보장이 필수적인가?**
  - YES → Provisioned (PTU) 선택.
  - NO → Standard (Pay-per-token) 선택.
- **질문 3: 실시간 응답(Interactive)이 필요한가?**
  - YES → Standard 또는 Provisioned.
  - NO → Batch (비용 50% 절감).

---

## 5. PTU (Provisioned Throughput Unit) 전략

PTU는 성능을 "돈으로 예약하는" 방식이다. 단순히 비싼 것이 아니라, 트래픽 규모에 따라 Standard보다 더 경제적일 수 있다.

### 5.1 PTU 계산법 및 손익분기점(Break-even Point)
Foundry 포털에서 제공하는 **PTU 계산기**를 사용하여 필요한 유닛 수를 산정한다. 
- 입력 값: 예상 평균 프롬프트 토큰, 응답 토큰, 초당 요청 수(RPS), 최대 피크 부하.
- **손익분기점**: 일반적으로 특정 모델의 TPM(Tokens Per Minute) 사용량이 전체 시간의 약 60% 이상 일정하게 유지될 때, Pay-per-token 방식보다 PTU 예약 방식이 더 저렴해진다.

### 5.2 예약 기간과 위약금
PTU는 월간(Monthly) 또는 연간(Yearly) 약정(Commitment)을 통해 추가 할인을 받을 수 있다. 

⚠️ **함정**: PTU 예약 기간은 물리적인 Lock-in을 의미한다. 비즈니스 수요를 잘못 예측하여 중간에 예약을 해지할 경우 남은 기간에 대한 위약금이 발생할 수 있으므로, 초기에는 소량의 PTU만 예약하고 트래픽 추이에 따라 확장하는 전략을 권장한다.

### 5.3 PTU 관리 및 모니터링 자동화 (Java)

PTU(Provisioned Throughput Unit)는 고정된 대역폭을 예약하므로, 예약된 대역폭을 초과하는 트래픽이 들어오면 `429 Too Many Requests`가 발생한다. 이를 지능적으로 처리하는 Java 로직을 구현한다.

```java
// PtuManager.java
package com.example;

import com.azure.ai.openai.models.ChatCompletionsOptions;
import com.azure.core.util.Context;

public class PtuManager {
    private final ResponsesClient ptuClient;
    private final ResponsesClient standardClient;

    public PtuManager(ResponsesClient ptu, ResponsesClient standard) {
        this.ptuClient = ptu;
        this.standardClient = standard;
    }

    public Response executeSmartRouting(String convId) {
        try {
            // 1. 우선적으로 예약된 PTU 엔드포인트 사용
            return ptuClient.createAzureResponse(
                new AzureCreateResponseOptions().setSkuOverride("Provisioned"),
                ResponseCreateParams.builder().conversation(convId).build()
            );
        } catch (HttpResponseException e) {
            // 2. PTU 쿼터가 꽉 찼을 경우(429), Standard(종량제)로 즉시 우회
            if (e.getResponse().getStatusCode() == 429) {
                System.out.println("PTU 대역폭 초과. Standard SKU로 오버플로우 처리합니다.");
                return standardClient.createAzureResponse(
                    new AzureCreateResponseOptions().setSkuOverride("Standard"),
                    ResponseCreateParams.builder().conversation(convId).build()
                );
            }
            throw e;
        }
    }
}
```

이 패턴을 **'PTU Overflow to Standard'**라고 부르며, 베이스라인 트래픽은 저렴하고 성능이 보장된 PTU로 처리하고, 순간적인 피크 트래픽만 Standard로 처리하여 비용과 가용성을 동시에 잡는 엔터프라이즈의 표준 기법이다.

---

## 5.4 Smart Load Balancing: 가중치 기반 라우팅 (Weighted Routing)

Model Router의 `auto` 모드 외에도, 개발자가 직접 여러 리전이나 모델 간의 트래픽 비중을 조절해야 할 때가 있다. 예를 들어, 새 모델(gpt-5.5)을 도입할 때 기존 모델(gpt-5)과 8:2 비중으로 테스트해보고 싶을 수 있다.

- **작동 원리**: 어플리케이션 레이어에서 가중치(Weight)를 계산하여 라우터에게 특정 모델 명칭을 전달한다.
- **Java 구현 예시**:

```java
// WeightedRouter.java
public class WeightedRouter {
    private final Random random = new Random();

    public String selectModel() {
        int chance = random.nextInt(100);
        if (chance < 20) {
            return "gpt-5.5-preview"; // 20% 트래픽
        } else {
            return "gpt-5-stable";    // 80% 트래픽
        }
    }
}
```

Foundry 2026-06 업데이트에서는 이러한 가중치 기반 라우팅을 포털 설정(Traffic Splitting)만으로도 가능하게 지원하기 시작했다. 코드 수정 없이 포털에서 슬라이더를 조절하여 `Canary Deployment`를 수행할 수 있다.

---

## 6. Region · Quota · Failover 및 서킷 브레이커 전략

하나의 리전에만 의존하는 시스템은 해당 리전의 장애나 쿼터 소진에 취약하다. 특히 생성형 AI 서비스는 모델의 업데이트나 리전별 점검으로 인해 일시적인 가용성 저하가 발생할 확률이 일반 API보다 높다.

### 6.1 Multi-region Failover 패턴
가장 권장되는 패턴은 **Primary (East US)** + **Backup (Sweden Central)** 조합이다. 
- **East US**: 최신 모델(GPT-5.5 등)이 가장 먼저 배포되고 기본 할당 쿼터가 가장 넉넉하다.
- **Sweden Central**: 유럽 리전 중 인프라가 매우 안정적이며, 북미 리전 장애 시 훌륭한 대체제가 된다.
- **데이터 위치 정책**: 만약 데이터 주권이 중요하다면 동일 구역 내(예: East US + West US 3)에서 페일오버를 구성해야 한다.

### 6.2 Resilience4j를 이용한 지능형 서킷 브레이커 구현

단순한 `try-catch` 페일오버는 "장애 리전"에 지속적으로 요청을 보내 시스템 전체의 부하를 높이는 단점이 있다. Java 진영의 표준 라이브러리인 **Resilience4j**를 사용하여 장애 발생 시 해당 리전으로의 통로를 일시적으로 차단하고 백업 리전으로 즉시 전환하는 구조를 설계한다.

```java
// CircuitBreakerApp.java
package com.example;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import java.time.Duration;
import java.util.function.Supplier;

public class CircuitBreakerApp {
    private final ResponsesClient primaryClient;
    private final ResponsesClient secondaryClient;
    private final CircuitBreaker circuitBreaker;

    public CircuitBreakerApp(ResponsesClient primary, ResponsesClient secondary) {
        this.primaryClient = primary;
        this.secondaryClient = secondary;

        // 1. 서킷 브레이커 설정: 5번 중 3번 실패(60%) 시 서킷 오픈 (10초간 대기)
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(60)
            .waitDurationInOpenState(Duration.ofSeconds(10))
            .slidingWindowSize(5)
            .recordExceptions(HttpResponseException.class)
            .build();

        this.circuitBreaker = CircuitBreakerRegistry.of(config).circuitBreaker("ai-router");
    }

    public Response getResponseWithFailover(String convId) {
        // 2. 서킷 브레이커를 통한 Primary 호출 실행
        Supplier<Response> primarySupplier = CircuitBreaker.decorateSupplier(circuitBreaker, () -> {
            return primaryClient.createAzureResponse(
                new AzureCreateResponseOptions(),
                ResponseCreateParams.builder().conversation(convId).model("auto").build()
            );
        });

        try {
            return primarySupplier.get();
        } catch (Exception e) {
            // 3. 서킷이 열렸거나 Primary 호출 실패 시 Secondary 호출 (Fallback)
            System.out.println("Primary 장애 감지. Secondary 리전으로 요청을 우회합니다. 원인: " + e.getMessage());
            return secondaryClient.createAzureResponse(
                new AzureCreateResponseOptions(),
                ResponseCreateParams.builder().conversation(convId).model("auto").build()
            );
        }
    }
}
```

- **Open State (열림)**: 에러가 임계치를 넘으면 모든 요청이 Primary로 가지 않고 즉시 Fallback(Secondary)으로 전달된다.
- **Half-Open (반열림)**: 일정 시간이 지나면 한두 개의 요청만 Primary로 보내 상태가 호전되었는지 확인한다.
- **Closed (닫힘)**: Primary가 정상화되면 다시 모든 트래픽을 Primary로 복구한다.

### 6.3 리전 간 쿼터 이동 (Self-Service Quota)
2026-06 출시된 기능을 통해, Foundry Management Center에서 **'Quota Trading'**이 가능해졌다. 예를 들어 `East US`의 gpt-5 쿼터가 남고 `West US 2`가 부족하다면, 관리자는 티켓을 생성할 필요 없이 포털 상에서 즉시 쿼터를 리전 간에 재배치하여 비용과 가용성을 최적화할 수 있다.

---

## 7. Global Batch API (50% 할인) 심층 분석

실시간성이 필요 없는 데이터 처리 작업은 Batch API를 사용하는 것이 압도적으로 유리하다.

### 7.1 작동 흐름
1.  **JSONL 생성**: 각 행이 개별 API 요청인 `.jsonl` 파일을 작성한다.
2.  **Upload**: 파일을 Foundry 스토리지에 업로드한다.
3.  **Submit**: Batch Job을 생성하여 제출한다.
4.  **Poll**: 작업 상태가 `completed`가 될 때까지 기다린다 (최대 24시간 SLA).
5.  **Download**: 결과 파일을 내려받아 파싱한다.

### 7.2 Java 예제: Batch Job 제출
```java
// BatchApp.java
package com.example;

import com.azure.ai.openai.OpenAIClient;
import com.azure.ai.openai.models.BatchCreateOptions;
import com.azure.ai.openai.models.BatchJob;
import java.util.Collections;

public class BatchApp {
    public void submitBatch(OpenAIClient client, String fileId) {
        // Global Batch 엔드포인트는 /openai/v1/batches 사용
        BatchCreateOptions options = new BatchCreateOptions(
            fileId, 
            "/openai/v1/chat/completions" // 대상 API 경로
        );
        
        options.setMetadata(Collections.singletonMap("purpose", "large_scale_eval"));

        BatchJob job = client.createBatchJob(options);
        System.out.println("배치 작업 시작: " + job.getId());
        
        // 이후 주기적으로 getBatchJob(jobId) 호출하여 상태 확인
    }
}
```

💡 **팁**: 대규모 모델 평가(Evaluation)를 수행할 때 Batch API를 사용하면 비용을 절반으로 줄이면서도 프로덕션 쿼터에 영향을 주지 않고 안전하게 대량의 테스트를 수행할 수 있다.

---

## 7. Global Batch API (50% 할인) 심층 활용

실시간성이 필요 없는 데이터 처리 작업은 Batch API를 사용하는 것이 압도적으로 유리하다. 단순히 비용 절감을 넘어, 프로덕션용 실시간 트래픽 쿼터(Rate Limit)를 전혀 소모하지 않는 '격리된 실행(Isolated Execution)' 환경을 제공한다는 점이 큰 장점이다.

### 7.1 작동 흐름 및 JSONL 구조 상세
Batch API는 개별 요청을 한 줄씩 JSON 객체로 담은 `.jsonl` 파일을 입력으로 받는다.

- **JSONL 구조 예시**:
```json
{"custom_id": "req-001", "method": "POST", "url": "/chat/completions", "body": {"model": "gpt-5", "messages": [{"role": "user", "content": "이 문장을 요약해줘: ..."}]}}
{"custom_id": "req-002", "method": "POST", "url": "/chat/completions", "body": {"model": "gpt-5", "messages": [{"role": "user", "content": "이 데이터를 분석해줘: ..."}]}}
```
각 요청은 `custom_id`를 통해 결과 파일에서 식별할 수 있다.

### 7.2 고성능 배치 모니터 및 결과 처리 (Java)

대량의 배치 작업은 완료까지 수 시간이 걸릴 수 있다. 다음은 Java에서 지수 백오프를 적용한 모니터링 및 결과 자동 다운로드 클래스 예시이다.

```java
// AdvancedBatchMonitor.java
package com.example;

import com.azure.ai.openai.OpenAIClient;
import com.azure.ai.openai.models.BatchJob;
import com.azure.ai.openai.models.BatchJobStatus;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.nio.file.StandardCopyOption;

public class AdvancedBatchMonitor {
    private final OpenAIClient client;

    public AdvancedBatchMonitor(OpenAIClient client) {
        this.client = client;
    }

    public void waitForCompletion(String jobId) throws Exception {
        int waitSeconds = 60; // 초기 대기 1분
        BatchJob job;

        while (true) {
            job = client.getBatchJob(jobId);
            System.out.printf("[Batch %s] Status: %s (Progress: %d/%d)\n", 
                jobId, job.getStatus(), job.getCompletedCount(), job.getTotalCount());

            if (job.getStatus() == BatchJobStatus.COMPLETED) {
                downloadResult(job.getOutputFileId(), "success_results.jsonl");
                break;
            } else if (job.getStatus() == BatchJobStatus.FAILED) {
                System.err.println("Batch 실패: " + job.getErrors());
                if (job.getErrorFileId() != null) {
                    downloadResult(job.getErrorFileId(), "error_details.jsonl");
                }
                break;
            }

            // 지수 백오프: 대기 시간을 점진적으로 늘리되 최대 30분으로 제한
            Thread.sleep(waitSeconds * 1000);
            waitSeconds = Math.min(waitSeconds * 2, 1800);
        }
    }

    private void downloadResult(String fileId, String fileName) throws Exception {
        InputStream stream = client.getFileContent(fileId);
        Files.copy(stream, Paths.get(fileName), StandardCopyOption.REPLACE_EXISTING);
        System.out.println("결과 파일 저장 완료: " + fileName);
    }
}
```

### 7.3 Batch API 사용 시 주의사항: 에러 관리
배치 작업 도중 특정 행(Row)에서 에러가 발생하더라도 전체 작업이 중단되지 않는다. 
- **Partial Success**: 10,000건 중 10건이 토큰 한도 초과로 실패하더라도 나머지 9,990건은 완료된다.
- **Retry Strategy**: 결과 파일의 `status`를 체크하여 실패한 `custom_id`만 모아 다시 배치 파일을 구성하는 로직이 필요하다.

---

## 8. Operational Monitoring & FinOps for Model Router

Model Router를 프로덕션에 도입하면, 비용과 성능에 대한 실시간 가시성 확보가 최우선 과제가 된다.

### 8.1 FinOps: 모델별 비용 대시보드 구축
Router가 여러 모델을 섞어 쓸 때, 월말 청구서만 보고서는 어떤 모델이 비용의 주범인지 파악하기 어렵다.
- **Tagging Strategy**: 각 배포 모델에 `CostCenter`, `ProjectID` 태그를 부여한다.
- **Usage Metrics**: Azure Monitor의 `InferenceTokens` 메트릭을 `ModelName` 차원으로 분할하여 대시보드를 구성한다.
- **Budget Alert**: 라우터 전체 예산의 80%가 소진되면 자동으로 `auto` 모드에서 강제적으로 `low-cost models`만 사용하도록 구성을 변경하는 자동화 스크립트를 운영팀에서 보유해야 한다.

### 8.2 Observability: OpenTelemetry 기반 트레이싱
자바 앱에서 `azure-core-tracing-opentelemetry`를 사용하면 다음과 같은 호출 체인을 시각화할 수 있다.
1. `Client App` -> `Model Router (auto)`
2. `Model Router` -> `GPT-5.5 (East US)` (Inference)
3. `Response Received`
이 트레이스를 통해 "왜 라우터가 이 질문에 gpt-5.5를 선택했는가?"에 대한 근거를 역추적할 수 있다.

---

## 8.3 Blue/Green 및 Canary 배포 전략 (Advanced)

모델의 성능이 실시간 서비스에 미치는 영향이 큰 만큼, 무중단 상태에서 안전하게 신규 모델을 배포하고 검증하는 전략이 필수적이다.

### 1) Blue/Green Deployment
- **Blue**: 현재 운영 중인 모델 버전 (예: gpt-5 v1).
- **Green**: 새로 배포할 모델 버전 (예: gpt-5 v1.1).
- **방법**: Green 모델을 별도의 엔드포인트에 배포하고 완벽히 테스트한 뒤, Model Router의 타겟을 Blue에서 Green으로 일시에 전환한다.
- **장점**: 문제 발생 시 즉시 Blue로 롤백이 가능하며, 다운타임이 없다.

### 2) Canary Release (Traffic Splitting)
- **방법**: 신규 모델(Canary)에게 전체 트래픽의 극소량(예: 1~5%)만 먼저 흘려보내고, 오류율과 지연 시간을 모니터링하며 비중을 점진적으로 늘린다.
- **Java 구현 전략**: 어플리케이션 레이어에서 사용자의 세션 ID를 해싱(Hashing)하여 특정 사용자는 항상 동일한 모델 버전(Stickiness)을 보게 설계한다.

```java
// CanaryRouter.java
public String selectDeployment(String userId) {
    // 사용자 ID 기반으로 0~99 사이의 버킷 결정
    int bucket = Math.abs(userId.hashCode()) % 100;
    
    if (bucket < 5) { // 5% 사용자는 Canary(v1.1) 모델 사용
        return "gpt-5-v1-1-canary";
    } else {
        return "gpt-5-v1-stable";
    }
}
```

---

## 8.4 모델 성능 벤치마크 및 비교 방법론

Router가 어떤 모델을 선택할지 결정하기 전, 혹은 `auto` 모드의 성능을 검증하기 위해 자체적인 벤치마크 파이프라인이 필요하다.

1.  **데이터셋 준비**: 실제 사용자의 과거 질문 셋(최소 100건 이상)을 준비한다.
2.  **병렬 실행**: 동일한 질문 셋을 각 모델(gpt-5, gpt-4o, phi-4 등)에게 동시에 던진다. (Batch API 활용 권장)
3.  **지표 측정**:
    - **유효성(Validity)**: 답변이 형식을 지켰는가? (JSON Schema 등)
    - **정확성(Accuracy)**: Ground Truth와 얼마나 일치하는가?
    - **비용 효율성**: 1,000 토큰당 처리 비용 대비 품질 비율.
4.  **Router Threshold 설정**: 벤치마크 결과를 바탕으로 "이 정도 난이도의 질문은 gpt-5-nano로 충분하다"는 임계치(Threshold)를 설정하여 라우터의 효율을 극대화한다.

---

## 8.5 Advanced Routing Patterns: Context-Aware Routing (심화)

단순히 가중치나 라우팅 모드에 의존하는 것을 넘어, 입력된 프롬프트의 '맥락'과 '복잡도'를 어플리케이션 레이어에서 1차 분석한 후 최적의 경로를 결정하는 고급 패턴이다.

1.  **Complexity Scorer**: 사용자의 질문이 들어오면 매우 가벼운 모델(예: Phi-4)이나 임베딩 유사도 검색을 통해 질문의 난이도를 0~1 사이로 점수화한다.
2.  **Route Dispatcher**:
    - 점수 < 0.3 (단순 문답): `gpt-5-nano`로 전송.
    - 0.3 <= 점수 < 0.7 (중간 난이도): `gpt-5` 표준 모델로 전송.
    - 점수 >= 0.7 (복잡한 추론/수학/코드): `gpt-5.5` 또는 `o-series` 모델로 전송.

이러한 패턴은 `auto` 모드보다 더 엄격하게 비용을 통제해야 하는 대규모 서비스에서 유효하며, Java의 `Filter`나 `Interceptor` 패턴을 사용하여 API 게이트웨이 레벨에서 구현하는 것이 정석이다.

---

## 8.6 모델 업데이트 및 라이프사이클 관리 (N-1 전략)

Foundry의 모델들은 주기적으로 업데이트되며, 이전 버전은 지원이 종료(End of Life)된다. 프로덕션 환경에서는 이를 관리하기 위한 **'N-1 전략'**을 권장한다.

- **N-1 전략**: 항상 가장 최신 버전(N)과 바로 직전의 안정 버전(N-1) 두 가지를 동시에 배포 상태로 유지한다.
- **업데이트 흐름**:
    1. 새 모델 버전 출시 시 `auto` 모드나 `latest` 태그를 사용하는 테스트 환경에서 먼저 검증한다.
    2. 검증 완료 후 Canary 방식으로 실서비스의 10% 트래픽을 신규 버전으로 돌린다.
    3. 일주일간 이상이 없으면 전체 트래픽을 신규 버전으로 전환하고, 구구버전(N-2) 배포를 제거한다.
- **Java SDK 대응**: `deploymentName`을 코드에 하드코딩하지 않고, 설정 파일이나 환경 변수에서 동적으로 읽어오도록 설계하여 재배포 없이 모델 버전을 교체할 수 있어야 한다.

---

## 8.7 Model Router & Semantic Caching 통합 전략

Model Router를 통과하는 요청의 비용과 지연 시간을 획기적으로 줄이는 또 다른 방법은 **Semantic Caching**을 도입하는 것이다. 

- **작동 원리**: 사용자의 질문이 들어오면 모델에게 바로 보내지 않고, 벡터 DB(Redis, Cosmos DB 등)에 저장된 과거의 유사한 질문과 답변이 있는지 먼저 확인한다. 
- **효과**: 완전히 동일한 질문이 아니더라도 '의미적으로 유사한' 질문에 대해 미리 저장된 고품질 답변을 즉시 반환함으로써, 모델 호출 비용(Router Cost)을 0으로 만들고 지연 시간을 수 밀리초(ms) 단위로 단축할 수 있다.

```java
// SemanticCacheInterceptor.java
public Response getCachedResponse(String userQuery) {
    // 1. 임베딩 모델을 통해 질문을 벡터로 변환
    float[] queryVector = embeddingClient.embed(userQuery);
    
    // 2. 벡터 DB에서 유사도 0.95 이상의 결과 탐색
    CacheResult cached = vectorDb.searchSimilar(queryVector, 0.95);
    
    if (cached != null) {
        System.out.println("캐시 적중! 모델 호출을 건너뜁니다.");
        return cached.toResponse();
    }
    
    // 3. 캐시 미스 시 Model Router 호출
    return routerClient.execute(userQuery);
}
```

### 8.8 Token Counting & Budget Guardrails (Java)

실시간 트래픽이 발생하는 환경에서는 모델 호출 전에 예상 비용을 계산하고 차단하는 '예산 가드레일'이 필요하다. Java 환경에서는 **JTokkit**과 같은 라이브러리를 사용하여 로컬에서 정확한 토큰 수를 계산할 수 있다.

```java
// BudgetGuardrail.java
public void checkBudget(String prompt) {
    Encoding registry = Encodings.newDefaultEncodingRegistry();
    Encoding enc = registry.getEncoding(EncodingType.CL100K_BASE);
    
    int tokenCount = enc.countTokens(prompt);
    double estimatedCost = (tokenCount / 1000.0) * CURRENT_MODEL_PRICE;
    
    if (estimatedCost > USER_SESSION_BUDGET) {
        throw new BudgetExceededException("이번 요청은 세션 예산을 초과합니다. 질문을 줄여주세요.");
    }
}
```

라우팅 전 이러한 '사전 검증' 단계를 거치면, 악의적인 대량 요청(Token Exhaustion Attack)으로부터 시스템을 보호하고 운영 비용의 예측 가능성을 높일 수 있다.

---

## 9. ⚠️ 함정: 실무에서 밟기 쉬운 지뢰들

1.  **`auto` 모드의 불확실성**: 자동 라우팅은 편리하지만, 때로는 사용자가 원하지 않는 성능의 모델로 라우팅될 수 있다. 특히 프롬프트의 미묘한 차이가 중요한 정교한 작업에서는 프로덕션 적용 전 충분한 벤치마크가 필요하다.
2.  **Data Zone 컴플라이언스**: Global SKU를 사용하면서 데이터 위치 정책(Data Residency)을 "US Zone"으로 설정하면 상충하는 설정으로 인해 배포가 실패하거나 규제 위반이 발생할 수 있다. 조직의 데이터 정책을 먼저 확인하라.
3.  **Batch API의 타임아웃**: Batch 작업은 최대 24시간의 SLA를 가진다. 10분 내에 끝날 것이라고 예상하고 Polling 로직을 짧게 가져가면 불필요한 API 호출 비용만 발생한다. 지수 백오프 기반의 여유 있는 Polling이 필요하다.
4.  **Quota Sharing**: 같은 프로젝트 내의 여러 배포가 동일한 쿼터 풀을 공유한다. Model Router를 쓰더라도 물리적인 전체 쿼터 한도는 변하지 않으므로, 전체 사용량을 모니터링해야 한다.

---

## 💡 팁

1.  **Partial PTU 예약**: 전체 트래픽을 PTU로 감당하기보다, 베이스라인 트래픽(예: 하위 40%)만 PTU로 예약하고 나머지 피크 트래픽은 Standard(Pay-per-token)로 흘려보내는 하이브리드 전략이 가장 가성비가 높다.
2.  **Region 간 Quota 이동**: 특정 리전에 쿼터가 남고 다른 리전이 부족하다면 Azure Support를 통하지 않고도 Foundry 포털의 Quota 관리 화면에서 리전 간 쿼터 재할당을 시도할 수 있다 (일부 SKU 한정).
3.  **Batch API로 비용 절감**: 야간 시간대에 수행해도 되는 벡터 임베딩 생성이나 과거 데이터 분석 작업은 무조건 Batch API로 돌려라. 연간 단위로 합산하면 무시할 수 없는 비용 차이가 발생한다.

---

## 🔴 2026 변경 사항

- **Model Router GA**: 베타 기간을 거쳐 정식 서비스로 전환되었다.
- **Responses API 기반**: 모든 라우팅은 Responses API v2 인터페이스를 통해 일관되게 관리된다.
- **Policy Governance**: Azure Policy를 통해 개발자의 모델 선택권을 제어하는 기능이 정식 도입되었다.
- **신규 배포 유형**: `GlobalProvisionedManaged`, `DataZoneBatch` 등 세분화된 SKU가 추가되어 선택의 폭이 넓어졌다.

---

## 📚 더 읽기

- [Model Router Deployment Guide](https://learn.microsoft.com/en-us/azure/foundry/how-to/deploy-model-router)
- [Governing AI with Azure Policy](https://learn.microsoft.com/en-us/azure/foundry/how-to/model-router-policy)
- [Azure OpenAI PTU Calculator](https://aka.ms/azure-openai-ptu-calculator)
- [Understanding Global and Regional SKU Quotas](https://learn.microsoft.com/en-us/azure/foundry/concepts/quotas)
- [Global Batch API Reference](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/batch)

---

[← Ch.7 Agent Service](Ch07_Agent_Service.md) | [Ch.9 Evaluation & Observability →](Ch09_Evaluation.md)
