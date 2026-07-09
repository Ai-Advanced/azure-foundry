# Chapter 10. 보안 · 거버넌스 · 비용 (Foundry Control Plane)

[← 목차로](README.md)

> **학습 목표**
> - 엔터프라이즈 보안 3층 모델 (Identity · Network · Content) 을 이해한다.
> - `azure-identity` 로 Managed Identity 인증을 구현한다.
> - `azure-ai-contentsafety` 로 Content Safety · Prompt Shield · Groundedness Detection 을 Python 으로 통합한다.
> - Azure Policy 로 모델 사용을 강제한다 (Ch.8 Policy 심화).
> - Foundry Control Plane 으로 fleet 컴플라이언스를 감시한다.
> - Cost Analysis · Budget Alert · 태깅으로 비용을 통제한다.

> **전제 조건**
> - [← Ch.3 API 연동 기초](Ch03_API_Basics.md), [← Ch.7 Agent Service](Ch07_Agent_Service.md), [← Ch.8 Model Router](Ch08_Model_Router.md) 완료
> - `uv add "azure-ai-contentsafety>=1.0.0" "azure-identity>=1.25.3" "azure-keyvault-secrets>=4.9"`

---

## 1. 엔터프라이즈 보안 3층 모델

| Layer | 관심사 | 도구 |
|---|---|---|
| **1. Identity / Access** | 누가 접근? | Entra ID · Managed Identity · RBAC · Agent Identity |
| **2. Network / Data** | 어디로 데이터 흐름? | Private Endpoint · VNet · CMK · Data Zone (Ch.12) |
| **3. Content / Behavior** | 모델이 뭘 말하나? | Content Safety · Prompt Shield · Purview DLP |

Ch.10 은 Layer 1 (Identity) + Layer 3 (Content) 를 다룬다. Layer 2 (Network) 는 Ch.12 에서 심화.

---

## 2. Layer 1: Identity — Managed Identity + `DefaultAzureCredential`

### 2.1 System-Assigned vs User-Assigned MI

- **System-Assigned**: Azure 리소스와 lifecycle 묶임. 리소스 삭제 시 자동 소멸. 단순 시나리오.
- **User-Assigned**: 독립 리소스. 여러 앱이 공유. 큰 조직 · CI/CD 파이프라인 표준.

### 2.2 `DefaultAzureCredential` 체인

Python 에서 인증 체인이 자동 fallback:

```
1. 환경변수 (AZURE_CLIENT_ID / TENANT_ID / CLIENT_SECRET)
2. Workload Identity (Container Apps / AKS)
3. Managed Identity (VM / Container Apps)
4. Azure CLI (az login)
5. Azure PowerShell
6. Azure Developer CLI (azd)
7. VS Code (Azure Account extension)
```

즉, 로컬 개발 = `az login` fallback, Container Apps 배포 = MI 자동 사용. **코드 변경 zero**.

```python
# src/foundry_app/auth.py
"""Managed Identity + DefaultAzureCredential."""
import os
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import AzureOpenAI

# 프로덕션: Managed Identity 우선, 로컬: az login fallback
credential = DefaultAzureCredential()

# Cognitive Services token scope (Foundry 는 CS 위에 구축됨)
token_provider = get_bearer_token_provider(
    credential,
    "https://cognitiveservices.azure.com/.default",
)

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    azure_ad_token_provider=token_provider,   # ← API Key 대신
    api_version="2025-01-01-preview",
)
```

⚠️ **함정**:
- Managed Identity 부여만 해서는 안됨. **RBAC 역할 명시 할당** 필수 (예: `Cognitive Services OpenAI User`).
- Token scope 은 반드시 `.default` suffix 로. `.default` 없으면 admin consent 오류.
- 로컬에서 `DefaultAzureCredential` 실패 시 `az login` 확인. 여러 계정 있으면 `az account set --subscription <id>`.

### 2.3 Agent Identity (2026 신규)

Foundry Agent Service 는 각 agent 에 **dedicated Entra ID** 부여. Agent 가 tool 호출 시 자신의 identity 로 인증 (Ch.13 OBO 와 함께).

