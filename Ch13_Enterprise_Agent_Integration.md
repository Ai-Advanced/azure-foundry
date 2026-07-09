# Chapter 13. 사내 툴 통합 Enterprise Agent 실전

[← 목차로](README.md)

> **학습 목표**
> - Foundry Agent 를 사내 툴(Teams, Outlook, SharePoint, GitHub, Slack, ERP)과 통합하는 3-tier 전략을 이해한다.
> - MSAL Java + OBO(On-Behalf-Of) 플로우로 사용자를 대신하여 사내 리소스에 접근하는 인증을 구현한다.
> - Custom MCP Server(Java)를 작성해 내부 REST API를 Foundry Agent 툴로 노출한다.
> - "메일 우선순위 매기기", "매일 답신 처리", "오늘 할 일 통합", "PR 리뷰 자동화" 같은 자연어 시나리오를 실제 코드로 구현한다.
> - Purview DLP 통합으로 사내 데이터 반출을 방지한다.

> **전제 조건**
> - [← Ch.5 Function Calling & Tool Use](Ch05_Function_Calling.md)
> - [← Ch.7 Foundry Agent Service](Ch07_Agent_Service.md)
> - [← Ch.12 폐쇄망 근접 배포](Ch12_Enterprise_Deployment.md) 완료 (VNet/Private Endpoint 셋업 완료)
> - Microsoft 365 라이선스 (Teams/Outlook/SharePoint 접근용)
> - Foundry Project 에서 OAuth Identity Passthrough 설정 가능한 권한

---

## 1. 왜 통합이 어려운가 — 그리고 어떤 3-tier 로 풀 것인가

사내 챗봇에서 "오늘 나에게 온 메일 중 중요한 것부터 우선순위 매겨줘" 를 처리하려면 세 가지가 동시에 필요하다.

1. **모델 (GPT-5)** — 이건 Ch.3~Ch.7까지 다 배웠다.
2. **사내 데이터 접근 (Outlook)** — 사용자의 mailbox 를 대신 읽으려면 **사용자 identity 로 delegated access** 필요.
3. **데이터가 외부로 새지 않는 보장** — Web Search 나 public MCP 로 프롬프트가 흘러가지 않도록 통제.

이걸 안정적으로 조합하는 방법이 **3-tier 통합 전략**이다.

| Tier | 방식 | 장점 | 단점 | 사용 시점 |
|---|---|---|---|---|
| **1** | Foundry **내장 툴** (File Search, Web Search, SharePoint) | 코드 zero, 빠른 시작 | SharePoint 는 delegated only, Web Search 는 public egress | 프로토타입, 사내 SharePoint 지식 검색 |
| **2** | **Custom MCP Server** (Java, VNet 내) | 유연, VNet egress, OAuth passthrough | 개발 필요 | **프로덕션 표준** — ERP·CRM·Outlook·Teams |
| **3** | **Custom Function** (Azure Functions, OpenAPI Tool) | 단일 함수 노출 간단 | 확장성 낮음 | 단발성 통합 (예: 특정 데이터베이스 조회) |

**결론**: Foundry Agent Service 로 사내 툴 통합을 진지하게 하려면 **Tier 2 (Custom MCP Server) 를 마스터**해야 한다. Ch.13의 대부분이 여기에 할애된다.

⚠️ **함정**: 관리자가 "간단하니 그냥 Custom Function 으로 다 해결" 이라 판단하는 경우가 흔한데, tool 이 5~10개를 넘어가면 각 함수마다 별도 인증·오류 처리·감사 로직이 파편화된다. MCP 는 **JSON-RPC 2.0 표준 스키마**로 tool 발견·인증·호출을 통일한다.

---

## 2. 인증 기초: 왜 OBO(On-Behalf-Of)가 핵심인가

Enterprise agent 인증의 핵심 원칙: **agent 는 자기 자신이 아니라 사용자를 대신하여 호출한다.**

### 2.1 Application vs Delegated Permission

Microsoft Graph API 권한은 두 종류.

| 종류 | 의미 | 예시 | 사내 agent 적합성 |
|---|---|---|---|
| **Application** | 앱 자체가 대신 (unattended) | `Mail.Read` (모든 사용자 mailbox 읽기) | ❌ 위험 — 관리자 승인 필요 + 감사 어려움 |
| **Delegated** | 로그인한 사용자로서 호출 | `Mail.Read` (내 mailbox 읽기) | ✅ 표준 — 각 사용자의 권한 그대로 |

**결론**: 사내 agent 는 항상 **Delegated 우선**. Application 권한은 "부서 전원의 메일 자동 분석" 같은 아주 좁은 시나리오에만.

### 2.2 OBO 플로우 그림

```
[Teams / Web UI]
    ↓ 사용자 로그인 (MSAL 팝업)
[Access Token: audience=Foundry, scope=user_impersonation]
    ↓
[Foundry Agent (Java 앱)]
    ↓ MSAL Java OBO 교환
[POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
 grant_type=on_behalf_of
 assertion={user_token}
 scope=https://graph.microsoft.com/.default]
    ↓
[Access Token: audience=Graph, delegated as user]
    ↓
[GET https://graph.microsoft.com/v1.0/me/messages]
    ↓
[사용자의 Inbox → agent가 프롬프트에 포함]
```

