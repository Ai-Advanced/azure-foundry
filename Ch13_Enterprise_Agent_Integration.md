# Chapter 13. 사내 툴 통합 Enterprise Agent 실전 (Python)

[← 목차로](README.md)

> **학습 목표**
> - Foundry Agent 를 사내 툴(Teams · Outlook · SharePoint · GitHub · Slack · ERP) 과 통합하는 3-tier 전략을 이해한다.
> - `msal` Python + OBO(On-Behalf-Of) 플로우로 사용자를 대신 사내 리소스에 접근한다.
> - `msgraph-sdk` Python 으로 Outlook 메일 · Teams · SharePoint 를 다룬다.
> - MCP Python (Ch.5) 로 사내 ERP · CRM 을 agent tool 로 노출한다.
> - Bot Framework Python 으로 Teams 프록시 봇을 구축한다.
> - Slack Bolt Python 으로 Slack 통합.
> - "메일 우선순위 매기기", "매일 답신 처리", "오늘 할 일 통합", "PR 리뷰 자동화" 자연어 시나리오를 Python 으로 구현한다.
> - Purview DLP 로 사내 데이터 반출 방지.

> **전제 조건**
> - [← Ch.5 Function Calling & MCP](Ch05_Function_Calling.md), [← Ch.7 Agent Service](Ch07_Agent_Service.md), [← Ch.12 폐쇄망 배포](Ch12_Enterprise_Deployment.md) 완료
> - Microsoft 365 라이선스 (Teams/Outlook/SharePoint 접근)
> - Foundry Project OAuth Identity Passthrough 설정 권한
> - `uv add "msal>=1.28" "msgraph-sdk>=1.0" "botbuilder-core>=4.16" "botbuilder-integration-aiohttp>=4.16" "slack-bolt>=1.20" "pygithub>=2.4" "aiohttp>=3.10"`

---

## 1. 왜 통합이 어려운가 — 그리고 3-tier 로 푸는 방법

사내 챗봇에서 "오늘 나에게 온 메일 중 중요한 것부터 우선순위 매겨줘" 를 처리하려면 셋이 동시에 필요:

1. **모델 (GPT-5)** — Ch.3-Ch.7 완료
2. **사내 데이터 접근 (Outlook)** — 사용자 mailbox 를 대신 읽으려면 **사용자 identity 로 delegated access**
3. **데이터 외부 유출 방지** — Web Search, public MCP 로 프롬프트 흘러가지 않게 통제

**3-tier 통합 전략**:

| Tier | 방식 | 장점 | 단점 | 사용 시점 |
|---|---|---|---|---|
| **1** | Foundry **내장 툴** (File Search, Web Search, SharePoint) | 코드 zero, 빠른 시작 | SharePoint 는 delegated only, Web Search 는 public egress | 프로토타입, SharePoint 지식 검색 |
| **2** | **Custom MCP Server** (Python, VNet 내) | 유연, VNet egress, OAuth passthrough | 개발 필요 | **프로덕션 표준** — ERP · CRM · Outlook · Teams |
| **3** | **Custom Function** (Azure Functions, OpenAPI Tool) | 단일 함수 노출 간단 | 확장성 낮음 | 단발성 통합 (예: 특정 DB 조회) |

**결론**: Foundry Agent Service 로 사내 툴 통합을 진지하게 하려면 **Tier 2 (Custom MCP Server) 를 마스터**. Python 은 `mcp` 1.28.1 + FastMCP 로 매우 빠르게 구축 가능 (Ch.5 재소환).

⚠️ **함정**: 관리자가 "간단하니 그냥 Custom Function 으로 다 해결" 이라 판단하는 경우 흔한데, tool 이 5-10개를 넘어가면 각 함수마다 별도 인증·오류 처리·감사 로직이 파편화됨. MCP 는 **JSON-RPC 2.0 표준 스키마**로 tool 발견·인증·호출 통일.

---

## 2. 인증 기초: OBO(On-Behalf-Of) 가 왜 핵심인가

Enterprise agent 인증 핵심: **agent 는 자기 자신이 아니라 사용자를 대신하여 호출한다.**

### 2.1 Application vs Delegated Permission

| 종류 | 의미 | 예시 | 사내 agent 적합성 |
|---|---|---|---|
| **Application** | 앱이 대신 (unattended) | `Mail.Read` = 모든 사용자 mailbox | ❌ 위험 — 관리자 승인 + 감사 어려움 |
| **Delegated** | 로그인 사용자로서 호출 | `Mail.Read` = 내 mailbox 만 | ✅ 표준 — 각 사용자 권한 그대로 |

**결론**: 사내 agent 는 항상 **Delegated 우선**. Application 은 "부서 전원 메일 자동 분석" 같은 특수 시나리오만.

### 2.2 OBO 플로우 그림

