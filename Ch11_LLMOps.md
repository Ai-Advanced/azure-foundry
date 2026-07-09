# Chapter 11. Production CI/CD & LLMOps (Python)

[← 목차로](README.md)

> **학습 목표**
> - LLMOps 가 MLOps 와 어떻게 다른지, 무엇을 자동화하는지 이해한다.
> - Bicep 과 Terraform 으로 Foundry 리소스 스택을 IaC 로 관리한다.
> - Prompt / Agent Version 관리 전략 (Git canonical + Foundry CMS sync) 을 세운다.
> - Canary / Blue-Green / Shadow 배포 전략을 Python 앱에 적용한다.
> - Evaluation-gated deployment 를 GitHub Actions 에 통합한다.
> - `uv` + Docker 로 재현 가능한 Python 이미지를 만든다.
> - App Insights alert 로 auto-rollback 을 트리거한다.

> **전제 조건**
> - [← Ch.8 Model Router](Ch08_Model_Router.md), [← Ch.9 Evaluation](Ch09_Evaluation.md), [← Ch.10 Security](Ch10_Security_Governance.md) 완료
> - GitHub Organization 계정 또는 Azure DevOps
> - Azure CLI + Bicep CLI (`az bicep install`)
> - Terraform 1.9+ (선택)
> - Docker

---

## 1. LLMOps 란 무엇인가

MLOps 에서 M(Machine Learning)이 빠지고 LLM 이 들어간다 = 이름만 바뀐 게 아니다. **훈련(Training)이 대상에서 사라지고, Prompt · RAG · Agent 구성이 대상**이 된다.

| 축 | MLOps | LLMOps |
|---|---|---|
| **주요 artifact** | 학습된 모델 weight | Prompt · Agent Definition · RAG Index |
| **CI 시간** | 모델 학습 (수시간~일) | 프롬프트 렌더 (초) |
| **품질 검증** | test set accuracy | LLM-as-judge evaluation (Ch.9) |
| **배포 단위** | 모델 파일 | Prompt version · Agent version · RAG snapshot |
| **롤백** | 이전 모델 로드 | 이전 prompt / agent version 활성 |

**LLMOps 4대 축:**
1. **IaC**: Foundry Resource · Project · Deployment · AI Search · Content Safety 를 코드로 선언
2. **Prompt / Agent 버전 관리**: git tag + Foundry-side version 병행
3. **Evaluation-gated deployment**: Ch.9 평가를 CI gate 로 통합
4. **Observability + Auto-rollback**: Ch.9 OTel trace 로 실시간 감시, 이상 시 자동 복귀

---

## 2. 🔧 실습 1 — Bicep 으로 Foundry 스택 IaC

Bicep 은 Azure ARM 의 DSL. Python 언어와 무관하나 배포 표준이므로 그대로 사용.

```bicep
// infra/foundry-stack.bicep
targetScope = 'resourceGroup'

@description('Foundry 리소스 이름')
param foundryName string = 'foundry-${uniqueString(resourceGroup().id)}'

@description('배포 지역')
param location string = 'eastus'

@description('GPT-5 배포 이름')
param gpt5DeploymentName string = 'gpt-5'

// 1. Foundry Resource (AIServices kind)
resource foundry 'Microsoft.CognitiveServices/accounts@2024-10-01' = {
  name: foundryName
  location: location
  kind: 'AIServices'
  sku: {
    name: 'S0'
  }
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    customSubDomainName: foundryName
    publicNetworkAccess: 'Enabled'   // 프로덕션은 'Disabled' + Private Endpoint (Ch.12)
    disableLocalAuth: true            // 프로덕션 표준: API Key 금지
  }
  tags: {
    env: 'prod'
    project: 'foundry-course'
    owner: 'team-ai-platform'
    'cost-center': 'CC-1024'
  }
}

// 2. GPT-5 Deployment
resource gpt5Deployment 'Microsoft.CognitiveServices/accounts/deployments@2024-10-01' = {
  parent: foundry
  name: gpt5DeploymentName
  sku: {
    name: 'GlobalStandard'
    capacity: 100   // TPM 단위 (100K TPM)
  }
  properties: {
    model: {
      format: 'OpenAI'
      name: 'gpt-5'
      version: '2025-08-preview'
    }
  }
}

// 3. Azure AI Search (RAG - Ch.6)
resource search 'Microsoft.Search/searchServices@2024-06-01-preview' = {
  name: 'search-${foundryName}'
  location: location
  sku: {
    name: 'standard'
  }
  properties: {
    replicaCount: 1
    partitionCount: 1
    semanticSearch: 'standard'
  }
}

// 4. Application Insights (Ch.9)
resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: 'ai-${foundryName}'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    IngestionMode: 'ApplicationInsights'
  }
}

output foundryEndpoint string = foundry.properties.endpoint
output gpt5DeploymentName string = gpt5Deployment.name
output searchEndpoint string = 'https://${search.name}.search.windows.net'
output appInsightsConnStr string = appInsights.properties.ConnectionString
```