핵심: **user token 을 direct 로 Graph 에 보내지 않는다.** 반드시 **OBO 교환**을 통해 Graph audience 의 토큰으로 바꿔서 사용.

### 2.3 MSAL Java 로 OBO 구현

```xml
<!-- pom.xml (Ch.3 base에 추가) -->
<dependency>
  <groupId>com.microsoft.azure</groupId>
  <artifactId>msal4j</artifactId>
  <version>1.17.2</version>
</dependency>
<dependency>
  <groupId>com.microsoft.graph</groupId>
  <artifactId>microsoft-graph</artifactId>
  <version>6.28.0</version>
</dependency>
```

```java
// OboExchange.java
package com.example.enterprise;

import com.microsoft.aad.msal4j.*;
import java.util.Collections;
import java.util.Set;

public class OboExchange {

    private final ConfidentialClientApplication app;

    public OboExchange(String clientId, String clientSecret, String tenantId) throws Exception {
        this.app = ConfidentialClientApplication.builder(
                clientId,
                ClientCredentialFactory.createFromSecret(clientSecret))
            .authority("https://login.microsoftonline.com/" + tenantId)
            .build();
    }

    /**
     * 사용자 토큰을 Graph 접근용 토큰으로 교환.
     * @param userToken Foundry가 넘겨준 사용자 access token
     * @param scopes 요청할 Graph scope (예: "https://graph.microsoft.com/.default")
     */
    public String exchangeForGraphToken(String userToken, Set<String> scopes) throws Exception {
        OnBehalfOfParameters params = OnBehalfOfParameters
            .builder(scopes, new UserAssertion(userToken))
            .build();

        IAuthenticationResult result = app.acquireToken(params).get();
        return result.accessToken();
    }

    public static void main(String[] args) throws Exception {
        String userToken = System.getenv("USER_ACCESS_TOKEN");  // Foundry가 전달
        String clientId = System.getenv("APP_CLIENT_ID");
        String clientSecret = System.getenv("APP_CLIENT_SECRET");
        String tenantId = System.getenv("TENANT_ID");

        OboExchange obo = new OboExchange(clientId, clientSecret, tenantId);
        String graphToken = obo.exchangeForGraphToken(
            userToken,
            Collections.singleton("https://graph.microsoft.com/.default"));

        System.out.println("Graph token acquired (audience=graph, scope=delegated)");
    }
}
```

⚠️ **함정**: `clientSecret` 을 코드에 하드코딩 금지. 실제로는 **Federated Identity Credential (OIDC)** 를 써서 secret 없이 인증. Ch.11의 GitHub OIDC 파이프라인과 결합.

💡 **팁**: `msal4j` 는 토큰 캐시 내장. 같은 사용자에게 여러 번 OBO 하면 재사용된다. Refresh는 `offline_access` scope 포함 시 자동.

---

## 3. MCP Java SDK — Custom Tool Server 만들기

**MCP (Model Context Protocol)** 는 Anthropic 이 발표하고 MS가 채택한 tool exchange 표준. Foundry Agent Service 는 2026-06 부터 MCP GA.

### 3.1 SDK 추가

```xml
<!-- pom.xml -->
<dependency>
  <groupId>io.modelcontextprotocol.sdk</groupId>
  <artifactId>mcp</artifactId>
  <version>2.0.0</version>
</dependency>
<dependency>
  <groupId>io.modelcontextprotocol.sdk</groupId>
  <artifactId>mcp-json-jackson3</artifactId>
  <version>2.0.0</version>
</dependency>
<!-- Spring Boot 사용 시 -->
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>mcp-spring-webflux</artifactId>
  <version>2.0.0</version>
</dependency>
```

### 3.2 최소 MCP Server (Spring Boot)

```java
// McpServerConfig.java
package com.example.mcp;

import io.modelcontextprotocol.sdk.server.McpServer;
import io.modelcontextprotocol.sdk.server.spec.McpServerFeatures;
import io.modelcontextprotocol.spec.McpSchema.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;
import java.util.Map;

@Configuration
public class McpServerConfig {

    @Bean
    public McpServer mcpServer() {
        return McpServer.async()
            .serverInfo("erp-mcp-server", "1.0.0")
            .capabilities(ServerCapabilities.builder()
                .tools(true)   // tool 노출
                .logging()
                .build())
            .tools(List.of(
                new McpServerFeatures.AsyncToolSpecification(
                    new Tool("get_customer",
                             "고객 ID로 고객 정보를 조회한다",
                             """
                             {
                               "type": "object",
                               "properties": {
                                 "customerId": {"type": "string", "description": "고객 고유 ID"}
                               },
                               "required": ["customerId"]
                             }
                             """),
                    this::handleGetCustomer)
            ))
            .build();
    }

    private Mono<CallToolResult> handleGetCustomer(McpAsyncServerExchange exchange, Map<String, Object> args) {
        String customerId = (String) args.get("customerId");
        // 사내 ERP REST 호출 (구현 생략)
        String customerJson = erpClient.getCustomer(customerId);
        return Mono.just(new CallToolResult(
            List.of(new TextContent(customerJson)),
            false));
    }
}
```

