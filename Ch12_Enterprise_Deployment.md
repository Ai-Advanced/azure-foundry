# Chapter 12. 폐쇄망 근접 배포 · 데이터 주권 · MS 계약 검증

[← 목차로](README.md)

> **학습 목표**
> - Microsoft 가 실제로 계약적으로 보증하는 것과 마케팅 문구를 구분한다.
> - Zero Data Retention (ZDR) 의 실제 커버리지와 신청 절차, 미커버 항목을 파악한다.
> - Customer Copyright Commitment 가 **데이터 유출 보상이 아니라 IP 인덤니티** 임을 이해한다.
> - Foundry 를 "폐쇄망 근접" 수준으로 배포하는 아키텍처 (Private Endpoint + VNet Injection + Managed VNet) 를 설계한다.
> - BYOS (Bring Your Own Storage) 로 데이터 주권을 확보한다.
> - 한국 금융권 / 공공 배포 시 체크리스트를 세운다.
> - Python `azure-identity` 로 폐쇄망 환경 Managed Identity 인증을 구현한다.

> **전제 조건**
> - [← Ch.10 Security · Governance · Cost](Ch10_Security_Governance.md) 완료
> - Azure ExpressRoute 또는 VPN 지식
> - 사내 네트워크 아키텍처 이해 (허브-스포크, NSG, UDR)

---

## 📌 시작 전 정직한 정정: 3가지 흔한 오해

이 챕터는 **사내 배포를 결정하는 담당자용**. 결정을 좌우하는 근본 사실부터.

### 오해 ① "유출되면 MS 가 보상한다"

**정답: 거짓이다.** Azure 계약 어디에도 "데이터 유출 시 MS 손해 보상" 조항 없음.

- **Azure SLA** = **가동시간 99.9%** 만 보증
- 위반 시 remedy = **월 서비스 요금의 10% (99.9% 미만) / 25% (99% 미만) / 100% (95% 미만) service credit**
- 이 credit 이 **sole and exclusive remedy** 로 명시 (Azure Cognitive Services SLA)
- **보안 사고 · 데이터 유출 · 사이버 공격은 SLA 대상 아님**
- 규제 벌금 · 사용자 배상 · 합의금 → **MS 는 자동 배상 안 함** (Q&A: Azure HIPAA BAA Liability)

**한국 금융권 시사점**: 유출 시 손해배상 원하면 표준 SLA 로 부족. 개별 MSA 협상 필요 (드묾).

### 오해 ② "Customer Copyright Commitment 로 뭐든지 방어"

**정답: 부분적으로만 참.** CCC 는 **AI 생성물이 제3자 저작권을 침해했다는 클레임에 대한 방어**. 데이터 유출 방어가 **아님**.

**CCC 적용 조건** (모두 만족):
- **Azure OpenAI 모델**만 (Claude, Llama 등 파트너 모델은 미커버)
- **Metaprompt** 로 저작권 침해 방지 지시
- **Protected material text 필터** filter mode
- **Protected material code 필터** filter or annotate mode
- **Jailbreak (Prompt Shield) 필터** filter mode
- Testing/평가 보고서 보유

**저작권 이외** (특허, 상표, 영업비밀) 은 CCC 대상 아님.

### 오해 ③ "Zero Data Retention 이면 데이터 전부 안 남는다"

**정답: ZDR 은 강력하나 커버리지 좁다.**

**ZDR 커버**:
- 프롬프트 / 완성 응답
- 임베딩

**ZDR 미커버** (⚠️ 놓치기 쉬움):
- **Fine-tuning 학습 데이터** (업로드 파일 자체)
- **Agent 상태 / conversation history** (Foundry Agent Service)
- **File Search 업로드 파일**
- **RAG 인덱스** (Azure AI Search)
- **Stored completions**
- **Application Insights 로그, OpenTelemetry trace**
- **클라이언트 사이드 저장 요청/응답**

**즉, 에이전트 대화 이력과 RAG 인덱스는 ZDR 과 무관하게 저장된다.** 사용자가 사내 문서 챗봇을 만들면 그 대화와 인덱스는 **여러분의 Azure 리소스에** 저장됨 (BYOS 참조). ZDR 은 관여 안 함.

