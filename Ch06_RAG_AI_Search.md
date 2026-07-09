# Chapter 6. RAG — Azure AI Search + LangChain/LlamaIndex

[← 목차로](README.md)

> **학습 목표**
> - RAG(Retrieval-Augmented Generation)의 원리와 필요성을 이해한다.
> - Embedding 모델의 특성을 파악하고 적절한 차원(Dimensions)을 결정한다.
> - 문서 분할(Chunking) 전략을 수립하고 효율적인 인덱싱 파이프라인을 Python 으로 구축한다.
> - Azure AI Search 로 Keyword + Vector + Semantic 이 결합된 Hybrid Search 를 구현한다.
> - LangChain / LlamaIndex 로 RAG 파이프라인을 고수준 추상화한다.
> - Foundry 'On Your Data' 관리형 RAG 를 언제 쓸지 판단한다.

> **전제 조건**
> - [← Ch.3 API 연동 기초](Ch03_API_Basics.md) 완료 (uv + openai + azure-ai-projects)
> - `text-embedding-3-small` 또는 `text-embedding-3-large` 배포 완료
> - Azure AI Search Standard tier 이상 리소스 생성

**추가 의존성:**
```bash
uv add "azure-search-documents>=12.0.0" \
       "langchain-azure-ai>=1.2.8" \
       "llama-index-llms-azure-openai>=0.5.5" \
       "llama-index-embeddings-azure-openai" \
       "llama-index-vector-stores-azureaisearch"
```

---

## 1. RAG 가 왜 필요한가

LLM 은 학습 데이터 안의 지식만 안다. 두 가지 치명적 한계:

1. **지식 컷오프**: 학습 이후 최신 정보 모름.
2. **사내 privacy 데이터 부재**: 사내 규정 · 미공개 프로젝트 문서 · 최신 기술 명세서는 학습 못함.

매번 파인튜닝은 비용·시간 비효율. RAG = 모델 재학습 없이 **질문과 관련된 문서를 외부에서 찾아 LLM 에게 참고자료로 제공**.

### 1.1 RAG 기본 흐름: Retrieve → Augment → Generate

1. **Retrieve**: 사용자 질문 → 벡터화 → 사내 문서 인덱스에서 top-k chunk 검색
2. **Augment**: 검색된 chunk 를 시스템 프롬프트에 grounding context 로 주입
3. **Generate**: 모델이 제공된 컨텍스트 기반으로 답변. "모르면 모른다"고 답하도록 강제 → hallucination 방어

---

## 2. Embeddings 개념과 모델 선택

컴퓨터는 텍스트 의미를 직접 이해 못 함 → 숫자 벡터로 변환 = **Embedding**. 유사 의미 텍스트는 벡터 공간에서 가까운 거리.

### 2.1 2026 주요 임베딩 모델

| 모델 | 차원 | 특성 |
|---|---|---|
| **text-embedding-3-small** | 1536 (조정 가능) | 가성비 최고. 대부분의 RAG 에 적합 |
| **text-embedding-3-large** | 3072 (조정 가능) | 최고 정확도. 법률·기술 문서 |
| **text-embedding-ada-002** (레거시) | 1536 | 신규 프로젝트에는 3-small 이 낫다 |

⚠️ **함정**: 인덱스의 벡터 차원 = 임베딩 모델 출력 차원 **일치 필수**. 3072 짜리 모델을 1536 차원 인덱스에 넣으면 인덱싱 단계 에러 → 인덱스 재생성해야 함. 초기 설계 중요.

💡 **팁**: `text-embedding-3-*` 계열은 출력 차원을 사후 조정 가능 (`dimensions` 파라미터). 비용 절감을 위해 1536 → 512 로 축소해도 성능 저하 미미.

### 2.2 Python 으로 임베딩 생성

```python
# src/foundry_app/embedding.py
"""Text → Vector 임베딩."""
from __future__ import annotations
import os
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    api_key=os.environ["AZURE_FOUNDRY_KEY"],
    api_version="2025-01-01-preview",
)


def embed(texts: list[str], deployment: str = "text-embedding-3-small") -> list[list[float]]:
    """텍스트 배열 → 벡터 배열. 한 번에 최대 2048개."""
    response = client.embeddings.create(
        model=deployment,
        input=texts,
        # dimensions=512,   # 옵션: 차원 축소
    )
    return [item.embedding for item in response.data]


if __name__ == "__main__":
    vectors = embed(["안녕하세요.", "Hello world."])
    print(f"차원: {len(vectors[0])}")
    print(f"첫 5 값: {vectors[0][:5]}")
```