### 3.3 Foundry Agent에 붙이기

Foundry Portal → Agent → Tools → **Add MCP Server** → URL 입력.

또는 Java 코드에서:
```java
// Agent 정의 시 MCP tool 등록
MCPTool erpTool = new MCPTool()
    .setServerUrl("https://mcp-erp.internal.corp/mcp")
    .setServerLabel("erp")
    .setProjectConnectionId(erpConnectionId);  // OAuth passthrough

PromptAgentDefinition agent = new PromptAgentDefinition("gpt-5")
    .setInstructions("ERP 시스템을 조회할 수 있다. 고객 정보 조회는 get_customer 툴 사용.")
    .setTools(List.of(erpTool));
```

⚠️ **함정**: MCP Server는 반드시 **HTTPS + Managed Identity** 인증. Public MCP endpoint 는 프롬프트 인젝션 벡터가 될 수 있다. VNet 내 Private Endpoint 로만 노출.

---

## 4. Teams 통합 — Bot Framework 프록시 패턴 (GA 권장)

Foundry의 **"Publish to Teams"** 는 아직 Early Access Preview 라 프로덕션 리스크가 있다. **Bot Framework SDK + Foundry backend 프록시**가 실전 표준.

### 4.1 아키텍처

```
Teams User → Message
    ↓
Azure Bot Service (Bot Framework endpoint)
    ↓ HTTP webhook
[Java Bot App - Spring Boot on Container Apps]
    ↓ 1. Teams SSO → user token 획득
    ↓ 2. OBO → Foundry scope token
    ↓ 3. Foundry Agent Service 호출 (Responses API v2)
    ↓ Agent → tools (Outlook MCP, ERP MCP, ...)
Response → Bot → Teams
```

### 4.2 pom.xml

```xml
<dependency>
  <groupId>com.microsoft.bot</groupId>
  <artifactId>bot-integration-spring</artifactId>
  <version>4.16.1</version>
</dependency>
<dependency>
  <groupId>com.microsoft.bot</groupId>
  <artifactId>bot-builder</artifactId>
  <version>4.16.1</version>
</dependency>
```

### 4.3 Handler 골격

```java
// FoundryTeamsBot.java
package com.example.bot;

import com.microsoft.bot.builder.*;
import com.microsoft.bot.schema.*;
import com.azure.ai.agents.AgentsClient;
import com.azure.ai.agents.AgentsClientBuilder;
import com.azure.identity.DefaultAzureCredentialBuilder;

import java.util.concurrent.CompletableFuture;

public class FoundryTeamsBot extends ActivityHandler {

    private final AgentsClient agentsClient;
    private final OboExchange obo;
    private final String agentId;

    public FoundryTeamsBot(String foundryEndpoint, String agentId,
                           OboExchange oboExchange) {
        this.agentsClient = new AgentsClientBuilder()
            .endpoint(foundryEndpoint)
            .credential(new DefaultAzureCredentialBuilder().build())
            .buildClient();
        this.agentId = agentId;
        this.obo = oboExchange;
    }

    @Override
    protected CompletableFuture<Void> onMessageActivity(TurnContext turnContext) {
        String userMessage = turnContext.getActivity().getText();
        String userAadObjectId = turnContext.getActivity().getFrom().getAadObjectId();
        // Teams SSO 로 얻은 사용자 토큰은 TokenExchangeState 를 통해 획득
        String userToken = getTeamsSsoToken(turnContext);

        try {
            // Foundry agent 호출 (사용자 컨텍스트 전달)
            // Responses API v2 사용
            String response = agentsClient.createResponse(agentId,
                new CreateResponseOptions()
                    .setInput(userMessage)
                    .setUserContext(userAadObjectId)  // 감사 로그용
                    .setDelegatedUserToken(userToken)); // MCP tools가 OBO에 사용

            return turnContext.sendActivity(MessageFactory.text(response))
                .thenApply(ra -> null);

        } catch (Exception e) {
            return turnContext.sendActivity(
                MessageFactory.text("에이전트 오류: " + e.getMessage())
            ).thenApply(ra -> null);
        }
    }

    private String getTeamsSsoToken(TurnContext ctx) {
        // Bot Framework Token Service → OAuth Connection Setting 활용
        // 실제 구현은 UserTokenClient.getUserToken() 호출
        return ctx.getTurnState().<String>get("USER_TOKEN");
    }
}
```

### 4.4 배포 순서

1. **Azure Bot Service** 리소스 생성 (Azure Portal)
2. Bot Service 에 **OAuth Connection Setting** 등록 (audience: Foundry)
3. Java 앱을 Container Apps 로 배포 (VNet integrated)
4. Bot Service 의 `messagingEndpoint` 를 앱 URL 로 설정
5. Teams App Studio 로 manifest 생성 후 조직 카탈로그 배포

💡 **팁**: Teams SSO 는 사용자에게 로그인 팝업을 한 번만 띄운다. 이후 refresh 자동. 이걸 안 쓰면 매 메시지마다 로그인 UX 파괴.