**ZDR 신청 조건**:
- **Enterprise Agreement (EA) 또는 Microsoft Customer Agreement (MCA)** 필수 — pay-as-you-go 불가
- Azure 지원 티켓 + 비즈니스 정당성 (Limited Access Form)
- 승인 2-5 영업일 (때로 7일)
- 신청서에 use-case · 데이터 민감도 · 컴플라이언스 요구사항 명시

---

## 1. 계약적으로 실제 보증되는 것

MS Learn, Product Terms, DPA 근거.

### 1.1 데이터 학습 미사용

| 항목 | 상태 | 근거 |
|---|---|---|
| 프롬프트·완성 응답 → foundation 모델 학습 안 함 | ✅ **계약 보증** | DPA + Data Privacy Doc |
| 임베딩 → foundation 모델 학습 안 함 | ✅ **계약 보증** | Data Privacy Doc |
| Fine-tuning 학습 데이터 → foundation 모델 학습 안 함 (허가 없이는) | ✅ **계약 보증** | Data Privacy Doc |
| Fine-tuned 모델은 해당 고객 전용 | ✅ **계약 보증** | "encrypted at rest, deleted anytime" |

**DPA (Data Processing Addendum)** 로 계약 구속력. **마케팅이 아니라 계약**.

### 1.2 Zero Data Retention (Modified Abuse Monitoring)

| 항목 | 상태 |
|---|---|
| 기본 abuse monitoring 저장 | **30일** (계약 보증) |
| ZDR 신청 대상 | EA/MCA 계약 고객만 |
| ZDR 승인 소요 | 2-7 영업일 |
| ZDR 활성화 검증 | `ContentLogging: FALSE` (Portal/CLI) |
| ZDR 커버 | 프롬프트, 완성 응답, 임베딩 |
| ZDR 미커버 | Agent state, RAG index, files, stored completions, app logs, OTel |
| Human review 정책 (기본) | 자동 우선, 필요시 authorized MS 직원 (SAW + JIT) |
| Human review (ZDR 후) | **완전 비활성** |
| Human reviewer 위치 (EEA 리소스) | EEA 내 authorized 직원만 |

### 1.3 Content Safety + Prompt Shield

CCC 자격 요건과 겹침 (오해 ②).

- **Content Safety (harm categories)**: Violence · Hate · Self-harm · Sexual — severity 0-6
- **Prompt Shield**: filter mode 활성 시 CCC 자격 획득
- **Protected Material Detection**: 저작권 텍스트·코드 필터
- **Groundedness Detection**: RAG 답변 컨텍스트 근거 자동 검증

⚠️ **함정**: 이 기능들 꺼두면 **CCC 자격 상실**. Foundry Agent Service default guardrail 함부로 비활성화 금지.

### 1.4 컴플라이언스 인증

Foundry 는 Azure 플랫폼 인증 상속.

| 인증 | 한국 시사점 |
|---|---|
| **K-ISMS-P** ✅ | 국내 금융권 필수 |
| ISO 27001, SOC 2 Type II ✅ | 기본 |
| HIPAA / HITECH ✅ | 의료 (BAA 필요) |
| PCI-DSS ✅ | 결제 |
| FedRAMP High/Moderate | Azure Gov Cloud 만 |
| EU Data Boundary (EUDB) ✅ | EU 개인정보 |

**Korean 금융권 참고**: [Azure K-ISMS Offering](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-korea-k-isms) — 80개 컨트롤. 금감원 클라우드 이용 가이드라인, 전자금융감독규정, 신용정보법 준수는 **고객 책임**. MS 는 플랫폼 레이어만.

---

## 2. 폐쇄망 근접 아키텍처

### 2.1 먼저 인정: 완전 air-gap 불가

**Foundry Agent Service 는 클라우드 native PaaS**. 다음 필수:
- **Azure ARM control plane** 접근
- **Model endpoint** 접근 (Azure OpenAI or 파트너)

인터넷 자체 완전 차단 불가. 하지만 다음 조합으로 **"이론상 노출 있으나 실질 통제 가능"** 수준 도달.