---

## 3. Chunking 전략

긴 문서를 통째로 벡터화하면 의미가 희석 → 검색 품질 저하. 적절한 크기로 자르는 청킹 필수.

### 3.1 분할 방식과 근거

- **Fixed-size Chunking**: 500-1000자 단위, 100-200자 overlap. 단순·범용.
- **Semantic Chunking**: 의미 변화 지점 감지 후 분할. 품질 ↑, 비용 ↑.
- **Recursive Character Splitting** (LangChain 표준): 문단 → 문장 → 단어 순으로 시도. Markdown/코드 구조 인식.

실무 기본: **800 토큰 크기 + 150 토큰 overlap**. 검색된 3-5개 chunk 가 LLM 컨텍스트 창을 과점하지 않으면서 충분한 정보를 담는 균형점.

### 3.2 LangChain 텍스트 splitter

```python
# src/foundry_app/chunking.py
from langchain_text_splitters import RecursiveCharacterTextSplitter, MarkdownHeaderTextSplitter


def chunk_plain_text(text: str, chunk_size: int = 800, overlap: int = 150) -> list[str]:
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=overlap,
        separators=["\n\n", "\n", ". ", " ", ""],  # 우선순위 순
    )
    return splitter.split_text(text)


def chunk_markdown(md: str) -> list[dict]:
    """Markdown 구조(제목) 기반 분할."""
    headers = [("#", "h1"), ("##", "h2"), ("###", "h3")]
    md_splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers)
    docs = md_splitter.split_text(md)
    return [{"content": d.page_content, "metadata": d.metadata} for d in docs]
```

⚠️ **함정**: 표(Table) · 코드 블록은 중간에 잘리면 의미 완전 상실. Markdown-aware splitter 나 코드 fence(` ``` `) 인식 로직 필요.

---

## 4. Azure AI Search 개요

Azure AI Search = RAG 저장소 핵심. 단순 벡터 DB 가 아니라 **키워드 + 벡터 + 시맨틱 랭킹** 통합 검색 엔진.

### 4.1 핵심 개념

- **Index**: 데이터 스키마 (테이블 유사)
- **Vector Field**: 임베딩 저장. HNSW(Hierarchical Navigable Small World) 알고리즘으로 수백만 벡터 중 유사한 것을 빠르게 찾음
- **Hybrid Search**: BM25(키워드) + HNSW(벡터) 결과를 RRF(Reciprocal Rank Fusion) 알고리즘으로 병합. 오타는 벡터가, 고유명사는 키워드가 잡음
- **Semantic Ranker**: 🔴 상위 50개 결과를 대형 모델로 재랭킹. Standard tier 이상 별도 SKU

💡 **팁**: HNSW `m` · `efConstruction` 파라미터는 대부분 기본값으로 충분. 극단적 상황 아니면 튜닝 지양.

---

## 5. 🔧 실습 1 — Index 생성 (Python SDK)

```python
# src/foundry_app/create_index.py
"""Azure AI Search 인덱스 생성."""
from __future__ import annotations
import os
from azure.core.credentials import AzureKeyCredential
from azure.identity import DefaultAzureCredential
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import (
    SearchIndex, SearchField, SearchFieldDataType,
    VectorSearch, VectorSearchProfile, HnswAlgorithmConfiguration,
    SemanticConfiguration, SemanticPrioritizedFields, SemanticField, SemanticSearch,
)

SEARCH_ENDPOINT = os.environ["AZURE_SEARCH_ENDPOINT"]
INDEX_NAME = "foundry-course-index"

# Managed Identity 우선, 로컬은 az login fallback
credential = DefaultAzureCredential()
# 또는 개발용: credential = AzureKeyCredential(os.environ["AZURE_SEARCH_KEY"])

index_client = SearchIndexClient(endpoint=SEARCH_ENDPOINT, credential=credential)

index = SearchIndex(
    name=INDEX_NAME,
    fields=[
        SearchField(name="id", type=SearchFieldDataType.String, key=True),
        SearchField(name="content", type=SearchFieldDataType.String,
                    searchable=True, retrievable=True,
                    analyzer_name="ko.microsoft"),   # 한국어 분석기
        SearchField(name="category", type=SearchFieldDataType.String,
                    filterable=True, facetable=True),
        SearchField(name="source", type=SearchFieldDataType.String, retrievable=True),
        SearchField(
            name="content_vector",
            type=SearchFieldDataType.Collection(SearchFieldDataType.Single),
            searchable=True, vector_search_dimensions=1536,
            vector_search_profile_name="my-hnsw-profile",
        ),
    ],
    vector_search=VectorSearch(
        algorithms=[HnswAlgorithmConfiguration(name="my-hnsw")],
        profiles=[VectorSearchProfile(name="my-hnsw-profile", algorithm_configuration_name="my-hnsw")],
    ),
    semantic_search=SemanticSearch(configurations=[
        SemanticConfiguration(
            name="my-semantic",
            prioritized_fields=SemanticPrioritizedFields(
                content_fields=[SemanticField(field_name="content")],
            ),
        ),
    ]),
)

# 인덱스 생성 (또는 업데이트)
result = index_client.create_or_update_index(index)
print(f"Index 생성됨: {result.name}")
```

---

## 6. 🔧 실습 2 — 문서 인덱싱 파이프라인

```python
# src/foundry_app/ingest.py
"""문서 → 청킹 → 임베딩 → Azure AI Search 업로드."""
from __future__ import annotations
import os, uuid
from pathlib import Path
from azure.identity import DefaultAzureCredential
from azure.search.documents import SearchClient
from openai import AzureOpenAI
from langchain_text_splitters import RecursiveCharacterTextSplitter

SEARCH_ENDPOINT = os.environ["AZURE_SEARCH_ENDPOINT"]
INDEX_NAME = "foundry-course-index"
EMBEDDING_DEPLOYMENT = "text-embedding-3-small"

credential = DefaultAzureCredential()
search_client = SearchClient(SEARCH_ENDPOINT, INDEX_NAME, credential)
openai_client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    api_key=os.environ["AZURE_FOUNDRY_KEY"],
    api_version="2025-01-01-preview",
)