---

## 5. Outlook 통합 — Custom MCP + Graph API

Outlook 접근을 Custom MCP 로 wrapping. Foundry의 Agent 365 MCP (Frontier Preview) 를 기다리는 대신 지금 GA 로 만들자.

### 5.1 Outlook MCP Server (핵심 3개 tool)

```java
// OutlookMcpServer.java
package com.example.mcp.outlook;

import com.microsoft.graph.serviceclient.GraphServiceClient;
import com.microsoft.graph.models.*;
import io.modelcontextprotocol.sdk.server.spec.McpServerFeatures.AsyncToolSpecification;
import io.modelcontextprotocol.spec.McpSchema.*;
import com.example.enterprise.OboExchange;

import java.util.List;
import java.util.Map;
import java.util.Collections;

public class OutlookMcpServer {

    private final OboExchange obo;

    public OutlookMcpServer(OboExchange obo) {
        this.obo = obo;
    }

    // Tool 1: 오늘 받은 메일 목록 (우선순위 판단용 원자료)
    public Mono<CallToolResult> listTodaysMail(McpAsyncServerExchange ex, Map<String, Object> args) {
        String userToken = extractUserToken(ex);
        try {
            String graphToken = obo.exchangeForGraphToken(userToken,
                Collections.singleton("https://graph.microsoft.com/.default"));

            GraphServiceClient graph = createGraphClient(graphToken);
            var messages = graph.me().messages().get(req -> {
                req.queryParameters.filter = "receivedDateTime ge " + todayIso();
                req.queryParameters.top = 50;
                req.queryParameters.select = new String[]{
                    "id", "subject", "from", "receivedDateTime", "importance", "bodyPreview"
                };
            });

            String json = serializeMessages(messages);
            return Mono.just(new CallToolResult(
                List.of(new TextContent(json)), false));

        } catch (Exception e) {
            return Mono.just(new CallToolResult(
                List.of(new TextContent("Error: " + e.getMessage())), true));
        }
    }

    // Tool 2: 메일 회신 초안 저장 (Drafts 폴더에)
    public Mono<CallToolResult> saveDraftReply(McpAsyncServerExchange ex, Map<String, Object> args) {
        String userToken = extractUserToken(ex);
        String messageId = (String) args.get("messageId");
        String replyBody = (String) args.get("replyBody");

        try {
            String graphToken = obo.exchangeForGraphToken(userToken,
                Collections.singleton("https://graph.microsoft.com/.default"));
            GraphServiceClient graph = createGraphClient(graphToken);

            // createReply → draft 저장 (send 하지 않음)
            var draft = graph.me().messages().byMessageId(messageId)
                .createReply().post(new CreateReplyPostRequestBody());

            // draft body update
            Message update = new Message();
            ItemBody body = new ItemBody();
            body.setContentType(BodyType.Html);
            body.setContent(replyBody);
            update.setBody(body);

            graph.me().messages().byMessageId(draft.getId()).patch(update);

            return Mono.just(new CallToolResult(
                List.of(new TextContent("Draft saved: " + draft.getId())), false));

        } catch (Exception e) {
            return Mono.just(new CallToolResult(
                List.of(new TextContent("Error: " + e.getMessage())), true));
        }
    }

    // Tool 3: 메일 전송 (초안이 아니라 즉시 발송 - 정책 확인 후만 사용)
    // 프로덕션에서는 이 tool 을 사용자 명시적 승인 후에만 활성화
    public Mono<CallToolResult> sendMailImmediate(McpAsyncServerExchange ex, Map<String, Object> args) {
        // 구현 생략 - saveDraftReply 와 유사 + graph.me().sendMail() 호출
        // ⚠️ 사용자 UX에서 "정말 보낼까요?" confirmation 필수
        return Mono.empty();
    }

    // 헬퍼: MCP session context에서 user token 추출
    private String extractUserToken(McpAsyncServerExchange ex) {
        return (String) ex.getRequestContext().get("user-token");
    }

    private GraphServiceClient createGraphClient(String token) {
        return new GraphServiceClient(new SimpleAccessTokenProvider(token));
    }

    private String todayIso() {
        return java.time.LocalDate.now().atStartOfDay(java.time.ZoneOffset.UTC).toString();
    }
    private String serializeMessages(Object messages) { return "..."; /* Jackson */ }
}
```

### 5.2 Foundry Agent 정의

```java
PromptAgentDefinition outlookAgent = new PromptAgentDefinition("gpt-5")
    .setInstructions("""
        너는 사용자의 이메일 비서다. 오늘 받은 메일을 분석하고, 우선순위를 매기고,
        회신 초안을 작성한다. 규칙:
        - 즉시 발송(send_mail_immediate) 은 사용자가 명시적으로 "보내" 라고 확인할 때만.
        - 그 외에는 무조건 draft 저장(save_draft_reply).
        - 답장 톤은 사용자의 최근 5통 스타일에 맞춘다.
        """)
    .setTools(List.of(
        new MCPTool()
            .setServerUrl("https://mcp-outlook.internal.corp/mcp")
            .setServerLabel("outlook")
            .setProjectConnectionId(outlookConnId)
    ));
```

