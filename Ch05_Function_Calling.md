# Chapter 5. Function Calling & Tool Use (MCP Python + FastAPI)

[← 목차로](README.md)

> **학습 목표**
> - LLM 이 외부 함수를 자율적으로 호출하는 Function Calling 패턴을 이해한다.
> - `openai` SDK 로 tool schema 정의 → tool_call → 실행 → 응답 round-trip 을 구현한다.
> - `asyncio` 로 parallel tool calls 를 처리하고 성능을 극대화한다.
> - MCP (Model Context Protocol) Python SDK (`mcp` 1.28.1) + FastAPI 로 custom tool server 를 구축한다.
> - Foundry Agent Service 에 MCP 서버를 부착하여 자체 시스템을 agent tool 로 노출한다.

> **전제 조건**
> - [← Ch.3 API 연동 기초](Ch03_API_Basics.md), [← Ch.4 Prompt Engineering](Ch04_Prompt_Engineering.md) 완료
> - `uv` 프로젝트에 다음 추가: `uv add "mcp>=1.28.1" "fastapi>=0.115" "uvicorn>=0.32"`

---

## 1. Function Calling: 개념과 왜 필요한가

LLM 은 학습 시점의 지식만 안다. 실시간 정보 · 사내 데이터 · 외부 API 결과를 답하려면 **모델이 스스로 함수를 부르게** 해야 한다.

**두 가지 패턴 비교:**

| 방식 | 흐름 | 단점 |
|---|---|---|
| **문자열 파싱** | 모델 응답에서 "날씨 API 호출해야 함" 을 감지 → 파싱 → 실행 | 파싱 실패, 프롬프트 fragile |
| **Function Calling** (표준) | 모델이 `tool_calls` 필드로 구조화된 함수 호출 → 실행 → 결과 다시 넣기 | 없음 (표준화) |

**Round-trip 흐름:**

```
1. User: "서울 날씨 어때?"
2. Model → tool_call: {name: "get_weather", args: {city: "Seoul"}}
3. Python 앱: get_weather("Seoul") 실행 → {temp: 24, condition: "sunny"}
4. Model 재호출 (tool result 첨부) → "서울은 현재 24도, 맑음입니다."
```

---

## 2. Tool 스키마 정의 — Pydantic 방식 (권장)

Java 는 JSON Schema 를 손으로 조립해야 했지만, Python 은 **Pydantic 모델 → 자동 스키마 생성** 이 가능하다.

### 2.1 방법 1: openai `pydantic_function_tool` 헬퍼

```python
# src/foundry_app/tools/weather.py
from pydantic import BaseModel, Field
from openai import pydantic_function_tool


class GetWeatherArgs(BaseModel):
    """도시의 현재 날씨 조회."""
    city: str = Field(description="도시 이름 (한글 또는 영문)")
    unit: str = Field(default="celsius", description="celsius | fahrenheit")


# Tool schema 자동 생성
weather_tool = pydantic_function_tool(
    GetWeatherArgs,
    name="get_weather",
    description="특정 도시의 현재 날씨를 조회한다. 온도(°C/°F)와 상태를 반환.",
)
```

### 2.2 방법 2: 수동 스키마 (외부 스펙 wrapper 등)

```python
weather_tool_manual = {
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "특정 도시의 현재 날씨를 조회한다.",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "도시 이름"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"], "default": "celsius"},
            },
            "required": ["city"],
            "additionalProperties": False,
        },
        "strict": True,
    },
}
```

💡 **팁**: `strict: True` 를 켜면 Ch.4 Structured Outputs 와 동일한 보장 — 모델이 스키마 이탈한 args 를 만들 확률 zero.

---

## 3. 🔧 완전한 Round-trip 예제

