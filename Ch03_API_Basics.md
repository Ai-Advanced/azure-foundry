# Chapter 3. API 연동 기초 (Python + REST)

[← 목차로](README.md)

> **학습 목표**
> - Foundry 프로젝트의 엔드포인트와 인증 구조를 이해한다.
> - `uv` 로 Python 3.11+ 프로젝트를 셋업하고 `pyproject.toml` (PEP 621) 로 의존성을 관리한다.
> - `openai` (2.44.0+) 와 `azure-ai-projects` (2.3.0+) 를 사용하여 GPT-5 모델과 대화하는 코드를 작성한다.
> - Semantic Kernel Python과 LangChain 대안을 이해한다.
> - REST API(curl)를 통해 SDK 없이 모델을 호출하는 방법을 익힌다.
> - Managed Identity 기반의 안전한 인증을 구현한다.

> **전제 조건**
> - [← Ch.2 첫 모델 배포](Ch02_First_Deployment.md) 완료 (GPT-5 배포 완료)
> - **Python 3.11+ (권장 3.13)** 설치
> - **`uv`** 설치 (`curl -LsSf https://astral.sh/uv/install.sh | sh`)
> - Azure CLI 2.60+ + `az login` 완료
> - 배포된 모델의 Endpoint 및 API Key 확보

---

## 1. API 경로와 라우팅 구조

Foundry 프로젝트를 생성하고 모델을 배포하면 호출 가능한 엔드포인트가 생성된다. 2026년 현재 Foundry는 더 단순하고 일관된 라우팅 체계를 제공한다.

### 1.1 주요 API 경로 (Stable Routes)

🔴 **2026 변경**: 과거에는 `api-version=2024-02-15-preview`처럼 모든 요청에 날짜 기반 쿼리 파라미터를 명시해야 했다. 현재는 `/openai/v1/`로 시작하는 고정된(stable) 경로를 통해 버전 관리의 복잡성을 줄였다.

- **Chat Completions**: `/openai/v1/chat/completions`
  - 가장 일반적으로 사용되는 대화형 API. GPT-5 계열 모델과의 텍스트 기반 상호작용에 사용.
- **Responses API**: `/openai/v1/responses`
  - 🔴 **2026 변경**: 기존 Assistants API(Threads, Messages 등)를 대체하는 차세대 에이전트 인터페이스. 멀티모달 처리 및 추론 모델(o-series) 연동 시 권장. Ch.7에서 상세.
- **Embeddings**: `/openai/v1/embeddings`
  - 텍스트를 벡터로 변환하여 RAG 엔진에 입력. Ch.6에서 상세.

### 1.2 엔드포인트 구성

Foundry의 엔드포인트는 세 가지 형태를 가진다.

| 형태 | 예시 | 사용 시점 |
|---|---|---|
| **Azure OpenAI 호환** | `https://<name>.openai.azure.com/` | Ch.2에서 GPT-5 배포한 경우 |
| **Foundry 통합 (권장)** | `https://<name>.services.ai.azure.com/` | 2026 신 통합 endpoint. Ch.8 모델 라우터에 필수 |
| **Cognitive Services** | `https://<name>.cognitiveservices.azure.com/` | 통합 AIServices kind 배포 |

`openai` SDK는 이 셋 다 지원. Foundry 통합 형태(`.services.ai.azure.com`)를 권장.

---

## 2. 두 가지 인증 방식

Foundry는 보안 수준과 운영 환경에 따라 두 가지 인증 방식을 제공한다.

### 2.1 API Key 방식

- **헤더**: `api-key: <YOUR_KEY>`
- **특징**: 로컬 개발이나 빠른 프로토타이핑에 적합. Foundry 포털의 'Project settings' → 'Keys' 에서 확인.
- **주의**: 소스 코드에 키를 직접 입력하지 말고 반드시 환경변수 또는 `.env` 파일 사용.

### 2.2 Entra ID (전 Azure AD) 방식 — 프로덕션 표준