def embed_batch(texts: list[str]) -> list[list[float]]:
    response = openai_client.embeddings.create(model=EMBEDDING_DEPLOYMENT, input=texts)
    return [item.embedding for item in response.data]


def ingest_document(doc_path: Path, category: str) -> int:
    text = doc_path.read_text(encoding="utf-8")

    splitter = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=150)
    chunks = splitter.split_text(text)

    # 임베딩은 배치 처리 (100개씩 - API 제한 안전)
    documents = []
    for i in range(0, len(chunks), 100):
        batch = chunks[i : i + 100]
        vectors = embed_batch(batch)
        for chunk, vector in zip(batch, vectors):
            documents.append({
                "id": str(uuid.uuid4()),
                "content": chunk,
                "category": category,
                "source": doc_path.name,
                "content_vector": vector,
            })

    # 인덱스 업로드 (upsert)
    result = search_client.upload_documents(documents)
    succeeded = sum(1 for r in result if r.succeeded)
    print(f"{doc_path.name}: {succeeded}/{len(documents)} chunks 업로드 성공")
    return succeeded


if __name__ == "__main__":
    from pathlib import Path
    total = 0
    for md_file in Path("docs").glob("*.md"):
        total += ingest_document(md_file, category="Tech")
    print(f"총 {total} chunks 인덱싱 완료")
```

⚠️ **함정**:
- 임베딩 batch 크기는 API 마다 다름. `text-embedding-3-small` 은 최대 2048 items · 8192 tokens per item. 100씩 배치가 안전선.
- `search_client.upload_documents` 는 1000 documents/request 상한. 큰 batch 는 분할.
- 인덱싱 실패 개별 확인: `result[i].succeeded` · `result[i].error_message`.

---

## 7. 🔧 실습 3 — Hybrid Search 쿼리

```python
# src/foundry_app/search.py
"""Keyword + Vector + Semantic 병합 검색."""
from __future__ import annotations
import os
from azure.identity import DefaultAzureCredential
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery, QueryType
from openai import AzureOpenAI

