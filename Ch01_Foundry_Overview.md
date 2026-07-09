# Chapter 1. Microsoft Foundry 개요 & 시작

[← 목차로](README.md)

> **학습 목표**
> - Microsoft Foundry의 정의와 2026년 리브랜딩 배경을 이해한다.
> - Azure OpenAI, Azure AI Services, Azure ML과 Foundry의 관계와 차이점을 구분할 수 있다.
> - Foundry Resource와 Project 기반의 새로운 아키텍처 구조를 완벽히 파악한다.
> - Azure CLI와 포털을 통해 실제 운영 가능한 Foundry 환경을 구축한다.
> - Foundry 포털의 각 메뉴(Models, Agents, Traces 등)의 상세 기능과 활용 시나리오를 익힌다.
> - 기업용 AI 애플리케이션 개발을 위한 리전 선택 및 비용 최적화 전략을 수립한다.

> **전제 조건**
> - Azure 구독(Subscription) 및 리소스 그룹 생성 권한
> - Azure CLI 설치 및 로그인 상태 (`az login`)
> - 기본적인 클라우드 개념(리전, 리소스 그룹, RBAC)에 대한 이해
> - Python 3.11+ 및 `uv` (실제 코딩은 Ch.3부터 진행하나 환경은 준비되어 있어야 함)

---

## 1. Microsoft Foundry란 무엇인가?

Microsoft Foundry는 기업이 생성형 AI 애플리케이션을 설계, 구축, 배포 및 관리할 수 있도록 지원하는 통합 클라우드 플랫폼이다. 과거 Azure AI Studio와 Azure AI Foundry로 파편화되어 불리던 서비스들이 2026년 6월부로 **Microsoft Foundry**라는 하나의 브랜드로 완전히 통합되었다. 

🔴 **2026 변경**: 2026년 6월 이전까지는 Azure AI Foundry라는 명칭을 사용했으나, 이제는 Microsoft Foundry가 공식 명칭이다. 단순히 이름만 바뀐 것이 아니라, Azure OpenAI Service, Azure AI Search, Azure AI Content Safety 등 흩어져 있던 AI 서비스들을 하나의 프로젝트 단위로 묶어 관리하는 '통합 AI 허브'로서의 정체성이 확립되었다. 이제 개발자는 여러 리소스를 개별적으로 관리할 필요 없이, Foundry 프로젝트 하나만으로 모든 AI 워크플로를 완결할 수 있다.

Foundry는 다음과 같은 세 가지 핵심 가치를 제공한다.

1.  **모델의 다양성 (Model Catalog)**: OpenAI의 GPT-5.5부터 Anthropic의 Claude, Meta의 Llama, Microsoft의 Phi-4까지 1,900개 이상의 모델을 한곳에서 탐색하고 배포할 수 있다.
2.  **통합된 도구 체인**: 프롬프트 엔지니어링을 위한 Playground, 데이터 인덱싱을 위한 Data + Indexes, 에이전트 구축을 위한 Agent Service를 단일 인터페이스에서 제공한다.
3.  **엔터프라이즈급 거버넌스**: 모든 AI 활동에 대한 트레이싱(Tracing), 평가(Evaluation), 보안 정책(Guardrails)을 프로젝트 레벨에서 중앙 집중식으로 관리한다.

### 1.1 등장 배경과 진화: 왜 통합되었는가?

초기 Azure의 AI 서비스는 개별 API 형태로 제공되었다. 개발자는 Azure OpenAI 리소스를 따로 만들고, 데이터를 저장할 저장소와 인덱싱을 위한 검색 서비스를 각각 관리해야 했다. 하지만 생성형 AI 앱이 복잡해지면서 '프로젝트' 중심의 관리 체계가 절실해졌고, 이를 위해 탄생한 것이 Foundry다. 이제 개발자는 인프라 구성 요소가 아닌 'AI 앱 그 자체'에 집중할 수 있다.

2026년에 접어들면서 Foundry는 단순한 '포털'을 넘어 **AI 운영 체제(OS)**와 같은 역할을 수행하기 시작했다. 모델 라우팅, 가드레일 적용, 멀티 에이전트 오케스트레이션이 모두 플랫폼 레벨에서 지원된다. 개발자는 더 이상 하부 인프라의 복잡성에 시달리지 않고, 비즈니스 로직과 프롬프트 엔지니어링에만 집중할 수 있는 환경을 갖게 되었다.

### 1.2 왜 Microsoft Foundry를 선택해야 하는가? (Value Proposition)

단순히 모델 API만 호출하는 수준을 넘어, 실제 엔터프라이즈 서비스에 적용하려면 다음과 같은 복잡한 문제들을 해결해야 한다.

-   **데이터 프라이버시 및 보안**: 기업 내부 데이터를 어떻게 안전하게 모델에 전달하고, 답변에 포함된 민감 정보를 어떻게 필터링할 것인가? Foundry는 데이터가 모델 학습에 사용되지 않음을 법적으로 보장하며, 엔터프라이즈급 가드레일(Prompt Shields 등)을 기본 제공한다.
-   **정량적 평가 (Evaluation)**: 모델이 내놓은 답변이 정확한가? 근거(Groundedness)가 있는가? 이를 사람이 일일이 검수하지 않고 자동화할 수 있는가? Foundry의 Evaluation suite는 수천 개의 답변을 단 몇 분 만에 AI가 직접 채점하도록 지원한다.
-   **멀티 모델 전략 (Model Agnostic)**: 특정 작업에는 GPT-5.5가 유리하지만, 단순 요약에는 가벼운 Phi-4가 유리할 수 있다. 이러한 모델 교체를 코드 수정 없이 할 수 있는가? Foundry는 통합 인퍼런스 API와 모델 라우터를 통해 이를 가능케 한다.
-   **관측 가능성 (Observability)**: 운영 중인 AI가 왜 이상한 답변을 했는지, 어떤 단계에서 병목이 생겼는지 OpenTelemetry 표준으로 추적할 수 있는가? Foundry Traces는 복잡한 체인(Chain)의 모든 호출 단계를 시각화해준다.

