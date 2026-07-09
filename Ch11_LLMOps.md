# Chapter 11. Production CI/CD & LLMOps

[← 목차로](README.md)

> **학습 목표**
> - LLMOps의 핵심 4대 축(IaC, 버전 관리, Evaluation-gate, Observability)을 이해하고 설계한다.
> - Bicep과 Terraform을 사용하여 Foundry 리소스를 코드로 관리(Infrastructure as Code)한다.
> - GitHub Actions를 활용해 Java 애플리케이션 빌드부터 모델 평가, 배포까지 이어지는 전 프로세스를 자동화한다.
> - Canary, Blue-Green, Shadow 배포 전략을 LLM 환경에 맞게 적용하고 자동 롤백 체계를 구축한다.
> - Federated Identity(OIDC)를 통해 비밀번호 없는 보안 파이프라인을 구현한다.

> **전제 조건**
> - [← Ch.10 Security · Governance](Ch10_Security_Governance.md) 과정의 보안 아키텍처 및 RBAC 지식
> - [← Ch.9 Evaluation](Ch09_Evaluation.md) 과정의 평가 지표 및 Python Runner 활용 능력
> - [← Ch.8 Model Router](Ch08_Model_Router.md) 과정의 배포 유형 및 라우팅 이해
> - GitHub 계정 및 Azure 구독 (Contributor 이상의 권한)

---

## 1. LLMOps: 프로덕션으로 가는 유일한 길

실험실에서 잘 작동하던 챗봇이 실제 서비스에 투입되는 순간, 개발팀은 "프롬프트를 바꿨는데 이전보다 답변이 나빠지면 어떡하지?", "인프라 설정을 누가 실수로 건드리면?", "비용이 갑자기 폭증하면?" 같은 실무적 공포에 직면한다. 이를 해결하는 체계가 바로 **LLMOps**다.

### 1.1 MLOps와 LLMOps의 결정적 차이
전통적인 MLOps는 '모델 학습(Training)'과 '데이터 드리프트' 관리에 집중한다. 반면, Foundry 기반의 LLMOps는 이미 완성된 기초 모델(Foundation Models)을 사용하므로 학습 과정이 거의 없다. 대신 다음의 구성 요소들을 어떻게 관리하느냐가 핵심이다.

- **Infrastructure**: Foundry Resource, Project, Model Deployment의 일관된 생성 (IaC).
- **Prompt & Agent**: 코드와 분리된 프롬프트 버전 및 에이전트 설정(Agent Versions) 관리.
- **Evaluation**: 배포 전 '골든셋(Golden Set)' 테스트를 통과했는지 검증하는 Gate.
- **Observability**: 실시간 토큰 비용, Latency, 그리고 답변의 유해성을 모니터링하고 문제 시 즉시 롤백.

### 1.2 LLMOps의 4대 축
1.  **IaC (Infrastructure as Code)**: Bicep이나 Terraform으로 인프라를 정의하여 환경 간 '설정 드리프트'를 방지한다.
2.  **Prompt/Agent 버전 관리**: Foundry CMS나 Git을 통해 실험과 운영 버전을 엄격히 분리한다.
3.  **Evaluation-Gated Deployment**: 평가 점수가 임계치(Threshold) 미만이면 배포 파이프라인을 중단한다.
4.  **Observability & Feedback Loop**: 운영 로그를 다시 평가 데이터셋(Trace-to-Dataset)으로 전환하여 시스템을 개선한다.

### 1.3 왜 Java 팀에게 LLMOps가 더 중요한가?

대부분의 LLM 실험은 Python 워크스테이션에서 시작되지만, 엔터프라이즈의 핵심 시스템은 Java(Spring Boot, Jakarta EE)로 구축되어 있다. Java 팀이 LLMOps를 도입해야 하는 결정적인 이유는 '안정성'과 '예측 가능성'이다.

- **Type Safety meets AI**: Java의 강력한 타입 시스템은 프롬프트 입출력의 구조를 엄격하게 관리(Pydantic vs Zod vs Java Records)할 수 있게 해준다. LLMOps는 이러한 코드 레벨의 엄격함을 인프라와 배포 영역까지 확장한다.
- **Microservices Orchestration**: 대규모 Java 마이크로서비스 환경에서 AI 기능을 추가할 때, 개별 서비스의 배포 주기와 AI 모델의 업데이트 주기를 맞추는 것은 매우 어렵다. LLMOps 파이프라인은 이 조율을 자동화한다.
- **Enterprise Standards**: 이미 조직 내에 구축된 CI/CD 표준(Jenkins, Azure DevOps, GitHub Actions)과 보안 표준(OAuth2, OIDC)을 LLM 워크로드에 그대로 이식할 수 있다.

---

## 2. 🔧 실습 1 — Bicep으로 Foundry 리소스 IaC

Azure 전용 언어인 Bicep은 Foundry의 복잡한 계층 구조를 가장 깔끔하게 표현한다.

### 2.1 멀티 리전(Multi-region) 가용성 아키텍처

실무에서는 단일 리전 배포만으로 부족하다. 특정 리전의 OpenAI 서비스가 장애가 나거나 Quota가 소진될 경우를 대비해 멀티 리전 아키텍처를 IaC로 자동화해야 한다.

```bicep
// multi-region.bicep
param locations array = [
  'eastus'
  'swedencentral'
]

module foundryResources './foundry-base.bicep' = [for loc in locations: {
  name: 'foundry-${loc}'
  params: {
    location: loc
    foundryName: 'foundry-${loc}-${uniqueString(loc)}'
  }
}]
```