SEARCH_ENDPOINT = os.environ["AZURE_SEARCH_ENDPOINT"]
INDEX_NAME = "foundry-course-index"

credential = DefaultAzureCredential()
search_client = SearchClient(SEARCH_ENDPOINT, INDEX_NAME, credential)
openai_client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    api_key=os.environ["AZURE_FOUNDRY_KEY"],
    api_version="2025-01-01-preview",
)


def hybrid_search(query: str, top_k: int = 5, category_filter: str | None = None) -> list[dict]:
    """Hybrid: BM25(query text) + Vector(query embedding) + Semantic reranker."""
    # 1) 질문을 벡터화
    query_vector = openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=[query],
    ).data[0].embedding

    # 2) Vector query 구성
    vector_query = VectorizedQuery(
        vector=query_vector,
        k_nearest_neighbors=top_k * 3,   # 벡터 검색은 넉넉히 뽑고 reranker 로 좁힘
        fields="content_vector",
    )

    # 3) Hybrid + Semantic Reranker
    results = search_client.search(
        search_text=query,   # BM25 keyword search
        vector_queries=[vector_query],
        query_type=QueryType.SEMANTIC,
        semantic_configuration_name="my-semantic",
        query_caption="extractive",       # 답변 발췌
        query_answer="extractive",        # 직접 답변 추출
        top=top_k,
        filter=f"category eq '{category_filter}'" if category_filter else None,
        select=["id", "content", "source", "category"],
    )

    return [
        {
            "score": r["@search.score"],
            "reranker_score": r.get("@search.reranker_score"),
            "content": r["content"],
            "source": r["source"],
        }
        for r in results
    ]


if __name__ == "__main__":
    hits = hybrid_search("Foundry Agent Service 는 뭘 하는가?", top_k=5, category_filter="Tech")
    for i, hit in enumerate(hits, 1):
        print(f"[{i}] reranker={hit['reranker_score']:.3f} source={hit['source']}")
        print(f"    {hit['content'][:100]}...")
```

💡 **팁**: `filter` 는 `$filter` OData 문법. 특정 카테고리·태그·날짜 범위로 검색 대상을 좁히면 성능·정확도 동시 개선.

---

## 8. 🔧 실습 4 — RAG 프롬프트 조립

검색 결과를 LLM 프롬프트에 grounding context 로 주입.

```python
# src/foundry_app/rag_chat.py
"""RAG 완전 예제: 검색 + 답변 생성."""
from __future__ import annotations
import os
from openai import AzureOpenAI
from foundry_app.search import hybrid_search   # 실습 3

openai_client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    api_key=os.environ["AZURE_FOUNDRY_KEY"],
    api_version="2025-01-01-preview",
)


SYSTEM_PROMPT = """너는 사내 문서 기반 답변 어시스턴트다.
아래 [Context] 만을 근거로 답한다.
Context 에 없는 내용이면 **"죄송하지만 해당 정보는 문서에서 찾지 못했습니다"** 라고만 답한다.
답변 끝에 참고 문서명을 반드시 명시한다.

[Context]
{context}
"""


def rag_answer(question: str, category: str | None = None) -> str:
    hits = hybrid_search(question, top_k=5, category_filter=category)
    if not hits:
        return "관련 문서를 찾지 못했습니다."

    context = "\n\n".join(
        f"--- {hit['source']} ---\n{hit['content']}" for hit in hits
    )

    response = openai_client.chat.completions.create(
        model=os.environ["AZURE_FOUNDRY_DEPLOYMENT"],
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT.format(context=context)},
            {"role": "user", "content": question},
        ],
        max_completion_tokens=1000,
        temperature=1.0,
    )
    return response.choices[0].message.content or ""


if __name__ == "__main__":
    answer = rag_answer("Foundry MCP 통합 언제 GA 됐어?", category="Tech")
    print(answer)
```

**RAG 프롬프트 엔지니어링의 핵심**: "**모르면 모른다고 답하라**" 강제. 이 한 줄이 hallucination 위험을 크게 줄인다.

---

## 9. LangChain 으로 고수준 추상화

`langchain-azure-ai` 로 RAG 파이프라인을 몇 줄로 요약.

```python
# src/foundry_app/langchain_rag.py
"""LangChain 을 사용한 RAG - 코드 절반으로 축약."""
import os
from langchain_azure_ai.chat_models import AzureAIChatCompletionsModel
from langchain_azure_ai.vectorstores import AzureSearchVectorStore
from langchain_azure_ai.embeddings import AzureAIEmbeddingsModel
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