⚠️ **함정**: `send_mail_immediate` 를 agent가 자유롭게 부르게 두면 재난이다. 정책 프롬프트로 반드시 draft 우선. **UX 레벨에서 confirmation** 을 추가.

---

## 6. SharePoint 통합 — 제약과 대안

SharePoint 는 Foundry의 **built-in tool** 로 제공되지만 두 가지 큰 제약이 있다.

1. **Delegated 인증만 지원** — application (service principal) 로 접근 불가. Unattended agent 시나리오에서 사용 불가.
2. **"Publish to Teams" 로 배포된 agent 에서는 미작동** — Teams-published agent는 app identity 로 실행되므로 delegated 접근 불가.

### 6.1 Attended 시나리오는 그대로

Foundry Portal 에서 SharePoint Connection 을 프로젝트에 등록 후:

```java
SharepointPreviewTool sharepointTool = new SharepointPreviewTool(
    new SharepointGroundingToolParameters()
        .setProjectConnections(List.of(
            new ToolProjectConnection(sharepointConnectionId)
        )));

PromptAgentDefinition agent = new PromptAgentDefinition("gpt-5")
    .setInstructions("사내 SharePoint 문서를 검색하여 답변한다. 반드시 인용 출처 명시.")
    .setTools(List.of(sharepointTool));
```

지원 파일 포맷: `.docx`, `.pptx`, `.pdf`, `.aspx`, `.one` (텍스트만 추출, 이미지·차트 미지원).

### 6.2 Unattended 대안: OneDrive + Custom MCP

SharePoint 대신 OneDrive Business 파일을 Ch.6의 RAG 파이프라인으로 자체 인덱싱.

```
1. Graph API `/me/drive/items` 로 OneDrive 파일 다운로드 (delegated OBO)
2. Ch.6의 chunking + embedding 파이프라인 (Azure AI Search)
3. Agent 는 File Search / Azure AI Search tool 로 자체 인덱스 검색
```

이 방식은 **Azure AI Search 를 통해 VNet 내에서 완결**되므로 폐쇄망에도 적합.

⚠️ **함정**: OneDrive/SharePoint 문서의 **sensitivity label** 은 자체 인덱싱 시 그대로 딸려오지 않는다. Purview DLP 정책으로 별도 보호 필요 (섹션 10 참조).

---

## 7. GitHub 통합 — MCP Migration (Copilot Extensions Deprecated)

**결정적 변화**: GitHub Copilot Extensions (GitHub App 기반) 은 **2025-11-10 deprecated**. Foundry 에서 GitHub 접근은 이제 **MCP Server 방식이 표준**.

### 7.1 GitHub PR Review MCP Server

```java
// GithubPrMcpServer.java (핵심만)
package com.example.mcp.github;

import org.kohsuke.github.*;
import io.modelcontextprotocol.spec.McpSchema.*;

import java.util.List;
import java.util.Map;

public class GithubPrMcpServer {

    private final GitHub github;

    public GithubPrMcpServer(String pat) throws Exception {
        this.github = new GitHubBuilder().withOAuthToken(pat).build();
    }

    // Tool: PR diff 가져오기
    public Mono<CallToolResult> getPrDiff(McpAsyncServerExchange ex, Map<String, Object> args) {
        String owner = (String) args.get("owner");
        String repo = (String) args.get("repo");
        int prNumber = ((Number) args.get("prNumber")).intValue();

        try {
            GHPullRequest pr = github.getRepository(owner + "/" + repo).getPullRequest(prNumber);
            String diff = pr.getDiffUrl().openConnection().getInputStream()
                .readAllBytes().toString();

            return Mono.just(new CallToolResult(
                List.of(new TextContent(diff)), false));

        } catch (Exception e) {
            return Mono.just(new CallToolResult(
                List.of(new TextContent("Error: " + e.getMessage())), true));
        }
    }

    // Tool: PR 코멘트 작성 (리뷰 결과 등록)
    public Mono<CallToolResult> postReviewComment(McpAsyncServerExchange ex, Map<String, Object> args) {
        String owner = (String) args.get("owner");
        String repo = (String) args.get("repo");
        int prNumber = ((Number) args.get("prNumber")).intValue();
        String body = (String) args.get("body");

        try {
            GHPullRequest pr = github.getRepository(owner + "/" + repo).getPullRequest(prNumber);
            pr.comment(body);
            return Mono.just(new CallToolResult(
                List.of(new TextContent("Comment posted")), false));

        } catch (Exception e) {
            return Mono.just(new CallToolResult(
                List.of(new TextContent("Error: " + e.getMessage())), true));
        }
    }
}
```

### 7.2 Agent Prompt (PR Review Bot)

```
너는 시니어 자바 엔지니어 리뷰어다. GitHub PR을 리뷰한다.
프로세스:
1. get_pr_diff 로 diff 가져오기
2. diff 를 분석하고 다음 카테고리별 findings 작성:
   - Bug (잠재 버그, null 체크 누락 등)
   - Style (사내 자바 코딩 규약 위반)
   - Performance (N+1, 불필요한 iteration)
   - Security (SQL injection, XSS, hardcoded secret)
3. post_review_comment 로 findings 를 PR 에 코멘트 등록 (Markdown format)
4. 코멘트 톤: 건설적, 예시 코드 포함
```