이렇게 모듈화된 Bicep 파일을 사용하면, 단 한 줄의 명령어로 전 세계 여러 리전에 동일한 보안 정책과 모델 배포 설정을 복제할 수 있다. 이는 수동 설정으로는 불가능한 '인프라의 복제 가능성(Reproducibility)'을 보장한다.

### 2.2 상태 관리(State Management)와 드리프트 탐지

IaC의 진정한 가치는 생성(Create)이 아니라 변경(Update)에 있다.
- **What-if 분석**: 배포 전 `az deployment group what-if` 명령을 실행하여, 현재의 클라우드 리소스와 Bicep 코드 간의 차이를 미리 확인한다. 이를 통해 실수로 중요한 배포(Deployment)를 삭제하는 사고를 방지한다.
- **리소스 잠금(Resource Locks)**: 실수로 프로덕션 리소스를 삭제하는 것을 막기 위해 `CanNotDelete` 잠금을 IaC 코드 내에 명시적으로 정의한다.

---

```bicep
// main.bicep
param location string = resourceGroup().location
param foundryName string = 'foundry-${uniqueString(resourceGroup().id)}'
param projectName string = 'foundry-project-prod'

// 1. Foundry Resource (AI Services Account)
resource aiAccount 'Microsoft.CognitiveServices/accounts@2024-10-01' = {
  name: foundryName
  location: location
  sku: {
    name: 'S0'
  }
  kind: 'AIServices'
  properties: {
    customSubDomainName: foundryName
    publicNetworkAccess: 'Enabled' // 실무에서는 Ch.10에 따라 Disabled 권장
  }
}

// 2. GPT-5 Model Deployment
resource gpt5Deployment 'Microsoft.CognitiveServices/accounts/deployments@2024-10-01' = {
  parent: aiAccount
  name: 'gpt-5-prod'
  properties: {
    model: {
      format: 'OpenAI'
      name: 'gpt-5'
      version: '2025-08'
    }
    versionUpgradeOption: 'OnceNewDefaultVersionAvailable'
  }
  sku: {
    name: 'GlobalStandard'
    capacity: 50 // 50K TPM
  }
}

// 3. Foundry Project
resource foundryProject 'Microsoft.MachineLearningServices/workspaces@2024-10-01-preview' = {
  name: projectName
  location: location
  kind: 'Project'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    hubResourceId: aiAccount.id
    friendlyName: 'Production AI Project'
  }
}

output foundryEndpoint string = aiAccount.properties.endpoint
output projectResourceId string = foundryProject.id
```

### 배포 실행
```bash
# bash
az deployment group create \
  --resource-group rg-llmops-prod \
  --template-file main.bicep
```

---

## 3. 🔧 실습 2 — Terraform 대안

멀티 클라우드 전략을 사용하거나 기존에 HCL(HashiCorp Configuration Language)을 주력으로 사용한다면 Terraform이 대안이다. `azurerm` 프로바이더의 최신 기능을 활용한다.

```hcl
# main.tf
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "rg" {
  name     = "rg-foundry-tf"
  location = "East US"
}

# Foundry Resource
resource "azurerm_ai_services" "foundry" {
  name                = "foundry-tf-resource"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku_name            = "S0"
}

# GPT-5 Deployment (azapi를 통한 최신 스펙 반영)
resource "azapi_resource" "gpt5_deployment" {
  type      = "Microsoft.CognitiveServices/accounts/deployments@2024-10-01"
  name      = "gpt-5-tf"
  parent_id = azurerm_ai_services.foundry.id

  body = jsonencode({
    properties = {
      model = {
        format  = "OpenAI"
        name    = "gpt-5"
        version = "2025-08"
      }
    }
    sku = {
      name     = "GlobalStandard"
      capacity = 50
    }
  })
}
```

💡 **팁**: 왜 Bicep인가, Terraform인가?
Azure 전용 기능을 가장 빠르게 반영(Day-0 support)하고 상태 파일(State file) 관리 부담을 줄이고 싶다면 **Bicep**을 추천한다. 반면 AWS/GCP 등과 함께 인프라를 통합 관리하고 팀의 숙련도가 HCL에 있다면 **Terraform**이 유리하다. Foundry는 두 방식 모두 완벽히 지원한다.

### 3.3 Terraform을 이용한 하이브리드/멀티 클라우드 운영 아키텍처

Azure뿐만 아니라 다른 클라우드 서비스(AWS, GCP)와 함께 사용하는 하이브리드 환경에서는 Terraform의 'Provider' 추상화가 강력한 힘을 발휘한다.

#### 1) 변수 관리와 환경 분리 (Variable Management)
운영(Prod), 스테이징(Stage), 개발(Dev) 환경을 동일한 코드로 관리하되, 파라미터만 분리하는 구조다.
```hcl
# variables.tf
variable "environment" {
  type    = string
  default = "dev"
}

variable "model_capacity" {
  type    = number
  default = 10
}
```

