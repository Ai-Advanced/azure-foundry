# Chapter 9. Evaluation & Observability (Python-native)

[← 목차로](README.md)

> **학습 목표**
> - LLM 서비스의 품질을 정량 측정하는 방법을 이해한다.
> - `azure-ai-evaluation` 1.18.1 (Python-native) 로 40+ evaluator 를 실전 사용한다.
> - Custom evaluator (LLM-as-judge · 코드 기반) 를 Pydantic + async 로 작성한다.
> - Continuous Evaluation 으로 프로덕션 트래픽 품질을 지속 감시한다.
> - AI Red Teaming Agent (PyRIT 기반) 로 배포 전 취약점을 자동 스캔한다.
> - OpenTelemetry + Application Insights 로 자동 계측한다.

> **전제 조건**
> - [← Ch.7 Agent Service](Ch07_Agent_Service.md), [← Ch.8 Model Router](Ch08_Model_Router.md) 완료
> - `uv add "azure-ai-evaluation>=1.18.1" "azure-monitor-opentelemetry>=1.6" "promptflow>=1.18"`

---

## 📌 Python 버전의 진정한 이점

Java 버전 Ch.9 는 "Evaluation SDK 가 Python only 라 어쩔 수 없이 Java+Python 병기" 라며 우회 pattern 을 다뤘다.

**Python 버전은 그런 apology 가 필요 없다.** `azure-ai-evaluation` 1.18.1 이 **GA, Python-primary, 40+ built-in evaluator** 를 제공한다. 이 챕터는 Python 의 진짜 실력을 보여준다.

---

## 1. 왜 evaluation 이 중요한가

LLM 서비스에서 "unit test 통과" = 품질 보장 부족.

1. **비결정성**: 같은 입력에 매번 다른 출력. temperature=0 도 완전 동일 보장 안 됨.
2. **평가 기준 주관성**: "이 답변이 좋은가?" 는 이분법 불가. Relevance · Groundedness · Coherence 등 다차원 지표.
3. **회귀(regression)**: 프롬프트 한 줄 수정이 다른 시나리오 망칠 수 있음.

**Evaluation 의 3가지 실무 목적:**
- **모델 선택**: GPT-5 vs Claude vs Phi-4 — 이 태스크에서 뭐가 더 좋은가
- **프롬프트 A/B**: v1.2 vs v1.3 — 통계적 유의미한 개선인가
- **회귀 방지**: 새 배포가 기존 시나리오를 망치지 않는가 (CI gate)

⚠️ **함정**: Evaluation 을 "리포트 만드는 것" 으로만 보면 실무 가치 없다. **배포 파이프라인 gate** 로 통합해야 의미 있다.

---

## 2. Foundry 내장 Evaluator (40+ 개, GA)

### 2.1 Quality 계열

| Evaluator | 측정 대상 | 언제 |
|---|---|---|
| **CoherenceEvaluator** | 답변 논리 일관성 | 긴 답변 검증 |
| **RelevanceEvaluator** | 질문·답변 관련성 | 엉뚱한 답변 탐지 |
| **FluencyEvaluator** | 문장 자연스러움 | 다국어 QA |
| **SimilarityEvaluator** | 기대 답변과 유사도 | Golden answer 대비 |
| **GroundednessEvaluator** | 컨텍스트 근거 유무 | RAG 필수 |
| **F1ScoreEvaluator** | Token 겹침 (정답 대비) | 추출형 태스크 |
| **BleuScoreEvaluator** / **RougeScoreEvaluator** | NLP 표준 지표 | 번역 · 요약 |

### 2.2 Safety 계열

- **ViolenceEvaluator** / **HateUnfairnessEvaluator** / **SexualEvaluator** / **SelfHarmEvaluator**: 4대 harm category, severity 0-6
- **ProtectedMaterialEvaluator**: 저작권 있는 텍스트/가사/코드 감지
- **IndirectAttackEvaluator**: XPIA (cross-prompt injection) 감지
- **CodeVulnerabilityEvaluator**: 생성된 코드의 보안 취약점

### 2.3 Agent 전용

- **IntentResolutionEvaluator**: 사용자 의도 파악 정확도
- **TaskAdherenceEvaluator**: 지시 이행률
- **ToolCallAccuracyEvaluator**: tool 호출 정확도

---

## 3. 🔧 실습 1 — 오프라인 배치 평가

기본 workflow: 데이터셋 준비 → evaluate() 실행 → 결과 확인.

