# Chapter 4. Prompt Engineering & Structured Outputs (Pydantic v2)

[← 목차로](README.md)

> **학습 목표**
> - LLM 의 메시지 구조(System · User · Assistant) 를 이해하고 각 역할을 효과적으로 정의한다.
> - Few-shot 프롬프팅과 Chain-of-Thought(CoT) 로 복잡한 추론 성능을 극대화한다.
> - GPT-5 Reasoning 모델의 `reasoning_effort` 파라미터를 상황별로 조절한다.
> - **Pydantic v2 로 100% 신뢰 가능한 구조화 출력(Structured Outputs)** 을 획득한다.
> - Vision 멀티모달 입력을 Python 으로 처리한다.
> - Foundry Prompts Playground 와 코드-사이드 템플릿을 병행 사용한다.

> **전제 조건**
> - [← Ch.3 API 연동 기초](Ch03_API_Basics.md) 완료 (`uv`, `openai`, `azure-ai-projects`)
> - GPT-5 또는 GPT-5-mini 배포 완료
> - Pydantic v2 이해 (기본 모델 · 필드 · 검증)

---

## 1. 역할(role)과 메시지 구조

LLM 대화는 **메시지 객체 배열** 이다. 각 메시지는 고유한 `role` 을 가지며, 이들의 조합이 모델의 출력 품질과 행동 양식을 결정한다. 2026년 기준, 이 역할 분담은 모델의 내부 어텐션(Attention) 이 정보를 가중 처리하는 기준이 된다.

### 1.1 주요 Role

- **`system`**: 페르소나, 지식 범위, 답변 형식, 제약 사항. 대화 최상단에 위치. **가장 신중하게 작성해야 함.**
- **`user`**: 실제 질문·명령. 대화 진행에 따라 여러 번 등장.
- **`assistant`**: 이전 턴의 모델 응답. 컨텍스트 유지용으로 다시 전달. 개발자가 mocking 하여 흐름을 유도하기도 함.
- **`tool`** (Ch.5): 함수 호출 결과. 모델이 도구를 부른 뒤 그 결과를 다시 넣을 때 사용.

⚠️ **함정**: 메시지 순서는 `system` → (`user` ↔ `assistant` 교차) 를 반드시 지킬 것. `assistant` 두 개 연속, `system` 중간 삽입은 400 에러 or 컨텍스트 혼동. 에이전트 loop 구현 시 이전 대화 잘라내는 과정에서 **`user` 로 끝나는지** 확인.

### 1.2 System Message 설계 패턴

- **나쁜 예**: `"너는 유능한 비서야. 친절하게 답해줘."`
  - '유능함' · '친절함' 을 모델이 자의적으로 해석 → 톤 · 깊이 불일치

- **좋은 예**:
  ```text
  너는 IT 보안 전문가 페르소나를 가진 기술 지원 에이전트다.
  - 모든 답변은 한국어로 하며, 전문 용어 뒤에는 괄호로 쉬운 설명을 덧붙인다.
  - 답변 끝에는 항상 "추가 보안 점검이 필요하신가요?" 를 포함한다.
  - 정치·종교 질문에는 "제 업무 범위를 벗어나는 질문입니다" 라고만 답한다.
  - 모르는 정보는 절대 추측하지 않고 "정보가 부족합니다" 라고 답한다.
  ```

💡 **팁**: **Boundary (경계) 지정** 이 핵심. '해야 할 일' 보다 '하지 말아야 할 일' 을 명시적으로 적어야 jailbreak / hallucination 이 줄어든다.

### 1.3 Python 에서 System Prompt 관리

작은 프로젝트는 문자열 상수로 충분하지만, 성장하면 파일로 분리 + Jinja2 템플릿화 권장.

```python
# src/foundry_app/prompts.py
from pathlib import Path
from jinja2 import Template

PROMPTS_DIR = Path(__file__).parent / "prompts"

def load_prompt(name: str, **variables) -> str:
    """prompts/security-agent.j2 → 렌더링 결과."""
    template_path = PROMPTS_DIR / f"{name}.j2"
    return Template(template_path.read_text(encoding="utf-8")).render(**variables)


# 사용
system_prompt = load_prompt("security_agent", company="Contoso")
messages = [
    {"role": "system", "content": system_prompt},
    {"role": "user", "content": "SQL injection 예방법?"},
]
```