#### 2) 원격 상태 저장소(Remote Backend) 보안
Terraform의 상태 파일(`terraform.tfstate`)에는 인프라의 민감한 정보가 포함될 수 있다. 이를 Azure Blob Storage에 저장하고 암호화하는 것이 필수다.
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stllmopsstate"
    container_name       = "tfstate"
    key                  = "prod.foundry.tfstate"
  }
}
```

#### 3) 자동화된 코드 검성 (TFLint & Checkov)
IaC 코드 자체가 보안 취약점을 포함하고 있는지 배포 전에 스캔한다. 예를 들어, 퍼블릭 네트워크 액세스가 허용된 AI 리소스를 생성하려 하면 CI 단계에서 차단한다.

---

LLM 애플리케이션에서 "코드의 버전"과 "지능의 버전(Prompt/Model)"은 생명주기가 다르다. 코드는 수주 단위로 배포될 수 있지만, 프롬프트는 실시간 피드백에 따라 매일 수정될 수도 있다.

### 4.1 Git-based vs CMS-based 관리

| 구분 | Git-based (Code-centric) | CMS-based (Foundry Prompts) |
|---|---|---|
| **저장소** | 애플리케이션 코드와 함께 Git에 저장 | Foundry 클라우드 저장소 |
| **수정 주체** | 개발자 (PR 필요) | 프롬프트 엔지니어 / 기획자 (UI에서 수정) |
| **배포 방식** | 전체 앱 재배포 또는 설정 파일 리로드 | 실시간 API 호출로 최신 프롬프트 획득 |
| **장점** | 코드와 프롬프트의 동기화가 완벽함 | 비개발자의 운영 참여도가 높고 대응이 빠름 |
| **추천** | 엄격한 품질 관리가 필요한 금융/공공 | 빠른 실험과 피드백이 필요한 B2C 서비스 |

### 4.2 Foundry Prompts CMS (Server-side) 활용

Foundry 포털의 Prompt Library를 사용하면 Java 애플리케이션의 재빌드 없이 지능을 교체할 수 있다.

- **작동 방식**: Java SDK에서 `PromptClient`를 생성하고, 고정된 `Prompt ID`를 호출한다. 이때 `version` 파라미터를 생략하면 항상 `latest` 혹은 `published` 상태의 프롬프트를 가져온다.
- **Rollback**: 만약 새로 배포한 프롬프트에서 부정확한 응답이 대량 발생하면, Foundry 포털에서 클릭 한 번으로 이전의 '안정 버전'으로 즉시 롤백할 수 있다. 이는 수 분이 소요되는 코드 롤백보다 훨씬 빠르고 안전하다.

### 4.3 Semantic Versioning for Agents (2026 GA)

🔴 **2026 변경**: **Agent Versions** 기능이 도입되어 에이전트의 설정(모델, 도구, 지침)을 통째로 스냅샷 찍어 관리한다.

- **Major (v2.0.0)**: 모델 아키텍처 변경 (예: GPT-4o → GPT-5), 에이전트 인터페이스(API)의 파괴적 변경.
- **Minor (v1.1.0)**: 새로운 도구(Tool) 추가, 주요 페르소나 변경, 지식 베이스(Index) 교체.
- **Patch (v1.0.1)**: 프롬프트 문구 미세 조정, 오타 수정, 출력 형식 강제 지침 추가.

---

## 5. Deployment 전략: 안정적인 이관

### 5.1 Canary Deployment (카나리 배포)와 Java Circuit Breaker

카나리 배포 시에는 [Ch.3](Ch03_API_Basics.md)에서 배운 Resilience4j와 결합하여 더욱 안전하게 운영할 수 있다.

1.  트래픽의 5%를 새 배포본으로 보낸다.
2.  만약 새 배포본에서 5xx 에러나 `Content Safety` 위반이 발생하면, Java 앱 내부의 Circuit Breaker가 작동하여 해당 요청을 즉시 기존의 안정 버전 모델로 우회(Fallback)시킨다.
3.  이를 통해 시스템 전체의 가용성을 해치지 않으면서 신규 모델을 실전에서 테스트할 수 있다.

### 5.2 Shadow Deployment (섀도 배포) 구현 로직

가장 안전한 배포 방식인 섀도 배포는 Java의 비동기 프로그래밍(`CompletableFuture`)을 활용해 구현한다.

```java
// ShadowDeploymentService.java
public String getResponse(String userQuery) {
    // 1. 현재 운영 중인 모델(Blue) 호출
    String liveResponse = azureFoundryClient.chat(userQuery, "v1.0-stable");

    // 2. 비동기로 신규 모델(Green)에게 동일한 쿼리 전송 (Shadow call)
    CompletableFuture.runAsync(() -> {
        try {
            String shadowResponse = azureFoundryClient.chat(userQuery, "v1.1-preview");
            // 3. 두 답변을 비교하고 로그를 남김 (Shadow Evaluation)
            logService.compareAndRecord(userQuery, liveResponse, shadowResponse);
        } catch (Exception e) {
            log.error("Shadow call failed", e);
        }
    });

    // 사용자는 지연 없이 운영 모델의 답변만 받음
    return liveResponse;
}
```

### 5.3 Traffic Mirroring과 Foundry Gateway

Foundry Gateway(Ch.8) 레벨에서 트래픽 미러링을 설정하면 애플리케이션 코드를 수정하지 않고도 섀도 배포를 수행할 수 있다. 게이트웨이는 들어오는 모든 요청을 복제하여 별도의 분석용 엔드포인트로 쏴주며, 이 데이터는 Ch.9의 Evaluation Dataset으로 자동 축적된다.

---


⚠️ **함정**: Canary 배포 시 **세션 유지(Sticky Session)**가 되지 않으면 유저가 대화 도중 모델이 바뀌어 답변 스타일이 오락가락하는 경험을 하게 된다. 반드시 `Conversation ID` 기반으로 동일한 모델이 응답하도록 라우팅 설정을 고정해야 한다.

---

## 6. Evaluation-Gated Deployment: 데이터 기반 배포 승인

배포 파이프라인의 중간에 "시험"을 넣는 과정이다. [Ch.9](Ch09_Evaluation.md)에서 배운 Evaluation SDK를 활용하여 '품질 검문소'를 구축한다.

### 6.1 품질 게이트(Quality Gate)의 3단계 프로세스

1.  **Ephemeral Environment 생성**: PR(Pull Request)이 생성되면 CI 서버가 테스트용 격리된 Foundry 프로젝트 환경을 즉시 생성한다. 이는 운영 환경에 영향을 주지 않고 안전하게 테스트하기 위함이다.
2.  **Golden Set Benchmarking**: 
    - 미리 정의된 100~500개의 표준 질문-답변 세트(Golden Set)를 새 모델/프롬프트에 주입한다.
    - `azure-ai-evaluation` SDK를 사용하여 Groundedness(근거성), Coherence(일관성), Relevance(관련성) 점수를 산출한다.
3.  **Threshold Enforcement**: 
    - 산출된 점수가 사전에 정의된 임계값(예: 평균 4.2점 이상)을 넘지 못하면 파이프라인은 즉시 중단된다.
    - 배포가 거절되면 해당 PR에 상세 리포트(어떤 질문에서 점수가 깎였는지)를 댓글로 자동 게시하여 개발자가 즉시 수정할 수 있게 돕는다.

### 6.2 Java CI 파이프라인과의 통합 기법

Java 프로젝트에서는 JUnit이나 TestNG와 같은 익숙한 테스트 프레임워크 내에서 이 품질 게이트를 호출할 수 있다.

```java
// EvaluationGateTest.java
@Test
public void verifyModelQualityBeforeDeployment() {
    // 1. Python Evaluation Runner 실행 결과(JSON)를 가져옴
    EvaluationResult result = evalClient.getLatestResult();
    
    // 2. 개별 지표 검증
    assertThat(result.getGroundedness()).isGreaterThanOrEqualTo(4.0);
    assertThat(result.getHateSeverity()).isLessThan(1);
    
    // 3. 이전 버전과의 성능 비교 (Regression Check)
    double baselineScore = evalClient.getBaselineScore("v1.0-stable");
    assertThat(result.getRelevance()).isGreaterThanOrEqualTo(baselineScore - 0.1); 
    // 0.1 이상의 성능 하락이 있으면 실패 처리
}
```

이렇게 하면 개발자들은 일반적인 단위 테스트(Unit Test)를 돌리는 것과 동일한 경험으로 AI 모델의 품질을 관리할 수 있다.

---

## 🔧 실습 3 — GitHub Actions 파이프라인 (전문)

Java 앱 빌드, Docker 이미지 생성, 배포, 그리고 평가 게이트까지 포함된 엔드 투 엔드 워크플로우다.

### pom.xml에 배포 플러그인 추가
```xml
<!-- pom.xml -->
<build>
    <plugins>
        <plugin>
            <groupId>com.microsoft.azure</groupId>
            <artifactId>azure-webapp-maven-plugin</artifactId>
            <version>2.13.0</version>
            <!-- Container Apps 또는 App Service 배포 설정 -->
        </plugin>
    </plugins>