```python
# Agent 생성 시 identity 자동 부여
agent = agents.create_agent(
    model="gpt-5",
    name="finance-agent",
    instructions="...",
    # identity 는 자동 - agent.identity_id 로 확인
)
print(f"Agent Entra ID: {agent.identity_id}")

# 이 identity 에 RBAC 역할 부여
# az role assignment create --assignee $AGENT_IDENTITY_ID \
#   --role "Storage Blob Data Reader" --scope $STORAGE_SCOPE
```

### 2.4 RBAC 역할 세분화

| 역할 | 대상 | 권한 |
|---|---|---|
| Foundry Account Owner | 관리자 | 리소스 + 과금 |
| Foundry Owner | 시니어 | 프로젝트 관리 |
| Foundry Project Manager | 팀 리더 | 모델 배포 · 데이터 · 평가 |
| Foundry User | 개발자 | Playground · API 호출 |
| Cognitive Services OpenAI User | 앱 (MI) | 모델 추론만 |

**Least Privilege 원칙**: 각 앱·팀에 **최소 필요 역할**만.

---

## 3. Layer 3: Content Safety (Python 통합)

### 3.1 `azure-ai-contentsafety` SDK

```python
# src/foundry_app/content_safety.py
"""Content Safety - 프롬프트/응답 harm 검사."""
from __future__ import annotations
import os
from azure.ai.contentsafety import ContentSafetyClient
from azure.ai.contentsafety.models import (
    AnalyzeTextOptions, TextCategory, AnalyzeTextOutputType,
)
from azure.identity import DefaultAzureCredential

cs_client = ContentSafetyClient(
    endpoint=os.environ["CONTENT_SAFETY_ENDPOINT"],
    credential=DefaultAzureCredential(),
)


def check_content(text: str, block_severity: int = 4) -> tuple[bool, dict]:
    """텍스트를 4대 harm category 로 검사.

    Args:
        text: 검사 대상
        block_severity: 이 값 이상이면 차단 (0-6 척도)

    Returns:
        (safe, details) 튜플. safe=False 면 details 로 이유 확인.
    """
    result = cs_client.analyze_text(AnalyzeTextOptions(
        text=text,
        categories=[
            TextCategory.HATE,
            TextCategory.SELF_HARM,
            TextCategory.SEXUAL,
            TextCategory.VIOLENCE,
        ],
        output_type=AnalyzeTextOutputType.EIGHT_SEVERITY_LEVELS,
    ))

    details = {}
    is_safe = True
    for cat_analysis in result.categories_analysis:
        severity = cat_analysis.severity
        details[cat_analysis.category] = severity
        if severity >= block_severity:
            is_safe = False

    return is_safe, details


# 사용
if __name__ == "__main__":
    safe, detail = check_content("일반적인 인사말입니다.")
    print(f"safe={safe}, detail={detail}")
```

### 3.2 Prompt Shield (Jailbreak Detection)

```python
from azure.ai.contentsafety.models import ShieldPromptRequest

def check_jailbreak(user_prompt: str, retrieved_docs: list[str] | None = None) -> bool:
    """User attack + XPIA (cross-prompt injection) 감지.

    Args:
        user_prompt: 사용자 입력
        retrieved_docs: RAG 로 가져온 문서들 (XPIA 검사 대상)

    Returns:
        True = 안전, False = 공격 감지
    """
    result = cs_client.shield_prompt(ShieldPromptRequest(
        user_prompt=user_prompt,
        documents=retrieved_docs or [],
    ))

    user_attack = result.user_prompt_analysis.attack_detected
    doc_attacks = any(d.attack_detected for d in result.documents_analysis)

    if user_attack:
        print("⚠️ User Jailbreak 시도 감지")
    if doc_attacks:
        print("⚠️ XPIA (문서 안에 프롬프트 인젝션) 감지")

    return not (user_attack or doc_attacks)
```

### 3.3 Groundedness Detection (RAG 답변 근거 검증)

Foundry Content Safety 의 Groundedness Detection API 는 RAG 답변이 제공된 context 를 정말 근거로 삼는지 자동 검증.

```python
from azure.ai.contentsafety.models import (
    AnalyzeTextGroundednessOptions, GroundingSources,
)

def check_groundedness(question: str, answer: str, context: str) -> dict:
    """RAG 답변이 context 근거하는지 검증. (Ch.9 Evaluation 과 별개, 실시간용)"""
    result = cs_client.detect_groundedness(AnalyzeTextGroundednessOptions(
        task="QnA",
        text=answer,
        grounding_sources=[GroundingSources(text=context)],
        qna={"query": question},
    ))
    return {
        "ungrounded": result.ungrounded_detected,
        "ungrounded_percentage": result.ungrounded_percentage,
        "ungrounded_details": result.ungrounded_details,
    }
```

