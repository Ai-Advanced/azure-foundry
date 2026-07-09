# Chapter 12. 폐쇄망 근접 배포 · 데이터 주권 · MS 계약 검증

[← 목차로](README.md)

> **학습 목표**
> - Microsoft가 실제로 계약적으로 보증하는 것과 마케팅 문구를 구분한다.
> - Zero Data Retention (ZDR)의 실제 커버리지와 신청 절차, 미커버 항목을 파악한다.
> - Customer Copyright Commitment 가 **데이터 유출 보상이 아니라 IP 인덤니티**임을 이해한다.
> - Foundry를 "폐쇄망 근접" 수준으로 배포하는 아키텍처(Private Endpoint + VNet Injection + Managed VNet)를 설계한다.
> - BYOS(Bring Your Own Storage)로 데이터 주권을 확보한다.
> - 한국 금융권/공공 배포 시 체크리스트를 세운다.

> **전제 조건**
> - [← Ch.10 Security · Governance · Cost](Ch10_Security_Governance.md) 완료
> - Azure ExpressRoute 또는 VPN 지식
> - 사내 네트워크 아키텍처 이해 (허브-스포크, NSG, UDR)

---

## 📌 시작 전 정직한 정정: 3가지 흔한 오해

이 챕터는 **사내 배포를 결정하는 담당자용**이다. 결정을 좌우하는 근본 사실부터 바로잡는다.

### 오해 ① "유출되면 MS가 보상한다"

**정답: 거짓이다.** Azure 계약 어디에도 "데이터 유출 시 MS가 손해를 보상" 이라는 조항은 없다.

- **Azure SLA**는 **가동시간(uptime) 99.9%** 만 보증한다.
- 위반 시 remedy는 **월 서비스 요금의 10% (99.9% 미만) / 25% (99% 미만) / 100% (95% 미만) service credit** 뿐이다.
- 이 credit이 **sole and exclusive remedy** 로 명시되어 있다 (Azure Cognitive Services SLA §).
- **보안 사고, 데이터 유출, 사이버 공격은 SLA 대상이 아니다.**
- 규제 벌금, 사용자 배상, 합의금 → **MS는 자동 배상하지 않는다** (Q&A: Azure HIPAA BAA Liability).

**한국 금융권 시사점**: 유출 시 손해배상을 원한다면 표준 SLA로는 부족하다. 개별 MSA(Master Services Agreement)에 별도 조항을 협상해야 하며, 이는 rare하다.

### 오해 ② "Customer Copyright Commitment 로 뭐든지 방어된다"

**정답: 부분적으로만 참.** CCC는 **AI 생성물이 제3자 저작권을 침해했다는 클레임에 대한 방어**다. 데이터 유출 방어가 **아니다**.

CCC 적용 조건 (모두 만족해야):
- **Azure OpenAI 모델**만 대상 (Claude, Llama 등 파트너 모델은 미커버)
- **Metaprompt**로 저작권 침해 방지 지시 명시
- **Protected material text 필터**를 filter mode로 활성화
- **Protected material code 필터**를 filter 또는 annotate mode로 활성화
- **Jailbreak (Prompt Shield) 필터**를 filter mode로 활성화
- 테스트/평가 보고서 보유

**저작권 이외의 클레임 (특허, 상표, 영업비밀)** 은 CCC 대상 아니다.

### 오해 ③ "Zero Data Retention 이면 데이터 전부 안 남는다"

**정답: ZDR은 강력하지만 커버리지가 좁다.**

**ZDR이 커버하는 것:**
- 프롬프트 / 완성 응답
- 임베딩

**ZDR이 커버하지 않는 것 (⚠️ 놓치기 쉬움):**
- **Fine-tuning 학습 데이터** (업로드된 파일 자체)
- **Agent 상태 / thread / conversation history** (Foundry Agent Service 사용 시)
- **File Search 업로드 파일**
- **RAG 인덱스** (Azure AI Search)
- **Stored completions**
- **Application Insights 로그, OpenTelemetry trace**
- **클라이언트 사이드 저장한 요청/응답**