- Foundry endpoint → **Public 완전 차단** (`publicNetworkAccess: Disabled`)
- Foundry endpoint → **Private Endpoint 로만 접근**
- Agent 아웃바운드 → **Managed VNet + Allow-only-approved-outbound**
- Foundry Resource → **VNet 내 delegated subnet 에서만 실행**
- BYOS → **Cosmos DB / Blob / AI Search 모두 customer VNet Private Endpoint**
- 온프렘 접근 → **ExpressRoute 또는 VPN**

⚠️ **함정**: "완전 폐쇄망" 요구받으면 **금감원·조달청과 사전 협의**. Foundry 는 **Azure China / Azure Local (Stack HCI) 미지원**. 정부·군은 **Azure Government (US Gov Cloud)** 만 (한국 조직 부적합).

### 2.2 아키텍처 다이어그램 (권장 표준)

```
┌────────────────────────────────────────────────────────────────┐
│ 온프레미스 (사내 네트워크)                                       │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ Legacy API (ERP/CRM) · Corporate DB · Internal DNS      │  │
│  └─────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
         ▲ ExpressRoute 또는 Site-to-Site VPN
┌────────────────────────────────────────────────────────────────┐
│ Azure Hub VNet (공유 서비스 계층)                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ Azure Firewall (FQDN allow-list · IP allow-list)        │  │
│  │ Application Gateway (온프렘 API L7 프록시)               │  │
│  │ VPN Gateway / ExpressRoute Gateway                      │  │
│  │ Private DNS Resolver                                     │  │
│  └─────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
         ▲ VNet Peering (allow-forwarded-traffic)
┌────────────────────────────────────────────────────────────────┐
│ Azure Spoke VNet (Foundry)                                     │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ Delegated Subnet /24 (Microsoft.App/environments)       │  │
│  │  - Foundry Hosted Agent Micro VM                        │  │
│  │  - Data Proxy (single-tenant per project)               │  │
│  │  - UDR → Hub Firewall (모든 아웃바운드)                  │  │
│  └─────────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ Private Endpoint Subnet                                 │  │
│  │  - Foundry Account (privatelink.services.ai.azure.com)  │  │
│  │  - Cosmos DB (agent thread state · 3000 RU/s 최소)      │  │
│  │  - Blob Storage (file uploads)                          │  │
│  │  - Azure AI Search (RAG index · semantic ranker)        │  │
│  │  - Key Vault (secrets)                                  │  │
│  │  - App Insights (관측)                                   │  │
│  └─────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

### 2.3 Private Endpoint 필수 DNS Zone (3개)

`publicNetworkAccess: Disabled` 후 Private Endpoint 로만 접근.

**Foundry Account용 Private DNS Zone 필수**:
- `privatelink.cognitiveservices.azure.com`
- `privatelink.openai.azure.com`
- `privatelink.services.ai.azure.com` (신 통합 endpoint)

⚠️ **함정**: BYOS 리소스 (Cosmos DB, Storage, AI Search) 의 Private Endpoint 는 **자동 생성 안 됨**. 각 리소스 별도 생성.

### 2.4 VNet Injection (Agent 아웃바운드)

Foundry Agent Service 만의 특별 기능. **에이전트 코드가 여러분 VNet 안에서 실행**.

**Subnet 요구사항:**

| 항목 | 값 |
|---|---|
| **Delegation** | `Microsoft.App/environments` (필수) |
| **최소 크기** | /27 (32 IP) — 프로덕션 위험 |
| **권장 크기** | **/24 (256 IP)** |
| 50 동시 세션 | /26 (64 IP) |
| IP 대역 | RFC 1918 only (`10.x`, `172.16-31.x`, `192.168.x`) |
| 미지원 | Public IP, CGNAT (`100.64.0.0/10`) |

### 2.5 Managed VNet (아웃바운드 통제 표준)

```bash
# Allow-only-approved-outbound 모드
az cognitiveservices account managed-network update \
  --resource-group rg-foundry-prod \
  --name foundry-prod-01 \
  --isolation-mode AllowOnlyApprovedOutbound
```

**아웃바운드 rule 예** (Python 앱 최소 allow-list):

```
Service Tags:
  - AzureActiveDirectory   (Entra ID 인증)
  - Storage                (BYOS storage)
  - AzureMonitor           (App Insights)
  - AzureMachineLearning   (Foundry service)