Foundry는 이 모든 질문에 대한 답을 미리 설계된 워크플로로 제공한다. 특히 2026년에 강화된 **Control Plane** 기능은 전사적인 AI 자산 관리와 보안 정책 준수 여부를 한눈에 파악하게 해준다. 이는 개별 API를 조합해서는 도달하기 힘든 압도적인 개발 생산성을 의미한다.

---

## 2. 서비스 비교: AOAI vs Foundry vs Azure ML

생성형 AI를 처음 접하는 개발자가 가장 혼란스러워하는 지점은 "어떤 서비스를 써야 하는가?"이다. 2026년 현재 기준의 비교는 다음과 같다.

| 비교 항목 | Azure OpenAI Service | Microsoft Foundry | Azure Machine Learning |
| :--- | :--- | :--- | :--- |
| **주요 대상** | OpenAI 모델만 필요한 개발자 | 생성형 AI 앱 전체를 구축하는 팀 | ML 모델을 직접 훈련/배포하는 데이터 과학자 |
| **제공 모델** | GPT 시리즈 전용 | GPT + Claude + Llama + Phi 등 1900+종 | 모든 오픈소스 모델 및 커스텀 모델 |
| **워크플로** | API 호출 위주 | 프로젝트 기반 통합 워크플로 | 파이프라인 및 실험 중심 |
| **인터페이스** | Azure Portal / API | **ai.azure.com (전용 포털)** | ml.azure.com (전용 포털) |
| **추천 케이스** | 단순 챗봇, 단순 텍스트 생성 | RAG, 멀티 에이전트, 복합 AI 앱 | 커스텀 모델 파인튜닝, 복합 ML 파이프라인 |

### 2.1 왜 Foundry가 엔터프라이즈의 표준인가? (Deep Dive)

단순히 API만 필요한 경우라면 Azure OpenAI Service로 충분할 수 있다. 하지만 기업 환경에서는 다음과 같은 요구사항이 반드시 뒤따른다.

1.  **모델 다각화 (Model Diversification)**: 특정 업무(예: 민감 정보 마스킹)에는 작고 빠른 Phi-4가 효율적이고, 복잡한 로직 설계에는 GPT-5.5가 필요할 수 있다. Foundry는 이들을 하나의 프로젝트 안에서 동일한 보안 정책 하에 관리하게 해준다.
2.  **데이터 주권과 검색의 결합**: AI가 기업 내부 문서를 읽게 하려면(RAG), 검색 서비스(Azure AI Search)와의 연동이 필수적이다. Foundry는 이 연동 과정을 'Click-to-Connect' 수준으로 단순화하여 개발자가 벡터 DB의 복잡한 파이프라인을 직접 코딩하지 않아도 되게 돕는다.
3.  **검증된 안전성**: 2026년 강화된 **Azure AI Content Safety** 기능이 Foundry에 내장되어 있어, 모델의 답변이 기업의 윤리 강령을 위반하는지 실시간으로 감시하고 차단한다. 이는 개별 API만으로는 구축하기 매우 까다로운 레이어다.

⚠️ **함정**: 포털에서 Azure OpenAI 리소스를 직접 생성할 수도 있지만, 엔터프라이즈 앱을 개발한다면 Foundry 프로젝트 내에서 'Connection'을 통해 Azure OpenAI를 사용하는 방식이 표준이다. 개별 리소스로 시작하면 나중에 Foundry의 평가 및 트레이싱 기능을 연동하기 매우 번거롭다. 

또한, Azure OpenAI 전용 리소스로 시작하면 나중에 Anthropic Claude 같은 타사 모델로 확장하고 싶을 때 아키텍처를 완전히 갈아엎어야 한다. Foundry는 모델 독립적인 'Inference API'를 제공하므로 미래 지향적인 선택이 된다.

---

## 3. Foundry Resource + Project 아키텍처

Foundry의 핵심 아키텍처는 리소스 관리 모델의 현대화에 있다. 2025년까지 사용되던 'Hub-based Project' 모델은 이제 레거시로 분류되며, 새로운 통합 리소스 모델로 대체되었다.

### 3.1 새로운 모델: Foundry Resource → Projects

🔴 **2026 변경**: 기존에는 'AI Studio Hub' 아래에 프로젝트를 생성했으나, 현재는 **Foundry Resource**가 최상위 컨테이너 역할을 한다. 이 구조는 Azure의 다른 리소스(예: Storage Account)와 일관성을 유지하면서도 AI 특유의 계층 구조를 지원한다.

-   **Foundry Resource (구 Hub)**: 보안 설정, 가상 네트워크(VNet), 공유 커넥션(Connection), 컴퓨팅 자원, 할당량(Quota)을 정의하는 물리적/관리적 단위다. 조직의 관리자나 클라우드 아키트가 한 번 설정하면 여러 개발 프로젝트에서 이를 공유한다.
-   **Foundry Project**: 실제 AI 애플리케이션 개발이 이루어지는 논리적 작업 영역이다. 모델 배포(Deployment), 데이터 인덱스, 프롬프트 라이브러리, 평가 결과 등이 프로젝트 단위로 격리되어 관리된다.

### 3.2 왜 바뀌었나? (The Power of Unification)

과거의 허브 모델은 권한 관리가 복잡하고 리소스 그룹 간의 의존성이 강해 대규모 프로젝트에서 충돌이 잦았다. 새로운 모델은 Foundry Resource 하나만으로도 하위 프로젝트들을 깔끔하게 관리할 수 있으며, 특히 엔터프라이즈 수준의 RBAC 적용이 훨씬 직관적으로 변했다. 또한, 리소스 그룹을 넘나드는 공유 자원 관리가 가능해져 사내 공통 AI 자산 중앙 관리(Central Governance)가 쉬워졌다.

### 3.3 마이그레이션 전략 및 팁

