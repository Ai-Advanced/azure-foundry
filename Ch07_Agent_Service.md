# Chapter 7. Foundry Agent Service (Responses API v2 + Semantic Kernel Python)

[← 목차로](README.md)

> **학습 목표**
> - 🔴 Assistants API 가 왜 deprecated 되었고 **Responses API v2** 가 어떻게 대체하는지 이해한다.
> - `azure-ai-agents` (1.1.0 GA) Python SDK 로 agent · conversation · response 를 만든다.
> - Prompt Agents (managed) vs Hosted Agents (your code) 선택 기준을 세운다.
> - Built-in tools (Code Interpreter · File Search · Web Search · Function · MCP) 를 조합한다.
> - Connected Agents 로 multi-agent orchestration 을 구현한다.
> - Semantic Kernel Python 으로 자유도 높은 agent 를 프로그래머틱하게 정의한다.

> **전제 조건**
> - [← Ch.3 API 연동 기초](Ch03_API_Basics.md), [← Ch.5 Function Calling & MCP](Ch05_Function_Calling.md) 완료
> - `uv add "azure-ai-agents>=1.1.0" "semantic-kernel>=1.43.1"`

---

## 1. 🔴 왜 새로운 API 인가

### 1.1 Assistants API 의 한계 (Legacy)

- `Thread` / `Message` / `Run` / `Assistant` 4개 오브젝트로 개념 분산
- Run step 이 복잡하여 tool orchestration 어려움
- polling 기반 상태 확인 → 오래된 UX 패턴
- 월별 `api-version` 파라미터 관리 부담

### 1.2 Responses API v2 (2026 표준)

- **stable `/openai/v1/responses` 라우팅** — api-version 관리 불필요
- Terminology 재정비:

| Assistants (구) | Responses v2 (신) |
|---|---|
| Thread | **Conversation** |
| Message | **Item** (더 확장된 타입 시스템) |
| Run | **Response** |
| Assistant | **Agent Version** |

- Item 은 message · tool_call · tool_result · reasoning 등을 통합 표현
- Multi-agent, MCP, memory 를 first-class 로 지원

⚠️ **함정**: Google 검색 시 Assistants API 예제가 여전히 많이 나온다. **`beta.threads.*` 로 시작하는 코드는 legacy** — 무시하고 Responses API 사용.

---

## 2. Responses API v2 아키텍처

### 2.1 4대 오브젝트

- **Conversation**: 사용자와 agent 의 지속되는 대화 컨텍스트. Thread 대체.
- **Item**: 대화 안의 각 발화 · tool_call · tool_result · reasoning 스텝. 확장된 타입 시스템.
- **Response**: 한 번의 agent 실행 결과. Run 대체.
- **Agent Version**: 배포된 agent 스냅샷 (prompt + tools + model). 버전 관리 · A/B 테스트.

### 2.2 Agent 두 가지 타입

| 타입 | 실행 위치 | 장점 | 단점 | 사용 시점 |
|---|---|---|---|---|
| **Prompt Agents** | Foundry 완전 관리 | 인프라 관리 zero, 빠른 프로덕션 진입 | 커스텀 orchestration 제한 | 표준 챗봇, 지식 조회, 사내 assistant |
| **Hosted Agents** | 여러분의 코드 | 자유도 max, Semantic Kernel / LangGraph 조합 | 인프라 관리 필요 | 복잡한 multi-agent, custom logic, 특수 도메인 |

---

## 3. 🔧 실습 1 — Prompt Agent 최소 예제

Portal 대신 SDK 로 agent 를 정의·실행하는 완전 흐름.