embeddings = AzureAIEmbeddingsModel(
    endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    credential=os.environ["AZURE_FOUNDRY_KEY"],
    model="text-embedding-3-small",
)

vectorstore = AzureSearchVectorStore(
    search_endpoint=os.environ["AZURE_SEARCH_ENDPOINT"],
    search_credential=os.environ["AZURE_SEARCH_KEY"],
    index_name="foundry-course-index",
    embedding_function=embeddings,
)

llm = AzureAIChatCompletionsModel(
    endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    credential=os.environ["AZURE_FOUNDRY_KEY"],
    model=os.environ["AZURE_FOUNDRY_DEPLOYMENT"],
    api_version="2025-01-01-preview",
)

prompt = ChatPromptTemplate.from_messages([
    ("system", "너는 사내 문서 기반 어시스턴트다. 다음 컨텍스트만 근거로 답한다.\n\n{context}"),
    ("user", "{question}"),
])

# LangChain LCEL - 파이프 연결
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

answer = chain.invoke("Foundry MCP 통합 언제 GA 됐어?")
print(answer)
```

## 10. LlamaIndex 대안 (데이터 중심)

문서량이 크고 여러 인덱스를 다뤄야 하면 LlamaIndex 가 적합.

```python
# src/foundry_app/llamaindex_rag.py
import os
from llama_index.core import VectorStoreIndex, StorageContext, Settings
from llama_index.core.node_parser import SentenceSplitter
from llama_index.llms.azure_openai import AzureOpenAI as LlamaAzureOpenAI
from llama_index.embeddings.azure_openai import AzureOpenAIEmbedding
from llama_index.vector_stores.azureaisearch import AzureAISearchVectorStore
from azure.search.documents import SearchClient
from azure.core.credentials import AzureKeyCredential

Settings.llm = LlamaAzureOpenAI(
    deployment_name=os.environ["AZURE_FOUNDRY_DEPLOYMENT"],
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    api_key=os.environ["AZURE_FOUNDRY_KEY"],
    api_version="2025-01-01-preview",
)
Settings.embed_model = AzureOpenAIEmbedding(
    deployment_name="text-embedding-3-small",
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    api_key=os.environ["AZURE_FOUNDRY_KEY"],
    api_version="2025-01-01-preview",
)
Settings.node_parser = SentenceSplitter(chunk_size=800, chunk_overlap=150)

search_client = SearchClient(
    endpoint=os.environ["AZURE_SEARCH_ENDPOINT"],
    index_name="foundry-course-index",
    credential=AzureKeyCredential(os.environ["AZURE_SEARCH_KEY"]),
)

vector_store = AzureAISearchVectorStore(search_or_index_client=search_client)
storage_context = StorageContext.from_defaults(vector_store=vector_store)

index = VectorStoreIndex.from_vector_store(vector_store)
query_engine = index.as_query_engine(similarity_top_k=5)

response = query_engine.query("Foundry MCP 통합 언제 GA 됐어?")
print(response)
```

**LangChain vs LlamaIndex 선택 기준:**

| 상황 | 권장 |
|---|---|
| RAG + Agent + 다양한 통합 | LangChain |
| 대규모 문서 인덱싱 · 리서치 파이프라인 | LlamaIndex |
| 최소 dep · 저수준 제어 | Azure SDK 직접 (실습 1-4) |
| 하이브리드 챗봇 with tool calling | LangChain |

---

## 11. Foundry 'On Your Data' (Managed RAG)

코드 zero. Foundry 포털에서 데이터 소스 연결만 하면 RAG 챗봇 완성.

1. Foundry Portal → **Data** → **+ Add data source** → Blob / SharePoint / OneDrive / Fabric 선택
2. AI Search 자동 연결 → 청킹 · 임베딩 자동 수행
3. Python 앱은 `AzureChatExtensionConfiguration` 하나로 검색+답변 통합 호출:

```python
# src/foundry_app/on_your_data.py
import os
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_FOUNDRY_ENDPOINT"],
    api_key=os.environ["AZURE_FOUNDRY_KEY"],
    api_version="2025-01-01-preview",
)