- **헤더**: `Authorization: Bearer <TOKEN>`
- **특징**: 프로덕션 환경에서 필수. API Key 노출 위험 없음, RBAC로 세밀한 권한 제어.
- **구현**: Python에서 `azure-identity`의 `DefaultAzureCredential` 사용. 로컬 `az login` 정보나 서버의 Managed Identity에서 자동 토큰 획득.

⚠️ **함정**: `Bearer` 토큰은 60분 후 만료. `azure-identity`는 자동 갱신(refresh)해주지만, 로컬 개발 중 세션 만료 시 `az login`을 다시 실행해야 할 수 있다.

---

## 🔧 실습 1 — `uv` 프로젝트 셋업

`uv` (astral-sh/uv)는 2026년 Python 개발의 사실상 표준이다. `pip`보다 10-100배 빠르고, lock 파일과 workspace를 기본 지원한다.

### Directory 구조

```text
foundry-app/
├── pyproject.toml       # PEP 621 프로젝트 정의
├── uv.lock              # 재현 가능한 dependency lock (자동 생성)
├── .env                 # 환경변수 (git ignore)
├── .python-version      # Python 버전 지정 (자동 생성)
└── src/
    └── foundry_app/
        ├── __init__.py
        └── main.py
```

### 프로젝트 초기화

```bash
# bash / powershell
uv init foundry-app --package
cd foundry-app

# Python 3.13 (free-threaded) 사용
uv python install 3.13
uv python pin 3.13

# 핵심 의존성 추가
uv add \
  "openai>=2.44.0" \
  "azure-ai-projects>=2.3.0" \
  "azure-identity>=1.25.3" \
  "pydantic>=2.13.4" \
  "python-dotenv>=1.0.0"

# 로깅
uv add "structlog>=24.4"
```

### `pyproject.toml` 결과 (PEP 621)

```toml
# pyproject.toml
[project]
name = "foundry-app"
version = "0.1.0"
description = "Microsoft Foundry Python Application"
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
    "openai>=2.44.0",
    "azure-ai-projects>=2.3.0",
    "azure-identity>=1.25.3",
    "pydantic>=2.13.4",
    "python-dotenv>=1.0.0",
    "structlog>=24.4",
]

[project.optional-dependencies]
langchain = [
    "langchain>=0.3.0",
    "langchain-azure-ai>=1.2.8",
]
semantic-kernel = ["semantic-kernel>=1.43.1"]
dev = [
    "pytest>=8.0",
    "ruff>=0.7.0",
    "basedpyright>=1.20",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.uv]
package = true
```

### 환경변수 설정 (`.env`)

```bash
# .env (반드시 .gitignore에 추가!)
AZURE_FOUNDRY_ENDPOINT=https://your-foundry.services.ai.azure.com
AZURE_FOUNDRY_KEY=your-api-key-here
AZURE_FOUNDRY_DEPLOYMENT=gpt-5
AZURE_FOUNDRY_API_VERSION=2025-01-01-preview
```

⚠️ **함정**: `azure-ai-projects`와 `azure-ai-openai`는 다른 패키지다. **`azure-ai-openai`는 사용하지 말 것** — 이는 Java의 beta 패키지에 대응하는 구버전. Python에서는 **`openai` Azure 호환 클라이언트**가 표준.

💡 **팁**: `uv sync` 로 `uv.lock`을 재현하면 팀원 간 완전히 동일한 환경 구성. CI/CD 파이프라인에도 `uv sync --frozen` 을 사용해 lock을 지키자.

---

## 🔧 실습 2 — `openai` SDK로 Hello Foundry

가장 직관적이고 유지보수 활발한 방식. GPT-5 Responses API도 곧바로 지원.