FQDN:
  - *.identity.azure.net
  - login.microsoftonline.com
  - *.blob.core.windows.net
  - settings.sdk.monitor.azure.com
  - mcr.microsoft.com          (컨테이너 이미지 pull)
  - pypi.org, files.pythonhosted.org  (필요 시 - runtime pip install)
```

⚠️ **함정**: Managed VNet 은 FQDN 룰이 **port 80/443 만 지원**. 다른 포트는 hub-and-spoke + Azure Firewall NVA 로.

---

## 3. BYOS — 데이터 주권의 실체

Foundry Agent Service **Standard Agent Setup** 사용 시 다음 데이터가 **여러분 Azure 리소스**에 저장됨. 데이터 주권의 핵심.

### 3.1 Cosmos DB — Agent Conversation State

Foundry Project 당 자동 생성 컨테이너:

| 컨테이너 | 저장 데이터 | RU/s |
|---|---|---|
| `{project-id}-thread-message-store` | 대화 메시지 | 1000 |
| `{project-id}-system-thread-message-store` | 시스템 메시지 | 1000 |
| `{project-id}-agent-entity-store` | Agent 메타데이터 | 1000 |
| `{project-id}-agent-definitions-v1` (Responses API) | Agent 버전 | 1000 |
| `{project-id}-run-state-v1` (Responses API) | 실행 상태 | 1000 |

**최소 처리량**: 3000 RU/s (기본 3컨테이너), 프로젝트 당 3000 RU/s 추가. Responses API 사용 시 +2000 RU/s.

**Bicep 사전 프로비저닝:**

```bicep
resource cosmos 'Microsoft.DocumentDB/databaseAccounts@2024-08-15' = {
  name: 'cosmos-foundry-agent'
  properties: {
    databaseAccountOfferType: 'Standard'
    locations: [{ locationName: location }]
    publicNetworkAccess: 'Disabled'  // Private Endpoint only
    capabilities: [{ name: 'EnableServerless' }]
  }
}
```

### 3.2 Blob Storage — File Uploads

두 개 컨테이너 필요:
- `-azureml-blobstore` (역할: **Storage Blob Data Contributor**) — 중간 데이터 (chunks, embeddings)
- `-agents-blobstore` (역할: **Storage Blob Data Owner**) — 사용자 업로드 파일

**⚠️ 제약**: Private Network 셋업에서 **Azure Blob Storage 기반 File Search 툴은 미지원**. OneDrive 나 Custom MCP 대안 (Ch.13).

### 3.3 Azure AI Search — RAG Index

Ch.6 RAG 파이프라인 연결. Private Endpoint + Semantic Ranker (Standard tier 이상).

**Indexer executionEnvironment**: Private Endpoint 통과 시 반드시 `"Private"`.

### 3.4 Foundry Prompts CMS — ⚠️ 결손

**현재 (2026-07)**: Prompts CMS 데이터는 **Foundry-managed storage**, customer-controlled 이관 불가. Private Endpoint 지원 미문서화.

**대안**:
- 프롬프트를 **Git canonical** (Ch.11 하이브리드)
- CI 파이프라인이 Prompts CMS 로 sync
- 민감 프롬프트는 CMS 대신 앱 코드 + Key Vault

---

## 4. 온프렘 데이터 접근 패턴

### 4.1 ExpressRoute (권장 - 프로덕션)

전용선 private 연결. **금융권 표준**.

- 대역폭: 50 Mbps ~ 100 Gbps
- Route filter 로 Microsoft peering vs Private peering 분리
- Foundry delegated subnet 은 spoke VNet 통해 ExpressRoute 로 (UDR)

### 4.2 Site-to-Site VPN (개발/스테이징)

VPN Gateway + on-prem VPN 기기. IPSec 터널. ExpressRoute 대비 저비용, 대역폭·지연·SLA 열등.

### 4.3 Application Gateway 프록시 (Managed VNet)

Managed VNet 사용 시 delegated subnet 을 사용자가 못 다루므로 Application Gateway 프록시.

```
Foundry Managed VNet
    ↓ Private Endpoint
Application Gateway (private frontend IP)
    ↓ VNet Peering + UDR