배포:

```bash
# bash
az group create --name rg-foundry-prod --location eastus

# What-if 로 미리 확인
az deployment group what-if \
  --resource-group rg-foundry-prod \
  --template-file infra/foundry-stack.bicep

# 실제 배포
az deployment group create \
  --resource-group rg-foundry-prod \
  --template-file infra/foundry-stack.bicep \
  --parameters foundryName=foundry-prod-01
```

⚠️ **함정**: `Microsoft.CognitiveServices/.../deployments` 는 **quota 없으면 조용히 실패**. Bicep validation 은 통과, 실제 배포에서 `InsufficientQuota` 에러. 배포 전 `az cognitiveservices usage list` 로 확인.

---

## 3. 🔧 실습 2 — Terraform 대안

멀티클라우드 팀이거나 Terraform 자산이 이미 많다면.

```hcl
# infra/main.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.10"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "foundry" {
  name     = "rg-foundry-tf"
  location = "eastus"
}

resource "azurerm_cognitive_account" "foundry" {
  name                = "foundry-tf-01"
  location            = azurerm_resource_group.foundry.location
  resource_group_name = azurerm_resource_group.foundry.name
  kind                = "AIServices"
  sku_name            = "S0"

  custom_subdomain_name = "foundry-tf-01"

  identity {
    type = "SystemAssigned"
  }
}

resource "azurerm_cognitive_deployment" "gpt5" {
  name                 = "gpt-5"
  cognitive_account_id = azurerm_cognitive_account.foundry.id

  model {
    format  = "OpenAI"
    name    = "gpt-5"
    version = "2025-08-preview"
  }

  sku {
    name     = "GlobalStandard"
    capacity = 100
  }
}
```

| 선택 기준 | 권장 |
|---|---|
| Azure-only, MS 생태계 집중 | **Bicep** (Azure 신기능 최우선 반영) |
| Multi-cloud (Azure + AWS + GCP) | **Terraform** (하나의 HCL) |
| 팀 Terraform state 관리 성숙 | Terraform 유지 |
| DevOps 인력 얕음 | Bicep (learning curve 완만) |

💡 **팁**: Foundry 신기능(Model Router, Agent 배포) 은 Bicep 에 먼저 반영. Terraform azurerm provider 는 몇 주-달 lag.

---

## 4. Prompt & Agent 버전 관리

프롬프트 하드코딩 → 한 줄 수정에 재배포. **하이브리드 전략** 이 실무 정답.

| 저장 위치 | 장점 | 단점 |
|---|---|---|
| **Git (코드와 함께)** | 원자적 커밋, PR 리뷰 | 프롬프트 수정 시 재배포 |
| **Foundry Prompts CMS** | 재배포 없이 갱신, PM 직접 수정 | git 과 동기화 필요 |

**하이브리드**: Git 이 canonical, Foundry Prompts CMS 로 sync (CI 자동).

### 4.1 프롬프트 파일 구조 (git 관리)

```
prompts/
├── customer-support/
│   ├── system.md              # 시스템 프롬프트
│   ├── VERSION                # 2.1.0
│   └── metadata.yaml          # 배포 대상 · 활성 여부
└── rag-agent/
    ├── system.md
    ├── VERSION
    └── metadata.yaml
```

### 4.2 Semantic Versioning

- **MAJOR**: 프롬프트 의도 변경 (답변 언어 한→영). Evaluation 재실행 필수.
- **MINOR**: 새 example 추가, tone 조정. 후방 호환.
- **PATCH**: 오타 수정, 공백 정리. 회귀 위험 낮음.