```python
# src/foundry_app/hello.py
"""Hello Foundry — 최소 예제 (API Key 인증)."""
from __future__ import annotations

import os
import sys
from dotenv import load_dotenv
from openai import AzureOpenAI, APIStatusError

load_dotenv()  # .env 파일 로드

ENDPOINT = os.environ["AZURE_FOUNDRY_ENDPOINT"]
API_KEY = os.environ["AZURE_FOUNDRY_KEY"]
DEPLOYMENT = os.environ["AZURE_FOUNDRY_DEPLOYMENT"]
API_VERSION = os.environ.get("AZURE_FOUNDRY_API_VERSION", "2025-01-01-preview")


def main() -> None:
    client = AzureOpenAI(
        azure_endpoint=ENDPOINT,
        api_key=API_KEY,
        api_version=API_VERSION,
    )

    try:
        response = client.chat.completions.create(
            model=DEPLOYMENT,  # ← 배포 이름 (모델 이름 아님!)
            messages=[
                {"role": "system", "content": "너는 Python 시니어 엔지니어다. 간결하게 답한다."},
                {"role": "user", "content": "안녕 Foundry! Python으로 보내는 첫 메시지야."},
            ],
            # 🔴 GPT-5 계열: max_completion_tokens (max_tokens 아님)
            max_completion_tokens=500,
            # 🔴 GPT-5 reasoning 모델: temperature=1.0 고정
            temperature=1.0,
        )
        reply = response.choices[0].message.content
        print(f"Foundry 응답:\n{reply}")

        # 사용량 확인
        usage = response.usage
        print(f"\n토큰 사용: prompt={usage.prompt_tokens}, "
              f"completion={usage.completion_tokens}, total={usage.total_tokens}")

    except APIStatusError as e:
        print(f"HTTP 에러 {e.status_code}: {e.message}", file=sys.stderr)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

### 실행

```bash
uv run python -m foundry_app.hello
```

### Entra ID (Managed Identity) 인증 버전

프로덕션에서 반드시 이 방식을 사용한다.

```python
# src/foundry_app/hello_entra.py
"""Entra ID 인증 - 프로덕션 표준."""
from __future__ import annotations

import os
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import AzureOpenAI

ENDPOINT = os.environ["AZURE_FOUNDRY_ENDPOINT"]
DEPLOYMENT = os.environ["AZURE_FOUNDRY_DEPLOYMENT"]

# DefaultAzureCredential 체인:
#  1. 환경변수 (AZURE_CLIENT_ID/TENANT_ID/CLIENT_SECRET)
#  2. Managed Identity (Container Apps / VM / Function)
#  3. Azure CLI (`az login`)
#  4. Azure PowerShell
#  5. VS Code (`Azure Account` extension)
credential = DefaultAzureCredential()

# Cognitive Services 전용 scope
token_provider = get_bearer_token_provider(
    credential,
    "https://cognitiveservices.azure.com/.default",
)

client = AzureOpenAI(
    azure_endpoint=ENDPOINT,
    azure_ad_token_provider=token_provider,   # ← API Key 대신
    api_version="2025-01-01-preview",
)

response = client.chat.completions.create(
    model=DEPLOYMENT,
    messages=[{"role": "user", "content": "인증 성공?"}],
    max_completion_tokens=100,
)
print(response.choices[0].message.content)
```

⚠️ **함정**:
- `DefaultAzureCredential` 이 로컬에서 실패하면 `az login` 부터. Managed Identity 는 Azure 내부에서만 동작.
- Token scope은 반드시 `https://cognitiveservices.azure.com/.default` (Foundry가 Cognitive Services 위에 구축됨).
- 토큰 만료(60분) 는 SDK가 자동 refresh — 코드에서 신경 쓸 필요 없음.

---

## 🔧 실습 3 — `azure-ai-projects` 로 Foundry 통합

`azure-ai-projects` 는 Foundry 프로젝트 전체를 관리하는 SDK. 여러 서비스(Chat, Agents, Evaluation, Connections)를 하나의 client로 다룰 수 있다.