</build>
```

### GitHub Actions Workflow
```yaml
# .github/workflows/llmops-pipeline.yml
name: Foundry LLMOps Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

permissions:
  id-token: write # OIDC 인증용
  contents: read

jobs:
  build-and-eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # 1. Java Build (Maven)
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      - name: Build with Maven
        run: mvn clean package -DskipTests

      # 2. Azure Login via OIDC (비밀번호 없는 인증)
      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      # 3. Infrastructure as Code (Bicep)
      - name: Deploy Infrastructure
        uses: azure/arm-deploy@v2
        with:
          resourceGroupName: rg-foundry-prod
          template: ./infra/main.bicep

      # 4. Evaluation Gate (Ch.9 Python Runner 활용)
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: Run Foundry Evaluation
        run: |
          pip install azure-ai-projects azure-ai-evaluation
          python ./scripts/run_eval.py --threshold 4.0
        env:
          AZURE_FOUNDRY_ENDPOINT: ${{ secrets.FOUNDRY_ENDPOINT }}

      # 5. Deploy to Azure Container Apps
      - name: Build and Push Docker Image
        run: |
          az acr login --name ${{ secrets.ACR_NAME }}
          docker build -t ${{ secrets.ACR_NAME }}.azurecr.io/foundry-app:${{ github.sha }} .
          docker push ${{ secrets.ACR_NAME }}.azurecr.io/foundry-app:${{ github.sha }}
      
      - name: Deploy to Container App
        uses: azure/container-apps-deploy-action@v1
        with:
          imageToDeploy: ${{ secrets.ACR_NAME }}.azurecr.io/foundry-app:${{ github.sha }}
          resourceGroup: rg-foundry-prod
          containerAppName: foundry-java-agent
```

---

## 7. Observability를 배포와 결합

성공적인 배포는 "배포 버튼을 누른 순간"이 아니라 "배포 후 서비스가 안정적임을 확인한 순간" 끝난다.

### 7.1 OTel 기반 자동 롤백 (Automatic Rollback) 구현

성공적인 배포는 "배포 버튼을 누른 순간"이 아니라 "배포 후 서비스가 안정적임을 확인한 순간" 끝난다. OTel 표준 계측 데이터를 기반으로 Java 애플리케이션에서 자동 롤백을 제어하는 로직을 살펴본다.

- **트리거 조건(Service Level Indicators)**:
  - **Error Rate**: API 에러율(5xx)이 지난 10분간 5% 이상 상승한 경우.
  - **Latency**: 답변의 지연 시간(P95)이 5초를 초과하는 경우.
  - **Safety Violation**: Content Safety 차단 건수가 평소보다 3배 이상 급증한 경우 (공격 탐지).
- **자동화 로직**:
  1. Application Insights의 Alert가 Azure Function을 트리거한다.
  2. Azure Function은 Foundry Gateway API를 호출하여 신규 모델(Green)의 가중치를 0%로 조정한다.
  3. 모든 트래픽은 즉시 구버전(Blue)으로 원복된다.

### 7.2 Java 코드를 통한 'Graceful Rollback' 처리

클라이언트 측에서도 롤백 시 발생할 수 있는 일시적인 단절을 방지하기 위해 'Soft-Switching' 로직을 가질 수 있다.

```java
// RollbackAwareRouter.java
public class RollbackAwareRouter {
    private final String STABLE_VERSION = "v1.0-stable";
    private final String CANARY_VERSION = "v1.1-canary";
    private final HealthMonitor healthMonitor;