```
[Teams / Web UI]
    ↓ 사용자 로그인 (MSAL 팝업)
[Access Token: audience=Foundry, scope=user_impersonation]
    ↓
[Foundry Agent (Python 앱)]
    ↓ MSAL Python OBO 교환
[POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
 grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
 assertion={user_token}
 requested_token_use=on_behalf_of
 scope=https://graph.microsoft.com/.default]
    ↓
[Access Token: audience=Graph, delegated as user]
    ↓
[GET https://graph.microsoft.com/v1.0/me/messages]
    ↓
[사용자 Inbox → agent 가 프롬프트에 포함]
```

핵심: **user token 을 direct 로 Graph 에 보내지 않음.** 반드시 **OBO 교환** 을 통해 Graph audience 의 토큰으로 바꿔 사용.

### 2.3 MSAL Python 으로 OBO 구현

```python
# src/foundry_app/enterprise/obo.py
"""On-Behalf-Of 토큰 교환 헬퍼."""
from __future__ import annotations
import os
from msal import ConfidentialClientApplication


class OboExchange:
    """User token → downstream API 토큰 교환."""

    def __init__(self, client_id: str, client_secret: str, tenant_id: str) -> None:
        self._app = ConfidentialClientApplication(
            client_id=client_id,
            client_credential=client_secret,
            authority=f"https://login.microsoftonline.com/{tenant_id}",
        )

    def exchange(self, user_token: str, scopes: list[str]) -> str:
        """OBO flow: user_token → downstream API token.

        Args:
            user_token: Foundry 로부터 받은 사용자 access token
            scopes: 요청할 downstream scope (예: ["https://graph.microsoft.com/.default"])

        Returns:
            downstream API 로 사용할 access token
        """
        result = self._app.acquire_token_on_behalf_of(
            user_assertion=user_token,
            scopes=scopes,
        )
        if "error" in result:
            raise RuntimeError(f"OBO 실패: {result.get('error_description')}")
        return result["access_token"]


# 사용
if __name__ == "__main__":
    obo = OboExchange(
        client_id=os.environ["APP_CLIENT_ID"],
        client_secret=os.environ["APP_CLIENT_SECRET"],
        tenant_id=os.environ["TENANT_ID"],
    )
    user_token = os.environ["USER_ACCESS_TOKEN"]  # Foundry 가 전달
    graph_token = obo.exchange(
        user_token,
        scopes=["https://graph.microsoft.com/.default"],
    )
    print(f"Graph token acquired (delegated)")
```

⚠️ **함정**:
- `client_secret` 하드코딩 금지. **Federated Identity Credential (OIDC)** 로 secret 없이 인증 (Ch.11 GitHub OIDC 와 결합).
- MSAL 은 토큰 캐시 내장. 같은 사용자 OBO 반복 호출 시 재사용.
- Refresh 는 `offline_access` scope 포함 시 자동.

💡 **팁**: 프로덕션은 Managed Identity + Federated Credential 조합. `ConfidentialClientApplication` 대신 `ManagedIdentityClient` 사용 가능.

---

## 3. MCP Python — Custom Tool Server (Ch.5 재소환)

Ch.5 에서 CRM MCP Server 를 만들었다. Enterprise 에서는 **Outlook · Teams · GitHub · Slack · ERP 각각을 MCP 서버로 노출**. 공통 pattern:

```python
# 기본 형태
from mcp.server.fastmcp import FastMCP

mcp = FastMCP(name="my-service", version="1.0.0")

@mcp.tool()
async def some_tool(param: str) -> dict:
    """도구 설명 - agent 에게 보이는 지시서."""
    # 실제 구현
    return {"result": "..."}

if __name__ == "__main__":
    mcp.run(transport="sse", host="0.0.0.0", port=8000)
```

이제 각 사내 시스템별 실제 MCP 서버를 만들어본다.

---

## 4. Teams 통합 — Bot Framework Python 프록시 (권장)

Foundry 의 **"Publish to Teams"** 는 아직 Early Access Preview. **Bot Framework SDK + Foundry backend 프록시** 가 실전 표준.

### 4.1 아키텍처

```
Teams User → Message
    ↓
Azure Bot Service (Bot Framework endpoint)
    ↓ HTTP webhook
[Python Bot App - aiohttp on Container Apps]
    ↓ 1. Teams SSO → user token 획득
    ↓ 2. OBO → Foundry scope token
    ↓ 3. Foundry Agent Service 호출 (Responses API v2)
    ↓ Agent → tools (Outlook MCP, ERP MCP, ...)
Response → Bot → Teams
```

### 4.2 Bot Handler 골격