### 3.1 평가 데이터셋 (JSONL)

```jsonl
# eval_dataset.jsonl - 각 라인은 dict
{"query": "Foundry Resource 와 Project 차이?", "response": "Resource 는 인프라 단위, Project 는 작업 단위다.", "context": "Foundry 는 Resource → Projects 계층 구조를 가진다..."}
{"query": "Global Standard 배포 특징?", "response": "종량제이고 quota 가 가장 높다.", "context": "Global Standard is pay-per-token with highest quota..."}
{"query": "ZDR 는 뭘 커버해?", "response": "프롬프트·완성·임베딩. Agent state 는 미커버.", "context": "Zero Data Retention covers prompts and completions..."}
```

### 3.2 평가 스크립트

```python
# src/foundry_app/evaluate_rag.py
"""Foundry 내장 evaluator 로 RAG 답변 채점."""
from __future__ import annotations
import os
from azure.ai.evaluation import (
    evaluate,
    GroundednessEvaluator,
    RelevanceEvaluator,
    CoherenceEvaluator,
    FluencyEvaluator,
)

# Judge model 설정 (evaluator 가 사용할 모델)
model_config = {
    "azure_endpoint": os.environ["AZURE_FOUNDRY_ENDPOINT"],
    "azure_deployment": "gpt-5",   # judge 는 GPT-5+ 강력 권장
    "api_version": "2025-01-01-preview",
    "api_key": os.environ["AZURE_FOUNDRY_KEY"],
}

# Foundry project 지정 - 결과가 포털 Evaluation 메뉴에 자동 업로드
azure_ai_project = {
    "subscription_id": os.environ["AZURE_SUBSCRIPTION_ID"],
    "resource_group_name": os.environ["AZURE_RESOURCE_GROUP"],
    "project_name": os.environ["FOUNDRY_PROJECT_NAME"],
}


result = evaluate(
    data="eval_dataset.jsonl",
    evaluators={
        "groundedness": GroundednessEvaluator(model_config),
        "relevance": RelevanceEvaluator(model_config),
        "coherence": CoherenceEvaluator(model_config),
        "fluency": FluencyEvaluator(model_config),
    },
    evaluation_name="rag-eval-v1",
    azure_ai_project=azure_ai_project,
)

# 요약
print("\n=== Aggregate scores (higher = better) ===")
for metric, score in result["metrics"].items():
    print(f"{metric}: {score:.3f}")

# Row-level 결과
print(f"\n총 {len(result['rows'])} 행 평가됨")
print(f"Foundry Portal 결과: {result['studio_url']}")
```

실행:
```bash
uv run python -m foundry_app.evaluate_rag
```

⚠️ **함정**:
- `GroundednessEvaluator` 는 반드시 `context` 필드 필요. RAG pipeline 의 retrieved chunks 를 그대로 넘기지 않으면 "판단 불가" 스코어.
- Judge model 은 GPT-5+ 권장. 저성능 judge = 신뢰 없는 평가.
- Judge cost 관리: `gpt-5-mini` 로 judge 하면 저렴하지만 신뢰도 저하.

💡 **팁**: `evaluate()` 는 자동으로 병렬 처리. `_parallel=True` (기본) · 데이터 수백 건도 몇 분 안에.

---

## 4. 🔧 실습 2 — Custom Evaluator (Pydantic + async)

내장 evaluator 로 부족한 도메인 특화 지표. Python 은 클래스 몇 줄로 완성.

### 4.1 LLM-as-Judge Custom Evaluator

```python
# src/foundry_app/evaluators/korean_formality.py
"""한국어 존댓말·격식체 준수 채점 (0.0-1.0)."""
from __future__ import annotations
from openai import AzureOpenAI
from pydantic import BaseModel, Field


class FormalityScore(BaseModel):
    """Judge 출력 스키마."""
    score: float = Field(ge=0.0, le=1.0, description="0.0=완전 반말, 1.0=완벽한 존댓말")
    reasoning: str = Field(description="채점 근거 (한 문장)")


class KoreanFormalityEvaluator:
    """한국어 답변이 존댓말·격식체를 지켰는지 채점."""

    def __init__(self, model_config: dict) -> None:
        self._client = AzureOpenAI(
            azure_endpoint=model_config["azure_endpoint"],
            api_key=model_config["api_key"],
            api_version=model_config["api_version"],
        )
        self._deployment = model_config["azure_deployment"]

    def __call__(self, *, query: str, response: str, **kwargs) -> dict:
        judge_prompt = f"""다음 답변이 한국어 존댓말·격식체를 지켰는지 0.0-1.0 사이로 채점하고 근거를 한 문장으로 설명하시오.

[질문] {query}
[답변] {response}
"""
        result = self._client.beta.chat.completions.parse(
            model=self._deployment,
            messages=[{"role": "user", "content": judge_prompt}],
            response_format=FormalityScore,   # ← Pydantic 으로 100% 스키마 보장
            temperature=1.0,
            max_completion_tokens=200,
        )
        parsed = result.choices[0].message.parsed
        return {
            "korean_formality": parsed.score,
            "korean_formality_reasoning": parsed.reasoning,
        }
```