💡 **팁**: 기존 Azure OpenAI 리소스를 이미 운영 중이라면, Foundry 포털 내에서 **'Upgrade to Foundry'** 옵션을 확인하라. 이 기능을 통해 기존 엔드포인트 URL과 API 키를 그대로 유지하면서 Foundry의 강력한 거버넌스 기능을 덧입힐 수 있다. 이는 '시스템 중단 없는 현대화'를 가능케 하며, 기존 앱의 가동 시간을 100% 유지하면서 새 플랫폼으로 옮겨갈 수 있는 가장 안전한 길이다.

---

## 4. 핵심 개념 상세 정의

Foundry를 능숙하게 다루기 위해 반드시 완벽히 이해해야 할 핵심 개념들이다. 이 개념들은 이후 Python SDK를 다룰 때 객체 모델(`AIProjectClient`, `AgentsClient`, `OpenAI` 등)로 그대로 전이된다.

1.  **Foundry Resource (AIServices kind)**: 모든 AI 활동의 기초가 되는 Azure 리소스다. 실제 Azure 리소스 종류(kind)는 `AIServices`로 생성된다. 이는 단순히 OpenAI뿐만 아니라 음성(Speech), 언어(Language), 비전(Vision), 콘텐츠 안전(Content Safety) 기능을 모두 포함하는 거대한 컨테이너다.
2.  **Project**: 개발팀이 작업하는 논리적 단위다. 프로젝트는 Foundry Resource의 할당량(Quota)을 나누어 쓴다. 예를 들어 하나의 Resource 아래 '고객 상담 봇 프로젝트'와 '내부 문서 요약 프로젝트'를 각각 독립적으로 운영할 수 있다. 프로젝트 간에는 기본적으로 데이터가 격리되지만, Resource를 통해 자원을 공유할 수 있다.
3.  **Connection**: 외부 세계와의 연결 고리다. Azure AI Search, SQL Database, Cosmos DB 등을 프로젝트에 연결할 때 사용한다. 특히 **Managed Identity**를 통한 인증을 Connection 레벨에서 설정함으로써, 코드에 API 키를 하드코딩하는 보안 사고를 근본적으로 방지한다.
4.  **Deployment**: 모델 카탈로그에서 선택한 모델(예: GPT-5.5)을 실제 API 엔드포인트로 활성화한 상태다. 'Global Standard'(종량제)나 'Provisioned'(예약제) 같은 SKU를 선택하게 되며, 이는 성능과 비용에 직전적인 영향을 미친다.
5.  **Endpoint**: 배포된 모델에 접근하기 위한 URL이다. Foundry는 프로젝트 내에서 여러 모델을 하나의 통합 엔드포인트처럼 관리할 수 있는 **Model Router** 기능을 제공하여, 클라이언트 코드 수정 없이 모델 버전을 스위칭할 수 있게 해준다.

💡 **팁**: Connection 설정 시 'Shared' 옵션을 적극 활용하라. 조직 내 공통으로 사용하는 Azure AI Search 인덱스가 있다면, 이를 Resource 레벨에서 Shared Connection으로 등록하여 모든 하위 프로젝트가 즉시 사용할 수 있다. 중복된 서비스 생성 비용을 획기적으로 줄일 수 있다.

---

## 5. RBAC 역할 및 거버넌스: 팀 협업의 기초

기업 환경에서 권한 관리는 보안의 핵심이다. Foundry는 AI 워크플로의 각 단계에 최적화된 역할을 제공한다.

### 5.1 주요 역할 상세

-   **Foundry Account Owner**: 리소스의 과금, 쿼터 요청, 전사적 보안 가드레일을 설정한다. 보통 클라우드 플랫폼 팀이 보유한다.
-   **Foundry Project Manager**: 특정 프로젝트 내에서 모델을 배포하고, 데이터를 업로드하며, 평가를 실행할 권한을 가진다. 프로젝트 리더나 시니어 개발자에게 할당한다.
-   **Foundry User**: Playground에서 테스트하거나, 배포된 엔드포인트에 추론 요청을 보낼 수 있는 권한이다. 일반 개발자 및 데이터 분석가용이다.
-   **Cognitive Services OpenAI User**: 실제 모델 API를 호출하기 위해 필요한 하위 역할이다. Managed Identity를 통해 앱이 모델에 접근할 때 필수적으로 요구된다.

### 5.2 협업 시나리오 및 권한 할당 팁

여러분이 팀장이라면, 팀원들을 프로젝트에 초대할 때 다음 가이드를 따르라.
1.  **시니어 개발자 / 아키텍트**: 모델 배포와 인덱싱을 직접 관리해야 하므로 **Foundry Project Manager** 권한이 필요하다.
2.  **주니어 개발자 / 데이터 분석가**: 배포된 모델을 사용해 Python 코딩을 하거나 Playground에서 프롬프트를 깎는 작업을 한다면 **Foundry User**만으로 충분하다.
3.  **애플리케이션(Managed Identity)**: 서버 앱이 배포될 때는 **Cognitive Services OpenAI User**와 **Foundry User** 권한이 조합되어야 한다.

⚠️ **함정**: Azure 구독의 'Owner' 권한이 있다고 해서 모든 것이 해결되지 않는다. Foundry 포털에서 모델을 배포하거나 데이터를 인덱싱할 때 "Access Denied"가 뜬다면, 본인에게 **'Foundry Project Manager'** 역할이 해당 리소스 혹은 프로젝트 레벨에서 명시적으로 할당되었는지 반드시 확인하라. Azure RBAC는 명시적 할당이 우선이다.

---

## 6. 모델 카탈로그 탐색: 2026년 주력 모델 라인업

Foundry의 모델 카탈로그는 단순한 목록이 아니라, 각 모델의 특성과 성능을 비교할 수 있는 대시보드다. 1,900개 이상의 모델 중 어떤 것을 골라야 할까?

### 6.1 플래그십 모델 상세 분석