```python
# src/foundry_app/enterprise/teams_bot.py
"""Bot Framework Python + Foundry Agent 프록시."""
from __future__ import annotations
import os
from botbuilder.core import ActivityHandler, TurnContext, MessageFactory
from botbuilder.schema import Activity
from azure.identity import DefaultAzureCredential
from azure.ai.agents import AgentsClient
from foundry_app.enterprise.obo import OboExchange


class FoundryTeamsBot(ActivityHandler):
    def __init__(self, agent_id: str) -> None:
        super().__init__()
        self._agents = AgentsClient(
            endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
            credential=DefaultAzureCredential(),
        )
        self._agent_id = agent_id
        self._obo = OboExchange(
            client_id=os.environ["APP_CLIENT_ID"],
            client_secret=os.environ["APP_CLIENT_SECRET"],
            tenant_id=os.environ["TENANT_ID"],
        )

    async def on_message_activity(self, turn_context: TurnContext) -> None:
        user_message = turn_context.activity.text
        user_aad_id = turn_context.activity.from_property.aad_object_id

        # Teams SSO 로 사용자 토큰 획득 (OAuthPrompt 별도 구현 필요)
        user_token = await self._get_sso_token(turn_context)

        # Foundry Agent 에게 사용자 컨텍스트 전달
        try:
            conversation = self._agents.conversations.create()
            self._agents.conversations.items.create(
                conversation_id=conversation.id,
                role="user",
                content=user_message,
                metadata={
                    "user_aad_id": user_aad_id,
                    "user_token": user_token,   # MCP tool 들이 OBO 에 사용
                },
            )
            response = self._agents.conversations.responses.create_and_poll(
                conversation_id=conversation.id,
                agent_id=self._agent_id,
            )

            # 응답 추출
            items = self._agents.conversations.items.list(
                conversation_id=conversation.id,
                order="desc", limit=1,
            )
            for item in items:
                for content in item.content:
                    if content.type == "text":
                        await turn_context.send_activity(MessageFactory.text(content.text.value))
                        return
        except Exception as e:
            await turn_context.send_activity(
                MessageFactory.text(f"에이전트 오류: {e}"),
            )

    async def _get_sso_token(self, turn_context: TurnContext) -> str:
        """Teams SSO 로 사용자 access token 획득.

        실제 구현: botbuilder-dialogs 의 OAuthPrompt 사용.
        여기선 상태에서 캐시된 토큰 반환 가정.
        """
        return turn_context.turn_state.get("USER_TOKEN", "")
```

### 4.3 aiohttp 서버 + Bot Adapter

```python
# src/foundry_app/enterprise/teams_server.py
"""aiohttp 로 Bot Framework 웹훅 서빙."""
from aiohttp import web
from botbuilder.core import BotFrameworkAdapter, BotFrameworkAdapterSettings
from botbuilder.schema import Activity
from foundry_app.enterprise.teams_bot import FoundryTeamsBot

import os

adapter_settings = BotFrameworkAdapterSettings(
    app_id=os.environ["MICROSOFT_APP_ID"],
    app_password=os.environ["MICROSOFT_APP_PASSWORD"],
)
adapter = BotFrameworkAdapter(adapter_settings)
bot = FoundryTeamsBot(agent_id=os.environ["FOUNDRY_AGENT_ID"])


async def messages(req: web.Request) -> web.Response:
    body = await req.json()
    activity = Activity().deserialize(body)
    auth_header = req.headers.get("Authorization", "")

    async def call_bot(turn_context):
        await bot.on_turn(turn_context)

    await adapter.process_activity(activity, auth_header, call_bot)
    return web.Response(status=200)


app = web.Application()
app.router.add_post("/api/messages", messages)


if __name__ == "__main__":
    web.run_app(app, host="0.0.0.0", port=3978)
```

### 4.4 배포 순서

1. **Azure Bot Service** 리소스 생성 (Portal)
2. Bot Service → **OAuth Connection Setting** 등록 (audience: Foundry)
3. Python 앱 Container Apps 배포 (VNet integrated)
4. Bot Service `messagingEndpoint` 를 앱 URL 로 설정
5. Teams App Studio 로 manifest 생성 → 조직 카탈로그 배포

💡 **팁**: Teams SSO 는 사용자에게 로그인 팝업을 한 번만. 이후 refresh 자동. 이걸 안 쓰면 매 메시지마다 로그인 UX 파괴.

---

## 5. Outlook 통합 — Custom MCP + Graph API

Agent 365 MCP (Frontier Preview) 대신 지금 GA 로 만드는 pattern.

### 5.1 Outlook MCP Server