response = client.chat.completions.create(
    model=os.environ["AZURE_FOUNDRY_DEPLOYMENT"],
    messages=[{"role": "user", "content": "회사 규정 몇 조에 휴가 관련 내용이 있어?"}],
    extra_body={
        "data_sources": [{
            "type": "azure_search",
            "parameters": {
                "endpoint": os.environ["AZURE_SEARCH_ENDPOINT"],
                "index_name": "company-policies",
                "authentication": {"type": "system_assigned_managed_identity"},
                "query_type": "semantic",
                "semantic_configuration": "default",
            },
        }],
    },
)
print(response.choices[0].message.content)
# 참고: response.choices[0].message.context 에 citations 포함됨
```

🔴 **2026 변경**: 'On Your Data' UI 개편으로 SharePoint / Fabric 실시간 sync 지원.

⚠️ **함정**: 관리형은 편리하나 청킹 크기 · 커스텀 임베딩 모델 조정 제한. 복잡한 로직 (다단계 검색, 조건부 filter) 필요하면 custom pipeline (실습 1-4).

---

## 12. 흔한 함정 정리

⚠️
1. **Chunk 크기**: 너무 크면 임베딩 부정확, 너무 작으면 컨텍스트 손실. **800 tokens + 150 overlap** 이 실무 기본.
2. **차원 mismatch**: 임베딩 모델 차원과 인덱스 벡터 차원 불일치 → 인덱싱 실패.
3. **HNSW 튜닝**: `m`·`efConstruction` 기본값이 대부분 최적. 섣부른 조정 금지.
4. **Semantic Ranker**: Standard tier 이상 별도 SKU. Free tier 사용 불가.
5. **'On Your Data' 관리형**: 편의 대신 커스터마이징 제한.
6. **Filter 없는 검색**: 수백만 chunk 중 상위 5개 뽑기 → 성능 저하 + 관련성 감소. category / date / user_id filter 는 항상 우선 고려.
7. **한국어 분석기**: `analyzer_name="ko.microsoft"` 필수. default 분석기는 한국어 형태소 처리 불가.

💡 **베스트 프랙티스**:
- 첫 배포: `text-embedding-3-small` 1536 차원 + 기본 HNSW + Semantic Ranker off → 나중에 필요시 upgrade.
- RAG 프롬프트에 항상 **"모르면 모른다"** + **출처 인용** 강제.
- Batch 임베딩으로 API 비용 절감 (single call > many small calls).
- Chunk metadata 에 `source`, `page`, `updated_at` 저장 → 사용자 UX 에 citation 노출.

---

## 요약 (Cheat Sheet)

- **RAG 흐름**: Retrieve (embedding 검색) → Augment (프롬프트 주입) → Generate (LLM 응답)
- **임베딩**: `text-embedding-3-small` (기본) or `-large` (고정확도). 차원 축소 가능
- **Chunking**: 800 tokens + 150 overlap, RecursiveCharacterTextSplitter 표준
- **Search**: Hybrid (BM25 + Vector) + Semantic Ranker 조합이 실무 정답
- **High-level**: LangChain (범용) · LlamaIndex (대규모 문서)
- **관리형**: Foundry 'On Your Data' (빠른 시작, 커스터마이징 제한)

## 📚 더 읽기

- [azure-search-documents Python SDK](https://learn.microsoft.com/en-us/python/api/overview/azure/search-documents-readme)
- [Azure OpenAI On Your Data](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/use-your-data)
- [Foundry Embedding Models](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/models#embeddings)
- [Hybrid Search + RRF 설명](https://learn.microsoft.com/en-us/azure/search/hybrid-search-ranking)
- [Semantic Ranking](https://learn.microsoft.com/en-us/azure/search/semantic-search-overview)
- [LangChain Azure 통합](https://python.langchain.com/docs/integrations/vectorstores/azuresearch)
- [LlamaIndex Azure AI Search](https://docs.llamaindex.ai/en/stable/examples/vector_stores/AzureAISearchIndexDemo/)

## 다음 챕터

[Ch.7 Foundry Agent Service (Responses API v2 + Semantic Kernel Python) →](Ch07_Agent_Service.md)

---