```python
# src/foundry_app/function_calling.py
"""Function Calling 완전 예제 - 날씨 조회 에이전트."""
from __future__ import annotations
import json, os
from typing import Any, Callable
from openai import AzureOpenAI, pydantic_function_tool
from pydantic import BaseModel, Field

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    api_key=os.environ["AZURE_FOUNDRY_KEY"],
    api_version="2025-01-01-preview",
)


class GetWeatherArgs(BaseModel):
    """도시의 현재 날씨 조회."""
    city: str = Field(description="도시 이름")
    unit: str = Field(default="celsius", description="celsius | fahrenheit")


def get_weather(city: str, unit: str = "celsius") -> dict[str, Any]:
    """실제 구현은 외부 API 호출. 여기선 mock."""
    mock = {"Seoul": 24, "Tokyo": 26, "New York": 15}
    temp = mock.get(city, 20)
    if unit == "fahrenheit":
        temp = int(temp * 9 / 5 + 32)
    return {"city": city, "temperature": temp, "unit": unit, "condition": "sunny"}


TOOL_REGISTRY: dict[str, Callable] = {"get_weather": get_weather}


def chat_with_tools(user_message: str, max_iterations: int = 5) -> str:
    """Tool calling loop."""
    tools = [pydantic_function_tool(GetWeatherArgs, name="get_weather",
             description="특정 도시의 현재 날씨 조회.")]

    messages: list[dict[str, Any]] = [
        {"role": "system", "content": "너는 날씨 정보 어시스턴트다. 필요하면 get_weather tool 을 호출한다."},
        {"role": "user", "content": user_message},
    ]

    for _ in range(max_iterations):
        response = client.chat.completions.create(
            model=os.environ["AZURE_FOUNDRY_DEPLOYMENT"],
            messages=messages, tools=tools, tool_choice="auto",
            max_completion_tokens=1000, temperature=1.0,
        )
        message = response.choices[0].message

        # 1) tool_calls 없으면 최종 답변
        if not message.tool_calls:
            return message.content or ""

        # 2) assistant message 를 messages 에 (원본 그대로!)
        messages.append(message.model_dump(exclude_none=True))

        # 3) 각 tool 실행 → tool role 로 결과 삽입
        for tool_call in message.tool_calls:
            fn_name = tool_call.function.name
            args = json.loads(tool_call.function.arguments)
            fn = TOOL_REGISTRY.get(fn_name)
            result = fn(**args) if fn else {"error": f"Unknown tool: {fn_name}"}
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result, ensure_ascii=False),
            })

    return "Tool loop 한도 초과"


if __name__ == "__main__":
    print(chat_with_tools("서울과 도쿄 날씨 알려줘."))
```

⚠️ **함정 (초보자 필독)**:
1. **tool_call 결과를 messages 에 append 안 하면** 모델이 자기가 뭘 물었는지 잊어버림 → 무한 loop.
2. **`assistant` message 는 반드시 원본 그대로** 재삽입. tool_calls 필드 제거하면 이후 tool result 가 orphan.
3. **`tool_call_id` 매칭 필수**. openai API 가 이 ID 로 request-response 를 짝짓는다.
4. Loop 방어: `max_iterations` 없으면 이론상 무한 재귀.

---

## 4. Parallel Tool Calls — asyncio 로 성능 극대화

GPT-5 는 한 번의 응답에 여러 tool 을 동시에 호출한다. 순차 실행하면 시간 낭비. `asyncio.gather` 로 병렬 처리.

```python
# src/foundry_app/parallel_tools.py
import asyncio, json, os
from openai import AsyncAzureOpenAI

async_client = AsyncAzureOpenAI(
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    api_key=os.environ["AZURE_FOUNDRY_KEY"],
    api_version="2025-01-01-preview",
)


async def async_get_weather(city: str) -> dict:
    """실제로는 httpx.AsyncClient 로 외부 API 호출."""
    await asyncio.sleep(0.5)   # 네트워크 지연 시뮬레이션
    return {"city": city, "temperature": 22, "condition": "cloudy"}


ASYNC_REGISTRY = {"get_weather": async_get_weather}


async def dispatch_parallel(tool_calls: list) -> list[dict]:
    """여러 tool call 을 병렬 실행."""
    async def _run(tool_call):
        fn = ASYNC_REGISTRY[tool_call.function.name]
        args = json.loads(tool_call.function.arguments)
        try:
            result = await fn(**args)
        except Exception as e:
            result = {"error": str(e)}
        return {
            "role": "tool",
            "tool_call_id": tool_call.id,
            "content": json.dumps(result, ensure_ascii=False),
        }

    # return_exceptions=True 로 하나 실패해도 나머지 진행
    return await asyncio.gather(*[_run(tc) for tc in tool_calls])
```

⚠️ **함정**: `asyncio.gather` 는 하나 실패하면 전체 취소가 default. `return_exceptions=True` 로 격리하거나 개별 `try/except`.

---

## 5. `tool_choice` 제어

| 값 | 의미 | 사용 시점 |
|---|---|---|
| `"auto"` (기본) | 모델이 판단 | 대부분 |
| `"none"` | 절대 호출 안 함 | 순수 응답 |
| `"required"` | 반드시 하나는 호출 | 정보 조회 강제 |
| `{"type":"function","function":{"name":"..."}}` | 특정 함수 강제 | 특정 workflow |

---

## 6. 🔴 MCP (Model Context Protocol) — 2026-06 GA

MCP 는 Anthropic 이 발표하고 MS · OpenAI 가 채택한 **tool exchange 표준**. JSON-RPC 2.0 기반.

**왜 MCP 인가:**
- Function calling 5-10개 하기 시작하면 각 함수마다 인증·오류 처리·감사 로직이 파편화된다.
- MCP 는 tool discovery · 실행 · 인증을 **표준 프로토콜** 로 통일.
- Foundry · Copilot · Claude Desktop 이 모두 동일 MCP 서버를 소비.