```python
# src/outlook_mcp_server/server.py
"""Outlook Graph API 를 MCP tool 로 노출."""
from __future__ import annotations
import os
from datetime import datetime, timezone
from mcp.server.fastmcp import FastMCP
from msgraph import GraphServiceClient
from azure.core.credentials import AccessToken, TokenCredential


class StaticTokenCredential(TokenCredential):
    """이미 얻은 access_token 을 GraphServiceClient 에 넘기는 helper."""
    def __init__(self, token: str) -> None:
        self._token = token

    def get_token(self, *scopes: str, **kwargs) -> AccessToken:
        # expires_on 은 대략 60분 후 (실제로는 OBO 결과의 expires_in 사용)
        return AccessToken(self._token, int(datetime.now(timezone.utc).timestamp()) + 3600)


mcp = FastMCP(name="outlook-server", version="1.0.0",
              instructions="사용자 Outlook 메일 조회·답장·전송 도구")


def _graph_client_for_user(user_token: str) -> GraphServiceClient:
    return GraphServiceClient(credentials=StaticTokenCredential(user_token))


@mcp.tool()
async def list_todays_mail(user_token: str, top: int = 50) -> list[dict]:
    """오늘 받은 이메일 목록 조회 (우선순위 판단용 원자료).

    Args:
        user_token: OBO 로 획득한 Graph token
        top: 최대 조회 수 (기본 50)

    Returns:
        메일 dict list (id, subject, from, receivedDateTime, importance, bodyPreview)
    """
    graph = _graph_client_for_user(user_token)
    today = datetime.now(timezone.utc).replace(hour=0, minute=0, second=0, microsecond=0).isoformat()

    messages = await graph.me.messages.get(
        request_configuration=lambda cfg: (
            setattr(cfg.query_parameters, "filter", f"receivedDateTime ge {today}"),
            setattr(cfg.query_parameters, "top", top),
            setattr(cfg.query_parameters, "select", [
                "id", "subject", "from", "receivedDateTime", "importance", "bodyPreview",
            ]),
        ),
    )
    return [
        {
            "id": m.id,
            "subject": m.subject,
            "from": m.from_.email_address.address if m.from_ else "",
            "received": m.received_date_time.isoformat() if m.received_date_time else "",
            "importance": m.importance.value if m.importance else "normal",
            "preview": m.body_preview,
        }
        for m in (messages.value or [])
    ]


@mcp.tool()
async def save_draft_reply(user_token: str, message_id: str, reply_body: str) -> dict:
    """메일에 대한 답장 초안 저장 (Drafts 폴더). 실제 발송하지 않음.

    Args:
        user_token: OBO Graph token
        message_id: 답장할 원본 메일 ID
        reply_body: 답장 내용 (HTML 가능)
    """
    from msgraph.generated.users.item.messages.item.create_reply.create_reply_post_request_body import (
        CreateReplyPostRequestBody,
    )
    from msgraph.generated.models.item_body import ItemBody
    from msgraph.generated.models.body_type import BodyType
    from msgraph.generated.models.message import Message

    graph = _graph_client_for_user(user_token)

    # createReply → draft 생성
    draft = await graph.me.messages.by_message_id(message_id).create_reply.post(
        body=CreateReplyPostRequestBody()
    )

    # draft body 업데이트
    update = Message(body=ItemBody(content_type=BodyType.Html, content=reply_body))
    await graph.me.messages.by_message_id(draft.id).patch(body=update)

    return {"draft_id": draft.id, "status": "saved"}


@mcp.tool()
async def send_mail_immediate(user_token: str, message_id: str, reply_body: str) -> dict:
    """즉시 메일 발송 (⚠️ 사용자 명시 승인 후만 사용).

    Args:
        user_token: OBO Graph token
        message_id: 답장할 원본 메일 ID
        reply_body: 답장 내용
    """
    from msgraph.generated.users.item.messages.item.reply.reply_post_request_body import (
        ReplyPostRequestBody,
    )
    from msgraph.generated.models.message import Message
    from msgraph.generated.models.item_body import ItemBody
    from msgraph.generated.models.body_type import BodyType

    graph = _graph_client_for_user(user_token)
    await graph.me.messages.by_message_id(message_id).reply.post(body=ReplyPostRequestBody(
        message=Message(body=ItemBody(content_type=BodyType.Html, content=reply_body)),
    ))
    return {"status": "sent"}


if __name__ == "__main__":
    mcp.run(transport="sse", host="0.0.0.0", port=8001)
```

### 5.2 Foundry Agent 에 부착

```python
# src/foundry_app/enterprise/outlook_agent.py
from azure.ai.agents import AgentsClient
from azure.ai.agents.models import MCPToolDefinition, PromptAgentDefinition
from azure.identity import DefaultAzureCredential
import os

agents = AgentsClient(
    endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    credential=DefaultAzureCredential(),
)

outlook_mcp = MCPToolDefinition(
    server_label="outlook",
    server_url="https://outlook-mcp.internal.corp/sse",
    project_connection_id=os.environ["OUTLOOK_CONN_ID"],   # OAuth passthrough
)

agent = agents.create_agent(
    model="gpt-5",
    name="email-assistant",
    instructions=(
        "너는 사용자의 이메일 비서다. 오늘 받은 메일을 분석·우선순위 매기고 답장 초안 작성.\n"
        "규칙:\n"
        "- 즉시 발송(send_mail_immediate) 은 사용자가 명시적으로 '보내' 라고 확인할 때만.\n"
        "- 그 외에는 무조건 draft 저장(save_draft_reply).\n"
        "- 답장 톤은 사용자 최근 5통 스타일에 맞춘다."
    ),
    tools=[outlook_mcp],
)
```