`evaluate()` 에 함께 등록:

```python
from foundry_app.evaluators.korean_formality import KoreanFormalityEvaluator

result = evaluate(
    data="eval_dataset.jsonl",
    evaluators={
        "groundedness": GroundednessEvaluator(model_config),
        "korean_formality": KoreanFormalityEvaluator(model_config),
    },
)
```

### 4.2 Code-based Evaluator (LLM 없이)

특정 정규식 매치 · 코드 문법 · 길이 제한 등은 LLM 없이 순수 코드로.

```python
# src/foundry_app/evaluators/length_check.py
"""답변 길이 제한 준수 여부 (0.0 or 1.0)."""
class ResponseLengthEvaluator:
    def __init__(self, max_chars: int = 500) -> None:
        self._max = max_chars

    def __call__(self, *, response: str, **kwargs) -> dict:
        return {
            "length_ok": 1.0 if len(response) <= self._max else 0.0,
            "actual_length": len(response),
        }
```

⚠️ **함정**: **Judge = evaluated 동일 모델** 이면 self-preference bias. 예: 답변도 GPT-5, judge 도 GPT-5 → GPT-5 는 자기 답변에 관대. **다른 계열 사용 권장** (예: judge=Claude, evaluated=GPT-5).

---

## 5. 🔧 실습 3 — CI Gate 로 통합 (배포 blocking)

Evaluation 을 배포 파이프라인 gate 로. threshold 미달이면 exit 1 → GitHub Actions/Azure DevOps fail.

```python
# src/foundry_app/eval_gate.py
"""Evaluation gate - CI 에서 임계값 미달 시 배포 blocking."""
from __future__ import annotations
import sys
import argparse
from azure.ai.evaluation import evaluate, GroundednessEvaluator, RelevanceEvaluator


THRESHOLDS = {
    "groundedness": 4.0,   # 5점 만점
    "relevance": 4.0,
}


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--data", required=True)
    args = parser.parse_args()

    model_config = {...}   # 위와 동일

    result = evaluate(
        data=args.data,
        evaluators={
            "groundedness": GroundednessEvaluator(model_config),
            "relevance": RelevanceEvaluator(model_config),
        },
    )

    failed = []
    for metric, threshold in THRESHOLDS.items():
        actual = result["metrics"].get(f"{metric}.mean")
        if actual is None:
            print(f"⚠️  {metric} score 미검출")
            continue
        if actual < threshold:
            print(f"❌ FAIL: {metric}={actual:.2f} < {threshold}")
            failed.append(metric)
        else:
            print(f"✅ PASS: {metric}={actual:.2f} ≥ {threshold}")

    if failed:
        print(f"\n💀 Evaluation gate 통과 실패: {failed}")
        return 1
    print("\n🎉 모든 threshold 통과")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

GitHub Actions 통합 (Ch.11 재소환):

```yaml
# .github/workflows/deploy.yml (일부)
- name: Evaluation Gate
  env:
    AZURE_FOUNDRY_ENDPOINT: ${{ vars.FOUNDRY_ENDPOINT }}
    AZURE_FOUNDRY_KEY: ${{ secrets.FOUNDRY_KEY }}
  run: |
    uv run python -m foundry_app.eval_gate --data test/eval_dataset.jsonl
  # exit 1 → 이후 deploy step skip