```python
# src/foundry_app/prompt_agent.py
"""Prompt Agent - Foundry 완전 관리형."""
from __future__ import annotations
import os, time
from azure.identity import DefaultAzureCredential
from azure.ai.agents import AgentsClient
from azure.ai.agents.models import (
    Agent, MessageRole, RunStatus,
    CodeInterpreterToolDefinition, FileSearchToolDefinition,
)

agents = AgentsClient(
    endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    credential=DefaultAzureCredential(),
)

# 1) Agent 생성 (Agent Version 이 자동 v1 로 부여됨)
agent: Agent = agents.create_agent(
    model=os.environ["AZURE_FOUNDRY_DEPLOYMENT"],   # e.g. "gpt-5"
    name="data-analyst-agent",
    instructions=(
        "너는 데이터 분석가다. 사용자가 CSV/데이터에 대해 질문하면 code_interpreter "
        "로 파이썬을 실행하여 답한다. 그래프가 도움되면 그린다."
    ),
    tools=[
        CodeInterpreterToolDefinition(),
        # File Search 도 함께 (업로드된 파일 검색)
        FileSearchToolDefinition(),
    ],
)
print(f"Agent 생성: {agent.id}")

# 2) Conversation 시작 (이전엔 Thread)
conversation = agents.conversations.create()

# 3) 사용자 메시지 추가 (Item = user role)
agents.conversations.items.create(
    conversation_id=conversation.id,
    role=MessageRole.USER,
    content="1부터 100까지 소수 개수는 몇 개고, 그 소수들의 합은 얼마야?",
)

# 4) Response 실행 (이전엔 Run)
response = agents.conversations.responses.create_and_poll(
    conversation_id=conversation.id,
    agent_id=agent.id,
)

if response.status != RunStatus.COMPLETED:
    print(f"실패: {response.status} - {response.last_error}")
else:
    # 5) 마지막 assistant Item 읽기
    items = agents.conversations.items.list(
        conversation_id=conversation.id,
        order="desc",
        limit=1,
    )
    for item in items:
        if item.role == MessageRole.AGENT:
            for content in item.content:
                if content.type == "text":
                    print(content.text.value)
            break

# 6) 정리 (dev 환경)
agents.delete_agent(agent.id)
```

💡 **팁**:
- `create_and_poll` 은 SDK 가 자동으로 상태를 폴링하며 완료 대기. 수동 폴링 필요 없음.
- Production 에서는 agent 를 매번 create 하지 말고 미리 만들어 두고 ID 만 재사용.

⚠️ **함정**:
- Conversation 을 정리하지 않으면 storage 가 계속 증가. Standard Setup 사용 시 Cosmos DB 요금 발생.
- `code_interpreter` 는 **별도 종량 과금** — 프로덕션 비용 모니터링 필수 (Ch.10).

---

## 4. 🔧 실습 2 — Function 도구 추가

Ch.5 에서 배운 function calling 을 agent 안에서. Agent 가 자동으로 tool call → 결과 처리.

```python
# src/foundry_app/agent_with_function.py
"""Agent + Custom Function."""
from __future__ import annotations
import os, json
from azure.identity import DefaultAzureCredential
from azure.ai.agents import AgentsClient
from azure.ai.agents.models import FunctionToolDefinition, ToolResources, RunStatus

agents = AgentsClient(
    endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    credential=DefaultAzureCredential(),
)


def get_stock_price(ticker: str) -> dict:
    """실제로는 야후 파이낸스 API 호출. 여기선 mock."""
    prices = {"MSFT": 425.30, "AAPL": 192.50, "GOOGL": 178.20}
    return {"ticker": ticker, "price": prices.get(ticker, 0.0), "currency": "USD"}


TOOL_REGISTRY = {"get_stock_price": get_stock_price}


agent = agents.create_agent(
    model="gpt-5",
    name="stock-agent",
    instructions="너는 주식 정보 어시스턴트다. 시세는 get_stock_price 로 조회한다.",
    tools=[
        FunctionToolDefinition(
            name="get_stock_price",
            description="특정 티커의 현재 주가 조회.",
            parameters={
                "type": "object",
                "properties": {"ticker": {"type": "string", "description": "종목 티커 (예: MSFT)"}},
                "required": ["ticker"],
                "additionalProperties": False,
            },
        ),
    ],
)

conversation = agents.conversations.create()
agents.conversations.items.create(
    conversation_id=conversation.id,
    role="user",
    content="MSFT, AAPL 주가 알려줘.",
)

# Manual response loop - function tool 호출 처리
response = agents.conversations.responses.create(
    conversation_id=conversation.id, agent_id=agent.id,
)

while True:
    response = agents.conversations.responses.get(
        conversation_id=conversation.id, response_id=response.id,
    )
    if response.status == RunStatus.REQUIRES_ACTION:
        # 모델이 tool 호출을 요구함
        tool_outputs = []
        for tool_call in response.required_action.submit_tool_outputs.tool_calls:
            fn = TOOL_REGISTRY[tool_call.function.name]
            args = json.loads(tool_call.function.arguments)
            result = fn(**args)
            tool_outputs.append({"tool_call_id": tool_call.id, "output": json.dumps(result)})

        agents.conversations.responses.submit_tool_outputs(
            conversation_id=conversation.id,
            response_id=response.id,
            tool_outputs=tool_outputs,
        )
    elif response.status in (RunStatus.COMPLETED, RunStatus.FAILED, RunStatus.CANCELLED):
        break

# 결과 출력
items = agents.conversations.items.list(conversation_id=conversation.id, order="desc", limit=1)
for item in items:
    for content in item.content:
        if content.type == "text":
            print(content.text.value)
```