⚠️ **함정**: `send_mail_immediate` 를 agent 가 자유롭게 부르게 두면 재난. 정책 프롬프트로 draft 우선 + UX 레벨 confirmation 강제.

---

## 6. SharePoint 통합 — 제약과 대안

SharePoint 는 Foundry 의 **built-in tool** 이나 두 가지 큰 제약:

1. **Delegated 인증만 지원** — application (service principal) 로 접근 불가. Unattended agent 시나리오 불가.
2. **"Publish to Teams" 로 배포된 agent 에서 미작동** — Teams-published agent 는 app identity 로 실행 → delegated 접근 불가.

### 6.1 Attended 시나리오

```python
from azure.ai.agents.models import SharepointToolDefinition

sharepoint_tool = SharepointToolDefinition(
    connection_id=os.environ["SHAREPOINT_CONN_ID"],
)

agent = agents.create_agent(
    model="gpt-5",
    name="sharepoint-agent",
    instructions="사내 SharePoint 문서 검색하여 답변. 반드시 인용 출처 명시.",
    tools=[sharepoint_tool],
)
```

지원 파일: `.docx`, `.pptx`, `.pdf`, `.aspx`, `.one` (텍스트만, 이미지/차트 미지원).

### 6.2 Unattended 대안: OneDrive + 자체 RAG (Ch.6 재소환)

- Graph API `/me/drive/items` 로 OneDrive 파일 다운로드 (delegated OBO)
- Ch.6 chunking + embedding 파이프라인 (Azure AI Search)
- Agent 는 Azure AI Search tool 로 자체 인덱스 검색

이 방식은 **Azure AI Search 통해 VNet 내 완결** → 폐쇄망 적합.

⚠️ **함정**: OneDrive/SharePoint 문서의 **sensitivity label** 은 자체 인덱싱 시 딸려오지 않음. Purview DLP 정책으로 별도 보호 (섹션 10).

---

## 7. GitHub 통합 — MCP Migration

**결정적 변화**: GitHub Copilot Extensions (GitHub App 기반) 은 **2025-11-10 deprecated**. Foundry 에서 GitHub 접근은 이제 **MCP Server 방식** 표준.

### 7.1 GitHub PR Review MCP Server

```python
# src/github_mcp_server/server.py
"""GitHub API 를 MCP tool 로 노출 - PR 리뷰 자동화용."""
from __future__ import annotations
import os
import httpx
from github import Github
from mcp.server.fastmcp import FastMCP

mcp = FastMCP(name="github-server", version="1.0.0",
              instructions="GitHub PR 리뷰 · 이슈 관리")

gh = Github(os.environ["GITHUB_PAT"])


@mcp.tool()
async def get_pr_diff(owner: str, repo: str, pr_number: int) -> str:
    """PR 의 diff 원본 가져오기."""
    pr = gh.get_repo(f"{owner}/{repo}").get_pull(pr_number)
    async with httpx.AsyncClient() as client:
        r = await client.get(
            pr.diff_url,
            headers={"Authorization": f"token {os.environ['GITHUB_PAT']}"},
        )
        return r.text


@mcp.tool()
async def post_review_comment(owner: str, repo: str, pr_number: int, body: str) -> dict:
    """PR 에 리뷰 코멘트 게시.

    Args:
        body: Markdown 지원. 리뷰 findings 로 사용.
    """
    pr = gh.get_repo(f"{owner}/{repo}").get_pull(pr_number)
    comment = pr.create_issue_comment(body)
    return {"comment_id": comment.id, "url": comment.html_url}


@mcp.tool()
async def assign_reviewers(owner: str, repo: str, pr_number: int, reviewers: list[str]) -> dict:
    """PR 에 리뷰어 지정.

    Args:
        reviewers: GitHub username list
    """
    pr = gh.get_repo(f"{owner}/{repo}").get_pull(pr_number)
    pr.create_review_request(reviewers=reviewers)
    return {"assigned": reviewers}


if __name__ == "__main__":
    mcp.run(transport="sse", host="0.0.0.0", port=8002)
```

### 7.2 PR Review Agent Prompt