💡 **팁**: PR review 는 지연이 허용되는 background task. **Global Batch API (Ch.8)** 를 활용하면 50% 비용 절감. GitHub Actions 로 야간 배치 실행.

---

## 8. Slack 통합 — Bolt SDK (Optional)

사내에 Slack 을 쓴다면.

```xml
<dependency>
  <groupId>com.slack.api</groupId>
  <artifactId>bolt</artifactId>
  <version>1.48.0</version>
</dependency>
<dependency>
  <groupId>com.slack.api</groupId>
  <artifactId>bolt-servlet</artifactId>
  <version>1.48.0</version>
</dependency>
```

```java
// SlackFoundryHandler.java
App slackApp = new App();

slackApp.command("/ask-agent", (req, ctx) -> {
    String question = req.getPayload().getText();
    String userId = req.getPayload().getUserId();

    // Foundry agent 호출
    String response = foundryClient.getAgent("slack-agent")
        .respond(question, Map.of("slackUserId", userId));

    return ctx.ack(response);
});

slackApp.event(MessageEvent.class, (payload, ctx) -> {
    // 채널 멘션 이벤트 처리
    if (payload.getEvent().getText().contains("<@" + botUserId + ">")) {
        // ... agent 호출 ...
    }
    return ctx.ack();
});
```

⚠️ **함정**: Slack 은 Enterprise Grid 가 아니면 SSO / Entra ID 통합이 약하다. 사용자 identity 매핑을 별도 관리해야 함 (Slack user_id ↔ Entra ID objectId).

---

## 9. 자연어 Use Case → 실제 구현 매핑

사용자님이 예시로 든 시나리오를 실제 agent 정의로 매핑.

### 9.1 "오늘 나에게 온 메일 중 중요한 것부터 우선순위 매겨줘"

**필요 tools**: Outlook MCP (`list_todays_mail`)

**Agent Instructions**:
```
너는 이메일 우선순위 매기기 비서다.
1. list_todays_mail 툴로 오늘 받은 메일 조회 (top=50).
2. 각 메일을 다음 기준으로 채점 (100점 만점):
   - CEO/임원 직속 (+40)
   - 회신 마감시간 명시 (+20)
   - "긴급" / "urgent" 키워드 (+15)
   - importance=high 플래그 (+10)
   - 초대 회신 필요 (+15)
3. 상위 10개를 markdown 표로 출력.
```

**호출 예시** (Java):
```java
String result = agent.respond("오늘 온 메일 중 중요한 것 위에서부터 알려줘");
// → Outlook MCP list_todays_mail 호출
// → GPT-5 채점 및 정렬
// → Markdown 결과 반환
```

### 9.2 "매일 답신 자동 처리해줘 (draft만 저장, 발송은 사용자 확인 후)"

**필요 tools**: Outlook MCP (`list_todays_mail`, `save_draft_reply`)

**Agent Instructions**:
```
너는 이메일 답신 초안 작성 비서다. 절대 send_mail_immediate 를 부르지 않는다.

프로세스:
1. list_todays_mail 로 미답신 메일 조회.
2. 각 메일에 대해 답신 필요 여부 판단:
   - 정보 공유용 메일은 skip
   - 질문 or 요청이 있으면 답신 필요
3. 답신 필요한 각 메일에 대해:
   - 최근 30일 사용자 답장 스타일 참조 (톤·서명·마무리 방식)
   - save_draft_reply 로 draft 저장
4. 결과 표로 요약: [발신자 | 제목 | 답신 초안 저장됨]
```

**정책 필터**: Purview DLP + Content Safety 로 답장에 사내 기밀 자동 감지 → 사용자 confirm 요구.

### 9.3 "오늘 내가 할 일 알려줘 (mail + slack + teams + outlook 통합)"

**필요 tools**: Outlook MCP, Teams MCP (custom), Slack MCP, Outlook Calendar MCP

**Toolbox 활용** — 여러 MCP 를 하나의 endpoint 로 묶어 agent 에 노출.

```java
// Toolbox 구성 (Foundry Portal Tools → Create Toolbox)
Toolbox unifiedToolbox = new Toolbox("daily-tasks-toolbox")
    .addMcpServer("outlook", outlookMcpUrl)
    .addMcpServer("teams", teamsMcpUrl)
    .addMcpServer("slack", slackMcpUrl)
    .addMcpServer("calendar", calendarMcpUrl);
```

**Agent Instructions**:
```
너는 사용자의 오늘 할 일을 통합 리스트로 만든다.

프로세스:
1. 병렬로 다음 tool 호출:
   - outlook.list_unreplied_mail (답신 대기 메일)
   - teams.list_mentions_today (오늘 나 멘션된 메시지)
   - slack.list_unread_priority (unread + 채널에서 나 멘션)
   - calendar.list_todays_meetings (오늘 미팅)
2. 각 항목을 소스/제목/우선순위/deadline 로 정규화
3. 마감시간 오름차순 정렬
4. Markdown 체크리스트로 출력. 각 항목 옆에 원본 링크.
```