On-Prem Resources (via ExpressRoute/VPN)
```

**⚠️ 제한**: L7 (HTTP/HTTPS) 만 지원. TCP/UDP 필요하면 Azure Firewall NVA.

### 4.4 Logic Apps Hybrid Connections — ❌ 미지원

공식적으로 "under development" (2026-07). Foundry Agent 에서 Logic Apps tool 사용 불가. 대체: Custom Function (Azure Functions) + Application Gateway.

---

## 5. Egress 통제 — 어디가 위험한가

각 툴이 **어느 IP 로 나가는지** 알면 데이터 유출 리스크 정확히 통제.

| 툴 | 아웃바운드 origin | 데이터 leak 리스크 |
|---|---|---|
| **Custom Function (Azure Functions)** | 여러분 VNet | ✅ 낮음 (VNet 방화벽 적용) |
| **OpenAPI Tool** | 여러분 VNet | ✅ 낮음 |
| **Custom MCP Server (VNet 내)** | 여러분 VNet | ✅ 낮음 |
| **Agent-to-Agent (A2A)** | 여러분 VNet | ✅ 낮음 |
| **Azure AI Search grounding** | 여러분 VNet (BYOS Search) | ✅ 낮음 |
| **File Search (Foundry-uploaded)** | Foundry managed | ⚠️ 중간 (BYOS Storage 격리 가능) |
| **Web Search / Bing Grounding** | **Foundry public endpoint** | ❌ 높음 |
| **SharePoint Grounding** | Foundry public (M365 API) | ⚠️ 중간 (M365 tenant 격리) |
| **Public MCP Server** | Public 인터넷 | ❌ 높음 |

⚠️ **결정적 함정 — Web Search**: `Bing Grounding` 툴은 여러분 VNet 이 아닌 **Foundry 공용 endpoint** 로 검색어를 보냄. 검색어에 사내 정보 실리면 사실상 **MS 공용 인프라 → Bing** 흘러감. 사내 문서 챗봇에는 **Web Search 툴 비활성** 또는 사용자 승인 후만 허용.

💡 **팁**: Foundry Toolbox (Ch.13) 로 툴별 activation 조건을 중앙 관리. 예: "회사 정보 카테고리 프롬프트에서는 Web Search 자동 비활성."

---

## 6. 한국 금융권 · 공공 배포 체크리스트

법·규제 준수는 고객 책임이나, Foundry 배포 시 담당자 체크:

### 6.1 법 · 규제

- [ ] **개인정보보호법**: 개인정보 처리 위탁 계약 (Azure DPA)
- [ ] **신용정보법** (신용정보 회사): 처리 시 사전 승인 · 감사
- [ ] **전자금융감독규정** (금융권): 클라우드 이용 승인, 3rd party 감사권
- [ ] **금감원 클라우드 이용 가이드라인** (2024/2025 개정): 정보자산 등급별 사용 범위
- [ ] **공공 클라우드 이용 가이드**: 국정원 CSAP 요구사항

### 6.2 계약 · 인증

- [ ] Azure DPA (Data Processing Addendum) 최신본 서명
- [ ] EA 또는 MCA 계약 (ZDR 신청 필수)
- [ ] K-ISMS-P 리포트 획득 → 내부 IT 감사팀 제출
- [ ] SOC 2 Type II 리포트 획득 → 외부 감사인 제출
- [ ] BAA (HIPAA 적용 시)

### 6.3 기술 통제

- [ ] `publicNetworkAccess: Disabled` 확인
- [ ] Private Endpoint (Foundry account) 활성
- [ ] Private Endpoint (Cosmos DB, Blob, AI Search, Key Vault) 활성
- [ ] Private DNS Zone 3개 등록 및 VNet 연결
- [ ] Managed VNet + Allow-only-approved-outbound 활성
- [ ] Firewall FQDN allow-list 최소화 (엄격 시 IP allow-list)
- [ ] BYOS: Cosmos DB · Blob · AI Search 모두 customer-owned + private
- [ ] Managed Identity 만으로 앱 인증 (API 키 하드코딩 금지)
- [ ] Customer Managed Keys (CMK) 활성 (Key Vault 통해)
- [ ] Content Safety + Prompt Shield 활성 (CCC 자격 유지)
- [ ] Purview DLP 정책 활성 (민감정보 프롬프트 차단)
- [ ] Application Insights + OTel 트레이싱 활성
- [ ] Foundry Control Plane 에서 fleet 컴플라이언스 모니터링

### 6.4 운영 · 거버넌스

- [ ] ZDR 신청서 제출 완료 → 승인 코멘트 문서화
- [ ] Cosmos DB / Blob 데이터 삭제 SOP (사용자 요청 시)
- [ ] Fine-tuning 데이터 삭제 SOP
- [ ] Audit log Retention (금감원: **5년**)
- [ ] 사용자 access review 분기별
- [ ] Model deployment 승인 워크플로 (Azure Policy - Ch.8)
- [ ] 데이터 유출 대응 SOP (MS 자동 보상 없음 — 자체)

---

## 7. 🔧 실습: 폐쇄망 근접 Foundry 배포 + Python 인증

### 7.1 pyproject.toml 추가

```bash
uv add "azure-identity>=1.25.3" "azure-keyvault-secrets>=4.9"
```

### 7.2 폐쇄망 Bicep 스택 (핵심)

```bicep
// infra/foundry-private.bicep
targetScope = 'resourceGroup'