---

## 2. Few-shot 프롬프팅

**입출력 예시(Example)** 를 제공하여 모델에게 패턴을 학습시킨다. 파인튜닝 없이 유사 효과 → **In-context Learning**.

- **Zero-shot**: 예시 없음. 상식 질문 · 모델 기본 성능이 높은 작업.
- **One-shot**: 예시 1개. 출력 형식 잡기.
- **Few-shot**: 3개 이상. 감성 분류, 특정 스타일 생성, 비정형 데이터 추출 등 필수.

### 2.1 최신 편향(Recency Bias)

🔴 **2026 변경**: GPT-5 계열은 컨텍스트 100만+ 토큰이지만 여전히 **프롬프트 뒷부분 정보에 민감**. Few-shot 예시의 **가장 마지막(사용자 질문 바로 앞)** 에 가장 중요한 패턴을 배치.

### 2.2 Python 구현 — Few-shot 헬퍼

```python
# src/foundry_app/few_shot.py
from typing import TypedDict

class Message(TypedDict):
    role: str
    content: str

def build_few_shot(
    system: str,
    examples: list[tuple[str, str]],   # [(user_input, assistant_output), ...]
    user_query: str,
) -> list[Message]:
    """Few-shot 메시지 배열 생성."""
    messages: list[Message] = [{"role": "system", "content": system}]
    for user_ex, asst_ex in examples:
        messages.append({"role": "user", "content": user_ex})
        messages.append({"role": "assistant", "content": asst_ex})
    messages.append({"role": "user", "content": user_query})
    return messages


# 사용: 조선시대 선비 말투 가이드
messages = build_few_shot(
    system="너는 조선시대 선비 말투를 쓰는 역사 가이드다. 모든 답변은 '하오'체로 끝낸다.",
    examples=[
        ("오늘 날씨가 어떤가?", "창밖을 보니 햇살이 눈부시게 내리쬐고 있소. 나들이 가기 좋구려."),
        ("맛있는 음식 추천해다오.", "여름철에는 시원한 냉면 한 그릇이 으뜸이라 하오. 고기 육수에 메밀면을 말아 드셔보시구려."),
    ],
    user_query="요즘 유행하는 게 뭔가?",
)
```

### 2.3 Few-shot 실패 케이스

예시 10+ 개 넘어가면 오히려 미세 차이에 혼동. 특정 도메인 지식 수천 개 필요 → **파인튜닝 or RAG (Ch.6)** 로 넘어갈 것.

---

## 3. Chain-of-Thought (CoT) 와 Reasoning 모델

복잡한 산술·법률·아키텍처 설계 → 모델이 '생각할 시간' 을 갖게 유도.

### 3.1 명시적 CoT vs 내부 CoT

- **명시적 CoT**: `"단계를 나누어 차근차근 생각해봐 (Let's think step by step)"` 를 프롬프트에 삽입. 사고 과정이 텍스트로 노출됨 → 디버깅 용이.
- **내부 CoT (Reasoning 모델)**: 🔴 **2026 변경** — GPT-5 reasoning 계열(o-series, GPT-5.1+) 은 사용자가 요청하지 않아도 내부적으로 여러 가설 검증. 최종 결과만 출력.

### 3.2 `reasoning_effort` 파라미터

GPT-5 reasoning 모델 전용. Latency-정확도 trade-off 조정.

| 값 | 특성 | 사용 시점 |
|---|---|---|
| **`low`** | 응답 속도 ≈ 일반 GPT-5, 저비용 | 간단한 논리 확인, 빠른 피드백 |
| **`medium`** (기본) | 균형 | 일반 코딩 · 문서 요약 · 논리 연관성 |
| **`high`** | 정확도 극대, latency 수십초~수분 | 과학·수학 증명, 대규모 코드 리뷰, 법률 해석 |

```python
# reasoning_effort 조절 예제
response = client.chat.completions.create(
    model="gpt-5",   # reasoning 모델 (Ch.2 배포명 그대로)
    messages=[
        {"role": "user", "content": "다음 시스템의 병목 지점을 분석하고 3가지 개선안 제시: ..."},
    ],
    max_completion_tokens=4000,
    reasoning_effort="high",   # ← 정밀 추론
)

# 내부 CoT 결과는 응답에 포함되지 않음 (모델 내부에서만 처리)
print(response.choices[0].message.content)
```