### 3.4 서비스 wrapper — 프로덕션 pattern

Content Safety 를 매 요청에 적용하는 wrapper.

```python
# src/foundry_app/safe_chat.py
"""Content Safety 로 감쌈 chat 호출 pattern."""
from openai import AzureOpenAI
from foundry_app.content_safety import check_content, check_jailbreak


class SafeChatClient:
    def __init__(self, openai_client: AzureOpenAI, deployment: str) -> None:
        self._client = openai_client
        self._deployment = deployment

    def chat(self, user_message: str, system: str = "") -> str:
        # 1) 입력 검사
        safe_in, _ = check_content(user_message)
        if not safe_in:
            return "요청에 부적절한 콘텐츠가 포함되어 있어 처리할 수 없습니다."

        # 2) Jailbreak 검사
        if not check_jailbreak(user_message):
            return "부적절한 프롬프트 조작 시도가 감지되었습니다."

        # 3) 모델 호출
        response = self._client.chat.completions.create(
            model=self._deployment,
            messages=[
                {"role": "system", "content": system} if system else None,
                {"role": "user", "content": user_message},
            ][-2:],   # None 필터
            max_completion_tokens=1000,
            temperature=1.0,
        )
        reply = response.choices[0].message.content or ""

        # 4) 응답 검사
        safe_out, _ = check_content(reply)
        if not safe_out:
            return "응답이 정책에 부합하지 않아 표시할 수 없습니다."

        return reply
```

⚠️ **함정**:
- Content Safety threshold 너무 낮음 → false positive 폭증 (정상 대화도 차단)
- Prompt Shield 는 **별도 SKU 과금**. Free tier 는 rate limit 낮음
- Groundedness Detection 은 preview → 프로덕션 시 SLA 없음. Ch.9 evaluation 병행 권장

---

## 4. 거버넌스: Azure Policy for Foundry

Ch.8 Model Router Policy 를 확장.

### 4.1 4가지 실무 정책

1. **Model 승인 리스트** (Ch.8): 사용 가능 모델 제한
2. **리전 제한**: 특정 지역에만 배포 허용 (예: Korea Central 만)
3. **SKU 제한**: Global 계열 금지, Regional 만
4. **Managed Identity 강제**: `disableLocalAuth: true` 필수

### 4.2 Enforcement vs Audit

- **Audit mode**: 위반 감지만, 배포 허용. 정책 도입 초기 파악용
- **Enforce mode**: 위반 시 배포 차단. 프로덕션 표준

먼저 Audit 로 몇 주 돌려 위반 패턴 파악 → Enforce 전환.

### 4.3 Python 으로 정책 준수 확인

```python
# src/foundry_app/compliance_check.py
"""배포 전 정책 준수 사전 확인."""
from __future__ import annotations
import os
from azure.identity import DefaultAzureCredential
from azure.mgmt.policyinsights import PolicyInsightsClient


def check_deployment_compliance(resource_id: str) -> list[dict]:
    """특정 리소스의 정책 위반 사항 조회."""
    client = PolicyInsightsClient(
        credential=DefaultAzureCredential(),
        subscription_id=os.environ["AZURE_SUBSCRIPTION_ID"],
    )
    states = client.policy_states.list_query_results_for_resource(
        policy_states_resource="latest",
        resource_id=resource_id,
    )
    violations = [
        {
            "policy": s.policy_definition_name,
            "compliance_state": s.compliance_state,
            "reason": s.compliance_reason_code,
        }
        for s in states if s.compliance_state == "NonCompliant"
    ]
    return violations
```

---

## 5. Foundry Control Plane (GA, 통합 관리)

Multiple Foundry Resource · Project · Agent 를 **하나의 관리 화면**에서.

**주요 기능:**
- **Fleet Overview**: 전사 리소스 한눈에
- **Compliance Dashboard**: Azure Policy 위반 · Guardrail 미활성 감지
- **Cost Anomaly Detection**: 급증 알림
- **Microsoft Defender for Cloud 연동**: 보안 이벤트
- **Microsoft Purview 연동**: 데이터 이동 audit + DLP