param location string = 'koreacentral'
param vnetName string = 'vnet-foundry-prod'
param foundryName string = 'foundry-prod-01'

// 1. VNet with subnets
resource vnet 'Microsoft.Network/virtualNetworks@2024-05-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: { addressPrefixes: ['10.0.0.0/16'] }
    subnets: [
      {
        name: 'foundry-agent-subnet'
        properties: {
          addressPrefix: '10.0.1.0/24'
          delegations: [
            {
              name: 'foundry-delegation'
              properties: { serviceName: 'Microsoft.App/environments' }
            }
          ]
        }
      }
      {
        name: 'private-endpoint-subnet'
        properties: {
          addressPrefix: '10.0.2.0/24'
          privateEndpointNetworkPolicies: 'Disabled'
        }
      }
    ]
  }
}

// 2. Foundry Resource - Public 차단, MI 강제
resource foundry 'Microsoft.CognitiveServices/accounts@2024-10-01' = {
  name: foundryName
  location: location
  kind: 'AIServices'
  sku: { name: 'S0' }
  identity: { type: 'SystemAssigned' }
  properties: {
    customSubDomainName: foundryName
    publicNetworkAccess: 'Disabled'   // ← 폐쇄망 핵심
    disableLocalAuth: true             // ← API Key 금지, Entra ID 강제
    networkAcls: {
      defaultAction: 'Deny'
      virtualNetworkRules: []
      ipRules: []
    }
  }
}

// 3. Private Endpoint (Foundry)
resource foundryPe 'Microsoft.Network/privateEndpoints@2024-05-01' = {
  name: 'pe-${foundryName}'
  location: location
  properties: {
    subnet: { id: '${vnet.id}/subnets/private-endpoint-subnet' }
    privateLinkServiceConnections: [
      {
        name: 'foundry-plsc'
        properties: {
          privateLinkServiceId: foundry.id
          groupIds: ['account']
        }
      }
    ]
  }
}

// 4. Private DNS Zones (3개 필수)
resource dnsAoai 'Microsoft.Network/privateDnsZones@2024-06-01' = {
  name: 'privatelink.openai.azure.com'
  location: 'global'
}
resource dnsCogSvc 'Microsoft.Network/privateDnsZones@2024-06-01' = {
  name: 'privatelink.cognitiveservices.azure.com'
  location: 'global'
}
resource dnsAiSvc 'Microsoft.Network/privateDnsZones@2024-06-01' = {
  name: 'privatelink.services.ai.azure.com'
  location: 'global'
}

// 5. Cosmos DB (BYOS)
resource cosmos 'Microsoft.DocumentDB/databaseAccounts@2024-08-15' = {
  name: 'cosmos-${foundryName}'
  location: location
  properties: {
    databaseAccountOfferType: 'Standard'
    locations: [{ locationName: location }]
    publicNetworkAccess: 'Disabled'
    ipRules: []
    virtualNetworkRules: []
  }
}