-   **GPT-5.5 (OpenAI)**: 2026년 4월 출시된 현세대 최강 모델이다. 105만 토큰의 거대한 컨텍스트 창을 지원하며, 'Reasoning' 모드를 활성화하면 복잡한 수학 문제나 로직 설계를 인간 전문가 수준으로 수행한다. 특히 **'Computer Use'** 기능을 통해 브라우저를 직접 조작하거나 스크린샷을 분석하여 사무 작업을 자동화하는 에이전트 구축에 최적이다.
-   **Claude 3.5 Sonnet (Anthropic)**: Foundry 내에서 'Inference API'를 통해 즉시 배포 가능하다. 특히 자연스러운 문체와 정교한 코드 작성 능력으로 Python 개발자들 사이에서도 인기가 높다. GPT 모델과는 다른 독특한 시각을 제공하여 앙상블 시스템 구축에 유리하며, 감성적인 대화가 필요한 서비스에 추천된다.
-   **Phi-4 (Microsoft)**: 마이크로소프트의 자체 소형 모델(SLM)이다. 성능 대비 전력 소모와 비용이 압도적으로 낮아, 실시간 채팅의 의도 분류(Intent Classification)나 간단한 개체명 인식(NER) 작업에 사용하면 비용을 90% 이상 절감할 수 있다. 소형 장치나 브라우저 내 추론에도 적합한 고효율 모델이다.
-   **Llama 3.1 / 4.0 (Meta)**: 오픈소스 생태계의 강자다. 특정 도메인(금융, 의료 등)의 전문 용어를 학습시키기 위한 파인튜닝(Fine-tuning) 베이스 모델로 가장 많이 사용된다. Foundry에서는 이를 관리형 인프라 위에서 클릭 몇 번으로 배포할 수 있어 운영 부담이 거의 없다.

### 6.2 모델 선택 가이드 및 주의사항

💡 **팁**: 모델을 선택할 때 'Model Card'를 꼼꼼히 읽어라. 각 모델이 지원하는 언어, 최대 토큰, 그리고 무엇보다 **'Data Zone'** 정보를 확인해야 한다. 특정 모델은 미국 리전에서만 데이터 처리가 가능할 수 있으므로, 유럽이나 한국 내 데이터 보관 규정이 중요한 경우 'Data Zone Standard' 배포 타입을 택해야 한다. 또한 모델별로 초당 호출 횟수(RPM) 제한이 다르므로 트래픽 예측에 따라 모델을 선정해야 하며, 필요 시 미리 쿼터를 확보해두어야 한다.

---

## 🔧 실습: 첫 리소스 및 프로젝트 생성

이론을 넘어 실제 환경을 구축해본다. 실무에서는 재현성을 위해 CLI를, 빠른 설정을 위해 포털을 혼용한다.

### 7.1 리전(Region) 선택의 전략: 왜 East US인가?

리전 선택은 단순히 지리적 거리가 아니다. 생성형 AI 리소스에서는 가용 자원과 기능 반영 속도가 핵심이다.
-   **East US / Sweden Central**: 최신 모델(GPT-5.5)이 가장 먼저 풀리고 쿼터가 넉넉하다. 신기능을 가장 먼저 써보고 싶다면 이 지역이 필수다. 또한 다수의 데이터 센터가 밀집되어 있어 서비스 중단 위험이 낮다.
-   **Korea Central**: 한국 내 데이터 보관이 법적 의무인 공공/금융 프로젝트에서 선택한다. 다만 최신 모델 업데이트가 해외 리전보다 수 주에서 수 개월 늦을 수 있음을 인지해야 한다. 네트워크 지연 시간(Latency)은 가장 낮아 빠른 반응 속도가 중요한 서비스에 유리하다.
-   **West US 3**: AI 연산 자원이 풍부하여 대규모 프로비저닝(Provisioned Managed) 배포 시 유리하며, 상대적으로 비용이 저렴한 경우가 많다.

### 7.2 Azure Portal 방식 (Step-by-Step)