```
너는 시니어 파이썬 엔지니어 리뷰어다. GitHub PR 을 리뷰한다.
프로세스:
1. get_pr_diff 로 diff 가져오기
2. diff 를 분석하고 다음 카테고리별 findings 작성:
   - Bug (잠재 버그, None 체크 누락 등)
   - Style (사내 파이썬 코딩 규약 위반, PEP 8)
   - Performance (N+1 쿼리, 불필요한 iteration)
   - Security (SQL injection, hardcoded secret)
   - Type (type hint 누락, Any 남용)
3. post_review_comment 로 findings 를 PR 에 등록 (Markdown)
4. 코멘트 톤: 건설적, 예시 코드 포함
```

💡 **팁**: PR review 는 지연 허용 background task. **Global Batch API (Ch.8)** 로 50% 비용 절감. GitHub Actions 로 야간 배치 실행.

---

## 8. Slack 통합 — Bolt Python

```python
# src/slack_bot/app.py
"""Slack Bolt Python + Foundry Agent."""
from __future__ import annotations
import os
from slack_bolt import App
from slack_bolt.adapter.socket_mode import SocketModeHandler
from azure.ai.agents import AgentsClient
from azure.identity import DefaultAzureCredential

app = App(
    token=os.environ["SLACK_BOT_TOKEN"],
    signing_secret=os.environ["SLACK_SIGNING_SECRET"],
)

agents = AgentsClient(
    endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    credential=DefaultAzureCredential(),
)
AGENT_ID = os.environ["FOUNDRY_AGENT_ID"]


@app.command("/ask-agent")
def handle_ask(ack, respond, command):
    """/ask-agent 명령."""
    ack()
    question = command["text"]
    user_id = command["user_id"]

    # Foundry Agent 호출
    conv = agents.conversations.create()
    agents.conversations.items.create(
        conversation_id=conv.id,
        role="user",
        content=question,
        metadata={"slack_user_id": user_id},
    )
    response = agents.conversations.responses.create_and_poll(
        conversation_id=conv.id, agent_id=AGENT_ID,
    )

    items = agents.conversations.items.list(conversation_id=conv.id, order="desc", limit=1)
    for item in items:
        for content in item.content:
            if content.type == "text":
                respond(content.text.value)
                return


@app.event("app_mention")
def handle_mention(event, say):
    """@botname 언급 처리."""
    text = event["text"]
    # ... agent 호출 로직 (위와 유사) ...


if __name__ == "__main__":
    SocketModeHandler(app, os.environ["SLACK_APP_TOKEN"]).start()
```

⚠️ **함정**: Slack 은 Enterprise Grid 아니면 SSO / Entra ID 통합 약함. 사용자 identity 매핑 별도 관리 (Slack user_id ↔ Entra ID objectId).

---

## 9. 자연어 Use Case → 실제 구현 매핑

사용자님이 예시로 든 시나리오를 실제 agent 정의로.

### 9.1 "오늘 나에게 온 메일 중 중요한 것부터 우선순위 매겨줘"

**필요 tools**: Outlook MCP (`list_todays_mail`)

**Agent Instructions**:
```
너는 이메일 우선순위 매기기 비서다.
1. list_todays_mail 로 오늘 받은 메일 조회 (top=50).
2. 각 메일을 다음 기준으로 채점 (100점 만점):
   - CEO/임원 직속 (+40)
   - 회신 마감시간 명시 (+20)
   - "긴급" / "urgent" 키워드 (+15)
   - importance=high 플래그 (+10)
   - 초대 회신 필요 (+15)
3. 상위 10개를 Markdown 표로 출력.
```

### 9.2 "매일 답신 자동 처리해줘 (draft만 저장)"

**필요 tools**: Outlook MCP (`list_todays_mail`, `save_draft_reply`)

**Agent Instructions**:
```
너는 이메일 답신 초안 작성 비서다. 절대 send_mail_immediate 를 부르지 않는다.

프로세스:
1. list_todays_mail 로 미답신 메일 조회.
2. 각 메일 답신 필요 여부 판단:
   - 정보 공유용 메일은 skip
   - 질문 or 요청 있으면 답신 필요
3. 답신 필요한 메일 각각:
   - 최근 30일 사용자 답장 스타일 참조 (톤 · 서명 · 마무리)
   - save_draft_reply 로 draft 저장
4. 결과 표: [발신자 | 제목 | 답신 초안 저장됨]
```

**정책 필터**: Purview DLP + Content Safety 로 답장에 사내 기밀 자동 감지 → 사용자 confirm 요구.

### 9.3 "오늘 내가 할 일 알려줘 (mail + slack + teams + calendar 통합)"

**필요 tools**: Outlook MCP, Teams MCP (custom), Slack MCP, Calendar MCP

**Toolbox 활용** — 여러 MCP 를 하나로.