// 6. AI Search (BYOS RAG)
resource search 'Microsoft.Search/searchServices@2024-06-01-preview' = {
  name: 'search-${foundryName}'
  location: location
  sku: { name: 'standard' }
  properties: {
    publicNetworkAccess: 'disabled'
    semanticSearch: 'standard'
    replicaCount: 1
    partitionCount: 1
  }
}

// (Private Endpoint for Cosmos, Search 는 별도 리소스 반복)

output foundryEndpoint string = foundry.properties.endpoint
```

배포:

```bash
az deployment group create \
  --resource-group rg-foundry-prod \
  --template-file infra/foundry-private.bicep \
  --parameters location=koreacentral
```

### 7.3 Python 앱 - Managed Identity 인증 (API Key 금지 환경)

폐쇄망에서는 **API Key 금지** (`disableLocalAuth: true`). Managed Identity + Entra ID 만.

```python
# src/foundry_app/private_client.py
"""폐쇄망 환경 Foundry 접근 - Managed Identity 만."""
from __future__ import annotations
import os
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import AzureOpenAI


def create_private_client() -> AzureOpenAI:
    """Container Apps / VM 에서 Managed Identity 로 접근.

    - Container Apps: System-Assigned MI 자동
    - 로컬 dev: az login fallback (dev subscription 별도 권장)
    """
    endpoint = os.environ["AZURE_FOUNDRY_ENDPOINT"]
    # 예: https://foundry-prod-01.services.ai.azure.com
    api_version = os.environ.get("AZURE_FOUNDRY_API_VERSION", "2025-01-01-preview")

    # DefaultAzureCredential 체인:
    #  1. Managed Identity (Container Apps / VM / Function)
    #  2. Workload Identity (AKS)
    #  3. Azure CLI (`az login`) - 로컬 dev
    #  4. ...
    credential = DefaultAzureCredential()

    # Cognitive Services 전용 scope
    token_provider = get_bearer_token_provider(
        credential,
        "https://cognitiveservices.azure.com/.default",
    )

    return AzureOpenAI(
        azure_endpoint=endpoint,
        azure_ad_token_provider=token_provider,   # ← API Key 아님
        api_version=api_version,
    )


if __name__ == "__main__":
    client = create_private_client()
    response = client.chat.completions.create(
        model=os.environ["AZURE_FOUNDRY_DEPLOYMENT"],
        messages=[{"role": "user", "content": "Private Endpoint 인증 성공?"}],
        max_completion_tokens=100,
        temperature=1.0,
    )
    print(response.choices[0].message.content)
```

⚠️ **함정**: 로컬 dev 는 `az login` fallback 이나 **폐쇄망 프로덕션 로컬 접근은 애초에 안 됨**. 개발 subscription 별도, 프로덕션은 Container Apps 내부에서만.

### 7.4 Key Vault Secret 로드

```python
# src/foundry_app/private_secrets.py
"""폐쇄망 Key Vault 접근 - Private Endpoint 통해."""
import os
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

kv_client = SecretClient(
    vault_url=os.environ["KEY_VAULT_URL"],   # e.g. https://kv-foundry.vault.azure.net/
    credential=DefaultAzureCredential(),
)