즉, **에이전트 대화 이력과 RAG 인덱스는 ZDR과 무관하게 저장된다.** 사용자님이 사내 문서 챗봇을 만들면, 그 대화와 인덱스는 **여러분의 Azure 리소스에** 저장된다 (Ch.10의 BYOS 참조). ZDR은 이 부분은 관여하지 않는다.

**ZDR 신청 조건:**
- **Enterprise Agreement (EA) 또는 Microsoft Customer Agreement (MCA)** 필수 — pay-as-you-go 계정은 불가
- Azure 지원 티켓 + 비즈니스 정당성 (Limited Access Form)
- 승인 2~5 영업일 (때로 7일)
- 신청서에 use-case와 데이터 민감도, 컴플라이언스 요구사항 명시

---

## 1. 계약적으로 실제 보증되는 것 (Contractual Guarantees)

MS Learn, Product Terms, DPA 를 근거로 정리.

### 1.1 데이터 학습 미사용 (Data Not Used for Training)

| 항목 | 상태 | 근거 |
|---|---|---|
| 프롬프트·완성 응답 → foundation 모델 학습 안 함 | ✅ **계약 보증** | Product Terms + [Data Privacy Doc](https://learn.microsoft.com/en-us/azure/foundry/responsible-ai/openai/data-privacy) |
| 임베딩 → foundation 모델 학습 안 함 | ✅ **계약 보증** | Data Privacy Doc |
| Fine-tuning 학습 데이터 → foundation 모델 학습 안 함 (허가 없이는) | ✅ **계약 보증** | Data Privacy Doc |
| Fine-tuned 모델은 해당 고객 전용 | ✅ **계약 보증** | Data Privacy Doc: "Fine-tuned models are exclusively available to the customer whose data was used to create the fine-tuned model, are encrypted at rest, and can be deleted by the customer at any time." |

이 부분은 **DPA (Data Processing Addendum)** 로 계약 구속력을 가진다. **마케팅이 아니라 계약이다.**

### 1.2 Zero Data Retention (모드별 Abuse Monitoring)

| 항목 | 상태 |
|---|---|
| 기본 abuse monitoring 저장 기간 | **30일** (계약 보증) |
| ZDR 신청 대상 | EA/MCA 계약 고객만 |
| ZDR 승인 소요 | 2-7 영업일 |
| ZDR 활성화 검증 | `ContentLogging: FALSE` (Azure Portal / CLI) |
| ZDR 커버 | 프롬프트, 완성 응답, 임베딩 |
| ZDR 미커버 | Agent state, RAG index, file uploads, stored completions, app logs, OTel trace |
| Human review 정책 (기본) | 자동 검토가 우선, 필요 시에만 authorized MS 직원 (SAW + JIT approval) |
| Human review 정책 (ZDR 승인 후) | **인간 검토 완전 비활성** |
| Human reviewer 위치 (EEA 리소스) | EEA 내 authorized 직원만 |

### 1.3 Content Safety + Prompt Shield

이 부분은 CCC 자격 요건과 겹친다 (오해 ② 참조).

- **Content Safety (harm categories)**: Violence / Hate / Self-harm / Sexual — 각 severity 0~6, threshold 설정
- **Prompt Shield (jailbreak detection)**: filter mode 활성화 시 CCC 자격 획득
- **Protected Material Detection**: 저작권 있는 텍스트(가사·기사)·코드 필터
- **Groundedness Detection**: RAG 답변의 컨텍스트 근거 자동 검증

**⚠️ 함정**: 이 기능들을 **꺼두면 CCC 자격 상실**. Foundry Agent Service의 default guardrail을 함부로 비활성화하지 말 것.

### 1.4 컴플라이언스 인증

Foundry는 Azure 플랫폼 인증을 상속받는다.

| 인증 | 상태 | Korean 시사점 |
|---|---|---|
| **K-ISMS-P** | ✅ Azure 획득 | 국내 금융권 필수 |
| **ISO 27001** | ✅ Azure 획득 | 기본 |
| **SOC 2 Type II** | ✅ Azure 획득 | 기본 |
| **HIPAA / HITECH** | ✅ BAA 가능 | 의료 |
| **PCI-DSS** | ✅ | 결제 |
| **FedRAMP High/Moderate** | ✅ Azure Gov Cloud | 미 정부용 |
| **EU Data Boundary (EUDB)** | ✅ | EU 개인정보 |

**Korean 금융권 참고**: [Azure K-ISMS Offering](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-korea-k-isms) — 80개 컨트롤. 금감원 클라우드 이용 가이드라인, 전자금융감독규정, 신용정보법에 대한 준수는 **고객 책임**이며 MS는 flatform 레이어만 커버함을 명심.

---

## 2. 폐쇄망 근접 아키텍처 — "완전 폐쇄망은 불가, 어디까지 가능한가"

### 2.1 먼저 인정할 것: 완전 air-gap 불가

**Foundry Agent Service는 클라우드 native PaaS다.** 다음이 필수적이다.

- **Azure ARM control plane** 접근 (관리 API)
- **Model endpoint** 접근 (Azure OpenAI 또는 파트너 모델)

즉, 인터넷 자체를 완전 차단할 수는 없다. 하지만 다음 조합으로 **"이론상 노출은 있으나 실질적으로 통제 가능"** 수준까지 갈 수 있다.

- Foundry endpoint → **Public 완전 차단** (`publicNetworkAccess: Disabled`)
- Foundry endpoint → **Private Endpoint 로만 접근**
- Agent 아웃바운드 → **Managed VNet + Allow-only-approved-outbound**
- Foundry Resource → **VNet 내 delegated subnet 에서만 실행**
- BYOS → **Cosmos DB / Blob / AI Search 모두 customer VNet 내 Private Endpoint**
- 온프렘 접근 → **ExpressRoute 또는 VPN**

⚠️ **함정**: "완전 폐쇄망" 을 요구받으면 **금감원·조달청과 사전 협의**하라. Foundry는 **Azure China / Azure Local (Stack HCI) 에서 사용 불가**하다. 정부·군은 **Azure Government (US Gov Cloud)** 만 옵션인데, 이는 한국 조직에는 적용 불가.

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

### 2.3 Private Endpoint 설정 요약

`publicNetworkAccess: Disabled` 후 Private Endpoint 로만 접근.

**Foundry Account용 Private DNS Zone (셋 다 필요):**
- `privatelink.cognitiveservices.azure.com`
- `privatelink.openai.azure.com`
- `privatelink.services.ai.azure.com` (신 통합 endpoint)

⚠️ **함정**: 관련 BYOS 리소스 (Cosmos DB, Storage, AI Search)의 Private Endpoint 는 **자동 생성되지 않는다**. 각 리소스 별도로 생성 필요.

### 2.4 VNet Injection (Agent 아웃바운드)

Foundry Agent Service 만의 특별 기능. **에이전트 코드가 여러분의 VNet 안에서 실행**된다.

**Subnet 요구사항:**
| 항목 | 값 |
|---|---|
| **Delegation** | `Microsoft.App/environments` (필수) |
| **최소 크기** | /27 (32 IP) — 프로덕션엔 위험 |
| **권장 크기** | **/24 (256 IP)** |
| **50 동시 세션** | /26 (64 IP) |
| **IP 대역** | RFC 1918 only (`10.x`, `172.16-31.x`, `192.168.x`) |
| **미지원** | Public IP, CGNAT (`100.64.0.0/10`) |

### 2.5 Managed VNet (아웃바운드 통제 표준)

```bash
# Allow-only-approved-outbound 모드로 전환
az cognitiveservices account managed-network \
  update \
  --resource-group rg-foundry-prod \
  --name foundry-prod-01 \
  --isolation-mode AllowOnlyApprovedOutbound
```

**아웃바운드 rule 예시** (Java 앱이 필요한 최소 allow-list):
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
```

⚠️ **함정**: Managed VNet 은 FQDN 룰이 **port 80/443 만 지원**. 다른 포트가 필요하면 hub-and-spoke + Azure Firewall NVA 로 확장.

---

## 3. BYOS — 데이터 주권의 실체

Foundry Agent Service의 **Standard Agent Setup** 을 쓰면 다음 데이터가 **여러분의 Azure 리소스**에 저장된다. 이게 데이터 주권 확보의 핵심이다.

### 3.1 Cosmos DB — Agent Conversation State

Foundry Project 당 다음 컨테이너가 자동 생성된다.

| 컨테이너 | 저장 데이터 | RU/s |
|---|---|---|
| `{project-id}-thread-message-store` | 대화 메시지 | 1000 |
| `{project-id}-system-thread-message-store` | 시스템 메시지 | 1000 |
| `{project-id}-agent-entity-store` | Agent 메타데이터 | 1000 |
| `{project-id}-agent-definitions-v1` (Responses API) | Agent 버전 | 1000 |
| `{project-id}-run-state-v1` (Responses API) | 실행 상태 | 1000 |

**최소 처리량**: 3000 RU/s (기본 3컨테이너), 프로젝트 당 3000 RU/s 추가. Responses API 사용 시 +2000 RU/s.

**설정법**:
```java
// Bicep으로 사전 프로비저닝
resource cosmos 'Microsoft.DocumentDB/databaseAccounts@2024-08-15' = {
  name: 'cosmos-foundry-agent'
  properties: {
    databaseAccountOfferType: 'Standard'
    locations: [{ locationName: location }]
    publicNetworkAccess: 'Disabled'  // Private Endpoint only
    capabilities: [{ name: 'EnableServerless' }]  // 또는 provisioned throughput
  }
}
```

Foundry Project 생성 시 이 Cosmos DB 를 **customer-owned resource**로 연결. 이후 모든 agent 대화가 여러분의 Cosmos DB 에 저장된다.

### 3.2 Blob Storage — File Uploads

두 개의 컨테이너 필요:
- `-azureml-blobstore` (역할: **Storage Blob Data Contributor**) — 중간 데이터 (chunks, embeddings)
- `-agents-blobstore` (역할: **Storage Blob Data Owner**) — 사용자 업로드 파일

**⚠️ 제약**: Private Network 셋업에서 **Azure Blob Storage 기반 File Search 툴은 미지원**. OneDrive 나 Custom MCP 를 대안으로.

### 3.3 Azure AI Search — RAG Index

Ch.6의 RAG 파이프라인과 연결. Search 서비스도 **Private Endpoint + Semantic Ranker (Standard tier 이상)** 조합.

**Indexer executionEnvironment**: Private Endpoint 통과 시 반드시 `"Private"` 로 설정.

### 3.4 Foundry Prompts CMS — ⚠️ 결손

**현재 (2026-07 기준)**: Prompts CMS 데이터는 **Foundry-managed storage** 에 저장되고, customer-controlled로 이관 불가. Private Endpoint 지원 미문서화.

**대안**:
- 프롬프트를 **Git 리포에 canonical source** 로 두고 (Ch.11의 하이브리드 전략)
- CI 파이프라인이 Prompts CMS 로 sync
- 민감한 프롬프트는 CMS 대신 앱 코드에 두고 Key Vault 에서 로드

---

## 4. 온프렘 데이터 접근 패턴

### 4.1 ExpressRoute (권장 - 프로덕션)

전용선 기반 private 연결. **금융권 표준**.

- 대역폭: 50 Mbps ~ 100 Gbps
- Route filter로 Microsoft peering vs Private peering 분리
- Foundry delegated subnet 은 spoke VNet 을 통해 ExpressRoute 로 나감 (UDR)

### 4.2 Site-to-Site VPN (개발/스테이징)

VPN Gateway + on-prem VPN 기기 조합. IPSec 터널. ExpressRoute 보다 비용 낮지만 대역폭·지연·SLA가 열등.

### 4.3 Application Gateway 프록시 패턴 (Managed VNet)

**Managed VNet 을 쓰는 경우** 에만 필요. Managed VNet은 VNet Injection 처럼 delegated subnet 을 사용자가 못 다루므로 Application Gateway 를 사이에 끼운다.

```
Foundry Managed VNet
    ↓ Private Endpoint
Application Gateway (private frontend IP)
    ↓ VNet Peering + UDR
On-Prem Resources (via ExpressRoute/VPN)
```

**⚠️ 제한**: Application Gateway 는 **L7 (HTTP/HTTPS) 만 지원**. TCP/UDP 필요하면 Azure Firewall NVA로.

```bash
# 아웃바운드 rule로 Application Gateway private frontend IP 등록
az cognitiveservices account managed-network \
  outbound-rule set \
  --resource-group rg-foundry-prod \
  --name foundry-prod-01 \
  --rule allow-onprem-erp \
  --type privateendpoint \
  --destination '{
    "serviceResourceId": "/subscriptions/.../applicationGateways/appgw-onprem",
    "subresourceTarget": "appGwPrivateFrontendIpIPv4",
    "sparkEnabled": false
  }'