### 6.1 Python SDK 상태 (2026-07)

| 버전 | 상태 | 권장 |
|---|---|---|
| **`mcp` 1.28.1** | ✅ Stable | 🎯 프로덕션 |
| `mcp` 2.0.0b1 | 🔄 Pre-release | 프로덕션 X |

Java 는 v2.0.0 GA 이나 **Python 은 v1.x 가 표준** — 2주 lag. Ch.11 CI 에서 v2 마이그레이션 계획.

---

## 7. 🔧 MCP Server 만들기 — FastMCP

**시나리오**: 사내 CRM 을 MCP 로 노출 → Foundry Agent 가 고객 정보를 자동 조회.

### 7.1 프로젝트 셋업

```bash
uv init crm-mcp-server --package
cd crm-mcp-server
uv add "mcp>=1.28.1" "fastapi>=0.115" "uvicorn[standard]>=0.32" "httpx>=0.27"
```

### 7.2 MCP Server 코드

```python
# src/crm_mcp_server/server.py
"""CRM MCP Server - 사내 REST API 를 MCP tool 로 노출."""
from __future__ import annotations
import os
import httpx
from mcp.server.fastmcp import FastMCP

# FastMCP - mcp 1.28.1 의 고수준 API, FastAPI-like decorator
mcp = FastMCP(
    name="crm-server",
    version="1.0.0",
    instructions="사내 CRM 시스템 조회 도구.",
)

CRM_BASE = os.environ["INTERNAL_CRM_URL"]
CRM_KEY = os.environ["INTERNAL_CRM_KEY"]


@mcp.tool()
async def get_customer(customer_id: str) -> dict:
    """고객 ID 로 CRM 에서 고객 정보 조회.

    Args:
        customer_id: 고객 고유 식별자 (예: 'CUST-12345')

    Returns:
        고객 정보 dict (name, email, tier, joined_date, ...)
    """
    async with httpx.AsyncClient(timeout=10.0) as client:
        r = await client.get(
            f"{CRM_BASE}/customers/{customer_id}",
            headers={"X-API-Key": CRM_KEY},
        )
        r.raise_for_status()
        return r.json()


@mcp.tool()
async def search_customers(query: str, limit: int = 10) -> list[dict]:
    """이름·이메일로 고객 검색."""
    async with httpx.AsyncClient(timeout=10.0) as client:
        r = await client.get(
            f"{CRM_BASE}/customers/search",
            params={"q": query, "limit": limit},
            headers={"X-API-Key": CRM_KEY},
        )
        r.raise_for_status()
        return r.json()["results"]


@mcp.tool()
async def create_support_ticket(
    customer_id: str, subject: str, description: str, priority: str = "normal",
) -> dict:
    """지원 티켓 생성.

    Args:
        priority: low | normal | high | urgent
    """
    async with httpx.AsyncClient(timeout=10.0) as client:
        r = await client.post(
            f"{CRM_BASE}/tickets",
            headers={"X-API-Key": CRM_KEY, "Content-Type": "application/json"},
            json={"customer_id": customer_id, "subject": subject,
                  "description": description, "priority": priority},
        )
        r.raise_for_status()
        return r.json()


if __name__ == "__main__":
    # STDIO - Claude Desktop / local dev
    mcp.run(transport="stdio")
```

### 7.3 HTTP 배포 (Foundry Agent 용)

```python
# src/crm_mcp_server/http_server.py
"""HTTP 배포 - Container Apps / Functions."""
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("crm-server", version="1.0.0")

# ... (@mcp.tool 정의 그대로) ...

if __name__ == "__main__":
    # SSE 모드 - Foundry Agent Service 가 소비
    mcp.run(transport="sse", host="0.0.0.0", port=8000)
```

### 7.4 실행 & 확인

```bash
# 로컬 실행
uv run python -m crm_mcp_server.http_server

# curl 로 tool list 확인
curl http://localhost:8000/sse
```

### 7.5 Foundry Agent 에 연결

```python
# src/foundry_app/agent_with_mcp.py
import os
from azure.ai.agents import AgentsClient
from azure.ai.agents.models import MCPToolDefinition
from azure.identity import DefaultAzureCredential

agents = AgentsClient(
    endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    credential=DefaultAzureCredential(),
)

crm_mcp = MCPToolDefinition(
    server_label="crm",
    server_url="https://crm-mcp.internal.corp/sse",
    require_approval="never",   # or "always" for sensitive ops
)

agent = agents.create_agent(
    model="gpt-5",
    name="customer-support-agent",
    instructions="너는 고객 지원 에이전트다. CRM 조회 및 티켓 생성에 crm 서버 tools 를 사용.",
    tools=[crm_mcp],
)
```