```

---

## 6. Continuous Evaluation (GA, 프로덕션 감시)

Foundry Portal → Evaluation → **Continuous** → Create.

- **Sampling rate**: 트래픽의 몇 % 를 평가할지 (실무 기본: **5-10%**)
- **Evaluators**: Groundedness + Task Adherence 조합 유용
- **Alert threshold**: 스코어 급락 시 Slack / Teams webhook

⚠️ **함정**: sampling 100% → evaluator 호출 비용이 서비스 원 호출을 뛰어넘음. **5-10% 가 실무 균형**. 중요 알람 시나리오만 100%.

---

## 7. AI Red Teaming Agent (Preview, PyRIT 기반)

배포 전 자동 취약점 스캔. Attacker Agent 가 jailbreak · prompt injection · harmful output 을 시도.

```python
# src/foundry_app/red_team.py
"""AI Red Teaming - 배포 전 취약점 스캔."""
from __future__ import annotations
import asyncio, os
from azure.identity import DefaultAzureCredential
from azure.ai.evaluation.red_team import RedTeam, RiskCategory, AttackStrategy


async def main() -> None:
    red_team = RedTeam(
        azure_ai_project={
            "subscription_id": os.environ["AZURE_SUBSCRIPTION_ID"],
            "resource_group_name": os.environ["AZURE_RESOURCE_GROUP"],
            "project_name": os.environ["FOUNDRY_PROJECT_NAME"],
        },
        credential=DefaultAzureCredential(),
        risk_categories=[
            RiskCategory.VIOLENCE,
            RiskCategory.HATE_UNFAIRNESS,
            RiskCategory.SEXUAL,
            RiskCategory.SELF_HARM,
        ],
        num_objectives=5,   # 카테고리 당 시도 수
    )

    # Target: 여러분의 agent endpoint (Ch.7 에서 만든 agent id)
    async def target_callable(query: str) -> str:
        # 실제로는 여러분 agent 호출
        from foundry_app.prompt_agent import call_agent
        return await call_agent(query)

    result = await red_team.scan(
        target=target_callable,
        scan_name="pre-release-scan-v1.3",
        attack_strategies=[
            AttackStrategy.EASY,       # 단순 injection
            AttackStrategy.MODERATE,   # 롤플레이 유도
            AttackStrategy.HARD,       # 복합 조작
        ],
    )

    print(f"발견된 취약점: {len(result.successful_attacks)}")
    print(f"Foundry Portal 결과: {result.studio_url}")


if __name__ == "__main__":
    asyncio.run(main())
```

⚠️ **함정**:
- Red Teaming 은 실제로 target endpoint 에 **harmful prompts 를 던진다**. **반드시 staging 환경**. Production 에 돌리면 (1) quota 순식간 소진, (2) audit log 에 harmful content 잔뜩 남음.
- 스캔 결과의 successful attack 은 진짜 취약점 → 즉시 프롬프트/가드레일 강화.

---

## 8. Observability — OpenTelemetry + App Insights

Python 은 `azure-monitor-opentelemetry` 한 줄로 자동 계측.

### 8.1 pyproject.toml 에 추가 (Ch.3 base)

```bash
uv add "azure-monitor-opentelemetry>=1.6"
```

### 8.2 app 시작 시 configure_azure_monitor 호출

```python
# src/foundry_app/main.py
"""앱 진입점 - 한 줄로 OpenTelemetry 자동 계측."""
import os
from azure.monitor.opentelemetry import configure_azure_monitor

# 반드시 openai import 이전에 호출
configure_azure_monitor(
    connection_string=os.environ["APPLICATIONINSIGHTS_CONNECTION_STRING"],
    enable_live_metrics=True,   # 실시간 대시보드
)

# 이후 openai / requests / httpx 등 모든 outgoing 호출이 자동 계측됨
from openai import AzureOpenAI

