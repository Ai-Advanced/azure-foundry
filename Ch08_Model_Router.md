# Chapter 8. Model Router & 배포 전략

[← 목차로](README.md)

> **학습 목표**
> - Model Router 가 왜 필요하고 어떻게 동작하는지 이해한다.
> - **Model Router Policy** 로 IT 관리자가 개발자의 모델 선택을 통제하는 방법을 익힌다.
> - Python `openai` 로 Model Router 를 auto 모드와 explicit 모드로 호출한다.
> - 10가지 배포 유형 (Global / Data Zone / Regional / Batch / Provisioned / Instant / Developer) 중 상황별로 선택한다.
> - PTU 경제성을 계산하고 언제 종량제에서 예약제로 전환할지 판단한다.
> - Multi-region failover 를 Python 으로 구현하고, Global Batch API 로 대량 async 작업을 처리한다.

> **전제 조건**
> - [← Ch.3 API 연동 기초](Ch03_API_Basics.md), [← Ch.7 Agent Service](Ch07_Agent_Service.md) 완료
> - Foundry 리소스에 최소 2개 모델 배포 (예: `gpt-5`, `gpt-5-nano`)

---

## 1. Model Router: 왜 필요한가

**단일 모델 잠금 문제:**
- 모든 요청을 GPT-5.5 로 → 간단한 분류·요약도 최고 성능 모델 낭비 → 비용 폭발
- 모든 요청을 GPT-5-nano 로 → 복잡한 추론 실패
- 코드에 model = "gpt-5" 하드코딩 → 새 모델 나올 때마다 배포 필요

**Model Router 원칙**: "**모델 선택을 코드가 아닌 서비스 레벨에서**"

Foundry Model Router 는 하나의 endpoint 뒤에 여러 모델을 놓고, 요청별로 자동/명시적 라우팅.

## 2. 아키텍처 (2026 GA)

- **엔드포인트**: `POST /openai/v1/responses` (또는 `/chat/completions`)
- **model 파라미터**:
  - `"auto"` — Foundry 가 task type · cost target · latency budget 을 판단하여 자동 선택
  - `"gpt-5"` / `"claude-3.5-sonnet"` / etc. — 명시적 지정

내부적으로 route table 을 관리하며, 새 모델 추가·구모델 제거를 관리자가 코드 배포 없이 반영.

---

## 3. Model Router Policy (거버넌스)