```python
# src/foundry_app/via_projects.py
"""azure-ai-projects: Foundry 통합 SDK 예제."""
from __future__ import annotations

import os
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient

PROJECT_ENDPOINT = os.environ["AZURE_FOUNDRY_ENDPOINT"]

# AIProjectClient - Foundry의 backbone client
project_client = AIProjectClient(
    endpoint=PROJECT_ENDPOINT,
    credential=DefaultAzureCredential(),
)

# .inference : Chat / Embeddings
# .agents    : Foundry Agent Service (Ch.7)
# .evaluations: Evaluation SDK (Ch.9)
# .connections: 외부 리소스 연결 (Search, Storage 등, Ch.6)

# Chat completions (openai 클라이언트 얻어서 사용)
inference_client = project_client.inference.get_azure_openai_client(
    api_version="2025-01-01-preview",
)

response = inference_client.chat.completions.create(
    model=os.environ["AZURE_FOUNDRY_DEPLOYMENT"],
    messages=[
        {"role": "user", "content": "Foundry 통합 SDK 좋은데?"},
    ],
    max_completion_tokens=100,
    temperature=1.0,
)
print(response.choices[0].message.content)
```

💡 **팁**: `AIProjectClient` 는 Ch.7 (Agents), Ch.9 (Evaluations) 에서도 재사용한다. 프로덕션 앱에서는 이 client를 앱 lifecycle 동안 하나만 만들어 재사용 (singleton pattern).

---

## 🔧 실습 4 — Semantic Kernel Python (오케스트레이션 대안)

복잡한 chain / plugin / memory 관리가 필요하다면 Semantic Kernel Python 이 강력하다. 2026년 현재 **활발히 개발 중** (Java 는 유지보수 모드).

```python
# src/foundry_app/via_sk.py
"""Semantic Kernel Python 예제."""
import asyncio
import os
from semantic_kernel import Kernel
from semantic_kernel.connectors.ai.open_ai import AzureChatCompletion
from semantic_kernel.contents import ChatHistory


async def main() -> None:
    kernel = Kernel()

    # Azure OpenAI 연결 추가
    kernel.add_service(AzureChatCompletion(
        service_id="foundry",
        deployment_name=os.environ["AZURE_FOUNDRY_DEPLOYMENT"],
        endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
        api_key=os.environ["AZURE_FOUNDRY_KEY"],
    ))

    chat_service = kernel.get_service("foundry")
    history = ChatHistory()
    history.add_system_message("너는 도움이 되는 어시스턴트다.")
    history.add_user_message("Semantic Kernel Python과 Foundry 조합 어때?")

    result = await chat_service.get_chat_message_content(chat_history=history, settings=None)
    print(result.content)


if __name__ == "__main__":
    asyncio.run(main())
```

Semantic Kernel은 Ch.5 (Function Calling), Ch.7 (Agents) 에서 더 깊게 다룬다.

## 🔧 실습 5 — LangChain 대안

LangChain 은 프롬프트 체인·RAG·에이전트 등 광범위한 생태계 갖춤. Foundry-native 통합 패키지 `langchain-azure-ai` 로 자연스럽게 연결.

```python
# src/foundry_app/via_langchain.py
"""LangChain + Foundry 예제."""
import os
from langchain_azure_ai.chat_models import AzureAIChatCompletionsModel

model = AzureAIChatCompletionsModel(
    endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    credential=os.environ["AZURE_FOUNDRY_KEY"],
    model=os.environ["AZURE_FOUNDRY_DEPLOYMENT"],
    api_version="2025-01-01-preview",
)

# LangChain의 표준 인터페이스
result = model.invoke([
    ("system", "너는 간결한 응답을 하는 어시스턴트다."),
    ("user", "LangChain + Foundry로 뭘 할 수 있어?"),
])
print(result.content)
```

**선택 기준:**
| 상황 | 권장 |
|---|---|
| 단순 chat 호출, 최소 dep | `openai` (실습 2) |
| Foundry 서비스(Agents, Eval) 통합 | `azure-ai-projects` (실습 3) |
| 복잡한 workflow · plugin · 메모리 | Semantic Kernel (실습 4) |
| RAG · 체인 · 다양한 LLM 통합 | LangChain (실습 5) |
| 데이터 분석·리서치 파이프라인 | LlamaIndex (Ch.6) |

---

## 3. REST curl 등가

SDK 없는 환경 (디버깅, curl 자동화, 임베디드) 에서 직접 HTTP 요청.

### Bash (Linux/macOS/WSL)