**Agent Instructions**:
```
너는 사용자의 오늘 할 일 통합 리스트를 만든다.

프로세스:
1. 병렬로 다음 tool 호출:
   - outlook.list_unreplied_mail (답신 대기)
   - teams.list_mentions_today (오늘 나 멘션)
   - slack.list_unread_priority (unread + 채널 멘션)
   - calendar.list_todays_meetings (오늘 미팅)
2. 각 항목 정규화: [소스 | 제목 | 우선순위 | deadline]
3. 마감시간 오름차순 정렬
4. Markdown 체크리스트로 출력. 각 항목에 원본 링크.
```

**성능**: Parallel tool calls (Ch.5) 로 4개 툴 동시 실행 → 지연 최소.

### 9.4 "코드 만들면 팀원 agent bot 에 review 받게 PR 만들어줘"

**필요 tools**: GitHub MCP (`get_pr_diff`, `post_review_comment`, `create_pr`, `assign_reviewers`)

**전체 워크플로**:
```
1. 사용자: "이 코드 PR 로 올리고 review bot 붙여줘"
2. Agent → github.create_pr (branch 자동 감지, title/body 생성)
3. Agent → github.assign_reviewers (팀 관례 lookup)
4. Agent → 팀의 review-bot Agent 를 Connected Agent (Ch.7) 로 호출
   - review-bot 은 get_pr_diff → diff 분석
   - 사내 파이썬 규약 checklist 자동 체크
   - post_review_comment 로 결과 등록
5. Agent → 사용자에게 PR URL + review 예정 알림
```

**Connected Agents (Ch.7 재소환)** 로 여러 agent 연쇄. review-bot 은 사내 공용 asset.

### 9.5 "Teams 팀 메시지에 대해서 알아서 잘 답변해줘"

**필요 tools**: Teams MCP (`list_channel_messages`, `post_reply`), Outlook MCP (문맥 참조)

⚠️ **위험도 높음** — agent 가 사용자 대신 팀 채팅 답변. 정책:

```
Agent Instructions:
1. 답변 자동화 대상은 명시적 화이트리스트 채널만 (예: #dev-team-questions)
2. 답변 조건:
   - 사용자 직접 멘션된 질문
   - 사용자 도메인 전문가 주제 (Foundry Prompts CMS 등록)
3. Draft 만 남기고 사용자 review 요청하는 케이스:
   - 다른 팀 담당자 언급된 답변
   - 결정/승인 필요 요청
   - 감정적/민감 대화
4. 매 답변 후 Purview DSPM audit log 자동 생성
```

**작동 방식**: Bot Framework 프록시 (섹션 4) + Teams SDK 로 채널 메시지 이벤트 구독. Foundry agent 답변 생성 후 post_reply.

---

## 10. Purview DLP 통합 — 사내 데이터 반출 방지

Ch.10 재소환.

### 10.1 지원 기능 (GA 2026-06)

- **Inline prompt DLP**: 프롬프트가 agent 도달 전 스캔 → 위반 시 차단
- **Sensitivity Label 인식**: SharePoint/OneDrive Confidential 라벨 자동 감지
- **DSPM for AI Activity Explorer**: 사용자별 agent 사용 감사

### 10.2 정책 예시

```
Policy 1: 결제 정보 반출 방지
- Condition: 프롬프트 or 답변에 credit card pattern
- Action: Block + user notification
- Scope: 전체 Foundry Project

Policy 2: 사내 소스코드 반출
- Condition: 답변에 특정 코드 패턴 (@Company decorator, foundry_app.* module)
- Action: Alert admin + audit log
- Scope: 개발자용 agent

Policy 3: PII 반출 방지
- Condition: 프롬프트에 주민번호/전화/이메일
- Action: Redact + audit
- Scope: 고객 상담 agent
```

### 10.3 Python 관측

DSPM audit log 는 KQL 로 App Insights 조회:

```kusto
customEvents
| where name == "PurviewDlpViolation"
| where timestamp > ago(24h)
| project timestamp, userAadObjectId, policy, action, redactedContent
| order by timestamp desc
```

⚠️ **함정**: Purview DLP 는 **프롬프트만 강제 enforce**. Agent **응답(output)** 은 alert 만, 자동 차단 안 됨. 응답 검증은 Content Safety + custom filter 로 별도.

---

## 11. 운영 · 보안 실무

**전사 배포 세부:**

- **Agent-per-team pattern**: 팀별 Foundry Project 분리. Marketing 팀 agent 가 Engineering 데이터 못 보게.
- **Session sticky**: Multi-turn 대화가 캐노리 배포 (Ch.11) 사이 왔다갔다 하지 않게 Conversation ID 를 버전 pin.
- **Token rotation**: MSAL refresh token 실패 시 사용자 재로그인 유도 (Teams SSO 자동).
- **Kill switch**: Foundry Portal 에서 특정 agent 즉시 disable. Incident 대응 SOP.
- **User consent UI**: MCP tool 이 "메일 발송 / 파일 삭제 / PR 머지" 등 write 액션 시 반드시 사용자 UI confirmation.
- **Audit retention**: 금감원 5년. App Insights 기본 90일 → Log Analytics workspace export + Storage archival.