🔴 **2026 변경**: **Azure Policy 로 개발자의 모델 선택을 강제**. 사용자가 처음 링크해준 [Model Router Policy 문서](https://learn.microsoft.com/en-us/azure/foundry/how-to/model-router-policy) 의 근간.

**정책명**: `Cognitive Services Deployments should only use approved Registry Models` (built-in policy)

### 3.1 IT 관리자 설정

Azure Portal → Policy → Assignments → 이 built-in policy 를 Foundry resource 또는 subscription 에 할당.

- **Allowed Asset IDs**: 허용 모델 리스트 (예: `["gpt-5", "gpt-5-mini", "gpt-5-nano"]`)
- **Enforcement mode**: `Enforce` (배포 차단) or `Audit` (경고만)
- 15분 propagation 후 활성

### 3.2 개발자 관점

Portal 에서 Model Router 배포 만들 때 정책 위반 모델의 체크박스가 disabled 됨. UI 배너에 "Azure Policy 로 제한됨" 표시. REST/CLI/Bicep 은 배포 시 400 에러.

### 3.3 사용 시나리오

- **금융권**: 승인된 모델만 (예: Azure OpenAI GA 만, 파트너 모델 제외)
- **정부**: 특정 리전 · 특정 SKU 만 (Regional standard, Global 제외)
- **일반 기업**: 실험 모델 (preview) 차단

⚠️ **함정**: Policy 는 **배포(deployment) 시점** 에 강제. 이미 배포된 모델을 나중에 policy 로 제거하면 Compliance dashboard 에 noncompliant 로 뜬다. 수동 삭제·재배포 필요.

---

## 4. 🔧 실습 — Python 으로 Model Router 3가지 방식 호출

```python
# src/foundry_app/model_router.py
"""Model Router - auto vs explicit vs 명시 모델 비교."""
from __future__ import annotations
import os
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    api_key=os.environ["AZURE_FOUNDRY_KEY"],
    api_version="2025-01-01-preview",
)


def call_with_model(model: str, question: str) -> dict:
    """model 파라미터에 따라 라우팅 결과 비교."""
    response = client.chat.completions.create(
        model=model,   # "auto" · "gpt-5" · "gpt-5-nano" · "gpt-5.5" ...
        messages=[
            {"role": "user", "content": question},
        ],
        max_completion_tokens=500,
        temperature=1.0,
    )
    return {
        "model_used": response.model,   # 실제로 사용된 모델
        "input_tokens": response.usage.prompt_tokens,
        "output_tokens": response.usage.completion_tokens,
        "response": response.choices[0].message.content,
    }


if __name__ == "__main__":
    question = "삼체문제(Three-body problem) 를 3문장으로 설명해."

    for model in ["gpt-5-nano", "gpt-5.5", "auto"]:
        result = call_with_model(model, question)
        print(f"\n=== model='{model}' → 실제 사용: {result['model_used']} ===")
        print(f"토큰: in={result['input_tokens']}, out={result['output_tokens']}")
        print(f"응답: {result['response'][:150]}...")
```

**결과 해석:**
- `gpt-5-nano`: 빠른 응답, 낮은 비용, 답변 간단
- `gpt-5.5`: 느림, 비쌈, 정교
- `auto`: Foundry 가 판단 → 대개 중급 모델 (gpt-5-mini 등) 선택

⚠️ **함정**: `auto` 는 실험/스테이징에서만. **프로덕션에는 명시 모델 지정** — auto 는 언젠가 저성능 모델로 라우팅되어 사용자 UX 저하 위험. SLA 산정도 불가.

💡 **팁**: `response.model` 로 실제 사용된 모델 확인 가능. audit log · 비용 attribution 에 활용.

---

## 5. 배포 유형 결정 매트릭스

Foundry 는 10가지 배포 유형 제공. 3축(비용 · 처리량 예측성 · 데이터 zone)으로 정리.

| 유형 | SKU | 과금 | 데이터 처리 | 사용 시점 |
|---|---|---|---|---|
| **Global Standard** ⭐ | `GlobalStandard` | 종량 | 어느 Azure 지역 | **기본 추천**. 트래픽 예측 어려운 개발/서비스 |
| Global Provisioned | `GlobalProvisionedManaged` | PTU 예약 | 어느 Azure 지역 | 지연시간 결정적으로 중요한 상용 |
| Global Batch | `GlobalBatch` | 50% 할인, 24h 비동기 | 어느 Azure 지역 | 대량 batch job, 야간 처리 |
| Data Zone Standard | `DataZoneStandard` | 종량 | US 또는 EU 존 | US/EU 데이터 거주성 |
| Data Zone Provisioned | `DataZoneProvisionedManaged` | PTU | US/EU | Zone + 처리량 예약 |
| Data Zone Batch | `DataZoneBatch` | 50% 할인 | US/EU | Zone + async |
| **Standard (Regional)** | `Standard` | 종량 | 단일 리전 (예: Korea Central) | **한국 금융권** — 국내 데이터 |
| Regional Provisioned | `ProvisionedManaged` | PTU | 단일 리전 | 리전 + 처리량 |
| Developer | `DeveloperTier` | 종량, 24h TTL | 어느 지역 | fine-tuning eval 만 |
| Instant (preview) | N/A | 종량, no deploy | 어느 지역 | 프로토타이핑 |

### 5.1 결정 트리

```
데이터 국내 처리 필수? (금감원 · 개인정보보호법)
├─ YES → Standard (Regional, Korea Central)
└─ NO  → 데이터 zone 필수? (EU GDPR)
         ├─ YES → Data Zone Standard/Provisioned
         └─ NO  → 처리량 예측 가능?
                 ├─ 종량제 OK → Global Standard
                 ├─ 예약제 유리 → Global Provisioned
                 └─ 비동기 OK → Global Batch (50% 할인)
```

⚠️ **함정**: **Global 계열은 데이터가 국경을 넘어 처리됨**. "우리 데이터는 한국 밖으로 나가면 안 된다" 요구가 있으면 반드시 Regional 또는 Data Zone.

---

## 6. PTU (Provisioned Throughput Unit) 경제학

**PTU** = 모델의 전용 차선 (예약 시간 단위 요금).

- **종량제 (Standard)**: 톨게이트, 사용량만큼. 트래픽 없으면 0원. 명절엔 지연·거부.
- **PTU (Provisioned)**: 월 정기권 + 전용 차선. 사용 없어도 요금. 어떤 상황에서도 일정 속도.

### 6.1 Break-even 계산

**대략**: 분당 토큰 사용량(TPM) 이 **100K TPM 이상 꾸준** 하면 PTU 가 20%+ 저렴 시작. 정확한 계산은 Foundry Portal 의 **PTU Calculator** 사용.

### 6.2 Python 으로 사용량 모니터링

```python
# src/foundry_app/usage_tracker.py
"""API 사용량 추적 - App Insights + 자체 로그."""
import os, time
from openai import AzureOpenAI
import structlog

log = structlog.get_logger()
client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    api_key=os.environ["AZURE_FOUNDRY_KEY"],
    api_version="2025-01-01-preview",
)


def tracked_call(model: str, messages: list) -> str:
    start = time.time()
    response = client.chat.completions.create(
        model=model, messages=messages, max_completion_tokens=1000, temperature=1.0,
    )
    duration = time.time() - start

    # 구조화 로그 (App Insights 로 자동 수집 - Ch.9)
    log.info(
        "foundry_call",
        model=model,
        actual_model=response.model,
        input_tokens=response.usage.prompt_tokens,
        output_tokens=response.usage.completion_tokens,
        total_tokens=response.usage.total_tokens,
        duration_ms=int(duration * 1000),
    )
    return response.choices[0].message.content or ""
```

⚠️ **함정**: PTU 는 **예약 기간 lock-in** — 1개월 최소. 중도 해지 시 위약. 트래픽 예측이 확실할 때만.

---

## 7. Multi-region Failover

Region 하나 장애 시 fallback. Python 으로 구현.

```python
# src/foundry_app/failover_client.py
"""Multi-region fallback client."""
from __future__ import annotations
import os
from openai import AzureOpenAI, APIStatusError, APITimeoutError
import structlog

log = structlog.get_logger()


class FailoverClient:
    """Primary → Secondary → Tertiary 순으로 fallback."""

    def __init__(self, endpoints: list[tuple[str, str, str]]) -> None:
        """endpoints: [(endpoint_url, api_key, deployment_name), ...]"""
        self.clients = [
            (AzureOpenAI(azure_endpoint=ep, api_key=key, api_version="2025-01-01-preview"),
             deployment)
            for ep, key, deployment in endpoints
        ]

    def chat(self, messages: list, max_tokens: int = 1000) -> str:
        last_error: Exception | None = None
        for idx, (client, deployment) in enumerate(self.clients):
            try:
                log.info("try_endpoint", index=idx, deployment=deployment)
                response = client.chat.completions.create(
                    model=deployment, messages=messages,
                    max_completion_tokens=max_tokens, temperature=1.0,
                )
                return response.choices[0].message.content or ""
            except (APIStatusError, APITimeoutError) as e:
                log.warning("endpoint_failed", index=idx, error=str(e))
                last_error = e
                continue

        raise RuntimeError(f"All {len(self.clients)} endpoints failed") from last_error


# 사용
client = FailoverClient([
    (os.environ["FOUNDRY_EAST_US"], os.environ["FOUNDRY_EAST_US_KEY"], "gpt-5"),
    (os.environ["FOUNDRY_SWEDEN"], os.environ["FOUNDRY_SWEDEN_KEY"], "gpt-5"),
    (os.environ["FOUNDRY_KOREA"], os.environ["FOUNDRY_KOREA_KEY"], "gpt-5"),
])
answer = client.chat([{"role": "user", "content": "Hello"}])
```

💡 **팁**: 429 (rate limit) 도 fallback 트리거로 삼으면 primary quota 부족 시 secondary 로 흘려보내 처리량 확보.

---

## 8. Global Batch API — 대량 async 처리

**시나리오**: 야간에 10만 건의 고객 리뷰를 요약. 종량제로 하면 비싸고 실시간 SLA 도 불필요.

**Batch API 흐름**:
1. JSONL 파일 준비 (한 줄 = 한 요청)
2. Files API 로 업로드
3. Batch API 로 job 생성 → 24시간 이내 완료
4. 결과 JSONL 다운로드
5. 종량 대비 **50% 할인**

```python
# src/foundry_app/batch_jobs.py
"""Global Batch API 예제 - 리뷰 요약 대량 처리."""
from __future__ import annotations
import os, json, time
from pathlib import Path
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    api_key=os.environ["AZURE_FOUNDRY_KEY"],
    api_version="2025-01-01-preview",
)


def build_batch_file(reviews: list[str], output_path: Path, deployment: str) -> None:
    """JSONL 포맷으로 batch input 생성."""
    with output_path.open("w", encoding="utf-8") as f:
        for i, review in enumerate(reviews):
            request = {
                "custom_id": f"review-{i:06d}",
                "method": "POST",
                "url": "/chat/completions",
                "body": {
                    "model": deployment,
                    "messages": [
                        {"role": "system", "content": "리뷰를 1문장으로 요약."},
                        {"role": "user", "content": review},
                    ],
                    "max_completion_tokens": 100,
                    "temperature": 1.0,
                },
            }
            f.write(json.dumps(request, ensure_ascii=False) + "\n")


def submit_batch(input_file: Path) -> str:
    """Batch job 제출 → batch ID 반환."""
    # 1) 파일 업로드
    uploaded = client.files.create(file=input_file.open("rb"), purpose="batch")

    # 2) Batch job 생성
    batch = client.batches.create(
        input_file_id=uploaded.id,
        endpoint="/chat/completions",
        completion_window="24h",
        metadata={"job_type": "review_summarization"},
    )
    print(f"Batch job 생성: {batch.id} (status={batch.status})")
    return batch.id


def wait_and_download(batch_id: str, output_dir: Path) -> Path:
    """완료 대기 + 결과 다운로드."""
    while True:
        batch = client.batches.retrieve(batch_id)
        print(f"Status: {batch.status} ({batch.request_counts.completed}/{batch.request_counts.total})")
        if batch.status == "completed":
            break
        if batch.status in ("failed", "cancelled", "expired"):
            raise RuntimeError(f"Batch failed: {batch.status}")
        time.sleep(60)   # 프로덕션엔 exponential backoff

    output_file = client.files.content(batch.output_file_id)
    output_path = output_dir / f"batch_{batch_id}_output.jsonl"
    output_path.write_bytes(output_file.read())
    return output_path


if __name__ == "__main__":
    reviews = ["리뷰1", "리뷰2", ...]  # 실제로는 DB 에서 로드
    input_path = Path("batch_input.jsonl")
    build_batch_file(reviews, input_path, deployment="gpt-5-mini")
    batch_id = submit_batch(input_path)
    result_path = wait_and_download(batch_id, Path("."))
    print(f"결과: {result_path}")
```

⚠️ **함정**:
- Batch input 파일 크기 상한 (~100MB). 큰 job 은 분할.
- 24h 이내 완료 SLA, 하지만 실제로 몇 분 만에 끝나는 경우도 흔함.
- `custom_id` 로 결과와 요청 매핑. 순서 보장 안 됨.

---

## 9. Region · Quota 전략

### 9.1 지역별 모델 가용성

최신 모델 (GPT-5.5) 은 **East US · Sweden Central** 이 가장 먼저. Korea Central 은 몇 주-몇 달 lag.

### 9.2 Quota 관리

- **Foundry Portal → Quota**: 현재 사용량 · TPM/RPM 상한 확인
- **증설 요청**: Request Quota → 24시간 내 승인 (일반적)
- 프로덕션 런칭 전 **필요 TPM × 1.5배** 미리 확보

```python
# 프로덕션 headroom 확인
import os
from azure.mgmt.cognitiveservices import CognitiveServicesManagementClient
from azure.identity import DefaultAzureCredential

mgmt = CognitiveServicesManagementClient(
    credential=DefaultAzureCredential(),
    subscription_id=os.environ["AZURE_SUBSCRIPTION_ID"],
)

usages = mgmt.usages.list(location="eastus")
for u in usages:
    if "OpenAI" in u.name.value:
        print(f"{u.name.value}: {u.current_value}/{u.limit}")
```

---

## 10. 흔한 함정 정리

⚠️
1. **`auto` 모드 프로덕션 사용** — 원치 않는 저성능 모델로 라우팅.
2. **PTU 예약 기간 lock-in** — 트래픽 예측 없이 예약하면 낭비.
3. **Data Zone + Global 혼용** — 데이터 거주성 위반.
4. **Model Router Policy 15분 propagation** — 즉시 반영 안 됨.
5. **Batch API 로 실시간 응답 기대** — 24시간 async job.
6. **Failover client 에 429 는 fallback 안 함** — 그러면 primary 부하 심할 때 secondary 활용 안 됨.

💡 **베스트 프랙티스**:

- 개발/스테이징 → `auto` 로 다양성 확보, 프로덕션 → 명시 모델
- 서비스 초기 → Global Standard, 트래픽 안정 후 → PTU 검토
- Batch API 를 evaluation (Ch.9) 대량 실행에 활용 (50% 절감)
- Multi-region 은 primary 지역 100% capacity 대비 secondary 30% 예약
- Model Router Policy 는 Enforce 전에 Audit mode 로 위반 사례 파악

---

## 요약 (Cheat Sheet)

- **Model Router**: `model="auto"` (개발) or 명시 (프로덕션)
- **Model Router Policy**: Azure Policy 로 IT 관리자가 모델 선택 통제
- **배포 유형 3축**: 비용 (종량/PTU/Batch) × 처리량 (예측성) × 지역 (Global/Zone/Regional)
- **한국 금융권 기본**: Standard (Regional, Korea Central)
- **PTU 진입선**: 100K TPM 지속 시 검토
- **Global Batch**: 대량 async, 50% 할인, 24h SLA
- **Failover**: primary → secondary → tertiary. 429 도 fallback 트리거로

## 📚 더 읽기

- [Model Router Policy 문서 (원본)](https://learn.microsoft.com/en-us/azure/foundry/how-to/model-router-policy)
- [Foundry Deployment Types](https://learn.microsoft.com/en-us/azure/foundry/foundry-models/concepts/deployment-types)
- [PTU Calculator](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/provisioned-throughput-onboarding)
- [Global Batch API](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/batch)
- [Quota Management](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/quota)

## 다음 챕터

[Ch.9 Evaluation & Observability (Python-native) →](Ch09_Evaluation.md)

---