Git tag 예: `prompt/customer-support/v2.1.0`

### 4.3 Agent Versions (Ch.7 재소환)

Foundry Agent Service 는 Agent Version 을 내장 지원. Ch.7 실습 예제 활용.

```python
# 새 버전 배포 (기존 v1 유지)
v2_agent = agents.create_agent_version(
    agent_id=existing_agent.id,
    instructions=open("prompts/customer-support/system.md").read(),
    tools=existing_agent.tools,
    model="gpt-5",
)

# Canary: 트래픽 10% v2
agents.update_agent(
    agent_id=existing_agent.id,
    traffic_distribution={"v1.2": 90, "v1.3": 10},
)
```

---

## 5. Deployment 전략

### 5.1 Canary (점진 확장)

새 버전에 소량 트래픽 (1-10%) → 지표 안정 시 확대.

```python
# 자동 확대 룰 예시
def should_expand_canary(metrics: dict) -> bool:
    """Ch.9 App Insights 지표 기반 확대 결정."""
    return (
        metrics["error_rate_5min"] < 0.005          # 0.5% 미만
        and metrics["groundedness_score"] > 0.85    # baseline 대비
        and metrics["p95_latency_ms"] < 3000        # p95 3초
    )
```

⚠️ **함정**: **User session sticky** 안 지키면 사용자가 신구 버전 왔다갔다. Conversation metadata 에 `assigned_version` 심고 다음 turn 은 같은 version 라우팅.

### 5.2 Blue-Green (원자적 스위치)

두 배포 slot 유지 (blue = 현재, green = 신규). 검증 완료 후 트래픽 100% green. 문제 시 blue 로 즉시 복귀.

Foundry 에서는 **Deployment name 두 개** 만들고 (예: `gpt-5-blue`, `gpt-5-green`) 앱 환경변수만 스위치.

### 5.3 Shadow (실시간 비교)

같은 요청을 신구 모델에 동시 → 사용자에게 구 모델 응답만. 신 모델 응답은 로깅 → 나중에 batch evaluation.

**프로덕션 트래픽으로 회귀 테스트 셋 자동 수집** 하는 강력한 pattern. 비용 2배지만 릴리스 리스크 극감. 결제 · 법률 답변 같은 고위험 도메인 필수.

```python
# src/foundry_app/shadow_deploy.py
"""Shadow 배포 - 신구 모델 동시 호출, 사용자에겐 stable 만."""
import asyncio
from openai import AsyncAzureOpenAI

stable_client = AsyncAzureOpenAI(deployment="gpt-5-stable", ...)
canary_client = AsyncAzureOpenAI(deployment="gpt-5-canary", ...)


async def shadow_chat(messages: list) -> str:
    """사용자에겐 stable 응답, canary 응답은 background 로깅."""
    stable_task = asyncio.create_task(stable_client.chat.completions.create(...))
    canary_task = asyncio.create_task(canary_client.chat.completions.create(...))

    stable_response = await stable_task

    # Canary 는 background - 실패해도 사용자엔 영향 X
    canary_task.add_done_callback(_log_canary_result)

    return stable_response.choices[0].message.content


def _log_canary_result(task: asyncio.Task) -> None:
    """Canary 응답을 shadow log 로 저장 - 나중에 evaluation."""
    if task.exception():
        # log 실패
        return
    canary_response = task.result()
    # save to shadow log storage (Blob, App Insights, etc.)
```

---

## 6. 🔧 실습 3 — Python Dockerfile (uv 기반)

`uv` 로 초고속 재현 가능한 이미지.