Ch.7 (Agent Service) 에서 이 패턴을 확장.

---

## 8. Foundry MCP Server (Preview, 2026-06)

Microsoft 가 Foundry Search · Evaluation · Storage 를 MCP endpoint 로 노출. 사용자가 직접 서버를 만들지 않아도 Foundry 기능을 MCP tool 로 소비.

- **인증**: Key-based, Microsoft Entra, OAuth (OBO)
- **상태**: Preview → 프로덕션은 표준 REST API

## 9. Toolbox — 여러 MCP 를 하나로 묶기

에이전트에 5개 MCP 서버 개별 등록 = 관리 복잡. **Toolbox** = 여러 MCP aggregation. Foundry Portal 에서 UI 로 생성.

Ch.13 사내 통합 시나리오에서 상세.

---

## 10. 인증 패턴 (실전)

| 방식 | 사용 시점 | 구현 |
|---|---|---|
| **Key-based** | 사내 개발 · POC | HTTP `X-API-Key` header |
| **Agent Identity** | agent 자체를 principal | Managed Identity |
| **Project Managed Identity** | 프로젝트 내 모든 agent 공유 | Foundry Project MI |
| **OAuth Identity Passthrough (OBO)** | 사용자를 대신 (Ch.13) | MSAL OBO flow |

FastMCP + FastAPI middleware 로 key auth 예:

```python
from fastapi import FastAPI, Header, HTTPException, Depends
from mcp.server.fastmcp import FastMCP

app = FastAPI()
mcp = FastMCP("crm-server")

async def verify_key(x_api_key: str = Header(...)):
    if x_api_key != os.environ["EXPECTED_KEY"]:
        raise HTTPException(status_code=401, detail="Invalid API key")

app.mount("/sse", mcp.sse_app(dependencies=[Depends(verify_key)]))
```

⚠️ **함정**: **인증 없는 public MCP 서버는 프롬프트 인젝션 벡터**. agent 가 다녀오는 내용이 감사 로그로 안 남으면 데이터 유출 원인.

---

## 11. 흔한 함정 (반복 강조)

1. **tool_call 결과 append 누락** → 모델이 자기 tool 사용을 인식 못함 → 무한 loop.
2. **`assistant` message 전체 재삽입** — tool_calls 만 뽑아 넣으면 orphan.
3. **Tool description 부실** → 엉뚱한 tool 호출. description = 모델을 위한 지시서.
4. **Parallel tool 예외 처리** — `return_exceptions=True` 로 각 실패 격리.
5. **`required` 필드 남용** — 모든 필드 required 로 넣으면 모델이 값을 fabrication.
6. **MCP 서버 트러스트** — 인증 없는 MCP 위험. 반드시 auth + logging.
7. **Strict + 유연한 스키마** — Union/Optional 은 `type: ["string", "null"]` 유니온.

💡 **베스트 프랙티스:**

- **Tool 이름 명확**: `get_customer_by_id` > `getCust`
- **description 에 예시 포함**: "예: `get_weather('Seoul')` → 서울 현재 온도"
- **Tool 수 15개 이하** — 넘어가면 Toolbox 로 그룹화
- **민감 작업**: `require_approval="always"` — 사용자 confirmation UI 강제
- **Cost 모니터링**: tool call 마다 프롬프트 크기 늘어남. Ch.9 OTel 로 추적.

---

## 요약 (Cheat Sheet)

- **Function Calling**: `tools=[...]` + `tool_choice="auto"` 로 시작. Round-trip 은 tool result 를 messages 에 재삽입.
- **Pydantic 방식**: `pydantic_function_tool(MyModel)` 로 스키마 자동. `strict: True` 로 준수 보장.
- **Parallel**: `asyncio.gather(..., return_exceptions=True)`.
- **MCP Python SDK**: `mcp` 1.28.1 stable. `FastMCP` decorator 로 서버 신속 구축.
- **Foundry MCP 부착**: `azure-ai-agents` 의 `MCPToolDefinition`.
- **인증**: 사내 → key-based, 사용자 대신 → OBO (Ch.13).

## 📚 더 읽기

- [OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling)
- [Azure OpenAI Function Calling](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/function-calling)
- [MCP 공식 스펙](https://modelcontextprotocol.io/)
- [`mcp` Python SDK GitHub](https://github.com/modelcontextprotocol/python-sdk)
- [Foundry MCP Get Started](https://learn.microsoft.com/en-us/azure/foundry/mcp/get-started)
- [FastMCP 고수준 API](https://github.com/jlowin/fastmcp)

## 다음 챕터

[Ch.6 RAG — Azure AI Search + LangChain/LlamaIndex →](Ch06_RAG_AI_Search.md)

---