```

### 4.4 Logic Apps Hybrid Connections — ❌ 미지원

**공식적으로 "under development"** 상태 (2026-07). Foundry Agent 에서 Logic Apps 를 툴로 못 씀. 대체: Custom Function (Azure Functions) + Application Gateway.

---

## 5. Egress 통제 — 어디가 위험한가

각 툴이 **어느 IP로 나가는지** 안다면 데이터 유출 리스크를 정확히 통제할 수 있다.

| 툴 | 아웃바운드 origin | 데이터 leak 리스크 |
|---|---|---|
| **Custom Function (Azure Functions)** | 여러분의 VNet | ✅ 낮음 (VNet 방화벽 적용) |
| **OpenAPI Tool** | 여러분의 VNet | ✅ 낮음 |
| **Custom MCP Server (VNet 내)** | 여러분의 VNet | ✅ 낮음 |
| **Agent-to-Agent (A2A)** | 여러분의 VNet | ✅ 낮음 |
| **Azure AI Search grounding** | 여러분의 VNet (BYOS Search) | ✅ 낮음 |
| **File Search (Foundry-uploaded)** | Foundry managed | ⚠️ 중간 (BYOS Storage로 격리 가능) |
| **Web Search / Bing Grounding** | **Foundry public endpoint** | ❌ 높음 |
| **SharePoint Grounding** | Foundry public endpoint (M365 Retrieval API) | ⚠️ 중간 (M365 tenant 격리) |
| **Public MCP Server** | Public 인터넷 | ❌ 높음 |

⚠️ **결정적 함정 — Web Search**: `Bing Grounding` 툴은 여러분의 VNet 이 아닌 **Foundry의 공용 endpoint** 를 통해 검색어를 보낸다. 검색어에 사내 정보가 실리면 그건 사실상 **Microsoft 공용 인프라를 거쳐 Bing 으로 흘러간다**. 사내 문서 챗봇에는 **Web Search 툴을 비활성화**하거나, 필요 시 사용자 승인 후에만 허용하도록 정책 설계.

💡 **팁**: Ch.13에서 다룰 Foundry Toolbox 를 활용하여 툴별 activation 조건을 중앙 관리. 예: "회사 정보 카테고리로 분류된 프롬프트에서는 Web Search 자동 비활성."

---

## 6. 한국 금융권 · 공공 배포 체크리스트

법적/규제 준수는 고객 책임이지만, Foundry로 배포 시 담당자가 반드시 체크할 항목:

### 6.1 법 · 규제

- [ ] **개인정보보호법**: 개인정보 처리 위탁 계약 (Azure DPA 로 처리)
- [ ] **신용정보법 (신용정보 회사인 경우)**: 신용정보 처리 시 사전 승인 · 감사
- [ ] **전자금융감독규정** (금융권): 클라우드 이용 승인, 3rd party 감사권
- [ ] **금감원 클라우드 이용 가이드라인 (2024/2025 개정 반영)**: 정보 자산 등급별 클라우드 사용 범위
- [ ] **공공기관 클라우드 컴퓨팅 서비스 이용 가이드**: 국정원 CSAP 요구사항

### 6.2 계약 · 인증

- [ ] Azure DPA (Data Processing Addendum) 최신본 서명
- [ ] EA 또는 MCA 계약 (ZDR 신청 필수 조건)
- [ ] K-ISMS-P 리포트 획득 → 내부 IT 감사팀에 제출
- [ ] SOC 2 Type II 리포트 획득 → 외부 감사인에 제출
- [ ] BAA (HIPAA 적용 시)

### 6.3 기술 통제

- [ ] `publicNetworkAccess: Disabled` 확인
- [ ] Private Endpoint (Foundry account) 활성
- [ ] Private Endpoint (Cosmos DB, Blob, AI Search, Key Vault) 활성
- [ ] Private DNS Zone 3개 등록 및 VNet 연결
- [ ] Managed VNet + Allow-only-approved-outbound 활성
- [ ] Firewall FQDN allow-list 최소화 (엄격 필요 시 IP allow-list 만)
- [ ] BYOS: Cosmos DB · Blob · AI Search 모두 customer-owned + private
- [ ] Managed Identity 만으로 앱 인증 (API 키 하드코딩 금지)
- [ ] Customer Managed Keys (CMK) 활성 (Key Vault 통해)
- [ ] Content Safety + Prompt Shield 활성 (CCC 자격 유지)
- [ ] Purview DLP 정책 활성 (민감정보 프롬프트 차단)
- [ ] Application Insights + OTel 트레이싱 활성
- [ ] Foundry Control Plane 에서 fleet 컴플라이언스 대시보드 모니터링

### 6.4 운영 · 거버넌스

- [ ] ZDR 신청서 제출 완료 → 승인 코멘트 문서화
- [ ] Cosmos DB / Blob 데이터 삭제 SOP (사용자 요청 시)
- [ ] Fine-tuning 데이터 삭제 SOP
- [ ] audit log Retention 정책 (금감원 요구: 5년)
- [ ] 사용자 access review 분기별
- [ ] Model deployment 승인 워크플로 (Azure Policy `Cognitive Services Deployments should only use approved Registry Models`)
- [ ] 데이터 유출 대응 SOP (MS 는 자동 보상 안 함 — 사고 대응은 자체)

---

## 7. 🔧 실습: 폐쇄망 근접 Foundry 배포 (Bicep)

Ch.11의 IaC 예제를 확장하여 Private Endpoint + Managed VNet 조합으로 배포.

### 7.1 pom.xml 추가 (Ch.3 base 에)

```xml
<!-- pom.xml -->
<dependency>
  <groupId>com.azure</groupId>
  <artifactId>azure-identity</artifactId>
  <version>1.18.4</version>