# 예: 외부 시스템 API key
external_api_key = kv_client.get_secret("external-crm-api-key").value
```

⚠️ **함정**: Key Vault 도 Private Endpoint + `privatelink.vaultcore.azure.net` DNS Zone 필수. Bicep 에 추가.

---

## 8. Foundry Control Plane 으로 fleet 감시

Ch.10 재소환. 폐쇄망 배포 후 여러 Foundry 리소스 · 프로젝트 · 배포를 통합 관리.

- **Fleet Overview**: 전사 Foundry 리소스 한눈에
- **Compliance Dashboard**: Azure Policy 위반 · Guardrail 미활성
- **Cost Anomaly**: 급증 알림
- **Defender for Cloud 통합**: 보안 이벤트
- **Purview 통합**: 데이터 이동 audit

Portal: `Foundry Portal → Management Center → Control Plane`

폐쇄망에서도 Control Plane 콘솔은 관리자용 개별 세션 (Managed Identity + 조건부 접근 정책 권장).

---

## 9. 흔한 함정 정리

⚠️ 재확인:

1. **"ZDR 만 켜면 다 안전"**: agent state, RAG index, file upload 는 ZDR 미커버. BYOS 격리.
2. **"CCC 로 유출 보호"**: CCC 는 저작권 인덤니티. 유출 보상 아님.
3. **Foundry account 만 Private Endpoint**: Cosmos, Blob, AI Search 각각도 필요. 자동 생성 안 됨.
4. **Private DNS Zone 3개 다 등록 안 함**: `services.ai.azure.com` 놓치면 신 통합 endpoint 안 뜸.
5. **Delegated subnet 크기 /28**: 프로덕션 IP 고갈. 최소 /24.
6. **Web Search 활성 사내 챗봇**: 검색어 Bing 으로 나감. 폐쇄망 정신 위배.
7. **`disableLocalAuth: false`**: API Key 접근 허용 → MI 원칙 위배.
8. **Prompts CMS 프로덕션 canonical**: 여전히 Foundry-managed. 민감 프롬프트는 Git.
9. **Global Standard 배포**: 데이터 여러 지역 처리. 국내 데이터 거주성 필요하면 **Regional (Korea Central) 또는 Data Zone**.
10. **Azure China / Azure Local 검토**: Foundry 미지원. 처음부터 배제.

💡 **베스트 프랙티스**:

- **최소 특권**: Foundry Project 마다 별도 Managed Identity, 최소 역할.
- **명시적 승인 필수**: Model deployment 는 Azure Policy 허용 리스트 강제 (Ch.8).
- **정기 감사**: 분기별 Control Plane 컴플라이언스 리포트, access review.
- **Fine-tuning 데이터**: 별도 Blob 컨테이너, encryption 강화, 삭제 SOP.
- **Purview DLP 실전 활성**: PII, 결제 정보, 사내 코드 leak 방지.

---

## 요약 (Cheat Sheet)

- **"유출되면 보상" — 거짓**. Azure SLA 는 uptime only, credit 이 sole remedy.
- **CCC** = AI 생성물 저작권 침해 인덤니티. 유출 보상 아님.
- **데이터 학습 미사용** = ✅ 계약 보증 (DPA + Product Terms).
- **ZDR** = ✅ 진짜지만 조건 (EA/MCA + 승인 + 프롬프트/완성/임베딩만 커버).
- **폐쇄망 완전 air-gap 불가**. Private Endpoint + VNet Injection + Managed VNet + BYOS 실질 최대치.
- **Azure China / Azure Local 미지원**, **Azure Government 는 한국 조직 부적합**.
- **한국 금융권 체크리스트** = K-ISMS-P + DPA 서명 + BYOS 격리 + MI + Purview DLP + audit 5년.
- **Web Search 툴은 폐쇄망 정신 위배**. 사내 챗봇 default 비활성.

## 📚 더 읽기

- [Data privacy for Foundry Models](https://learn.microsoft.com/en-us/azure/foundry/responsible-ai/openai/data-privacy)
- [Zero Data Retention 신청](https://learn.microsoft.com/en-us/answers/questions/5834904/enable-zero-data-retention)
- [Customer Copyright Commitment 요건](https://learn.microsoft.com/en-us/azure/foundry/responsible-ai/openai/customer-copyright-commitment)
- [Configure network isolation for Foundry](https://learn.microsoft.com/en-us/azure/foundry/how-to/configure-private-link)
- [Foundry Agent Service Networking Deep Dive](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agents-networking-deep-dive)
- [Standard Agent Setup (BYOS)](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/standard-agent-setup)
- [Access on-premises resources from Foundry](https://learn.microsoft.com/en-us/azure/foundry/how-to/access-on-premises-resources)
- [Azure K-ISMS Offering](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-korea-k-isms)
- [Azure OpenAI Service SLA (uptime only)](https://www.azure.cn/en-us/support/sla/cognitive-services/)
- [Microsoft Products and Services DPA](https://www.microsoft.com/licensing/docs/view/Microsoft-Products-and-Services-Data-Protection-Addendum-DPA/)

## 다음 챕터

[Ch.13 사내 툴 통합 Enterprise Agent 실전 (Python) →](Ch13_Enterprise_Agent_Integration.md)

---