```bash
# bash
curl "${AZURE_FOUNDRY_ENDPOINT}/openai/v1/chat/completions?api-version=2025-01-01-preview" \
  -H "Content-Type: application/json" \
  -H "api-key: ${AZURE_FOUNDRY_KEY}" \
  -d '{
    "model": "gpt-5",
    "messages": [{"role": "user", "content": "REST 테스트"}],
    "max_completion_tokens": 100,
    "temperature": 1.0
  }'
```

### Windows PowerShell

```powershell
# powershell
$headers = @{
    "api-key" = $env:AZURE_FOUNDRY_KEY
    "Content-Type" = "application/json"
}

$body = @{
    model = $env:AZURE_FOUNDRY_DEPLOYMENT
    messages = @(
        @{ role = "user"; content = "PowerShell 테스트" }
    )
    max_completion_tokens = 100
    temperature = 1.0
} | ConvertTo-Json

$uri = "$($env:AZURE_FOUNDRY_ENDPOINT)/openai/v1/chat/completions?api-version=2025-01-01-preview"

Invoke-RestMethod -Uri $uri -Method Post -Headers $headers -Body $body
```

### Python `httpx` (SDK 대안)

```python
# 순수 HTTP 호출 (진단·저수준 제어)
import httpx, os

response = httpx.post(
    f"{os.environ['AZURE_FOUNDRY_ENDPOINT']}/openai/v1/chat/completions",
    params={"api-version": "2025-01-01-preview"},
    headers={
        "api-key": os.environ["AZURE_FOUNDRY_KEY"],
        "Content-Type": "application/json",
    },
    json={
        "model": os.environ["AZURE_FOUNDRY_DEPLOYMENT"],
        "messages": [{"role": "user", "content": "httpx 테스트"}],
        "max_completion_tokens": 100,
        "temperature": 1.0,
    },
    timeout=60.0,
)
print(response.json())
```

---

## 4. 에러 처리와 재시도 정책

LLM 호출은 네트워크 지연 / 할당량 제한(Rate Limit) / 서버 오류로 실패 가능성 상존.

| 상태 | 의미 | 대응 |
|---|---|---|
| **429** Too Many Requests | TPM/RPM 초과 | Exponential backoff (5s→15s→45s...) |
| **500** Internal Server Error | Foundry 내부 | 즉시 재시도 (최대 3회) |
| **503** Service Unavailable | 일시 과부하 | Backoff 후 재시도 |
| **400** Bad Request | 요청 형식 오류 | **재시도 안 함**. 로그 확인 |
| **401** Unauthorized | 인증 실패 | 토큰 재발급 or 재로그인 |
| **403** Forbidden | 권한 없음 | RBAC 역할 확인 |

### `openai` SDK 내장 재시도

`openai` 는 기본적으로 지수 백오프 재시도를 내장한다. Client 초기화 시 조정 가능.

```python
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    api_key=os.environ["AZURE_FOUNDRY_KEY"],
    api_version="2025-01-01-preview",
    max_retries=3,     # 기본 2회, 필요시 증가
    timeout=60.0,      # 초 단위, 기본 10분
)
```

### 커스텀 재시도 (`tenacity`)

```python
# uv add tenacity
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
from openai import APIStatusError

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=2, min=5, max=60),
    retry=retry_if_exception_type(APIStatusError),
)
def call_with_retry(client: AzureOpenAI, messages: list) -> str:
    response = client.chat.completions.create(
        model=os.environ["AZURE_FOUNDRY_DEPLOYMENT"],
        messages=messages,
        max_completion_tokens=500,
        temperature=1.0,
    )
    return response.choices[0].message.content
```

⚠️ **함정**: 429 에러에는 응답 헤더 `Retry-After` 가 포함된다. Blind exponential backoff보다 이 값을 준수하는 게 정확. `openai` SDK 는 이미 이걸 반영한다.

---

## 5. VS Code / Copilot Chat 연동

앞서 사용자와 나눈 대화(Azure vs Custom Endpoint 판단) 를 실전으로.

**Azure AI Foundry 확장 프로그램** (Marketplace 검색: "Azure AI Foundry") 을 설치하면:
- Foundry 프로젝트 리스트 자동 로드
- 배포된 모델 목록 확인
- `.env` 자동 생성 (endpoint / key 자동 채움)
- Playground 인라인 실행