---

## 5. 🔧 실습 3 — MCP Tool 부착 (Ch.5 재사용)

Ch.5 에서 만든 CRM MCP 서버를 agent 에 붙인다. Function 을 하나씩 정의하는 것보다 **훨씬 유지보수 유리**.

```python
from azure.ai.agents.models import MCPToolDefinition

crm_mcp = MCPToolDefinition(
    server_label="crm",
    server_url="https://crm-mcp.internal.corp/sse",
    require_approval="never",   # sensitive ops 는 "always"
)

agent = agents.create_agent(
    model="gpt-5",
    name="customer-support",
    instructions="사내 CRM 조회·티켓 생성에 crm MCP 서버 사용.",
    tools=[crm_mcp],
)
```

MCP 서버가 노출한 모든 tool 이 자동으로 agent 에 등록됨. 새 tool 추가는 MCP 서버 코드만 수정 → agent 재정의 불필요.

---

## 6. Built-in Tools (2026 GA 목록)

| 툴 | 상태 | 설명 |
|---|---|---|
| **Code Interpreter** | ✅ GA | 격리된 Python sandbox. 데이터 분석·플로팅·수학 |
| **File Search** | ✅ GA | 업로드된 파일 (PDF/DOCX 등) 벡터 검색 |
| **Web Search** | ✅ GA | Bing 기반 실시간 웹 검색. ⚠️ **폐쇄망 정신 위배 주의** (Ch.12) |
| **Bing Grounding** | ✅ GA | Web Search 고급 버전. citation 강화 |
| **Azure AI Search Grounding** | ✅ GA | 여러분의 Search 인덱스 (Ch.6) 통합 |
| **Azure Functions** | ✅ GA | 사내 Function 호출 |
| **SharePoint Grounding** | 🔄 Preview | SharePoint 문서 (⚠️ delegated only) |
| **Microsoft Fabric** | 🔄 Preview | Fabric 데이터 에이전트 |
| **Image Generation** | 🔄 Preview | DALL-E 3 통합 |
| **Browser Automation** | 🔄 Preview | agent 가 브라우저 조작 |
| **Computer Use** | 🔄 Preview | agent 가 OS UI 조작 |
| **Memory** | 🔄 Preview | 세션 간 컨텍스트 유지 (2026-06) |
| **MCP Tools** | ✅ GA | 외부 MCP 서버 (Ch.5) |

⚠️ **함정**: Preview 툴은 SLA 없음. 프로덕션엔 GA 만.

---

## 7. Connected Agents (Multi-agent Orchestration)

Agent 하나로 부족한 경우. **각 agent 가 서로를 tool 로 호출**.