</dependency>
<dependency>
  <groupId>com.azure</groupId>
  <artifactId>azure-security-keyvault-secrets</artifactId>
  <version>4.9.6</version>
</dependency>
```

### 7.2 폐쇄망 Bicep 스택 (핵심 조각만)

```bicep
// foundry-private.bicep
targetScope = 'resourceGroup'

param location string = 'koreacentral'
param vnetName string = 'vnet-foundry-prod'
param foundryName string = 'foundry-prod-01'

// 1. VNet with subnets
resource vnet 'Microsoft.Network/virtualNetworks@2024-05-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: ['10.0.0.0/16']
    }
    subnets: [
      {
        name: 'foundry-agent-subnet'  // delegated for agent runtime
        properties: {
          addressPrefix: '10.0.1.0/24'
          delegations: [
            {
              name: 'foundry-delegation'
              properties: {
                serviceName: 'Microsoft.App/environments'
              }
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

// 2. Foundry Resource with public access disabled
resource foundry 'Microsoft.CognitiveServices/accounts@2024-10-01' = {
  name: foundryName
  location: location
  kind: 'AIServices'
  sku: { name: 'S0' }
  identity: { type: 'SystemAssigned' }
  properties: {
    customSubDomainName: foundryName
    publicNetworkAccess: 'Disabled'   // ← 폐쇄망 근접 핵심
    disableLocalAuth: true             // ← API Key 사용 금지, Entra ID 강제
    networkAcls: {
      defaultAction: 'Deny'
      virtualNetworkRules: []
      ipRules: []
    }
  }
}

// 3. Private Endpoint (Foundry account)
resource foundryPe 'Microsoft.Network/privateEndpoints@2024-05-01' = {
  name: 'pe-${foundryName}'
  location: location
  properties: {
    subnet: {
      id: '${vnet.id}/subnets/private-endpoint-subnet'
    }
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
resource dnsZoneAoai 'Microsoft.Network/privateDnsZones@2024-06-01' = {
  name: 'privatelink.openai.azure.com'
  location: 'global'
}
resource dnsZoneCogSvc 'Microsoft.Network/privateDnsZones@2024-06-01' = {
  name: 'privatelink.cognitiveservices.azure.com'
  location: 'global'
}
resource dnsZoneAiSvc 'Microsoft.Network/privateDnsZones@2024-06-01' = {
  name: 'privatelink.services.ai.azure.com'
  location: 'global'
}

// 5. Cosmos DB (Standard Agent Setup - BYOS)
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

// (Private Endpoint for Cosmos DB, Search 는 별도 리소스로 반복 — 생략)

output foundryEndpoint string = foundry.properties.endpoint
```

배포:
```bash
# bash
az deployment group create \
  --resource-group rg-foundry-prod \
  --template-file foundry-private.bicep \
  --parameters location=koreacentral
```

### 7.3 Java 앱에서 Managed Identity 로 접근

폐쇄망 환경에서는 **API Key 사용 금지** (`disableLocalAuth: true`). Managed Identity + Entra ID 로만 접근.

```java
// PrivateFoundryClient.java
package com.example;

import com.azure.ai.projects.AIProjectClientBuilder;
import com.azure.ai.openai.OpenAIClient;
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.azure.core.credential.TokenCredential;

public class PrivateFoundryClient {

    public static void main(String[] args) {
        String endpoint = System.getenv("AZURE_FOUNDRY_ENDPOINT");
        // 예: https://foundry-prod-01.services.ai.azure.com
        String deployment = System.getenv("AZURE_FOUNDRY_DEPLOYMENT");

        if (endpoint == null || deployment == null) {
            System.err.println("환경변수 미설정.");
            return;
        }

        // Managed Identity 우선, 로컬 개발 시 az login fallback
        TokenCredential credential = new DefaultAzureCredentialBuilder()
            .build();

        try {
            OpenAIClient client = new AIProjectClientBuilder()
                .endpoint(endpoint)
                .credential(credential)   // API Key 아님
                .buildOpenAIClient();

            // 이제 여기서 client 호출 → Private Endpoint 경유 → Foundry
            System.out.println("Private Endpoint 로 인증 성공");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

Container Apps / VM 에서 실행 시 System-Assigned Managed Identity 가 자동 사용됨.

⚠️ **함정**: 로컬에서 테스트할 때는 `az login` 이후 `DefaultAzureCredential` 이 Azure CLI credential 을 사용한다. 하지만 **폐쇄망 실환경에서 로컬 접근은 애초에 안 될 것**. 개발은 개발 subscription 별도 유지, 프로덕션은 사내 Container Apps 에서만.

---

## 8. Foundry Control Plane 으로 fleet 감시

폐쇄망 배포 후 **여러 Foundry 리소스 · 프로젝트 · 배포를 통합 관리**하는 도구 (Ch.10 재소환).

- **Fleet Overview**: 전사 Foundry 리소스 한눈에
- **Compliance Dashboard**: Azure Policy 위반 · Guardrail 미활성 감지
- **Cost Anomaly**: 급증 알림
- **Defender for Cloud 통합**: 보안 이벤트
- **Purview 통합**: 데이터 이동 audit

Portal: `Foundry Portal → Management Center → Control Plane`

폐쇄망 환경에서도 Control Plane 콘솔은 관리자용 개별 세션으로 접근 (Managed Identity + 조건부 접근 정책 조합 권장).

---

## 9. 흔한 실수 정리

⚠️ **함정 총정리:**

1. **"ZDR 만 켜면 다 안전"**: agent state, RAG index, file upload 는 ZDR 미커버. 이들은 BYOS 로 격리.
2. **"CCC 로 유출 보호"**: CCC는 저작권 인덤니티. 유출 보상 아님.
3. **Foundry account 만 Private Endpoint**: Cosmos DB, Blob, AI Search 각각도 Private Endpoint 걸어야 함. 자동 생성 안 됨.
4. **Private DNS Zone 3개 다 등록 안 함**: `services.ai.azure.com` 을 놓치면 새 통합 endpoint 는 안 뜬다.
5. **Delegated subnet 크기 /28**: 프로덕션에서 IP 고갈. 최소 /24 권장.
6. **Web Search 활성 상태로 사내 챗봇**: 검색어가 Bing 으로 나감. 폐쇄망 정신 위배.
7. **`disableLocalAuth: false`**: API Key 로 접근 허용 → Managed Identity 원칙 위배.
8. **Prompts CMS 를 프로덕션 canonical 로 사용**: 여전히 Foundry-managed. 민감 프롬프트는 Git 관리.
9. **Global Standard 배포**: 데이터 처리가 여러 지역으로. 한국 내 데이터 거주성 필요하면 **Regional (Korea Central) 또는 Data Zone** 선택.
10. **Azure China 나 Azure Local 검토**: Foundry 미제공. 이 옵션은 처음부터 배제.

💡 **베스트 프랙티스:**

- **최소 특권 원칙**: Foundry Project 마다 별도 Managed Identity, 필요한 최소 역할만.
- **명시적 승인 필수**: Model deployment는 Azure Policy로 허용 리스트 강제 (Ch.8 재참조).
- **정기 감사**: 분기별 Foundry Control Plane 컴플라이언스 리포트, 사용자 access review.
- **Fine-tuning 데이터는 별도 Blob 컨테이너**: encryption 강화, 삭제 SOP 명확.
- **Purview DLP 정책 실전 활성**: PII, 결제 정보, 사내 코드 리크 방지.

---

## 요약 (Cheat Sheet)

- **"유출되면 보상" — 거짓**. Azure SLA는 uptime only, credit이 sole remedy. 유출 보상은 개별 MSA 협상 필요.
- **CCC** = AI 생성물 저작권 침해 인덤니티. 유출 보상 아님.
- **데이터 학습 미사용** = ✅ 계약 보증 (DPA + Product Terms).
- **ZDR** = ✅ 진짜지만 조건 (EA/MCA + 승인 + 커버리지 제한: fine-tune/agent state/RAG index 미커버).
- **폐쇄망 완전 air-gap 불가**. Private Endpoint + VNet Injection + Managed VNet + BYOS 조합이 실질 최대치.
- **Azure China / Azure Local 미제공**, **Azure Government 는 한국 조직 부적합**.
- **한국 금융권 체크리스트** = K-ISMS-P 확인 + DPA 서명 + BYOS 격리 + Managed Identity + Purview DLP + audit log 5년 유지.
- **Web Search 툴은 폐쇄망 정신 위배**. 사내 챗봇에는 비활성 default.

## 📚 더 읽기 (모두 계약·기술 원문)

- [Data privacy for Foundry Models (Data Privacy Doc)](https://learn.microsoft.com/en-us/azure/foundry/responsible-ai/openai/data-privacy)
- [Zero Data Retention 신청 안내](https://learn.microsoft.com/en-us/answers/questions/5834904/enable-zero-data-retention)
- [Customer Copyright Commitment 요건](https://learn.microsoft.com/en-us/azure/foundry/responsible-ai/openai/customer-copyright-commitment)
- [Configure network isolation for Foundry](https://learn.microsoft.com/en-us/azure/foundry/how-to/configure-private-link)
- [Foundry Agent Service Networking Deep Dive](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agents-networking-deep-dive)
- [Standard Agent Setup (BYOS)](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/standard-agent-setup)
- [Access on-premises resources from Foundry](https://learn.microsoft.com/en-us/azure/foundry/how-to/access-on-premises-resources)
- [Azure K-ISMS Offering](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-korea-k-isms)
- [Azure OpenAI Service SLA (uptime only)](https://www.azure.cn/en-us/support/sla/cognitive-services/)
- [Microsoft Products and Services DPA](https://www.microsoft.com/licensing/docs/view/Microsoft-Products-and-Services-Data-Protection-Addendum-DPA/)

## 다음 챕터

[Ch.13 사내 툴 통합 Enterprise Agent 실전 →](Ch13_Enterprise_Agent_Integration.md)

---