**접근**: `Foundry Portal → Management Center → Control Plane`.

Python API 는 preview — 대부분 Portal / Bicep 로 관리. 프로그래머틱 접근 필요 시 REST 로.

---

## 6. Purview DLP 통합 (2026-06 GA)

Data Loss Prevention 을 프롬프트 inline enforcement.

**설정 흐름:**
1. Purview 콘솔에서 DLP 정책 생성 (예: "credit card pattern 감지 → block")
2. Foundry Project 에 Purview integration 활성
3. 이후 모든 프롬프트가 자동 스캔 → 위반 시 차단 + audit log

**예시 정책:**

| 정책 | 조건 | 액션 |
|---|---|---|
| 결제 정보 반출 방지 | 프롬프트에 신용카드 번호 | Block + user notification |
| 소스코드 유출 방지 | 응답에 `@Company` annotation, `com.corp.*` package | Alert admin + audit |
| PII 반출 방지 | 프롬프트에 주민번호/전화/이메일 | Redact + audit |

⚠️ **함정**: DLP 는 **프롬프트 (input)** 만 강제 enforce. **응답 (output)** 은 alert 만 발생하고 자동 차단 안 됨. 응답 검증은 Content Safety + custom filter (섹션 3.4) 로 병행.

---

## 7. Customer Managed Keys (CMK)

기본 MS-managed encryption 은 대부분 상황에 충분하나, 금융/의료/공공 은 **CMK 필수**.

- Foundry Resource → Encryption → CMK
- Azure Key Vault 에 encryption key 저장
- 정기 key rotation policy 설정

Python 접근:

```python
from azure.keyvault.keys import KeyClient
from azure.identity import DefaultAzureCredential

kv_client = KeyClient(
    vault_url=os.environ["KEY_VAULT_URL"],
    credential=DefaultAzureCredential(),
)
key = kv_client.get_key("foundry-cmk-v1")
```

⚠️ **함정**: CMK 사용 중 key 삭제 = 데이터 접근 불가. Key Vault 의 **soft delete + purge protection 필수 활성**.

---

## 8. 비용 관리

### 8.1 3층 접근

1. **Cost Analysis** (Portal): 리소스별·태그별 소비 조회
2. **Budget Alert**: 월 예산 % 도달 시 Action Group 트리거
3. **Cost Anomaly Detection**: ML 기반 이상 사용 자동 감지

### 8.2 태깅 표준

모든 리소스에 최소 4개 태그:

| 태그 | 예시 |
|---|---|
| `env` | `prod` / `staging` / `dev` |
| `project` | `customer-chatbot` |
| `owner` | `team-ai-platform` |
| `cost-center` | `CC-1024` |

Bicep 에서 자동 상속:

```bicep
resource foundry 'Microsoft.CognitiveServices/accounts@2024-10-01' = {
  name: 'foundry-prod-01'
  location: location
  tags: {
    env: 'prod'
    project: 'customer-chatbot'
    owner: 'team-ai-platform'
    'cost-center': 'CC-1024'
  }
  // ...
}
```

### 8.3 Python 으로 비용 조회

```python
# src/foundry_app/cost_query.py
"""이번 달 Foundry 소비 조회."""
from __future__ import annotations
import os
from datetime import datetime, timedelta
from azure.identity import DefaultAzureCredential
from azure.mgmt.costmanagement import CostManagementClient
from azure.mgmt.costmanagement.models import (
    QueryDefinition, QueryDataset, QueryAggregation,
    QueryTimePeriod, TimeframeType, QueryGrouping,
)

client = CostManagementClient(credential=DefaultAzureCredential())

scope = f"/subscriptions/{os.environ['AZURE_SUBSCRIPTION_ID']}"

query = QueryDefinition(
    type="ActualCost",
    timeframe=TimeframeType.MONTH_TO_DATE,
    dataset=QueryDataset(
        granularity="Daily",
        aggregation={"totalCost": QueryAggregation(name="Cost", function="Sum")},
        grouping=[
            QueryGrouping(type="Dimension", name="ResourceType"),
            QueryGrouping(type="Tag", name="project"),
        ],
    ),
)
result = client.query.usage(scope=scope, parameters=query)
for row in result.rows:
    print(row)   # [date, cost, resource_type, project, currency]
```