```python
# src/foundry_app/connected_agents.py
"""Multi-agent - routing agent + specialist agents."""
from azure.ai.agents.models import ConnectedAgentToolDefinition

# 1) Specialist agents 먼저 생성
sql_agent = agents.create_agent(
    model="gpt-5", name="sql-specialist",
    instructions="너는 SQL 전문가다. 자연어 → SQL 변환.",
)

finance_agent = agents.create_agent(
    model="gpt-5", name="finance-analyst",
    instructions="너는 재무 분석가다. 수치 해석 · 트렌드 분석.",
)

# 2) Routing agent 가 specialist 들을 tool 로 소환
router = agents.create_agent(
    model="gpt-5", name="router",
    instructions=(
        "너는 라우터다. 사용자 질문 성격에 따라 적절한 specialist 를 호출한다:\n"
        "- SQL/데이터 조회: call_sql_specialist\n"
        "- 재무/수치 분석: call_finance_analyst"
    ),
    tools=[
        ConnectedAgentToolDefinition(
            name="call_sql_specialist",
            description="자연어 → SQL 변환 및 실행",
            agent_id=sql_agent.id,
        ),
        ConnectedAgentToolDefinition(
            name="call_finance_analyst",
            description="재무 수치 분석 및 인사이트",
            agent_id=finance_agent.id,
        ),
    ],
)
```

⚠️ **함정**:
- **순환 참조 방지**: A → B → A → B ... 무한 loop 위험. instructions 에 명시적 종료 조건 필수.
- 각 agent 호출마다 별도 model 호출 = 비용 배수 증가. 필요한 경우에만 라우팅.

🔴 **2026 신규**: **Agent-to-Agent (A2A, preview)** — 서로 다른 Foundry Project 의 agent 간 discovery + call. **Routines (preview, 2026-06)** — 재사용 가능한 multi-step workflow 정의.

---

## 8. Semantic Kernel Python — Hosted Agent 대안

Semantic Kernel Python 은 활발히 개발 중 (Java 는 유지보수 모드). Plugin · Memory · Planner 를 프로그래머틱하게 구성.

### 8.1 최소 예제

```python
# src/foundry_app/sk_agent.py
"""Semantic Kernel Python + Foundry."""
import asyncio, os
from semantic_kernel import Kernel
from semantic_kernel.connectors.ai.open_ai import AzureChatCompletion
from semantic_kernel.agents import ChatCompletionAgent
from semantic_kernel.functions import kernel_function


class WeatherPlugin:
    @kernel_function(description="특정 도시의 현재 날씨 조회")
    def get_weather(self, city: str) -> str:
        # 실제로는 외부 API
        return f"{city}: 24°C, sunny"


async def main() -> None:
    kernel = Kernel()
    kernel.add_service(AzureChatCompletion(
        service_id="foundry",
        deployment_name=os.environ["AZURE_FOUNDRY_DEPLOYMENT"],
        endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
        api_key=os.environ["AZURE_FOUNDRY_KEY"],
    ))

    # Plugin 등록
    kernel.add_plugin(WeatherPlugin(), plugin_name="weather")

    # ChatCompletionAgent 생성
    agent = ChatCompletionAgent(
        kernel=kernel,
        service_id="foundry",
        name="WeatherBot",
        instructions="너는 날씨 어시스턴트다. weather 플러그인 사용.",
    )

    # 대화
    async for response in agent.invoke("서울 날씨 알려줘."):
        print(response.content)


if __name__ == "__main__":
    asyncio.run(main())
```

### 8.2 언제 SK 를 쓰나

- **복잡한 planner** (auto-planner, sequential planner) 가 필요
- **여러 LLM provider** 를 통합 (Azure + Anthropic + local)
- **Memory / Vector store 통합** 을 프로그래머틱 제어
- 오픈소스 프레임워크로 벤더 lock-in 회피

SK 는 Foundry Agent Service 를 대체하는 게 아니라 **보완**. Agent 인프라는 Foundry, orchestration 로직은 SK 조합이 실무 유리.

---

## 9. Agent Versioning (2026 신규)

Foundry Agent Service 는 각 agent 를 **immutable version** 으로 관리.