#### 단계 1: Foundry Resource 생성
1.  [Azure Portal](https://portal.azure.com) 로그인.
2.  상단 검색창에 'Azure AI Foundry' 입력 후 서비스 진입.
3.  **+ Create → Foundry Resource** 클릭.
4.  기본 정보 입력:
    -   **Resource Group**: `rg-foundry-lab-01`
    -   **Name**: `foundry-res-primary`
    -   **Region**: `East US` (GPT-5.5 가용성 확인)
5.  **Identity** 탭: **System Assigned Managed Identity**를 반드시 **On**으로 설정하라. 이후 Python 앱에서 `DefaultAzureCredential`로 키 없이 리소스에 접근할 때 이 설정이 없으면 고생하게 된다. 보안 강화를 위해 권장되는 표준 설정이다.
6.  **Review + Create** 클릭 후 약 2~3분 대기.

#### 단계 2: Foundry Project 생성 (핵심 과정)
1.  생성된 Foundry Resource 페이지로 이동한다.
2.  상단의 **'Launch Foundry Portal'** 버튼 혹은 직접 [ai.azure.com](https://ai.azure.com)에 접속한다.
3.  좌측 하단의 **'Management Center'** 아이콘을 클릭한다. (여기가 인프라와 개발 영역의 교차점이다)
4.  **Projects** 메뉴에서 **+ New Project** 버튼을 클릭한다.
5.  프로젝트 이름(예: `enterprise-ai-app`)을 입력하고 앞서 만든 Resource가 정확히 선택되었는지 확인한다.
6.  프로젝트 생성이 완료되면 화면이 '프로젝트 대시보드'로 전환된다. 이제 모델을 배포할 준비가 된 것이다.

### 7.3 Azure CLI 방식 (자동화 및 전문가용)

터미널에서 다음 명령어를 실행한다. `AIServices` 종류로 생성하는 것이 포인트다.

```bash
# 리소스 그룹 생성 (이미 있다면 생략)
az group create --name rg-foundry-cli --location eastus

# Foundry Resource (AIServices) 생성
# 'S0'은 표준 계층이며, 대부분의 엔터프라이즈 기능을 포함한다.
az cognitiveservices account create \
    --name my-foundry-cli-res \
    --resource-group rg-foundry-cli \
    --kind AIServices \
    --sku S0 \
    --location eastus \
    --yes

# 생성된 리소스의 상세 정보 및 엔드포인트 확인
az cognitiveservices account show \
    --name my-foundry-cli-res \
    --resource-group rg-foundry-cli \
    --query "{Name:name, Endpoint:properties.endpoint, ID:id}"
```

⚠️ **함정**: 간혹 `az cognitiveservices` 명령어가 작동하지 않는다면, 최신 Azure CLI 확장 프로그램이 설치되지 않은 것이다. `az extension add --name cognitiveservices`를 먼저 실행하라. 또한, Foundry 전용 CLI인 `az ai-foundry`는 현재 프리뷰 중이므로 일부 명령어가 변경될 수 있다. 실습 중 권한 에러가 나면 `az login`을 통해 최신 토큰을 확보했는지 확인하라.

---

## 8. 포털 투어 및 활용 시나리오 (https://ai.azure.com)

리소스가 준비되었다면 전용 포털인 [ai.azure.com](https://ai.azure.com)으로 접속한다. 이 사이트는 개발자 경험(DX)에 최적화되어 있으며, 각 메뉴는 AI 앱 생애주기의 특정 단계를 담당한다.

### 8.1 주요 메뉴 상세 가이드 및 팁

-   **Home**: 최근 작업한 프로젝트와 빠른 시작 튜토리얼을 보여준다. 현재 리소스의 상태, 쿼터 소모 현황, 그리고 MS의 최신 AI 공지사항을 확인하는 통합 대시보드다.
-   **Models**: 모델을 탐색하고 배포한다. 'Serverless API' 방식으로 배포하면 인프라 관리 없이 호출 횟수만큼만 비용을 낼 수 있다. 수천 개의 모델 중 성능, 비용, 리전 가용성을 따져 최적의 조합을 찾는 곳이다. 배포 전 각 모델의 가격표(Pricing)를 반드시 확인하라.
-   **Agents**: 2026년의 핵심인 **Responses API v2**를 기반으로 지능형 에이전트를 설계한다. 파일 검색, 코드 실행, 웹 검색 등의 도구를 에이전트에게 장착시키는 과정을 GUI로 직관적으로 수행한다. 이곳에서 만든 에이전트 정의(Definition)는 나중에 Python 코드에서 `AgentsClient` (`azure-ai-agents`)를 통해 호출된다.
-   **Data + Indexes**: RAG 시스템의 기반이 되는 벡터 DB를 구축한다. Azure AI Search와 연동하여 PDF, 마크다운, 워드 문서를 AI가 검색 가능한 '지식베이스'로 변환한다. 데이터 정제와 청킹(Chunking) 전략을 여기서 수립하며, 대규모 인덱싱 시 발생하는 비용을 관리 센터에서 모니터링해야 한다.
-   **Evaluation**: "내 AI의 답변이 얼마나 신뢰할 수 있는가?"를 검증한다. 'Groundedness'(답변의 근거 유무) 지표는 생성형 AI의 고질적 문제인 환각(Hallucination)을 잡아내는 데 필수적이다. 대량의 테스트 데이터셋을 돌려 모델의 정확도를 수치화하며, 리플레이 기능을 통해 실패한 사례를 분석하고 프롬프트를 개선한다.
-   **Traces**: OpenTelemetry 표준을 사용하여 애플리케이션 로그를 추적한다. Python(FastAPI 등) 앱에서 보낸 요청이 Foundry 내부에서 어떻게 처리되었는지 타임라인으로 보여준다. `azure-monitor-opentelemetry` 로 자동 계측 가능. 병목 지점을 찾거나 에러 원인을 분석할 때 사용하며, 시각화된 호출 트리를 통해 복잡한 AI 워크플로의 성능을 튜닝한다.
-   **Management Center**: 현재 사용 중인 모든 프로젝트의 쿼터와 보안 연결을 중앙 관리한다. 사내 AI 거버넌스 담당자가 가장 많이 보게 될 화면이며, 사용하지 않는 배포 모델을 삭제하여 불필요한 비용 지출을 막는 곳이기도 하다.
-   **Playgrounds**: 'Chat Playground'에서 시스템 프롬프트를 테스트하라. 여기서 성공한 프롬프트는 우측 상단의 'View Code'를 통해 즉시 Java/Python 코드로 추출할 수 있다. 초반 코드 작성 시 훌륭한 뼈대가 되며, 온도(Temperature)나 Top-P 같은 파라미터를 실시간으로 튜닝하며 응답의 품질을 확인해볼 수 있다.

💡 **팁**: Playgrounds의 **'Parameters'** 설정을 유심히 보라. `Temperature`를 낮추면 일관된 답변을, 높이면 창의적인 답변을 얻을 수 있다. 실무용 챗봇은 대개 0.3~0.5 사이를 권장한다. 또한 `Max Tokens`를 적절히 설정하여 예상치 못한 긴 답변으로 인한 과금을 막아야 한다. 'Top P' 설정은 답변의 다양성을 조절하는 또 다른 핵심 인자다.

---

## 9. 비용 및 성능 최적화 전략 (SKU 가이드)

운영 단계에서 비용 폭탄을 맞지 않으려면 배포 타입(SKU)을 이해해야 한다.

1.  **Global Standard (Pay-per-token)**: 가장 유연하다. 트래픽이 적거나 예측 불가능한 개발/테스트 단계에 적합하다. 사용한 만큼만 지불하므로 초기 프로젝트에 가장 권장된다. 하지만 부하가 몰릴 때 지역 간 로드밸런싱으로 인해 속도 지연이 발생할 수 있다.
2.  **Global Provisioned (Managed PTU)**: 처리량을 예약한다. 응답 속도(Latency)가 비즈니스에 치명적인 경우(예: 실시간 결제 상담) 사용한다. 최소 1개월 단위 예약이 일반적이며, 높은 트래픽이 지속될 때 토큰당 단가는 Global Standard보다 저렴해질 수 있다. 기업용 핵심 서비스에 필수적이다.
3.  **Global Batch**: 비실시간 처리에 최적이다. 전날 쌓인 대량의 로그를 분석하거나, 수만 건의 기사를 요약하는 작업 등에 사용하면 비용을 최대 50% 아낄 수 있다. 24시간 이내 처리가 보장되며, 대용량 데이터 전처리에 매우 경제적이다.

⚠️ **함정**: 'Global Standard'는 전 세계 유휴 자원을 사용하므로 가끔 리전 간 네트워크 지연이 발생할 수 있다. 극도의 실시간성이 필요하다면 해당 리전 전용인 'Standard (Regional)' 배포를 고려해야 하지만, 이 경우 해당 리전의 쿼터 제한을 엄격히 적용받게 되며 신규 모델 반영이 늦을 수 있다. 또한 Regional 배포는 가용 영역(AZ) 장애 시 대처가 Global보다 까다롭다.

---

## 10. 엔터프라이즈 보안: Connections와 Managed Identity

Foundry의 진가는 보안에서 드러난다. 더 이상 `.env` 파일에 API 키를 저장하지 마라.

-   **Connections**: Azure AI Search나 Blob Storage 연결 시 키 대신 **Managed Identity**를 선택하라. 이렇게 하면 Foundry 프로젝트 자체가 Azure 리소스에 대한 권한을 갖게 된다. 이는 보안 감사 시 매우 강력한 증거가 되며, 키 유출 사고를 원천 차단한다.
-   **Content Safety**: 모든 모델 입출력은 자동으로 Content Safety 레이어를 거치게 설정할 수 있다. 혐오 표현, 성적 표현 등을 플랫폼 레벨에서 필터링한다. 이를 통해 개발자는 유해 콘텐츠 차단 로직을 일일이 짤 필요가 없어진다.
-   **Networking**: Foundry Resource는 가상 네트워크(VNet) 내에 격리될 수 있다. 사내 망에서만 접근 가능한 AI 시스템을 구축할 때 이 기능은 필수적이다. 외부 인터넷과의 접점을 최소화하여 데이터 유출 위험을 낮춘다.

💡 **팁**: `Ch.10 Security & Governance`에서 자세히 다루겠지만, 지금 당장은 **'키 없는 인증'**이 Foundry의 표준임을 기억하라. 실무 프로젝트에서는 보안 팀의 승인을 받기 훨씬 쉬워진다. 또한, `azure-identity` 라이브러리를 사용하면 로컬 개발 환경과 클라우드 운영 환경 간의 코드 변경 없이 인증을 처리할 수 있어 개발 편의성이 비약적으로 향상된다.

### 10.1 Network Security Architecture

엔터프라이즈 환경에서 AI 모델은 단순한 API가 아니라, 기업의 핵심 자산과 데이터에 접근하는 통로다. 따라서 Foundry는 다음과 같은 3중 네트워크 보안 계층을 제공한다.

1.  **Public Access Disable**: 기본적으로 Foundry Resource의 공용 인터넷 접근을 차단하고, 승인된 가상 네트워크(VNet) 내에서만 트래픽을 허용한다.
2.  **Private Endpoints**: Azure Private Link를 통해 AI 서비스에 사설 IP를 부여한다. 이를 통해 트래픽이 공용 인터넷을 거치지 않고 Microsoft 백본 네트워크 내에서만 이동하게 되어, 중간자 공격(MITM) 위협을 원천 차단한다.
3.  **Trusted Inter-Service Communication**: Foundry Resource와 연동된 Storage Account나 Azure AI Search 간의 통신 시, Managed Identity 기반의 신뢰 관계를 구축하여 별도의 방화벽 예외 규칙 없이도 안전한 데이터 교환이 가능하게 한다.

### 10.2 Azure Policy를 활용한 거버넌스 자동화

클라우드 관리자는 Azure Policy를 사용하여 조직 전체의 AI 사용 규칙을 코드로 강제할 수 있다(Governance as Code).

-   **Allowed Regions Policy**: 데이터 주권 준수를 위해 특정 국가(예: 한국, 미국) 이외의 리전에 Foundry 리소스를 생성하는 것을 금지한다.
-   **Enforce Managed Identity**: 시스템 할당 Managed Identity가 활성화되지 않은 리소스 생성을 거부하여, 모든 앱이 키 없이 인증하도록 강제한다.
-   **SKU Restriction**: 과도한 비용 발생을 막기 위해 Provisioned Managed(PTU) SKU 생성을 특정 부서로 제한하거나, 승인된 모델만 배포 가능하도록 설정한다.

### 10.3 리소스 태깅(Tagging) 및 비용 추적 전략

대규모 기업 환경에서는 여러 부서가 하나의 Foundry Resource를 공유할 수 있다. 이때 비용을 정확히 배분(Chargeback)하기 위해 태깅 전략이 필수적이다.

-   **BusinessUnit**: `Finance`, `HR`, `Engineering` 등으로 부서를 구분한다.
-   **Environment**: `Dev`, `Test`, `Prod`로 환경을 구분하여 운영 비용을 별도로 관리한다.
-   **ProjectID**: 특정 프로젝트 코드나 비용 센터(Cost Center) 정보를 담는다.

Foundry는 이러한 태그 정보를 모든 하위 프로젝트의 호출 로그와 연동하여 Azure Cost Management에서 부서별 AI 사용량 보고서를 자동으로 생성할 수 있게 돕는다.

### 10.4 레거시 Azure AI Studio에서의 마이그레이션

2026년 이전에 구축된 Azure AI Studio 허브(Hub) 기반 시스템을 운영 중이라면, 다음 절차를 통해 Microsoft Foundry로 전환할 수 있다.

1.  **Hub Discovery**: 현재 운영 중인 허브 리소스를 식별한다.
2.  **Resource Conversion**: Azure Portal의 'Foundry Upgrade' 위저드를 사용하여 허브를 Foundry Resource로 변환한다. 이 과정에서 기존의 Project들은 자동으로 Foundry Project로 계층이 재조정된다.
3.  **Connection Validation**: 변환 후 기존의 Azure AI Search나 OpenAI Connection이 정상적으로 동작하는지 'Connection Test' 도구로 검증한다.
4.  **SDK Update**: Python `azure-ai-projects` 2.3.0(GA) 이상으로 업데이트하여 새로운 `AIProjectClient` API를 사용하도록 코드를 수정한다. Python은 Java보다 feature-complete 상태 (Hosted Agents 등 preview 기능 다수 포함).

### 10.5 데이터 보안 및 개인정보 보호 (Data Residency)

글로벌 기업이 AI를 도입할 때 가장 큰 걸림돌은 데이터가 물리적으로 어느 국가에 저장되고 처리되는가 하는 문제다. Foundry는 이를 해결하기 위해 **Data Residency** 옵션을 프로젝트 레벨에서 제공한다.

-   **Data Zone Standard**: 특정 지리적 경계(예: EU, US) 내에서만 데이터가 이동하도록 보장한다. 이는 GDPR이나 미국의 의료 데이터 보안 표준(HIPAA)을 준수해야 하는 프로젝트에 필수적이다.
-   **No Data Logging Policy**: 기본적으로 Microsoft는 고객의 데이터를 모델 학습에 사용하지 않으며, 승인된 엔터프라이즈 리소스에 대해서는 디버깅을 위한 일시적인 로깅조차 수행하지 않는 'No Logging' 옵션을 활성화할 수 있다. 이를 통해 기밀 정보 유출 가능성을 원천 차단한다.

### 10.6 리소스 정리 및 비용 최적화 체크리스트

프로젝트가 종료되거나 개발 단계에서 사용하지 않는 자원을 방치하면 의도치 않은 과금이 발생할 수 있다.

-   **Unused Deployments**: 사용하지 않는 모델 배포(특히 Provisioned SKU)는 즉시 삭제한다.
-   **Idle Compute Instances**: 인덱싱이나 평가를 위해 생성한 컴퓨팅 인스턴스가 실행 중인지 확인하고 정지시킨다.
-   **Old Data Indexes**: 테스트용으로 생성한 대규모 벡터 인덱스는 데이터 저장 비용과 검색 유닛 비용을 발생시키므로 정기적으로 정리한다.

---

## 11. 거버넌스 및 규정 준수 (Compliance)

엔터프라이즈 환경에서는 단순히 기술적으로 훌륭한 것만으로는 부족하다. Foundry는 다음과 같은 거버넌스 도구를 제공한다.

-   **Audit Logs**: 누가 어떤 모델을 언제 호출했는지, 어떤 설정을 변경했는지 모든 이력이 기록된다. 이는 규제 산업(금융, 의료 등)에서 필수적인 요구사항이다.
-   **Policy Integration**: Azure Policy와 연동하여 특정 리전에서만 모델을 배포하도록 강제하거나, 특정 SKU 사용을 제한할 수 있다.
-   **Compliance Dashboard**: 현재 시스템이 ISO, SOC2, HIPAA 등의 국제 표준을 얼마나 준수하고 있는지 한눈에 보여준다.

---

## 12. 개발자 워크플로: 아이디어에서 상용화까지 (SDLC for AI)

Foundry를 사용하는 전형적인 개발 흐름은 다음과 같다. 이 흐름을 익히는 것이 툴 자체를 익히는 것보다 중요하다.

1.  **Exploration**: 모델 카탈로그에서 프로젝트에 적합한 모델을 고른다. 벤치마크 데이터를 확인하며 가성비를 따진다.
2.  **Prototyping**: Playgrounds에서 시스템 프롬프트를 깎고 도구(Tool)들을 연동해본다. 에이전트 서비스 v2를 통해 로직을 시각화한다.
3.  **Integration**: 'View Code'를 통해 얻은 설정값으로 Python 백엔드 앱을 작성한다. FastAPI · Flask · Django 등 원하는 프레임워크와 조합. (Ch.3에서 본격적으로 시작한다)
4.  **Testing & Evaluation**: 실제 사용자의 예상 질문 셋을 돌려보고 성능 지표(Relevance, Coherence 등)를 뽑는다.
5.  **Deployment**: Global Standard에서 Global Provisioned로 업그레이드하여 실제 서비스에 런칭한다. 리전별 부하를 확인한다.
6.  **Monitoring**: Traces와 App Insights를 통해 실시간으로 시스템을 감시하고, 이상 징후 발생 시 리플레이 기능을 통해 원인을 분석한다.

---

## 13. 성공적인 시작을 위한 체크리스트

프로젝트를 시작하기 전, 다음 항목들을 체크하라.

- [ ] Azure 구독에서 `Microsoft.CognitiveServices` 리소스 공급자가 등록되어 있는가?
- [ ] 충분한 Quota(할당량)가 확보되었는가? (특히 최신 모델)
- [ ] 팀원들에게 적절한 RBAC 역할이 할당되었는가?
- [ ] 비용 알림(Cost Alert)이 설정되었는가?

---

## 14. 자주 묻는 질문 (FAQ)

**Q: 기존 Azure AI Studio와 무엇이 다른가요?**
A: 브랜드가 통합되었을 뿐만 아니라, 하부 리소스 모델이 AIServices 기반으로 단일화되었습니다. 더 이상 여러 리소스를 연결하느라 고생할 필요가 없습니다.

**Q: 비즈니스 계정이 아닌 개인 계정으로도 사용 가능한가요?**
A: 네, Azure 구독만 있다면 가능합니다. 다만 일부 최신 모델(GPT-5.5 등)은 리전과 계정 유형에 따라 초기 쿼터가 제한될 수 있습니다.

---

---

## 15. 트러블슈팅 및 일반적인 오류 (Troubleshooting)

실제 환경을 구축하다 보면 마주치게 되는 흔한 문제들과 해결책이다.

1.  **"Resource provider 'Microsoft.CognitiveServices' not registered"**
    -   **원인**: Azure 구독에 AI 서비스를 생성할 권한이 활성화되지 않았다.
    -   **해결**: Azure Portal의 'Subscriptions' 메뉴에서 해당 구독을 선택하고, 좌측의 'Resource providers'에서 `Microsoft.CognitiveServices`를 찾아 'Register'를 클릭한다. 완료까지 약 1~5분 정도 소요될 수 있다.

2.  **"Access Denied" (Foundry Portal 내 모델 배포 시)**
    -   **원인**: Azure RBAC 역할이 부족하다. 구독 소유자(Owner)라 하더라도 프로젝트 레벨에서 명시적인 매니저 권한이 없을 수 있다.
    -   **해결**: IAM 설정에서 본인에게 **'Foundry Project Manager'** 역할을 부여했는지 확인한다. 또한 리소스 그룹 레벨에서의 상속 여부도 체크해야 한다.

3.  **"Capacity exceeded" (Quota 에러)**
    -   **원인**: 해당 리전의 특정 모델(예: GPT-5.5)에 할당된 초당 토큰 수(TPM) 한도를 초과했다.
    -   **해결**: 'Management Center'의 'Quota' 탭에서 현재 사용량을 확인하고, 필요 없는 배포를 삭제하거나 추가 쿼터를 요청(Request Quota)한다. 혹은 사용량이 적은 다른 리전(예: Sweden Central)으로 리소스를 새로 생성한다.

4.  **"Connection to Data Source failed"**
    -   **원인**: 데이터 소스(Azure AI Search 등)의 방화벽 설정이나 인증 방식(Managed Identity) 설정이 잘못되었다.
    -   **해결**: 데이터 소스 리소스의 'Networking' 설정에서 'Allow Azure services'가 켜져 있는지 확인하고, Foundry Resource의 Managed Identity가 데이터 소스에 대해 'Search Index Data Reader' 권한을 가지고 있는지 검증한다.

---

## 16. 미래 전망 및 로드맵: 2026년 그 이후

Microsoft Foundry는 단순한 개발 도구를 넘어 기업의 'AI 중추'로 진화하고 있다. 향후 주목해야 할 기술적 변화는 다음과 같다.

-   **Autonomous Multi-Agent Systems**: 사람이 일일이 트리거하지 않아도 에이전트끼리 협업하여 복잡한 목표를 달성하는 시스템이 표준이 될 것이다. Foundry는 이러한 에이전트 간의 통신과 보안을 플랫폼 레벨에서 조율한다.
-   **On-device & Cloud Hybrid**: 보안이 극도로 중요한 데이터는 온디바이스 SLM(Phi-4 등)으로 처리하고, 복잡한 추론은 클라우드 LLM(GPT-5.5)으로 처리하는 하이브리드 워크플로가 강화될 것이다.
-   **Deep Integration with Microsoft 365**: 기업 내부의 엑셀, 팀즈, 아웃룩 데이터와의 연동이 더욱 깊어져, 별도의 코딩 없이도 사내 데이터 기반 AI 앱을 즉시 출시할 수 있는 환경이 조성될 것이다.

---

## 17. 결론

본 챕터에서는 Microsoft Foundry의 근본적인 철학과 2026년 리브랜딩 이후의 변화된 아키텍처를 살펴보았다. 리소스를 생성하고 프로젝트를 설정하는 것은 단순한 작업처럼 보이지만, 그 속에 담긴 RBAC, 리전 선택 전략, 보안 철학을 이해하는 것이 엔터프라이즈 AI 앱 개발의 성패를 가른다. 이제 여러분은 강력한 AI 엔진을 다룰 준비를 마쳤다. 다음 장에서는 이 엔진에 실제 모델을 얹고, Playground에서 그 능력을 시험해보는 과정을 다룰 것이다.

---

## 요약 (Cheat Sheet)

-   **Microsoft Foundry**는 단순 모델 제공을 넘어 AI 앱의 생애주기 전체를 관리하는 플랫폼이다.
-   **Foundry Resource**는 전사 관리 단위, **Project**는 개발팀 작업 단위다. (Resource 생성 후 반드시 Project를 생성하라)
-   보안의 핵심은 **Managed Identity**와 **RBAC** 설정이다. (API 키 하드코딩은 이제 금기 사항이다)
-   최신 모델과 넉넉한 쿼터를 위해 **East US** 리전에서 시작하는 것을 추천한다.
-   **Playground**의 'View Code' 기능을 활용하여 초기 Python 코딩의 뼈대를 잡아라 (Python/curl 스니펫 자동 생성).
-   비용 최적화를 위해 **Global Standard**(개발)와 **Global Batch**(대량 처리)를 적절히 혼용하라.
-   모든 모델 호출은 **Tracing**을 통해 투명하게 기록되며, 이는 디버깅과 성능 평가의 기초가 된다.

## 📚 더 읽기

-   [Microsoft Foundry 공식 문서 개요](https://learn.microsoft.com/en-us/azure/foundry/)
-   [Foundry의 새로운 기능 (2026-07 업데이트)](https://learn.microsoft.com/en-us/azure/foundry/whats-new-foundry)
-   [Foundry 아키텍처 및 리소스 모델 가이드](https://learn.microsoft.com/en-us/azure/foundry/concepts/architecture)
-   [기존 Azure OpenAI 프로젝트를 Foundry로 업그레이드하기](https://learn.microsoft.com/en-us/azure/foundry/how-to/upgrade-azure-openai)

## 다음 챕터

[Ch.2 첫 모델 배포와 Playground →](Ch02_First_Deployment.md)

---