client = AzureOpenAI(...)
response = client.chat.completions.create(...)
# 이 호출은 App Insights 에 자동 trace 됨
```

### 8.3 🔴 OpenTelemetry GenAI Semantic Conventions

GenAI 전용 attribute 표준 (2026 GA):

| Attribute | 예시 |
|---|---|
| `gen_ai.system` | `az.ai.openai` |
| `gen_ai.request.model` | `gpt-5` |
| `gen_ai.request.temperature` | `1.0` |
| `gen_ai.usage.input_tokens` | `1245` |
| `gen_ai.usage.output_tokens` | `312` |
| `gen_ai.response.finish_reasons` | `["stop"]` |

App Insights KQL 로 토큰 사용량 · 지연 · 실패율 즉시 조회:

```kusto
requests
| where cloud_RoleName == "foundry-app"
| extend model = tostring(customDimensions["gen_ai.request.model"])
| summarize avg_duration=avg(duration), total_calls=count() by model, bin(timestamp, 5m)
| order by timestamp desc
```

⚠️ **함정**: OTel semantic conventions 은 자주 업데이트. `opentelemetry-semantic-conventions-ai` 버전 pin 필수. 어느 날 필드명 바뀌면 대시보드 깨짐.

### 8.4 Live Metrics

App Insights 포털 → Live Metrics → 실시간 트래픽 · 오류율 · 종속성 관측. `enable_live_metrics=True` 옵션.

---

## 9. Trace Viewer + Trace Replay (Preview)

Foundry Portal 의 **Traces** 메뉴:

- **Trace Viewer**: 각 conversation timeline · tool call · latency 시각화
- **Conversation View**: 사용자 이력 pretty-formatted
- 🔴 **Trace Replay** (2026-06 preview): 과거 conversation 을 **재실행**. 신 프롬프트/모델로 같은 입력 → diff 비교. 프롬프트 A/B 매우 빠르게.

💡 **팁**: Trace-to-Dataset (preview) — production trace 를 evaluation dataset 으로 변환. 실제 사용자 시나리오로 회귀 테스트 셋 자동 수집.

---

## 10. Prompt Flow 통합 (Python only)

`promptflow` 패키지는 Python only. Java 에서는 REST 뿐이었으나 Python 은 native.

```bash
uv add "promptflow>=1.18"
```

```python
# flows/rag_flow/flow.dag.yaml (Prompty format)
# ... flow 정의 ...

# 실행
from promptflow.client import PFClient
pf = PFClient()
run = pf.run(flow="flows/rag_flow", data="eval_data.jsonl")
metrics = pf.get_metrics(run)
```

Prompt Flow 는 복잡한 chain 을 시각적 DAG 로 관리 → 팀 협업에 유리.

---

## 11. 함정 정리 & 베스트 프랙티스

⚠️
1. **Groundedness 없이 RAG 평가** → context 반드시 전달
2. **Judge = evaluated 동일 모델** → self-preference bias
3. **Continuous Eval 100% sampling** → 비용 폭발
4. **Red Teaming 을 production 에** → 반드시 staging
5. **OTel semconv 버전 pin 안 함** → 대시보드 어느 날 깨짐
6. **Threshold 너무 엄격** → 배포 늘 fail → 실용 재조정
7. **Judge cost 저감 위해 gpt-5-mini** → 평가 신뢰도 저하

💡 **베스트 프랙티스**:
- 평가 dataset git 관리 (`data/eval/*.jsonl`)
- PR 마다 minimum eval → 회귀 조기 발견
- Judge model 명시적 pin (`gpt-5` 등)
- Red Teaming 결과를 매 release note 첨부
- App Insights 알람 = threshold + baseline 조합 (계절성 고려)
- Batch API (Ch.8) 로 evaluation 대량 실행 → 50% 절감

---

## 요약 (Cheat Sheet)

- **`azure-ai-evaluation` 1.18.1 GA**: 40+ 내장 evaluator, Python native
- **내장 3대 축**: Quality · Groundedness · Safety
- **Custom Evaluator**: Pydantic + async 로 몇 줄이면 완성
- **CI Gate**: `evaluate()` → threshold 검증 → exit 1
- **Continuous Eval**: 5-10% sampling, threshold alert
- **AI Red Teaming**: staging 만, PyRIT 기반 자동 취약점 스캔
- **Observability**: `configure_azure_monitor()` 한 줄로 OTel + App Insights
- **Trace Replay**: 과거 conversation 재실행 → 프롬프트 A/B 가속

## 📚 더 읽기

- [`azure-ai-evaluation` Python SDK README](https://learn.microsoft.com/en-us/python/api/overview/azure/ai-evaluation-readme)
- [Foundry Evaluation Overview](https://learn.microsoft.com/en-us/azure/foundry/observability/how-to/evaluate-agent)
- [AI Red Teaming Agent](https://learn.microsoft.com/en-us/azure/foundry/how-to/develop/run-scans-ai-red-teaming-agent)
- [Azure Monitor OpenTelemetry Python](https://learn.microsoft.com/en-us/python/api/overview/azure/monitor-opentelemetry-readme)
- [OpenTelemetry GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [Prompt Flow Python](https://microsoft.github.io/promptflow/)
- [Trace Replay & Conversation View](https://learn.microsoft.com/en-us/azure/foundry/observability/how-to/view-trace)

## 다음 챕터

[Ch.10 보안 · 거버넌스 · 비용 →](Ch10_Security_Governance.md)

---