    public String routeRequest(String query) {
        // 실시간 헬스 체크 데이터 기반 라우팅
        if (healthMonitor.isHealthy(CANARY_VERSION)) {
            try {
                return callModel(query, CANARY_VERSION);
            } catch (Exception e) {
                // 신규 버전 호출 실패 시 즉시 안정 버전으로 Fallback
                log.warn("Canary version failed, falling back to stable", e);
                return callModel(query, STABLE_VERSION);
            }
        }
        return callModel(query, STABLE_VERSION);
    }
}
```

### 7.3 비용 회귀(Cost Regression) 방지 및 예산 관리

PR 단계에서 `Foundry Cost Estimation` 도구를 실행하여, 이번 변경이 적용되었을 때 예상되는 토큰 비용 변화를 리포트한다.

- **Token Budgeting**: 각 부서나 프로젝트별로 월별 토큰 사용량 상한선(Quota)을 설정한다.
- **Hard Limit vs Soft Limit**: 소프트 리밋 도달 시 경고 메일 발송, 하드 리밋 도달 시 해당 API Key 비활성화 또는 저비용 모델(Phi-4)로 강제 전환.
- **FinOps Dashboard**: 토큰 1,000개당 비즈니스 가치(예: 해결된 상담 건수)를 계산하여 LLM 투자의 투자자본수익률(ROI)을 실시간으로 추적한다.

### 8.1 GitHub Actions와 Azure 간의 OIDC 설정 (Step-by-Step)

보안은 자동화의 전제 조건이다. OIDC(OpenID Connect)를 설정하는 구체적인 프로세스를 숙지하여 비밀번호 없는 배포 환경을 구축하라.

1.  **Azure Entra ID App Registration 생성**: CI/CD 파이프라인을 위한 서비스 주체(Service Principal)를 만든다.
2.  **Federated Credential 추가**: GitHub 리포지토리의 `main` 브랜치나 `environment`를 신뢰하도록 설정한다.
3.  **RBAC 할당**: 해당 서비스 주체에게 Foundry 리소스 그룹에 대한 `Contributor` 및 `Cognitive Services OpenAI User` 권한을 부여한다.
4.  **GitHub Secret 구성**: `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`만 저장한다. (Client Secret은 절대 필요 없다.)

### 8.2 보안이 강화된 CI/CD Runner (Self-hosted Runners)

금융권이나 공공기관처럼 네트워크가 폐쇄된 환경에서는 GitHub의 공용 러너(Hosted Runner)를 쓸 수 없다.

- **VNet 통합 러너**: Azure Virtual Network 내에 배포된 가상 머신이나 Container Apps를 Self-hosted Runner로 등록한다.
- **Private Endpoint 통신**: 러너는 공용 인터넷을 통하지 않고 Azure Private Link를 통해 Foundry 엔드포인트에 접속한다. 이를 통해 프롬프트 데이터나 평가 데이터가 공용 망으로 유출되는 것을 원천 차단한다.
- **JIT (Just-In-Time) 권한**: 배포가 진행되는 동안에만 일시적으로 권한을 활성화하는 PIM(Privileged Identity Management)과 파이프라인을 연동한다.

---

## 9. Governance & Policy: 인프라 수준의 통제

LLMOps는 기술적 배포를 넘어 조직의 정책을 강제하는 수단이 되어야 한다.

### 9.1 Azure Policy를 이용한 AI 거버넌스

인프라가 코드로 관리(IaC)되므로, 규정 위반을 배포 단계에서 차단할 수 있다.

- **Model Whitelisting**: 조직에서 승인하지 않은 고비용 모델이나 미검증 모델의 배포를 정책적으로 차단한다.
- **Enforce Content Safety**: 모든 OpenAI 배포 시 반드시 'High' 레벨의 콘텐츠 필터가 적용되어야만 배포가 성공하도록 강제한다.
- **Region Restriction**: 데이터 주권(Data Residency) 법규를 준수하기 위해, 데이터가 한국 외의 리전으로 흘러 나가는 것을 방지한다.

### 9.2 Tagging & Cost Allocation

수백 개의 에이전트가 운영될 때 비용을 추적하는 유일한 방법은 태깅(Tagging)이다.

- **Required Tags**: `CostCenter`, `ProjectID`, `Environment` 태그가 없는 리소스는 생성을 거부하도록 정책을 설정한다.
- **Cost Analysis**: Azure Cost Management에서 태그별로 비용을 분류하여, 어떤 서비스가 가장 많은 토큰을 소비하는지 리포트를 자동 생성한다.

---

## 11. Case Study: 글로벌 제조 기업의 LLMOps 도입기 (From 40min to 1sec)

실제 사례를 통해 LLMOps가 비즈니스 가치를 어떻게 창출하는지 살펴보자. 글로벌 가전 제조사인 B사는 사내 기술 문서 검색 에이전트를 구축하면서 다음과 같은 LLMOps 여정을 거쳤다.

### 11.1 문제 상황: '프롬프트 지옥'과 배포 공포증

초기 B사는 프롬프트를 Java 코드 안에 하드코딩하여 관리했다. 
- **문제 1**: 프롬프트 한 단어를 고칠 때마다 40분이 걸리는 전체 CI/CD 파이프라인(빌드-테스트-이미지 생성-배포)을 돌려야 했다.
- **문제 2**: 여러 명의 프롬프트 엔지니어가 각자 다른 버전의 프롬프트를 로컬에서 테스트하면서, 실제 운영 환경에 어떤 버전이 배포되었는지 추적하기가 불가능해졌다.
- **문제 3**: 특정 국가의 언어(예: 독일어)에서 갑자기 성능이 떨어지는 회귀 현상이 발생해도 이를 배포 전에 감지할 방법이 없었다.

### 11.2 해결책: Foundry LLMOps 체계 구축

B사는 본 커리큘럼에서 배운 내용을 바탕으로 체계를 전면 개편했다.

1.  **Prompt CMS 도입**: 모든 프롬프트를 Foundry Prompt Library로 이전했다. 이제 기획자가 포털 UI에서 프롬프트를 수정하고 'Publish'를 누르면, 운영 중인 Java 앱이 이를 실시간으로 감지하여 반영한다. 배포 시간은 40분에서 1초로 단축되었다.
2.  **OIDC 기반 보안 강화**: GitHub Secret에 저장했던 관리자 암호를 모두 삭제하고, OIDC를 통한 Managed Identity 인증으로 전환했다. 보안 감사팀의 지적 사항을 100% 해결했다.
3.  **Shadow Deployment 운영**: 매일 밤, 낮 동안 들어온 실제 사용자 질문 1,000개를 추출하여 신규 프롬프트 후보군에 섀도 배포 형식으로 흘려보냈다. 다음날 아침 자동 생성된 비교 리포트(Groundedness 비교)를 보고 팀장이 '머지(Merge)' 여부를 최종 결정했다.

### 11.3 성과: 품질과 비용의 두 마리 토끼

- **품질**: 배포 전 자동 평가 게이트를 통과한 프롬프트만 상용화함으로써, 사용자 불만(Hallucination 보고) 건수가 이전 대비 70% 감소했다.
- **비용**: 평가 점수 데이터를 근거로, 단순 인사말이나 날씨 문의 같은 'Low-complexity' 질문에는 GPT-5 대신 훨씬 저렴한 모델을 사용하도록 라우팅을 최적화하여 월간 운영 비용을 45% 절감했다.

---

## 12. 🔧 실습 4 — 고급 IaC: Private Link 보안 구성 (Bicep)

엔터프라이즈 환경에서는 보안을 위해 모든 AI 서비스가 가상 네트워크(VNet) 내부에 숨겨져야 한다. 이를 코드로 구현하는 방법이다.

### 12.1 Private Endpoint 아키텍처 정의

```bicep
// private-link.bicep
param vnetName string = 'vnet-foundry-prod'
param subnetName string = 'snet-ai-services'