---

## 12. 🔧 실습 요약: 사내 통합 Agent 최소 배포

Ch.13 종합 실습.

**시나리오**: "메일 우선순위 + Teams 챗봇" agent

**구성 요소**:
1. Foundry Project (Ch.12 폐쇄망 근접 셋업)
2. Outlook MCP Server (섹션 5) → Container Apps (VNet integrated)
3. Foundry Agent 정의 (섹션 9.1 Instructions)
4. Bot Framework Python 앱 (섹션 4) → Container Apps
5. Azure Bot Service 리소스 → Teams 채널 연결
6. Purview DLP 정책 (섹션 10.2) 활성

**배포 순서**:

```bash
# 1. Ch.12 Bicep 배포 (Foundry + BYOS + Private Endpoint)
az deployment group create --template-file infra/foundry-private.bicep ...

# 2. Outlook MCP 서버 배포 (uv + Docker + ACR + Container Apps)
uv sync
docker build -t $ACR/outlook-mcp:1.0.0 -f Dockerfile.mcp .
docker push $ACR/outlook-mcp:1.0.0
az containerapp create --name outlook-mcp --image $ACR/outlook-mcp:1.0.0 \
   --environment $CAE --vnet-configuration ...

# 3. Foundry Agent 생성 (Python 코드 or Portal)
uv run python -m foundry_app.enterprise.create_agent

# 4. Bot Framework Bot 배포 (동일 파이프라인)
docker build -t $ACR/teams-bot:1.0.0 -f Dockerfile.bot .
docker push $ACR/teams-bot:1.0.0
az containerapp create --name teams-bot ...

# 5. Azure Bot Service 리소스 + Teams 채널 (Portal)

# 6. Purview 정책 활성 (Purview compliance portal)
```

---

## 요약 (Cheat Sheet)

- **3-tier 통합**: Foundry 내장 (Tier 1) · Custom MCP (Tier 2, 프로덕션 표준) · Custom Function (Tier 3, 단발성)
- **인증 원칙**: Delegated + OBO 표준. Application permission 은 예외
- **`msal` Python** + **`mcp` Python 1.28.1 + FastMCP** = 커스텀 통합의 두 축
- **Teams**: Bot Framework Python 프록시 (GA). Foundry "Publish to Teams" 는 Preview
- **SharePoint**: Delegated only, Teams-published agent 미작동. Unattended 는 OneDrive + 자체 RAG
- **GitHub**: Copilot Extensions deprecated. MCP server 새 표준
- **Slack**: `slack-bolt` Python
- **자연어 use case** → agent instruction + MCP tools 조합
- **Purview DLP** = prompt-level enforcement. 응답 검증 별도
- **운영**: Session sticky · Kill switch · Audit 5년

## 📚 더 읽기

- [Foundry Agent Service Overview](https://learn.microsoft.com/en-us/azure/foundry/agents/overview)
- [`mcp` Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [MSAL Python (OBO Flow)](https://learn.microsoft.com/en-us/entra/identity-platform/msal-python)
- [Microsoft Graph Python SDK](https://learn.microsoft.com/en-us/graph/sdks/sdks-overview)
- [Bot Framework Python](https://github.com/microsoft/botbuilder-python)
- [Teams SSO with Bot Framework](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/authentication/auth-aad-sso-bots)
- [Foundry SharePoint Tool](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/sharepoint)
- [Foundry MCP Authentication](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/mcp-authentication)
- [GitHub Copilot Extensions Deprecation](https://github.blog/changelog/2025-09-24-deprecate-github-copilot-extensions-github-apps/)
- [Slack Bolt Python](https://slack.dev/bolt-python/tutorial/getting-started)
- [Microsoft Purview DLP for AI](https://learn.microsoft.com/en-us/purview/ai-agents)
- [Toolbox for MCP Aggregation](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/toolbox)

## 다음 단계

Ch.13 은 커리큘럼 응용편 종착점이다. 실제 배포에 들어가면:

1. **PoC (2-4주)**: 한 팀 · 한 use case (예: 메일 우선순위) 로 검증
2. **Beta (1-2개월)**: 팀 내부 배포, Purview DLP 실전 검증
3. **GA (3-6개월)**: 조직 전체 롤아웃, Foundry Control Plane 으로 fleet 관리
4. **지속 개선**: Ch.9 Continuous Evaluation + Ch.11 CI/CD 적용

**Foundry 는 매주 신기능** — Foundry MCP Server (preview), Agent-to-Agent (A2A) preview, Routines preview 등. 본 커리큘럼을 뼈대로 [What's New](https://learn.microsoft.com/en-us/azure/foundry/whats-new-foundry) 정기 모니터링 + 확장.

[← 목차로](README.md)

---