💡 **팁**: Copilot Chat에서 Azure OpenAI 를 붙일 때는 `Custom Endpoint` 필드 비우기. Azure 전용 preset 이 인증 로직과 API version 을 자동 처리. 자체 Gateway 를 거치는 특수 상황 아니면 Azure 기본 경로 사용이 안전.

**VS Code 설정** (`settings.json` 예):

```json
// .vscode/settings.json
{
  "python.defaultInterpreterPath": ".venv/bin/python",
  "python.analysis.typeCheckingMode": "strict",
  "azure.aiFoundry.defaultProject": "prod-foundry-eastus",
  "editor.formatOnSave": true,
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff"
  }
}
```

---

## 6. 로깅과 디버깅

프로덕션 앱은 반드시 structured logging + trace ID 를 도입.

### `structlog` 기본 셋업

```python
# src/foundry_app/logging_setup.py
import logging
import structlog

def setup_logging(level: str = "INFO") -> None:
    logging.basicConfig(level=level, format="%(message)s")
    structlog.configure(
        processors=[
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
        logger_factory=structlog.stdlib.LoggerFactory(),
    )

log = structlog.get_logger()
```

### Azure Monitor OpenTelemetry (Ch.9에서 상세)

```python
# uv add azure-monitor-opentelemetry
from azure.monitor.opentelemetry import configure_azure_monitor

configure_azure_monitor(
    connection_string=os.environ["APPLICATIONINSIGHTS_CONNECTION_STRING"],
)
# 이후 openai 호출은 자동으로 App Insights 로 trace
```

### OpenAI raw HTTP 로그

```bash
# 모든 요청 / 응답 원문 확인
export OPENAI_LOG=debug   # or set OPENAI_LOG=debug on Windows

# 실행
uv run python -m foundry_app.hello
```

⚠️ **함정**: `OPENAI_LOG=debug` 는 프롬프트/응답 전체를 stdout 에 찍는다. **프로덕션에는 절대 켜지 말 것** — 민감 정보 누출. 개발·트러블슈팅 전용.

---

## 요약 (Cheat Sheet)

- **패키지 관리**: `uv` + `pyproject.toml` (PEP 621) 표준
- **인증**: API Key (개발/테스트) vs `DefaultAzureCredential` (프로덕션 필수)
- **API 경로**: 2026년부터 `/openai/v1/` 고정 경로
- **주 SDK**: `openai` (Foundry chat native) + `azure-ai-projects` (Foundry 통합)
- **오케스트레이션**: 단순 → `openai` / 복잡 → Semantic Kernel or LangChain
- **에러 대응**: `max_retries` + `tenacity` 재시도. `Retry-After` 헤더 준수.
- **GPT-5**: `max_completion_tokens` (max_tokens 아님), `temperature=1.0` 고정
- **로깅**: `structlog` 로 JSON 로그, `azure-monitor-opentelemetry` 로 App Insights trace

## 📚 더 읽기

- [Azure AI Foundry Python SDK Hub](https://learn.microsoft.com/en-us/python/api/overview/azure/)
- [azure-ai-projects (Python) README](https://learn.microsoft.com/en-us/python/api/overview/azure/ai-projects-readme)
- [openai Python SDK (PyPI)](https://pypi.org/project/openai/)
- [uv 공식 문서](https://docs.astral.sh/uv/)
- [Azure OpenAI Service REST API Reference](https://learn.microsoft.com/en-us/azure/ai-services/openai/reference)
- [DefaultAzureCredential (azure-identity)](https://learn.microsoft.com/en-us/python/api/azure-identity/azure.identity.defaultazurecredential)
- [Semantic Kernel Python](https://learn.microsoft.com/en-us/semantic-kernel/overview/)
- [LangChain Azure AI 통합](https://python.langchain.com/docs/integrations/chat/azure_ai/)

## 다음 챕터

[Ch.4 Prompt Engineering & Structured Outputs (Pydantic v2) →](Ch04_Prompt_Engineering.md)

---