resource aiAccount 'Microsoft.CognitiveServices/accounts@2024-10-01' existing = {
  name: 'foundry-resource-prod'
}

resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-05-01' = {
  name: 'pe-foundry-prod'
  location: resourceGroup().location
  properties: {
    subnet: {
      id: resourceId('Microsoft.Network/virtualNetworks/subnets', vnetName, subnetName)
    }
    privateLinkServiceConnections: [
      {
        name: 'foundry-connection'
        properties: {
          privateLinkServiceId: aiAccount.id
          groupIds: [
            'account'
          ]
        }
      }
    ]
  }
}

// Private DNS Zone 설정 (이름 해석을 위해 필수)
resource dnsZoneGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-05-01' = {
  parent: privateEndpoint
  name: 'default'
  properties: {
    privateDnsZoneConfigs: [
      {
        name: 'privatelink-cognitiveservices'
        properties: {
          privateDnsZoneId: resourceId('Microsoft.Network/privateDnsZones', 'privatelink.cognitiveservices.azure.com')
        }
      }
    ]
  }
}
```

이 코드는 Foundry 리소스에 대한 '전용 통로'를 가상 네트워크 안에 뚫어준다. Java 애플리케이션은 이제 공용 인터넷이 아닌 이 전용 통로를 통해 모델과 통신하며, 이는 데이터 유출을 막는 가장 강력한 인프라 방어선이 된다.

---

## 13. Troubleshooting LLMOps Pipelines: 흔히 발생하는 오류와 해결책

자동화된 파이프라인은 편리하지만, 복잡한 인프라와 모델의 상호작용 때문에 예기치 못한 곳에서 문제가 터지곤 한다.

1.  **"Resource Temporarily Unavailable (Quota Exceeded)"**: 
    - **원인**: 파이프라인이 리소스를 생성하려 할 때 해당 리전의 쿼터가 부족함. 특히 전역 표준(Global Standard) 배포는 인기가 많아 쿼터 확보가 어렵다.
    - **해결**: Bicep 코드에 리전 분산 로직을 추가하거나, Azure Portal에서 쿼터 증설 요청을 자동화하는 API를 연동한다. 또한, 비프로덕션 환경은 `Data Zone` 배포를 사용하여 쿼터 간섭을 피한다.
2.  **"Evaluation Script Failed with Timeout"**:
    - **원인**: 테스트 데이터셋이 너무 커서 Python Runner가 600초(GitHub Actions 기본 타임아웃) 이내에 완료하지 못함.
    - **해결**: 평가 태스크를 병렬화(`parallel_batch` 사용)하거나, 중요도가 높은 '코어 테스트 셋'만 우선 실행하도록 분리한다. `azure-ai-evaluation` SDK의 비동기 처리 옵션을 활용하라.
3.  **"OIDC Authentication Failed (Subject mismatch)"**:
    - **원인**: GitHub 환경 변수와 Azure의 Federated Credential 설정이 일치하지 않음. 주로 브랜치 명칭(main vs master)이나 리포지토리 소유자 이름 불일치.
    - **해결**: Azure Portal의 Entra ID 메뉴에서 'Identity' 로그를 확인하여 거부 사유(Subject mismatch)를 확인하고 GitHub 워크플로 파일의 `permissions` 설정을 재점검한다.
4.  **"Model Version Deprecated"**:
    - **원인**: 기초 모델의 수명 주기가 종료되어 더 이상 배포할 수 없음.
    - **해결**: Bicep 코드의 모델 버전을 `latest`로 두지 말고 명시적인 날짜 기반 버전(예: `2025-08`)으로 고정한 뒤, 주기적으로 상위 버전으로의 마이그레이션 테스트를 수행한다.

---


기술적인 자동화만큼 중요한 것이 사람의 프로세스다. 프로덕션 운영을 위한 최소한의 Runbook 요구사항은 다음과 같다.

1.  **SOP (Standard Operating Procedure)**: 장애 발생 시 모델을 다른 리전으로 수동 전환하는 순서.
2.  **On-call Rotation**: 장애 알림을 받을 담당자 순번 및 연락처.
3.  **Post-mortem Template**: 장애 원인 분석 및 재발 방지 대책 기록 양식.

---

## ⚠️ 함정 (Pitfalls)

1.  **Quota 부족으로 인한 배포 실패**: Bicep으로 모델 배포 리소스를 만들 때, 리전에 쿼터가 부족하면 `Conflict` 에러가 발생하며 중단된다. 배포 전 `az cognitiveservices usage list`로 쿼터 가용량을 먼저 확인해야 한다.
2.  **프롬프트 하드코딩**: 프롬프트를 Java 코드 내에 String으로 박아두면, 문구 하나 고칠 때마다 빌드-평가-배포라는 무거운 과정을 거쳐야 한다. Foundry Prompts CMS로 분리하라.
3.  **임계값(Threshold)의 딜레마**: Evaluation Gate의 점수 임계값을 너무 높게(예: 4.8/5.0) 잡으면 아주 사소한 변동에도 배포가 매번 실패하여 생산성이 저하된다. 실무 데이터에 기반한 합리적인 임계값을 조율하라.
4.  **불완전한 롤백**: 인프라(Bicep)는 롤백되었는데 프롬프트 CMS 버전이 롤백되지 않으면 시스템이 꼬인다. 모든 구성 요소의 버전을 하나의 '릴리스 번호'로 묶어 관리하라.

---

## 💡 팁 (Tips)

1.  **GitHub Actions Matrix 활용**: 여러 리전(East US, Sweden Central 등)에 동일한 인프라를 배포해야 할 때 Matrix 전략을 사용하면 코드를 중복하지 않고 병렬 배포할 수 있다.
2.  **Shadow Deployment의 로그 활용**: 섀도 배포에서 발생한 신모델의 답변 데이터를 [Ch.9](Ch09_Evaluation.md)의 AI Red Teaming 에이전트에게 입력값으로 주어 미리 취약점을 파악하라.
3.  **Secret-less 아키텍처**: CI/CD뿐만 아니라 애플리케이션 코드에서도 `DefaultAzureCredential`을 사용하여 Key Vault 접근까지 모두 Managed Identity로 처리하라.

---

## 🔴 2026 변경 사항 요약

- **Agent Versions GA**: 에이전트 구성을 버전화하여 배포와 롤백의 원자성 확보.
- **Foundry Control Plane GA**: 플릿 관리 및 OTel 기반 자동 롤백 성숙.
- **Federated Identity 권장**: `client_secret` 방식의 인증은 레거시로 간주됨.
- **Trace-to-Dataset GA**: 운영 로그를 즉시 평가용 데이터셋으로 변환 가능.

---

## 요약 (Cheat Sheet)

- **IaC**: Bicep/Terraform으로 환경 간 일관성을 유지하라.
- **Versioning**: 에이전트와 프롬프트는 고유의 시맨틱 버전을 가져야 한다.
- **Gate**: 평가 점수가 검증되지 않은 모델은 절대 프로덕션에 올리지 마라.
- **Strategy**: Canary나 Shadow 배포로 사용자 영향을 최소화하며 이전하라.
- **Secret**: OIDC를 통해 키 없는 안전한 파이프라인을 구축하라.

---

## 📚 더 읽기

- [Bicep for Azure AI Services](https://learn.microsoft.com/en-us/azure/templates/microsoft.cognitiveservices/accounts)
- [Terraform azurerm_ai_services Resource](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/ai_services)
- [Foundry CLI for Prompt Management](https://learn.microsoft.com/en-us/azure/foundry/how-to/prompt-flow-cli)
- [GitHub Actions Azure Login (OIDC)](https://github.com/Azure/login#configure-deployment-credentials)
- [LLMOps Deployment Strategies in Azure](https://learn.microsoft.com/en-us/azure/architecture/guide/ai/llmops-deployment)
- [Azure Key Vault + Managed Identity for Java](https://learn.microsoft.com/en-us/azure/key-vault/general/tutorial-net-create-vault-azure-web-app)

---

---

## 14. 실무자를 위한 LLMOps 성숙도 모델 (Maturity Model)

우리 팀의 LLMOps 수준을 객관적으로 진단하고 다음 단계로 나아가기 위한 구체적인 로드맵을 제시한다. 성숙도는 단순히 기술의 도입 여부가 아니라, '데이터를 얼마나 신뢰하고 자동화에 의존하느냐'로 결정된다.

- **Level 0: Ad-hoc (수동 운영 및 실험)**: 
    - 특징: 프롬프트를 Java 코드나 별도 텍스트 파일에 하드코딩한다. 배포 후 '눈대중'으로 결과를 확인하며, 장애 발생 시 원인을 파악하는 데 수 시간이 걸린다.
    - 리스크: 사람이 바뀌면 시스템의 동작 원리를 알 수 없으며, 사소한 변경이 대규모 장애(환각 파괴)로 이어질 확률이 매우 높다.
- **Level 1: Repeatable (기초 자동화 및 가시성)**: 
    - 특징: Bicep/Terraform으로 인프라 생성을 자동화한다. 프롬프트를 외부 설정(Config)으로 분리하고, Application Insights를 통해 기본적인 에러율을 모니터링한다.
    - 이득: 동일한 인프라를 여러 번 생성할 수 있는 '복제 가능성'을 확보한다.
- **Level 2: Reliable (데이터 기반 거버넌스)**: 
    - 특징: 모든 PR 단계에서 자동 평가 게이트(Ch.9)가 작동한다. OIDC 보안 인증을 사용하여 인증서 만료 걱정 없는 파이프라인을 운영한다. 모델별 토큰 비용과 지연시간을 대시보드로 시각화한다.
    - 이득: "이 배포는 안전하다"라는 확신을 데이터로 증명할 수 있다.
- **Level 3: Optimized (지능적 자가 진화)**: 
    - 특징: 섀도 배포를 통해 실시간 트래픽 기반으로 모델을 검증한다. 운영 로그를 자동으로 평가 데이터셋으로 변환(Trace-to-Dataset)하여 데이터셋이 스스로 성장한다. 품질 저하 감지 시 사람의 개입 없이 자동으로 이전 버전으로 롤백되는 'Self-healing' 체계를 갖춘다.
    - 이득: 시장 변화와 사용자 피드백에 실시간으로 대응하며, 운영 공수를 최소화한다.

---

## 15. Resource Lifecycle Management: 비용을 지배하는 자가 승리한다

LLMOps에서 가장 흔히 간과하는 것이 '리소스의 수명 주기'다. 특히 자동화된 CI/CD 파이프라인은 수많은 임시 리소스를 생성하므로, 이를 제대로 관리하지 않으면 월말 청구서에 경악하게 된다.

### 15.1 자동 삭제(Auto-Cleanup) 루틴
- **TTL (Time-to-Live) 설정**: 테스트용으로 생성된 Foundry 프로젝트나 모델 배포는 24시간 후 자동으로 삭제되도록 Bicep 태그를 기반으로 Azure Automation이나 Logic Apps를 연동한다.
- **Pruning Policy**: 사용하지 않는 이전 버전의 모델 배포나 인덱스(Index)를 주기적으로 스캔하여 삭제하는 스케줄러를 운영한다.

### 15.2 Quota Reservation 전략
- 프로덕션 환경의 안정성을 위해 주요 리전의 Quota를 미리 예약(Reserved Capacity)하고, 개발 환경은 전역(Global) 가용 쿼터를 공유하도록 설계하여 비용과 성능의 균형을 맞춘다.

---

## 16. 에필로그: Java 개발자가 만드는 AI의 미래

본 커리큘럼의 모든 여정을 마친 여러분은 이제 단순한 API 호출자를 넘어, 엔터프라이즈급 AI 시스템을 설계하고 운영할 수 있는 **Full-stack AI Engineer**로 거듭났다.

우리가 학습한 Java의 견고한 객체 지향 원칙과 Microsoft Foundry의 최첨단 AI 생태계는 환상의 짝꿍이다. AI 기술은 매달 변하지만, 우리가 배운 **'검증 가능한 시스템'**, **'다중 방어 보안'**, **'자동화된 운영'**이라는 본질적인 원칙은 흔들리지 않을 것이다.

이제 여러분의 전공 분야(금융, 제조, 의료, 유통 등)에 이 기술을 투사하여 비즈니스 가치를 창출해 보라. 여러분이 작성한 단 한 줄의 Java 코드가 수백만 명의 사용자에게 안전하고 지능적인 경험을 제공하는 그날을 응원한다.

---

이로써 **Microsoft Foundry 실무 커리큘럼**의 모든 과정을 마쳤다.

커리큘럼 완주를 진심으로 축하한다. 우리는 기초적인 모델 배포(Ch.1-3)부터 시작하여 RAG와 에이전트(Ch.4-7)를 구축하고, 이를 엔터프라이즈급으로 운영하기 위한 전략과 보안(Ch.8-11)을 모두 학습했다. 

**3줄 요약:**
1.  **지능보다 시스템**: 모델 자체의 성능보다 이를 검증(Eval)하고 관리(Ops)하는 체계가 서비스의 품질을 결정한다.
2.  **보안은 타협 불가**: Managed Identity와 다층 방어 체계는 선택이 아닌 필수다.
3.  **작게 시작하고 자동화하라**: 우선 단일 모델로 시작하되, 점진적으로 Model Router와 CI/CD 파이프라인을 구축하여 확장성을 확보하라.

이제 배운 내용을 바탕으로 사내 PoC를 시작하거나, 팀원들을 위한 온보딩 자료로 이 커리큘럼을 활용해 보기 바란다. Microsoft Foundry의 강력한 생태계가 여러분의 비즈니스 혁신을 가속화할 것이다.

[← 목차로](README.md)