⚠️ **함정**:
- Reasoning 모델은 `temperature=1.0` 고정, 사용자 조정 불가.
- `top_p` 등 샘플링 파라미터도 제한.
- `reasoning_effort=high` 는 응답 시간이 예측 불가 → 사용자 UX 에 "생각 중..." 로딩 표시 필수.
- 내부 CoT 도 토큰 소비함 → 비용 산정 시 `usage.completion_tokens_details.reasoning_tokens` 확인.

---

## 4. Structured Outputs (Pydantic v2) — Python 의 진가

애플리케이션 개발자에게 가장 환영받는 기능. 모델이 생성하는 JSON 이 사용자가 정의한 스키마를 **100% 준수** 하도록 강제.

### 4.1 왜 Pydantic v2 인가

Python 에서는 **Pydantic v2 + `client.beta.chat.completions.parse()`** 조합이 표준. Java의 Jackson JSON Schema 수동 작성과 비교 불가능한 편의성.

- 모델 클래스 → JSON Schema 자동 생성
- 응답 → 모델 인스턴스 자동 파싱 및 검증
- Optional / Union / Enum 등 Python 타입을 그대로 활용

### 4.2 기본 사용법

```python
# src/foundry_app/structured.py
"""Pydantic v2 로 영화 리뷰 구조화 추출."""
from __future__ import annotations
import os
from typing import Literal
from pydantic import BaseModel, Field
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    api_key=os.environ["AZURE_FOUNDRY_KEY"],
    api_version="2025-01-01-preview",
)


class MovieReview(BaseModel):
    """영화 리뷰 구조. Docstring 이 모델에 힌트로 전달됨."""
    title: str = Field(description="영화 제목")
    score: int = Field(ge=0, le=10, description="0-10 점수")
    highlights: list[str] = Field(min_length=1, max_length=5, description="주요 장점 1-5개")
    genre: Literal["액션", "드라마", "코미디", "SF", "공포", "기타"]
    watched_again: bool = Field(description="다시 볼 의사")


completion = client.beta.chat.completions.parse(
    model=os.environ["AZURE_FOUNDRY_DEPLOYMENT"],
    messages=[
        {"role": "system", "content": "너는 영화 평론가다. 리뷰를 분석하고 정해진 JSON 형식으로만 답한다."},
        {"role": "user", "content": "어제 '인터스텔라' 봤는데 최고였어. 블랙홀 묘사가 압권이고 음악이 소름 돋더라. 10점 만점에 10점. 또 볼 거야."},
    ],
    response_format=MovieReview,   # ← Pydantic 모델을 그대로 전달
    max_completion_tokens=500,
    temperature=1.0,
)

# 100% 스키마 준수 보장 (Foundry 서버가 토큰 생성 제약)
review: MovieReview = completion.choices[0].message.parsed
print(f"제목: {review.title}")
print(f"점수: {review.score}/10")
print(f"장르: {review.genre}")
print(f"장점: {', '.join(review.highlights)}")
print(f"재관람: {'예' if review.watched_again else '아니오'}")
```

**한 줄로 요약**: `response_format=PydanticModel` → `parsed` 로 타입 안전 객체 획득. 파싱 실패 가능성 zero.

### 4.3 중첩·복합 스키마

Pydantic 모델을 중첩해서 복잡한 구조도 자유롭게.

```python
from datetime import date

class Actor(BaseModel):
    name: str
    role: str

class Movie(BaseModel):
    title: str
    release_year: int = Field(ge=1900, le=2100)
    genres: list[Literal["Action", "Drama", "Comedy", "SF", "Horror"]]
    actors: list[Actor] = Field(min_length=1)
    imdb_rating: float | None = Field(default=None, ge=0.0, le=10.0)

# response_format=Movie 로 넘기면 위와 동일하게 동작
```

### 4.4 Strict Mode 규칙

🔴 **2026 변경**: Structured Outputs Strict Mode 는 **GA**. Pydantic v2 는 자동으로 다음 규칙을 만족한다.

1. `type: "object"` 최상위
2. 모든 속성이 `required` 에 포함 (Optional 은 `["string", "null"]` union)
3. `additionalProperties: false`

Pydantic 이 이걸 자동 생성하므로 사용자가 직접 JSON Schema 작성 필요 없음.