```python
# 새 버전 배포 (기존 v1 유지)
v2_agent = agents.create_agent_version(
    agent_id=existing_agent.id,
    instructions="v2: 톤을 더 친근하게. Emoji 사용 허용.",
    tools=existing_agent.tools,
    model="gpt-5",
)

# 트래픽 분할 (Canary - Ch.11 재소환)
agents.update_agent(
    agent_id=existing_agent.id,
    traffic_distribution={"v1": 90, "v2": 10},   # v2 에 10% 만
)
```

Ch.11 LLMOps 에서 이 패턴을 CI/CD 파이프라인과 결합.

---

## 10. Agent 상태 · 로그 · 정리

### 10.1 진행 중 Response 취소

```python
agents.conversations.responses.cancel(
    conversation_id=conversation.id,
    response_id=response.id,
)
```

### 10.2 Conversation 정리 (Storage 관리)

```python
# 오래된 conversation 삭제
conversations = agents.conversations.list(before="2026-06-01T00:00:00Z")
for conv in conversations:
    agents.conversations.delete(conv.id)
```

### 10.3 Trace 확인

Ch.9 의 `azure-monitor-opentelemetry` 로 자동 계측. Foundry Portal → Traces 에서 agent 실행 timeline · tool call · latency 확인.

---

## 11. 흔한 함정 (Trap Summary)

⚠️ 반복 강조:

1. **Assistants API 예제 사용 금지** — `beta.threads.*` 로 시작하는 코드는 legacy. Responses API v2 사용.
2. **Conversation cleanup 안 함** → Cosmos DB 스토리지 누적. 주기적 삭제 SOP 필수.
3. **Built-in tool 별도 과금** — Code Interpreter, Web Search 는 종량. Ch.10 비용 관리.
4. **Connected Agents 순환 참조** — A → B → A 무한 loop.
5. **Web Search 를 사내 챗봇에** — 검색어 public egress → 데이터 유출 벡터 (Ch.12).
6. **Preview tool 을 프로덕션에** — SLA 없음. Memory · Browser Automation 등.
7. **MCP tool 트러스트** — 인증 없는 MCP 는 프롬프트 인젝션 벡터.

💡 **베스트 프랙티스**:

- Agent 는 미리 만들어두고 ID 재사용 (매 요청 create 하지 말 것)
- **`require_approval="always"`** 를 sensitive tool 에 (메일 발송, 파일 삭제 등)
- Instructions 에 **negative constraint** ("절대 X 하지 마라") 명시
- Multi-agent 는 명확한 종료 조건 (예: "max 3 hops")
- Semantic Kernel + Foundry 조합으로 유연성 + 관리 편의성 병행

---

## 요약 (Cheat Sheet)

- **Responses API v2** = 2026 표준. Assistants API 는 legacy.
- **4대 오브젝트**: Conversation · Item · Response · Agent Version
- **Agent 타입**: Prompt (managed) vs Hosted (your code)
- **Built-in tools GA**: Code Interpreter, File Search, Web Search, Bing Grounding, Azure AI Search, Functions, MCP
- **Connected Agents** = agent-as-tool for multi-agent
- **Semantic Kernel Python** = orchestration 자유도 원할 때 조합
- **Agent Versioning** = 배포 immutable 스냅샷 + traffic splitting

## 📚 더 읽기

- [Foundry Agent Service Overview](https://learn.microsoft.com/en-us/azure/foundry/agents/overview)
- [azure-ai-agents Python SDK README](https://learn.microsoft.com/en-us/python/api/overview/azure/ai-agents-readme)
- [Responses API Reference](https://learn.microsoft.com/en-us/azure/ai-foundry/openai/api-version-lifecycle)
- [Connected Agents Guide](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/connected-agents)
- [MCP Tools in Agent Service](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/model-context-protocol)
- [Semantic Kernel Python Docs](https://learn.microsoft.com/en-us/semantic-kernel/overview/)
- [Agent Identity](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-identity)

## 다음 챕터

[Ch.8 Model Router & 배포 전략 →](Ch08_Model_Router.md)

---