```dockerfile
# Dockerfile
# Multi-stage: builder → runtime (이미지 크기 최소화)

FROM python:3.13-slim AS builder

# uv 설치 (10-100x faster than pip)
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app

# 의존성 lock 파일만 먼저 (Docker layer cache)
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project

# 앱 코드 복사
COPY src/ ./src/
RUN uv sync --frozen --no-dev

# ────────── Runtime stage ──────────
FROM python:3.13-slim

WORKDIR /app

# builder 에서 venv 만 가져옴
COPY --from=builder /app/.venv /app/.venv
COPY --from=builder /app/src /app/src

ENV PATH="/app/.venv/bin:$PATH"
ENV PYTHONUNBUFFERED=1

# App Insights 자동 계측 (Ch.9)
ENV OTEL_PYTHON_LOG_CORRELATION=true

EXPOSE 8000

# FastAPI 앱 실행 예
CMD ["python", "-m", "uvicorn", "foundry_app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**이미지 크기**: ~150MB (Python 3.13-slim + uv + venv). Alpine 은 glibc 이슈로 azure SDK 호환성 낮으므로 slim 권장.

### 6.1 .dockerignore

```
.venv/
__pycache__/
*.pyc
.git/
.env
.pytest_cache/
```

### 6.2 로컬 빌드 & 실행

```bash
docker build -t foundry-app:local .
docker run --rm -p 8000:8000 --env-file .env foundry-app:local
```

---

## 7. 🔧 실습 4 — GitHub Actions 파이프라인 (전문)

Evaluation gate 통합 완전 워크플로.

```yaml
# .github/workflows/deploy.yml
name: Deploy Foundry Python App

on:
  push:
    branches: [main]
  pull_request:

env:
  AZURE_RG: rg-foundry-prod
  ACR_NAME: myfoundryacr
  APP_NAME: foundry-agent-app
  PYTHON_VERSION: "3.13"

permissions:
  id-token: write   # OIDC federated identity
  contents: read