**성능**: Parallel tool calls (Ch.5) 로 4개 툴 동시 실행 → 지연시간 최소.

### 9.4 "코드 만들면 팀원들의 agent bot 에 review 받게 PR 만들어줘"

**필요 tools**: GitHub MCP (`get_pr_diff`, `post_review_comment`, `create_pr`, `assign_reviewers`)

**전체 워크플로**:
```
1. 사용자: "이 코드 PR 로 올리고 review bot 붙여줘"
2. Agent → github.create_pr (branch 자동 감지, title/body 생성)
3. Agent → github.assign_reviewers (팀 관례 lookup)
4. Agent → 팀의 review-bot Agent 를 A2A(Agent-to-Agent) 로 호출
   - review-bot 은 get_pr_diff 로 diff 분석
   - 사내 자바 규약 checklist 자동 체크
   - post_review_comment 로 결과 등록
5. Agent → 사용자에게 PR URL + review 예정 알림
```

**Connected Agents (Ch.7 재소환)** 로 여러 agent 를 연쇄. review-bot 은 사내 공용 asset.

### 9.5 "Teams 팀 메시지에 대해서 알아서 잘 답변해줘"

**필요 tools**: Teams MCP (`list_channel_messages`, `post_reply`), Outlook MCP (문맥 참조용)

**⚠️ 이건 위험도 높음** — agent 가 사용자를 대신해 팀 채팅에 답을 남긴다. 정책:

```
Agent Instructions:
1. 답변 자동화 대상은 명시적으로 화이트리스트 한 채널만 (예: #dev-team-questions)
2. 다음 케이스만 답변:
   - 사용자에게 직접 멘션된 질문
   - 사용자가 도메인 전문가인 주제 (Foundry Prompts CMS에 등록)
3. 다음 케이스는 draft 만 남기고 사용자 review 요청:
   - 다른 팀 담당자 언급된 답변
   - 결정/승인이 필요한 요청
   - 감정적/민감한 대화
4. 매 답변 후 Purview DSPM audit log 자동 생성
```

**작동 방식**: Bot Framework 프록시 (섹션 4) + Teams SDK 로 채널 메시지 이벤트 구독. Foundry agent 가 답변 생성 후 post_reply.

---

## 10. Purview DLP 통합 — 사내 데이터 반출 방지

Ch.10에서 개념만 다뤘던 Purview 를 실전 시나리오에.

### 10.1 지원 기능 (GA 2026-06)

- **Inline prompt DLP**: 프롬프트가 agent 에 도달하기 전 스캔 → 정책 위반 시 차단
- **Sensitivity Label 인식**: SharePoint/OneDrive 문서의 Confidential 라벨 자동 감지
- **DSPM for AI Activity Explorer**: 사용자별 agent 사용 감사

### 10.2 정책 예시 (Purview 콘솔에서 설정)

```
Policy 1: 결제 정보 반출 방지
- Condition: 프롬프트 or 답변에 credit card number pattern
- Action: Block + user notification
- Scope: 전체 Foundry Project

Policy 2: 사내 소스코드 반출 방지
- Condition: 답변에 특정 코드 패턴 (@Company annotation, com.corp.* package)
- Action: Alert admin + audit log
- Scope: 개발자용 agent

Policy 3: PII 반출 방지
- Condition: 프롬프트에 주민번호/전화번호/이메일 등 PII
- Action: Redact + audit
- Scope: 고객 상담 agent
```

### 10.3 Java 앱 관측

DSPM audit log는 KQL 로 App Insights 에서 조회:
```kusto
customEvents
| where name == "PurviewDlpViolation"
| where timestamp > ago(24h)
| project timestamp, userAadObjectId, policy, action, redactedContent
| order by timestamp desc
```

⚠️ **함정**: Purview DLP 는 **프롬프트에 대해서만 강제 enforce**. Agent 의 **응답(output)** 에는 알림만 발생하고 자동 차단 안 됨. 응답 검증은 Content Safety + custom filter 로 별도 구현 필요.

---

## 11. 운영 · 보안 실무 팁

**전사 배포 시 실무 세부:**

- **Agent-per-team pattern**: 각 팀별 Foundry Project 분리. Marketing 팀 agent 가 실수로 Engineering 데이터를 못 보게.
- **Session sticky**: Multi-turn 대화가 캐노리 배포 (Ch.11) 사이를 왔다갔다 하지 않도록 Conversation ID 를 버전 pin.
- **Token rotation**: MSAL refresh token 실패 시 사용자를 재로그인 유도 (Teams SSO 자동 처리).
- **Kill switch**: Foundry Portal 에서 특정 agent 즉시 disable 가능. Incident 대응 SOP 에 포함.
- **User consent UI**: MCP tool 이 "메일 발송/파일 삭제/PR 머지" 등 write 액션 수행 시 반드시 사용자 UI confirmation.
- **Audit retention**: 금감원 요구 = 5년. App Insights 는 기본 90일 → Log Analytics workspace 로 export + Storage 로 archival.