⚠️ **함정**:
- 첫 요청에서 스키마 컴파일 시간이 몇 초 추가. 이후 캐싱됨.
- `Optional[str]` 또는 `str | None` 필드는 자동으로 union type 처리됨.
- 모델이 최선 응답이 없을 때 `refusal` 필드에 이유가 온다: `completion.choices[0].message.refusal`.

```python
# refusal 처리
msg = completion.choices[0].message
if msg.refusal:
    print(f"모델이 답변 거부: {msg.refusal}")
else:
    review = msg.parsed
    # ... 정상 처리
```

### 4.5 원시 JSON Schema 방식 (Pydantic 없이)

Pydantic 사용 못 하는 상황(외부 스키마 파일, 다이내믹 스키마) 에서.

```python
schema = {
    "type": "object",
    "properties": {
        "title": {"type": "string"},
        "score": {"type": "integer", "minimum": 0, "maximum": 10},
    },
    "required": ["title", "score"],
    "additionalProperties": False,
}

response = client.chat.completions.create(
    model=os.environ["AZURE_FOUNDRY_DEPLOYMENT"],
    messages=[...],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "movie_review",
            "schema": schema,
            "strict": True,
        },
    },
)

import json
data = json.loads(response.choices[0].message.content)
```

Pydantic 방식보다 verbose. 특별한 이유 없으면 Pydantic 권장.

---

## 5. Vision Multimodal 입력

GPT-5는 텍스트 + 이미지 통합 처리. 차트 해석, 영수증 데이터 추출, 시각적 검수 등.

### 5.1 이미지 전달 두 방법

1. **HTTPS URL**: 공개 이미지 링크 전달. 모델이 직접 fetch. 요청 body 가벼움.
2. **Base64 Data-URI**: 로컬 파일 인코딩. 보안 중요 데이터 · 임시 이미지 처리.

### 5.2 Python 구현

```python
# src/foundry_app/vision.py
"""Vision — 로컬 이미지 분석."""
from __future__ import annotations
import base64
import os
from pathlib import Path
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    api_key=os.environ["AZURE_FOUNDRY_KEY"],
    api_version="2025-01-01-preview",
)


def analyze_image(image_path: Path, question: str = "이 이미지 설명해줘.") -> str:
    """로컬 이미지 → Base64 → GPT-5 분석."""
    if image_path.stat().st_size > 20 * 1024 * 1024:
        raise ValueError("이미지 크기 20MB 초과 - 리사이즈 필요")

    # 확장자 → MIME 매핑
    mime_map = {".png": "image/png", ".jpg": "image/jpeg", ".jpeg": "image/jpeg", ".webp": "image/webp"}
    mime = mime_map.get(image_path.suffix.lower())
    if not mime:
        raise ValueError(f"미지원 이미지 포맷: {image_path.suffix}")

    b64 = base64.b64encode(image_path.read_bytes()).decode("ascii")
    data_uri = f"data:{mime};base64,{b64}"

    response = client.chat.completions.create(
        model=os.environ["AZURE_FOUNDRY_DEPLOYMENT"],
        messages=[
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": question},
                    {
                        "type": "image_url",
                        "image_url": {
                            "url": data_uri,
                            "detail": "high",   # low: 65 tokens, high: 정밀 분석
                        },
                    },
                ],
            },
        ],
        max_completion_tokens=1000,
        temperature=1.0,
    )
    return response.choices[0].message.content


if __name__ == "__main__":
    result = analyze_image(Path("./sample.png"), "차트를 분석하고 요약해줘.")
    print(result)
```

### 5.3 여러 이미지 동시 전달

```python
messages = [{
    "role": "user",
    "content": [
        {"type": "text", "text": "이 두 이미지의 차이를 비교해줘."},
        {"type": "image_url", "image_url": {"url": data_uri_1}},
        {"type": "image_url", "image_url": {"url": data_uri_2}},
    ],
}]
```

⚠️ **함정**:
- 이미지 한 장당 **20MB** 상한, **최대 8192x8192 픽셀**. 초과 시 업로드 단계 에러.
- `detail: "high"` 는 이미지를 512x512 격자로 쪼개 각각 분석 → 토큰 소비 크게 증가. 문서 인식 · 미세 항목 검출에만 사용.
- Vision 은 GPT-5.5 · GPT-4o 등 특정 모델에서만 지원. `gpt-5-mini` 는 텍스트 전용.

