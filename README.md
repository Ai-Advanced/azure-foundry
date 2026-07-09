# Microsoft Foundry 실무 커리큘럼 (Python 버전)

> 📢 **명명 변경 안내 (2026-06)** — Microsoft가 **"Azure AI Foundry" → "Microsoft Foundry"** 로 리브랜딩했습니다.
> 포털 URL(**https://ai.azure.com**)은 그대로이고 기존 리소스도 그대로 동작합니다. 본 커리큘럼은 새 명명을 사용합니다.

> 🌿 **Git 브랜치 안내**: 이 커리큘럼은 두 언어 버전으로 제공됩니다.
> - `main` 브랜치 = **Java 버전** (v1.2 · 13챕터 · 531 KB)
> - `python` 브랜치 = **Python 버전** (v2.0 · **현재 브랜치**)
> - 전환: `git checkout main` / `git checkout python`

Microsoft Foundry(구 Azure AI Foundry)를 **처음 만지는 사람부터 프로덕션 배포까지** 이어지는 실무 심화 교재.
챕터당 5~10페이지, **개념 → 실습 코드 → 함정/베스트 프랙티스** 3단 구조.

---

## 대상 독자

- Azure OpenAI / Azure ML 경험이 없거나 얕은 개발자
- Foundry 기반 GenAI 서비스 실무 담당자 (백엔드 / 플랫폼 / 데이터 사이언스)
- MLOps · DevOps · 보안/거버넌스 담당자
- 사내 PoC → 프로덕션 이관을 준비 중인 팀

## 학습 방식

- **주 언어: Python 3.11+ (권장 3.13)** with **`uv`** 패키지 관리 (pyproject.toml PEP 621)
- **부 언어: REST curl** — SDK가 없는 환경/디버깅용
- **오케스트레이션 프레임워크**: LangChain (`langchain-azure-ai`), Semantic Kernel Python, LlamaIndex 중 선택
- 각 챕터는 앞 챕터를 전제로 함. 순서대로 읽는 것을 권장.

## Python SDK 스택 결정 (중요)

| 용도 | 패키지 | 상태 | 버전 |
|---|---|---|---|
| **Foundry 통합 (backbone)** | `azure-ai-projects` | ✅ **GA** | 2.3.0 |
| **Agents (Responses API v2)** | `azure-ai-agents` | ✅ GA | 1.1.0 |
| **Chat / LLM 클라이언트 (권장)** | `openai` (Azure 호환) | ✅ **GA** | 2.44.0 |
| **Evaluation (Python-native)** | `azure-ai-evaluation` | ✅ **GA** | 1.18.1 |
| **Content Safety** | `azure-ai-contentsafety` | ✅ GA | 1.0.0 |
| **AI Search (RAG)** | `azure-search-documents` | ✅ GA | 12.0.0 |
| **Entra ID 인증** | `azure-identity` | ✅ GA | 1.25.3 |
| **OpenTelemetry + App Insights** | `azure-monitor-opentelemetry` | ✅ GA | latest |
| **Prompt Flow** | `promptflow` | ✅ GA | 1.18.5 |
| **AI Inference (unified)** | `azure-ai-inference` | ⚠️ Beta | 1.0.0b9 |
| **MCP Python SDK** | `mcp` | ✅ Stable | 1.28.1 |
| **오케스트레이션 (LangChain)** | `langchain-azure-ai` | ✅ GA | 1.2.8+ |
| **오케스트레이션 (Semantic Kernel)** | `semantic-kernel` | ✅ **GA (활발)** | 1.43.1 |
| **RAG (LlamaIndex)** | `llama-index-llms-azure-openai` | ✅ GA | 0.5.5 |
| **Structured Outputs** | `pydantic` | ✅ GA | 2.13.4 |
| **OBO (Enterprise 인증)** | `msal` | ✅ GA | 1.28.x |
| **Microsoft Graph** | `msgraph-sdk` | ✅ GA | 1.x |
| **Teams Bot** | `botbuilder-core` + `botbuilder-integration-aiohttp` | ✅ GA | 4.x |
| **Slack** | `slack-bolt` | ✅ GA | 1.x |
| **GitHub API** | `pygithub` | ✅ GA | 2.x |

> 💡 **Python이 Java보다 우세한 지점** (2026-07 기준):
> - **`openai` 2.44.0 GA** — Foundry chat의 primary client (Java의 `azure-ai-openai`는 아직 beta.16)
> - **`azure-ai-evaluation` 1.18.1 GA** — 40+ evaluator (Java 사실상 없음)
> - **`promptflow` 1.18.5 GA** — DAG-based flow orchestration (Java 없음)
> - **`semantic-kernel` 1.43.1 GA** — 활발히 개발 중 (Java는 유지보수 모드)
> - **`langchain-azure-ai` 1.2.8+** — Foundry-native, Responses API 라우팅 포함

### Python 결손 부분 (알아둘 것)

- **MCP Python SDK**: v1.28.1 stable, **v2.0.0b1 pre-release** (Java v2.0.0 GA에 2주 lag). production은 v1.x 권장, v2 spec 확정 후 마이그레이션.
- **HTTP transport**: MCP Python은 STDIO + SSE 중심, Streamable HTTP는 preview.

## 실습 환경 준비물

| 항목 | 버전 / 요구사항 |
|---|---|
| Python | **3.11+ (권장 3.13, free-threaded 옵션)** |
| 패키지 관리 | **`uv`** (Astral, 10-100x faster than pip) — 대안: `pip` / `poetry` / `pdm` |
| 프로젝트 파일 | **`pyproject.toml`** (PEP 621) |
| IDE | VS Code + Python extension / PyCharm / Cursor |
| Azure 구독 | Free tier 가능. Foundry 리소스 배포 권한 필요 |
| Azure CLI | 2.60+ (`az login` 인증용) |
| Docker | Container Apps 배포 시 (Ch.11) |
| Git | 최신 |

**`uv` 설치** (Linux/macOS):
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**`uv` 설치** (Windows PowerShell):
```powershell
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Foundry 리소스 만드는 법은 **Ch.1** 참조.

---

## 커리큘럼

### 🟢 초급 (Beginner) — 기초 이해

| Ch | 제목 | 파일 |
|---|---|---|
| 1 | Microsoft Foundry 개요 & 시작 | [Ch01_Foundry_Overview.md](Ch01_Foundry_Overview.md) |
| 2 | 첫 모델 배포와 Playground (GPT-5/5.5 · Claude · Llama · Phi) | [Ch02_First_Deployment.md](Ch02_First_Deployment.md) |
| 3 | API 연동 기초 (Python `openai` + `azure-ai-projects` + REST) | [Ch03_API_Basics.md](Ch03_API_Basics.md) |

### 🟡 중급 (Intermediate) — 실무 응용

| Ch | 제목 | 파일 |
|---|---|---|
| 4 | Prompt Engineering & Structured Outputs (Pydantic v2) | [Ch04_Prompt_Engineering.md](Ch04_Prompt_Engineering.md) |
| 5 | Function Calling & Tool Use (**MCP `mcp` 1.28.1** + FastAPI) | [Ch05_Function_Calling.md](Ch05_Function_Calling.md) |
| 6 | RAG — Azure AI Search + LangChain/LlamaIndex | [Ch06_RAG_AI_Search.md](Ch06_RAG_AI_Search.md) |
| 7 | Foundry Agent Service (**Responses API v2** + Semantic Kernel Python) | [Ch07_Agent_Service.md](Ch07_Agent_Service.md) |

### 🔴 고급 (Advanced) — 프로덕션

| Ch | 제목 | 파일 |
|---|---|---|
| 8 | Model Router & 배포 전략 | [Ch08_Model_Router.md](Ch08_Model_Router.md) |
| 9 | Evaluation & Observability (`azure-ai-evaluation` **Python-native**) | [Ch09_Evaluation.md](Ch09_Evaluation.md) |
| 10 | 보안 · 거버넌스 · 비용 (Foundry Control Plane) | [Ch10_Security_Governance.md](Ch10_Security_Governance.md) |
| 11 | Production CI/CD & LLMOps (uv + Docker + GitHub Actions) | [Ch11_LLMOps.md](Ch11_LLMOps.md) |

### 🟣 엔터프라이즈 확장 (Enterprise Extension) — 폐쇄망 · 사내 툴 통합

| Ch | 제목 | 파일 |
|---|---|---|
| 12 | 폐쇄망 근접 배포 · 데이터 주권 · MS 계약 검증 (ZDR / CCC / Private Endpoint) | [Ch12_Enterprise_Deployment.md](Ch12_Enterprise_Deployment.md) |
| 13 | 사내 툴 통합 Enterprise Agent 실전 (`msal` OBO + `msgraph-sdk` + MCP Python + Bot Framework) | [Ch13_Enterprise_Agent_Integration.md](Ch13_Enterprise_Agent_Integration.md) |

---

## 📌 2026 주요 변경 반영 사항

| 영역 | 변경 내용 | 영향 챕터 |
|---|---|---|
| **명명** | Azure AI Foundry → **Microsoft Foundry** (2026-06) | 전체 |
| **Agent API** | Assistants API → **Responses API v2** | Ch.7, Ch.8 |
| **리소스 모델** | Hub-based Project (legacy) → **Foundry Project** | Ch.1 |
| **모델 카탈로그** | GPT-5.5 (2026-04), Claude in Foundry, Sora-2, Grok | Ch.2 |
| **MCP** | Agent Service **MCP 통합 GA**, `mcp` Python SDK 1.28.1 | Ch.5, Ch.7, Ch.13 |
| **배포 유형** | Global Standard/Provisioned/Batch / Data Zone / Regional / Developer | Ch.2, Ch.8 |
| **평가·관측** | `azure-ai-evaluation` **1.18.1 GA (Python)**, Continuous Eval, AI Red Teaming | Ch.9 |
| **거버넌스** | **Foundry Control Plane** GA (Defender/Purview 연동) | Ch.10, Ch.12 |
| **API 버저닝** | `/openai/v1/` stable routes | Ch.3, Ch.7 |
| **폐쇄망 배포** | Private Endpoint + VNet Injection + Managed VNet | Ch.12 |
| **사내 툴 통합** | Bot Framework Python + `msal` OBO + `mcp` custom server | Ch.13 |
| **⚠️ 정정: 유출 보상** | Customer Copyright Commitment는 IP 인덤니티, **데이터 유출 보상 아님** | Ch.12 |
| **Python 패키징** | `uv` + `pyproject.toml` (PEP 621) 표준화 | Ch.3, Ch.11 |

---

## 챕터 간 연결 (Learning Path)

```
Ch.1 개요 ─┬─▶ Ch.2 첫 배포 ─▶ Ch.3 API 연동 (Python)
           │                        │
           │                        ▼
           │              Ch.4 Prompt/Pydantic Structured Outputs
           │                        │
           │                        ▼
           │              Ch.5 Function Calling + MCP ──┐
           │                        │                    │
           │                        ▼                    ▼
           │              Ch.6 RAG (LangChain) ──▶ Ch.7 Agent Service (SK Python)
           │                                             │
           │                                             ▼
           └─────────────────▶ Ch.8 Model Router (배포 전략 종합)
                                                         │
                                                         ▼
                                          Ch.9 Evaluation (Python-native)
                                                         │
                                                         ▼
                                          Ch.10 Security · Governance · Cost
                                                         │
                                                         ▼
                                          Ch.11 Production CI/CD (uv + Docker)
                                                         │
                    ┌──────────────────────────────────────┤
                    │                                       │
                    ▼                                       ▼
       Ch.12 폐쇄망 근접 배포                Ch.13 Enterprise Agent 실전
       (ZDR · CCC · Private Endpoint)      (msal OBO · msgraph-sdk · MCP)
                    └─────────────────┬────────────────────┘
                                      │
                                      ▼
                          🎯 사내 배포 준비 완료
```

- **Ch.1~3**: 최소 실무 진입선. "Foundry에서 GPT-5로 뭐 하나 만들어봐" 해결.
- **Ch.4~7**: 사내 RAG 챗봇 / Responses API v2 에이전트 PoC 완결.
- **Ch.8~11**: 프로덕션 이관 · 다지역/다모델 · 평가 파이프라인 · IaC + CI/CD.
- **Ch.12~13 (Enterprise Extension)**: 폐쇄망/제한환경 배포 + 사내 툴(M365/Teams/Outlook/GitHub/Slack/ERP) 통합. **금융권·공공·엔터프라이즈 고객사용**.

---

## 진행 상황 (Python Version Delivery Checklist)

### 코어 커리큘럼 (Ch.1~11)
- [x] Ch.1 — Microsoft Foundry 개요 & 시작
- [x] Ch.2 — 첫 모델 배포와 Playground
- [x] Ch.3 — API 연동 기초 (Python + uv + openai)
- [x] Ch.4 — Prompt Engineering & Structured Outputs (Pydantic v2)
- [x] Ch.5 — Function Calling & Tool Use (MCP Python + FastMCP)
- [x] Ch.6 — RAG (Azure AI Search + LangChain + LlamaIndex)
- [x] Ch.7 — Agent Service (Responses API v2 + Semantic Kernel Python)
- [x] Ch.8 — Model Router & 배포 전략
- [x] Ch.9 — Evaluation & Observability (**Python-native · 40+ evaluator**)
- [x] Ch.10 — Security · Governance · Cost
- [x] Ch.11 — Production CI/CD & LLMOps (uv Docker + GitHub Actions)

### 엔터프라이즈 확장 (Ch.12~13)
- [x] Ch.12 — 폐쇄망 근접 배포 · 데이터 주권 · MS 계약 검증
- [x] Ch.13 — 사내 툴 통합 Enterprise Agent 실전 (msal + msgraph + MCP + Bot Framework Python)

**13개 챕터 전량 Python 변환 완료 (2026-07, v2.0)**

---

## 표기 규약

- **⚠️ 함정** : 실무에서 자주 밟는 지뢰
- **💡 팁** : 알아두면 시간 절약되는 요령
- **🔧 실습** : 손으로 따라 하는 hands-on
- **📚 더 읽기** : Microsoft Learn 원문 링크
- **🔴 2026 변경** : 최근 12개월 내 바뀐 부분 (구 자료와 다름)
- 코드 블록 상단에 `# pyproject.toml` / `# main.py` / `# bash` 등 파일/맥락 표기

## 주요 참고 URL

| 리소스 | URL |
|---|---|
| 포털 | https://ai.azure.com |
| Docs 허브 | https://learn.microsoft.com/en-us/azure/foundry/ |
| What's New | https://learn.microsoft.com/en-us/azure/foundry/whats-new-foundry |
| 모델 카탈로그 | https://learn.microsoft.com/en-us/azure/foundry/foundry-models/ |
| Agent Service | https://learn.microsoft.com/en-us/azure/foundry/agents/overview |
| Evaluation (Python) | https://learn.microsoft.com/en-us/python/api/overview/azure/ai-evaluation-readme |
| Python SDK 허브 | https://learn.microsoft.com/en-us/python/api/overview/azure/ |
| azure-ai-projects (Python) | https://learn.microsoft.com/en-us/python/api/overview/azure/ai-projects-readme |
| openai (PyPI) | https://pypi.org/project/openai/ |
| uv (패키지 관리) | https://docs.astral.sh/uv/ |
| MCP Python SDK | https://pypi.org/project/mcp/ |

## 버전 정보

- 커리큘럼 버전: **v2.0 (Python edition)** — Java v1.2 커리큘럼의 Python 포팅
- Git 브랜치: `python` (main = Java 원본)
- 기준 시점: **2026-07** (Foundry / Python SDK 최신 GA 반영)
- 상위 Java 버전과의 관계:
  - **동일**: 커리큘럼 구조, Foundry 개념, Bicep IaC, 아키텍처 다이어그램
  - **차이**: 언어 스택 전체 · Ch.9 대폭 단순화 (Python-native) · Ch.5 MCP SDK 버전 · Ch.13 인증/통합 라이브러리
- Python이 Java보다 유리한 이유: `openai` GA, `azure-ai-evaluation` GA, `promptflow` GA, `semantic-kernel` 활발 개발
</content>