---

## 12. 🔧 실습 요약: 사내 통합 Agent 최소 배포

Ch.13 내용 종합 실습.

**시나리오**: "메일 우선순위 + Teams 챗봇" agent

**구성 요소:**
1. Foundry Project (Ch.12의 폐쇄망 근접 셋업)
2. Outlook MCP Server (섹션 5) → Container Apps 배포 (VNet integrated)
3. Foundry Agent 정의 (Instruction: 섹션 9.1)
4. Bot Framework Bot (섹션 4) → Container Apps 배포
5. Azure Bot Service 리소스 → Teams 채널 연결
6. Purview DLP 정책 (섹션 10.2) 활성

**배포 순서:**
```bash
# 1. Ch.12 Bicep 스택 배포 (Foundry + BYOS + Private Endpoint)
az deployment group create --template-file foundry-private.bicep ...

# 2. Outlook MCP 서버 배포 (Maven → Docker → ACR → Container Apps)
mvn package
docker build -t $ACR/outlook-mcp:1.0.0 .
docker push $ACR/outlook-mcp:1.0.0
az containerapp create --name outlook-mcp --image $ACR/outlook-mcp:1.0.0 \
   --environment $CAE --vnet-configuration ...

# 3. Foundry Agent 생성 (Java 코드 or Portal)
java -jar create-agent.jar

# 4. Bot Framework Bot 배포 (동일 파이프라인)
mvn package && docker build && push && containerapp create

# 5. Azure Bot Service 리소스 생성 + Teams 채널 연결 (Portal)

# 6. Purview 정책 활성 (Purview compliance portal)
```

---

## 요약 (Cheat Sheet)

- **3-tier 통합 전략**: Foundry 내장 (Tier 1) · Custom MCP (Tier 2, 프로덕션 표준) · Custom Function (Tier 3, 단발성).
- **인증 원칙**: Delegated + OBO 가 표준. Application permission 은 예외적일 때만.
- **MSAL Java (`msal4j`)** + **MCP Java SDK (`io.modelcontextprotocol.sdk:mcp` 2.0.0)** 조합이 커스텀 통합의 두 축.
- **Teams**: Bot Framework 프록시가 GA. Foundry "Publish to Teams" 는 Preview 라 프로덕션 조심.
- **SharePoint**: Delegated only, Teams-published agent 에선 미작동. Unattended 는 OneDrive + 자체 RAG 로.
- **GitHub**: Copilot Extensions deprecated. MCP server 가 새 표준.
- **자연어 use case → agent instruction + MCP tools 조합**으로 대부분 시나리오 커버.
- **Purview DLP** = prompt-level enforcement. 응답 검증은 별도 (Content Safety + custom).
- **Session sticky · kill switch · audit retention 5년** 은 운영 필수.

## 📚 더 읽기

- [Foundry Agent Service Overview](https://learn.microsoft.com/en-us/azure/foundry/agents/overview)
- [MCP Java SDK (io.modelcontextprotocol)](https://java.sdk.modelcontextprotocol.io/latest/)
- [MSAL for Java (OBO Flow)](https://learn.microsoft.com/en-us/entra/identity-platform/msal-java)
- [Microsoft Graph API — Mail](https://learn.microsoft.com/en-us/graph/api/resources/message)
- [Bot Framework SDK for Java](https://github.com/microsoft/botbuilder-java)
- [Teams SSO with Bot Framework](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/authentication/auth-aad-sso-bots)
- [Foundry SharePoint Tool](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/sharepoint)
- [Foundry MCP Authentication Guide](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/mcp-authentication)
- [GitHub Copilot Extensions Deprecation Notice](https://github.blog/changelog/2025-09-24-deprecate-github-copilot-extensions-github-apps/)
- [Slack Bolt for Java](https://docs.slack.dev/tools/java-slack-sdk/)
- [Microsoft Purview DLP for AI Agents](https://learn.microsoft.com/en-us/purview/ai-agents)
- [Toolbox for MCP Aggregation](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/toolbox)
- [junwoojeong100/microsoft-foundry-labs (참고 워크샵)](https://github.com/junwoojeong100/microsoft-foundry-labs)

## 다음 단계

Ch.13 은 **커리큘럼의 응용편 종착점**이다. 실제 배포에 들어가면:

1. **PoC (2-4주)**: 한 팀 · 한 use case (예: 메일 우선순위) 로 검증
2. **Beta (1-2개월)**: 팀 내부 배포, Purview DLP 실전 검증
3. **GA (3-6개월)**: 조직 전체 롤아웃, Foundry Control Plane 으로 fleet 관리
4. **지속 개선**: Ch.9 Continuous Evaluation + Ch.11 CI/CD 파이프라인 적용

**Foundry 는 매주 신기능이 나온다** — Foundry MCP Server (preview), Agent-to-Agent (A2A) preview, Routines preview 등. 본 커리큘럼을 뼈대로 삼고 [What's New](https://learn.microsoft.com/en-us/azure/foundry/whats-new-foundry) 를 정기 모니터링하며 확장.

[← 목차로](README.md)

---