jobs:
  # ────────── 1. Build & Test ──────────
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true

      - name: Set up Python
        run: uv python install ${{ env.PYTHON_VERSION }}

      - name: Install deps (frozen)
        run: uv sync --frozen

      - name: Lint (ruff)
        run: uv run ruff check src/

      - name: Type check (basedpyright)
        run: uv run basedpyright src/

      - name: Unit tests
        run: uv run pytest -v --cov=src

  # ────────── 2. Evaluation Gate ──────────
  evaluate:
    needs: build-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: astral-sh/setup-uv@v3

      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}      # OIDC, no secret
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Install eval deps
        run: uv sync --frozen --extra eval

      - name: Run Evaluation Gate (Ch.9)
        env:
          AZURE_FOUNDRY_ENDPOINT: ${{ vars.FOUNDRY_ENDPOINT }}
          AZURE_FOUNDRY_KEY: ${{ secrets.FOUNDRY_KEY }}
          AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          AZURE_RESOURCE_GROUP: ${{ env.AZURE_RG }}
          FOUNDRY_PROJECT_NAME: prod-foundry-eastus
        run: uv run python -m foundry_app.eval_gate --data test/eval_dataset.jsonl
        # exit 1 시 gate fail → 이후 job skip

  # ────────── 3. Deploy (Bicep + App) ──────────
  deploy:
    needs: evaluate
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      # 3a. Foundry 리소스 IaC (실습 1)
      - name: Deploy Foundry stack (Bicep)
        run: |
          az deployment group create \
            --resource-group $AZURE_RG \
            --template-file infra/foundry-stack.bicep \
            --parameters foundryName=foundry-prod-01

      # 3b. Prompt sync (git canonical → Foundry Prompts CMS)
      - name: Sync prompts to Foundry
        run: |
          for prompt in prompts/*/; do
            name=$(basename $prompt)
            version=$(cat $prompt/VERSION)
            content=$(cat $prompt/system.md | jq -Rs .)
            az rest --method PUT \
              --url "${{ vars.FOUNDRY_ENDPOINT }}/openai/v1/prompts/$name/versions/$version" \
              --body "{\"content\": $content}"
          done

      # 3c. Docker build & push
      - name: Docker build & push
        run: |
          az acr login --name $ACR_NAME
          docker build -t $ACR_NAME.azurecr.io/$APP_NAME:${{ github.sha }} .
          docker push $ACR_NAME.azurecr.io/$APP_NAME:${{ github.sha }}
          docker tag $ACR_NAME.azurecr.io/$APP_NAME:${{ github.sha }} \
                     $ACR_NAME.azurecr.io/$APP_NAME:latest
          docker push $ACR_NAME.azurecr.io/$APP_NAME:latest

      # 3d. Container Apps 배포 (Canary 10%)
      - name: Deploy canary revision (10%)
        run: |
          az containerapp update \
            --name $APP_NAME \
            --resource-group $AZURE_RG \
            --image $ACR_NAME.azurecr.io/$APP_NAME:${{ github.sha }} \
            --revision-suffix v${{ github.run_number }}

          # 최신 revision 에 10% 트래픽
          az containerapp ingress traffic set \
            --name $APP_NAME \
            --resource-group $AZURE_RG \
            --revision-weight latest=10 \
                              $APP_NAME--stable=90

  # ────────── 4. Smoke Test ──────────
  smoke-test:
    needs: deploy
    runs-on: ubuntu-latest
    steps:
      - name: Health check + sample query
        run: |
          curl -f https://${{ vars.APP_URL }}/health
          curl -f -X POST https://${{ vars.APP_URL }}/query \
            -H "Content-Type: application/json" \
            -d '{"query": "smoke test"}'
```

⚠️ **함정**:
- `permissions: id-token: write` 최상단에 없으면 OIDC 토큰 못 받음. 자주 놓침.
- `astral-sh/setup-uv@v3` 는 uv 캐시 자동. lock 파일 hash 로 layer 재사용.

💡 **팁**: `uv` 는 pip 대비 CI 시간 5-10x 단축. GitHub Actions minutes 절감 효과 큼.

---

## 8. Azure DevOps 대안

`azure-pipelines.yml` 로 같은 파이프라인. 차이:

- **Approval Gates**: Azure DevOps 는 stage 사이 manual approval UI 강력. GitHub Actions 는 `environments` 로 유사 구현.
- **Self-hosted Agents**: Azure DevOps 는 사내망 self-hosted agent 오래 지원. 사내 방화벽 뒤 리소스 접근 시 유리.
- **Enterprise 통합**: Azure AD 조직 계정 자연 연동.

기능적으로 동등. 팀 선호에 맞춤.

---

## 9. Auto-Rollback (App Insights → Action Group)

Ch.9 의 OTel + App Insights 를 배포 결정과 결합.

### 9.1 Alert 조건 (KQL)

```kusto
// 5분 window 에러율 spike
requests
| where timestamp > ago(5m)
| where cloud_RoleName == 'foundry-agent-app'
| summarize error_rate = countif(success == false) * 100.0 / count()
| where error_rate > 5   // 5% 초과 시 alert
```

### 9.2 Action Group → Logic App → Rollback

Alert 트리거 → Action Group → Logic App / Function → `az containerapp ingress traffic set` 로 canary 트래픽 0%.

```python
# Function 예 (Azure Functions Python)
# src/rollback_function/__init__.py
import logging
import azure.functions as func
from azure.identity import DefaultAzureCredential
from azure.mgmt.appcontainers import ContainerAppsAPIClient


def main(msg: func.QueueMessage) -> None:
    """App Insights alert → Container Apps rollback."""
    logging.info("Rollback trigger received")

    client = ContainerAppsAPIClient(
        credential=DefaultAzureCredential(),
        subscription_id="...",
    )

    # Canary 트래픽 0%
    client.container_apps.begin_update(
        resource_group_name="rg-foundry-prod",
        container_app_name="foundry-agent-app",
        container_app_envelope={
            "properties": {
                "configuration": {
                    "ingress": {
                        "traffic": [
                            {"revisionName": "foundry-agent-app--stable", "weight": 100},
                        ],
                    },
                },
            },
        },
    ).wait()
    logging.info("Rollback complete")
```

---

## 10. Zero-Secret CI/CD

**Azure Key Vault + GitHub OIDC** 조합이 2026 표준.

- **API Key CI 하드코딩 금지** (감사 fail)
- **Federated Identity** 로 GitHub → Azure 인증 (토큰 60분 자동 만료)
- **앱 런타임 Managed Identity** 로 Key Vault 에서 secret 로드

```python
# src/foundry_app/secrets.py
"""앱 런타임 Key Vault 접근."""
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
import os

kv_client = SecretClient(
    vault_url=os.environ["KEY_VAULT_URL"],
    credential=DefaultAzureCredential(),   # Managed Identity
)

db_password = kv_client.get_secret("db-password").value
```

⚠️ **함정**: OIDC 설정 후 예전 `AZURE_CLIENT_SECRET` GitHub secret 안 지우면 감사에서 지적. 마이그레이션 완료 후 즉시 rotate + delete.

---

## 11. 비용 회귀 방지

배포는 성공했는데 비용 30% 증가 흔함. 원인: 프롬프트 길어짐, 새 tool 추가로 왕복 늘어남, model 이 GPT-5.5 (더 비쌈) 로 바뀜.

### 11.1 PR 마다 Cost Estimation

```yaml
# .github/workflows/cost-check.yml
- name: Foundry cost estimation
  run: |
    uv run python scripts/estimate_cost.py \
      --baseline prompts/main/ \
      --candidate prompts/pr-${{ github.event.pull_request.number }}/ \
      --requests-per-day 100000
    # $50/day 초과 시 exit 1 → PR block
```

### 11.2 Budget Alert (Ch.10 재소환)

Foundry Resource budget 80% / 100% / 120% 3단계 alert.

---

## 12. Runbook 문서화 요구

기술만으론 부족. 새벽 3시 alert 받으면 뭘 해야 하는지 문서화 필수.

**최소 runbook 목록:**
- **Deployment failure**: 배포 실패 시 rollback 절차
- **High error rate**: 5% 초과 시 조사 순서
- **Quota exhaustion**: 429 폭증 시 Ch.8 failover client 활성
- **Content Safety false positive spike**: threshold 재조정
- **Prompt regression**: 이전 prompt version 즉시 배포
- **On-call rotation**: 팀 로테이션 + escalation path

Runbook 은 git 에 두고 alert 메시지에 link embed. 새벽 3시에 위키 뒤지지 않게.

---

## 13. 흔한 함정 정리

⚠️
1. **`quota` 없이 deployment 만들기** → InsufficientQuota 에러 (Bicep validation 은 통과)
2. **Prompt 하드코딩** → 한 줄 수정에 재배포. Prompts CMS 로 옮기면 재배포 불필요
3. **Canary session sticky 안 지킴** → UX 파탄
4. **Evaluation threshold 너무 엄격** → 배포 늘 fail → 실용 재조정
5. **OIDC 후 old secret 남김** → 감사 fail
6. **`uv sync` 대신 `uv add`** → CI 마다 최신 버전 pulling, lock 무시
7. **Docker slim 대신 alpine** → azure SDK glibc 이슈

💡 **베스트 프랙티스**:
- 프롬프트 canonical 은 git, deployed 는 Foundry CMS (하이브리드)
- Canary 룰은 error rate + quality score + latency 3축
- Shadow 배포는 고위험 도메인 (결제, 법률) 필수
- Key Vault 는 항상 soft delete + purge protection
- `uv.lock` 반드시 git commit
- Runbook 은 alert 링크에 embed

---

## 요약 (Cheat Sheet)

- **LLMOps 4대 축**: IaC · Prompt/Agent 버전 · Evaluation gate · Auto-rollback
- **IaC**: Azure-only → Bicep, multi-cloud → Terraform
- **Prompt 저장**: Git canonical + Foundry Prompts CMS sync (하이브리드)
- **배포**: Canary (점진) · Blue-Green (원자적) · Shadow (트래픽 복제)
- **Evaluation gate**: Ch.9 evaluator → CI, threshold 미달 시 배포 fail
- **Zero-Secret**: GitHub OIDC + Federated Identity + Managed Identity + Key Vault
- **Docker**: `uv` + Python slim, ~150MB, 캐시 layer 활용
- **Auto-rollback**: App Insights alert → Action Group → 트래픽 0%
- **Runbook**: git 에 두고 alert 에 링크

## 📚 더 읽기

- [Bicep for Azure Cognitive Services](https://learn.microsoft.com/en-us/azure/templates/microsoft.cognitiveservices/accounts)
- [Terraform azurerm_cognitive_deployment](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/cognitive_deployment)
- [Foundry CLI (Prompts sync)](https://learn.microsoft.com/en-us/cli/azure/foundry)
- [GitHub Actions + Azure OIDC 설정](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure)
- [Azure Container Apps Canary / Traffic Splitting](https://learn.microsoft.com/en-us/azure/container-apps/revisions)
- [`uv` in Docker (best practices)](https://docs.astral.sh/uv/guides/integration/docker/)
- [Key Vault + GitHub OIDC pattern](https://learn.microsoft.com/en-us/azure/key-vault/general/security-features)

## 다음 챕터

[Ch.12 폐쇄망 근접 배포 · 데이터 주권 · MS 계약 검증 →](Ch12_Enterprise_Deployment.md)

---