### 8.4 비용 절감 실전 팁

- **Batch API (Ch.8)**: 실시간성 없는 작업 → 50% 할인
- **`gpt-5-mini`, `gpt-5-nano` 사용**: 간단한 태스크는 저렴 모델로 90%+ 절감
- **Model Router `auto`**: 개발/테스트 트래픽만 (프로덕션은 명시)
- **PTU**: 100K TPM 지속 시 종량제 대비 20%+ 절감 (Ch.8)
- **RAG chunk 크기 축소**: 8192 → 4096 시 검색 정확도 미미 감소, 비용 대폭 감소
- **불필요 배포 삭제**: Developer tier · 사용 안 하는 legacy deployment

---

## 9. 감사(Audit) & 규정 준수

- **Activity Log**: 관리 작업 (배포/삭제/역할 부여) 자동 기록
- **Diagnostic Settings**: Foundry API 호출 로그를 Log Analytics workspace 로 export
- **Log Retention**: 금감원 요구 **5년** — App Insights 기본 90일이므로 Storage 로 archival 필수

```python
# 감사 로그 쿼리 예시 (KQL)
"""
AzureActivity
| where ResourceProvider == "Microsoft.CognitiveServices"
| where OperationNameValue endswith "deployments/write"
| project TimeGenerated, Caller, ResourceGroup, Resource, ActivityStatus
| order by TimeGenerated desc
"""
```

**컴플라이언스 인증** (Ch.12 상세):
- K-ISMS-P (금융권 필수)
- SOC 2 Type II · ISO 27001
- HIPAA (의료), PCI-DSS (결제)
- EU Data Boundary

---

## 10. 흔한 함정 정리

⚠️
1. **MI 부여만 하고 RBAC 역할 안 붙임** → 401
2. **Content Safety threshold 너무 낮음** → false positive 폭증
3. **Prompt Shield 별도 SKU 미인지** → 무료 시 rate limit
4. **CMK key 삭제** → 데이터 접근 불가 (soft delete + purge protection)
5. **DLP 로 응답까지 자동 차단 기대** → 응답은 alert 만
6. **Global 배포 → 데이터 국경 초과** → Ch.12 Regional 선택
7. **태그 빈 리소스** → Cost Analysis 안 됨. Bicep에서 강제

💡 **베스트 프랙티스**:
- **Least Privilege**: 각 앱 최소 역할
- **Managed Identity default**: API Key 는 로컬 개발만
- **정책은 Audit → Enforce** 순서로 도입
- **분기별 access review** + policy compliance report
- **비용 태그 4개 필수** — Bicep 강제
- **Budget alert 80% + 100% + 120%** 3단계

---

## 요약 (Cheat Sheet)

- **Identity**: `DefaultAzureCredential` + Managed Identity, 프로덕션 표준
- **Content Safety**: `azure-ai-contentsafety` — 4대 harm + Prompt Shield + Groundedness Detection
- **거버넌스**: Azure Policy (모델·리전·SKU·MI 강제)
- **Control Plane**: fleet 관리 · 컴플라이언스 · cost anomaly
- **Purview DLP**: 프롬프트 inline 검사, 응답은 alert 만
- **CMK**: 금융/의료/공공 필수, Key Vault soft delete 필수
- **비용**: 태그 표준 + Budget + Cost Anomaly + Batch/PTU 최적화

## 📚 더 읽기

- [`azure-identity` Python SDK](https://learn.microsoft.com/en-us/python/api/overview/azure/identity-readme)
- [`azure-ai-contentsafety` Python](https://learn.microsoft.com/en-us/python/api/overview/azure/ai-contentsafety-readme)
- [Prompt Shields](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/concepts/jailbreak-detection)
- [Groundedness Detection](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/concepts/groundedness)
- [Foundry Control Plane](https://learn.microsoft.com/en-us/azure/foundry/control-plane/overview)
- [Purview DLP for AI](https://learn.microsoft.com/en-us/purview/ai-agents)
- [Model Router Policy](https://learn.microsoft.com/en-us/azure/foundry/how-to/model-router-policy)
- [Azure Cost Management](https://learn.microsoft.com/en-us/azure/cost-management-billing/)

## 다음 챕터

[Ch.11 Production CI/CD & LLMOps →](Ch11_LLMOps.md)

---