🔴 **2026 변경**: GPT-5.5 는 이미지 인식을 넘어 **Computer Use** 로 확장. 브라우저 · OS UI 요소를 직접 클릭 · 텍스트 입력. Ch.7 에이전트 도구에서 상세.

---

## 6. Prompt 재사용 & 버전 관리

프롬프트를 코드에 하드코딩하면 유지보수 재앙. Foundry 는 체계적 관리 도구 제공.

### 6.1 Foundry Prompts Playground

- **Prompt Assets**: 작성한 프롬프트를 '자산' 으로 저장 → 고유 ID 부여
- **버전 관리**: 수정 이력 자동 기록, 버전별 성능 비교 테스트
- **View Code**: Playground 검증된 프롬프트를 Python/curl 코드로 즉시 변환
- **환경별 배포**: 개발 vs 운영 서로 다른 버전 배포 가능

### 6.2 변수 치환 두 전략

프롬프트에 `{{customer_name}}` 같은 자리표시자 → 런타임 데이터 주입.

- **클라이언트 사이드** (Python `str.format()` or Jinja2): 개발자 완전 제어. 프롬프트가 코드에 섞임.
  ```python
  system = f"고객명: {customer.name}. 다음 규칙을 따라 답변: ..."
  ```

- **서버 사이드** (Foundry Prompts): 프롬프트 템플릿은 Foundry 에 등록, 앱은 변수만 넘김. **엔터프라이즈 권장**.
  ```python
  # Foundry Prompts API 호출 (2026 preview)
  prompt = project_client.prompts.get(name="customer-support-v3")
  rendered = prompt.render(variables={"customer_name": "홍길동"})
  ```

💡 **팁**: 두 방식 병행 가능. 코드에 canonical version 두고 (git 관리), Foundry 에 sync 하는 하이브리드. Ch.11 CI/CD 에서 자동화.

---

## 7. Python 특유의 실무 팁

Java 커리큘럼 대비 Python 만의 강점.

- **`asyncio` 병렬 호출**: 여러 프롬프트를 동시에 처리 가능. `openai` 는 `AsyncAzureOpenAI` 클래스 제공.
  ```python
  from openai import AsyncAzureOpenAI
  import asyncio

  async_client = AsyncAzureOpenAI(...)

  async def process_many(queries: list[str]) -> list[str]:
      tasks = [async_client.chat.completions.create(model=..., messages=[{"role":"user","content":q}]) for q in queries]
      responses = await asyncio.gather(*tasks)
      return [r.choices[0].message.content for r in responses]
  ```

- **Type hint + `basedpyright`**: 프롬프트 응답 타입까지 완전 검증. IDE 자동완성 강력.
- **`ruff` 로 코드 스타일 자동화**: Ch.11 pre-commit hook 에서 활용.
- **`pytest` + VCR** 로 LLM 응답 mocking: 테스트 반복 시 API 호출 비용 zero.

---

## 요약 (Cheat Sheet)

- **System Message**: Boundary(경계) 중심으로 명시. '하지 말아야 할 일' 을 적어야 hallucination 방지.
- **Few-shot**: 가장 중요한 예시를 마지막에 배치 (recency bias).
- **GPT-5 Reasoning**: `reasoning_effort` = low/medium/high 로 지연·정확도 조절. `temperature=1.0` 고정.
- **Structured Outputs**: `client.beta.chat.completions.parse(response_format=PydanticModel)` 이 표준. 100% 스키마 준수.
- **Vision**: 20MB · 8192px 상한. `detail="high"` 는 토큰 크게 소비.
- **Prompt 버전 관리**: Foundry Prompts CMS + Git 하이브리드가 실무 정답.

## 📚 더 읽기

- [Structured Outputs Guide (OpenAI)](https://platform.openai.com/docs/guides/structured-outputs)
- [Azure OpenAI Structured Outputs](https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/structured-outputs)
- [Pydantic v2 공식 문서](https://docs.pydantic.dev/latest/)
- [OpenAI Vision Guide](https://platform.openai.com/docs/guides/vision)
- [GPT-5 Reasoning Models](https://platform.openai.com/docs/guides/reasoning)
- [Foundry Prompt Engineering](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/prompt-engineering)

## 다음 챕터

[Ch.5 Function Calling & Tool Use (MCP Python + FastAPI) →](Ch05_Function_Calling.md)

---
