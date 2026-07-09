# Chapter 6. RAG : Azure AI Search 연동

[← 목차로](README.md)

> **학습 목표**
> - RAG(Retrieval-Augmented Generation)의 기술적 메커니즘과 엔터프라이즈 환경에서의 비즈니스적 가치를 심도 있게 이해한다.
> - Embedding 모델의 아키텍처적 특성을 파악하고, 비용과 성능 사이의 최적화된 차원(Dimensions) 선택 전략을 수립한다.
> - 비정형 데이터의 특성에 따른 정교한 문서 분할(Chunking) 기법을 익히고 실무 수준의 인덱싱 파이프라인을 구축한다.
> - Azure AI Search의 Hybrid Search(Vector + Keyword + Semantic) 기능을 구현하고, 검색 품질을 극대화하는 튜닝 기술을 습득한다.
> - Foundry 'On Your Data'를 활용한 서버리스 RAG 구현 방법과 커스텀 파이프라인으로의 전환 시점을 정확히 판단한다.

> **전제 조건**
> - Chapter 3에서 구성한 Java 개발 환경 (JDK 21, Maven 3.9+)
> - Foundry 프로젝트에서 text-embedding-3-small 및 gpt-5-mini 모델 배포 완료
> - Azure AI Search 서비스(Standard SKU 이상 권장) 및 API Key 확보
> - 기본 Java 객체 지향 프로그래밍 및 RESTful API 통신에 대한 숙련도

---

## 1. RAG가 왜 필요한가: AI 지식의 한계 극복과 신뢰성 구축

LLM(Large Language Model)은 인류의 방대한 공용 데이터를 학습하여 일반적인 대화와 논리적 추론에는 매우 능숙하다. 하지만 기업 현장에서 LLM을 실무에 투입하려고 할 때, 우리는 두 가지 거대한 장벽에 직면하게 된다.

첫째는 **지식 컷오프(Knowledge Cut-off)** 현상이다. 아무리 강력한 모델이라도 학습 데이터가 수집된 시점 이후의 세상은 알지 못한다. 예를 들어 어제 발표된 새로운 기술 규정이나 오늘 아침에 바뀐 제품의 가격 정보에 대해 모델은 알 수 없으며, 심지어는 모르는 내용을 사실인 것처럼 꾸며내는 할루시네이션(Hallucination, 환각)을 일으켜 비즈니스에 치명적인 오류를 초래할 수 있다. 할루시네이션은 모델이 다음 단어를 확률적으로 예측하는 특성 때문에 발생하며, 정보가 부족할수록 확률적으로 '가장 그럴듯한' 오답을 선택하게 된다. RAG는 이러한 모델의 내부적 한계를 외부의 검증된 데이터로 보완하는 기술이다.

둘째는 **데이터 격리(Data Isolation)** 문제이다. 기업의 경쟁력은 외부에 공개되지 않은 사내 규정, 미공개 프로젝트 보고서, 기술 설계도, 그리고 고객의 개인 정보에서 나온다. 이러한 데이터는 보안상의 이유로 LLM의 기본 학습 데이터에 포함될 수 없는 폐쇄된 영역의 정보이다. 모델은 이 데이터를 읽어본 적이 없으므로, 우리 회사만의 독특한 프로세스나 전문 용어에 대해 정확하게 답할 수 없다.

이러한 한계를 극복하기 위해 모델 자체를 파인튜닝(Fine-tuning)하는 방식이 논의되기도 하지만, 지식이 업데이트될 때마다 수만 달러의 비용과 수주의 시간을 들여 모델을 재학습시키는 것은 현실적으로 불가능하다. **RAG(Retrieval-Augmented Generation, 검색 증강 생성)**는 모델을 재학습시키지 않고, 모델에게 '참고 자료가 가득한 디지털 도서관'을 제공하는 방식이다. 사용자의 질문이 들어올 때마다 관련 있는 문서 조각을 실시간으로 찾아내어 모델의 입력 프롬프트에 끼워 넣어주는 이 기법은, 모델이 항상 최신이고 정확한 근거를 바탕으로 답변할 수 있도록 보장한다. RAG는 모델을 학습시키는 대신 '검색 능력을 갖춘 비서'로 변모시키는 전략이며, 이는 오늘날 엔터프라이즈 AI 아키텍처의 표준으로 자리 잡았다.

### 1.1 RAG의 주요 비즈니스 유스케이스

1. **지능형 고객 지원 및 셀프 서비스**: 최신 제품 업데이트 내역, 서비스 약관, FAQ 데이터를 기반으로 24시간 정확한 답변을 제공한다. 이는 고객 만족도를 높이는 동시에 상담원의 단순 반복 업무를 획기적으로 줄여준다. 상담원은 이제 복잡한 감정적 케어나 특수 사례에만 집중할 수 있게 되어 전체적인 서비스 품질이 향상된다.
2. **사내 지식 자산 통합 검색 (KM)**: 위키 시스템, 슬랙 대화록, PDF 기획서, 엑셀 명세서 등에 파편화된 정보를 통합 검색한다. 신입 사원이 "우리 회사의 원격 근무 비용 청구 규정이 뭐야?"라고 물으면 여러 사규를 조합하여 즉시 답변과 함께 출처 링크를 제공한다. 이는 불필요한 사내 문의 시간을 줄여 조직의 생산성을 높인다.
3. **전문 분야의 심층 문서 분석**: 법률 계약서 검토, 의료 논문 요약, 기술 표준 분석 등 고도의 전문성이 요구되는 분야에서 수천 페이지의 문서를 단 몇 초 만에 분석한다. 사람은 모델이 찾아낸 핵심 구절만 최종 검토함으로써 업무 효율성을 극대화할 수 있다. 이는 단순한 키워드 검색을 넘어 고차원적인 지식 추출과 논리적 요약의 영역이다.

### 1.2 RAG의 3단계 메커니즘 (Retrieve : Augment : Generate)

- **Retrieve (검색)**: 사용자의 자연어 질문을 벡터로 변환한 뒤, 사전에 구축된 검색 인덱스에서 의미적으로 가장 관련성이 높은 상위 K개의 문서 조각(Chunk)을 선별한다. 이때 단순히 텍스트가 겹치는 것을 넘어 질문의 '의도'와 '맥락'을 파악하는 것이 기술적 핵심이다.
- **Augment (강화)**: 검색된 문서 조각들을 모델에게 전달할 시스템 메시지나 프롬프트 사이에 주입(Injection)한다. 이때 단순한 텍스트 나열이 아니라, 모델이 질문과 참고 자료를 명확히 구분할 수 있도록 메타데이터(문서 제목, 날짜, 작성자, 출처 URL 등)와 함께 구조화하여 전달한다.
- **Generate (생성)**: LLM은 자신의 사전 학습 지식이 아닌, 바로 눈앞에 제시된 문서들(Context)을 근거로 삼아 답변을 생성한다. "제공된 정보 내에서만 답변하고 정보가 없으면 모른다고 하라"는 지침을 통해 답변의 신뢰도를 확보한다. 모델은 이제 '창작자'가 아니라 '엄격한 요약자 및 전달자'의 역할을 수행하게 된다.

---

## 2. Embeddings: 텍스트를 기계가 이해하는 좌표로 변환하기

인간은 단어의 맥락과 감정을 이해하지만, 컴퓨터는 오직 숫자만을 처리한다. 따라서 RAG를 구현하기 위해서는 텍스트를 숫자의 나열인 벡터(Vector)로 변환하는 과정이 필요하다. 이를 **임베딩(Embedding)**이라고 한다.

임베딩 기술의 핵심은 유사한 의미를 가진 텍스트들이 고차원 벡터 공간 내에서 서로 가까운 거리에 위치하게 만드는 것이다. 예를 들어 '축구'와 '야구'는 '스포츠'라는 상위 개념에서 가깝게 위치하고, '축구'와 '미분방정식'은 공간상에서 매우 멀리 위치하게 된다. 이 거리 계산은 주로 코사인 유사도(Cosine Similarity)를 통해 이루어지며, 이는 두 벡터 사이의 각도를 측정하여 의미적 유사성을 판별한다. 각도가 작을수록 두 텍스트는 비슷한 의미를 담고 있다는 뜻이다.

### 2.1 Foundry의 최신 임베딩 모델 라인업 (2026 기준)

Foundry Model Catalog에서는 성능과 운영 비용을 고려하여 선택할 수 있는 다양한 임베딩 모델을 제공한다. 모델을 선택할 때는 데이터의 언어적 특성, 평균 문장 길이, 검색 대상의 규모를 종합적으로 고려해야 한다.

| 모델 카테고리 | 모델명 | 차원 (Dim) | 주요 특징 및 비즈니스 추천 용도 |
|---|---|---|---|
| **Standard** | **text-embedding-3-small** | 1536 | 속도와 정확도의 완벽한 조화. 대부분의 일반적인 RAG 시나리오와 대규모 인덱싱 작업에 최적화됨. 한국어 형태소와 문맥 이해 성능도 매우 안정적임. |
| **High-End** | **text-embedding-3-large** | 3072 | 최고 수준의 의미 파악 능력 제공. 법률, 의료 등 정밀한 검색이 필수적인 전문 도메인 추천. 단, 인덱스 저장 비용이 small 모델 대비 높음. |
| **Multimodal** | **text-embedding-4-preview** | 2048 | 🔴 **2026 변경**: 이미지와 텍스트를 동일 공간에 매핑 가능. 그림이나 도표가 많은 기술 문서 검색에 유리. OCR 없이도 이미지의 시각적 문맥을 바로 이해함. |

### 2.2 차원 결정 전략과 마트료시카 학습 (MRL)

	ext-embedding-3 계열 모델은 **마트료시카 표현 학습(Matryoshka Representation Learning)**이라는 혁신적인 기법을 적용했다. 이는 1536차원으로 생성된 벡터의 앞부분 일부(예: 256차원, 512차원)만 잘라내어 사용하더라도 검색 품질이 급격히 떨어지지 않도록 설계된 방식이다. 러시아의 마트료시카 인형처럼, 커다란 벡터 안에 작은 핵심 벡터가 중첩되어 포함되어 있다고 이해하면 쉽다.

벡터의 차원을 줄이면 Azure AI Search의 저장 용량이 획기적으로 줄어들고 검색 속도가 빨라진다. 만약 수억 건의 문서를 인덱싱해야 하는 대규모 프로젝트라면, 1536차원을 그대로 쓰기보다 내부적인 성능 테스트를 거쳐 512차원 정도로 축소하여 사용하는 것이 경제적으로 매우 유리하다. 이는 인프라 비용 절감의 핵심 기술이다.

⚠️ **함정**: 검색 인덱스를 한 번 생성하면 그 인덱스 내의 벡터 필드 차원(Dimensions) 수는 고정된다. 만약 1536차원 인덱스를 운영하다가 더 정밀한 3072차원 모델로 변경하고 싶다면, 기존 인덱스를 모두 삭제하고 전 문서를 다시 임베딩하여 넣어야 하는 **재인덱싱(Re-indexing)** 작업이 필요하다. 따라서 프로젝트 초기 설계 단계에서 향후 확장성과 성능 요구치를 고려하여 차원을 신중히 결정해야 한다.

---

## 3. Chunking 전략: 검색 품질을 좌우하는 데이터 설계

문서 한 권을 통째로 임베딩하여 저장하면 해당 문서의 구체적인 세부 정보는 희석되고 전체적인 주제나 톤만 벡터에 남게 된다. 사용자의 세밀한 질문에 정확히 답변하려면 문서를 작은 조각(Chunk)으로 잘라야 한다. 이 **청킹(Chunking)** 과정은 RAG 성능의 70퍼센트 이상을 결정짓는 핵심 단계이다. 청킹이 잘못되면 아무리 비싼 검색 엔진을 써도 정확한 결과를 얻을 수 없다.

### 3.1 실무에서 자주 사용되는 분할 전략

- **고정 크기 분할 (Fixed-size Chunking)**: 단순히 500자나 1000자 단위로 자르는 방식이다. 구현이 가장 간단하지만 중요한 의미를 담은 문장이 중간에 잘려 나가는 치명적인 단점이 있다. 예를 들어 주어가 앞 청크에 있고 서술어가 뒤 청크에 있으면 검색 시 정보가 깨지게 된다.
- **재귀적 문자 분할 (Recursive Character Splitting)**: 줄바꿈, 마침표, 공백 등의 순서로 우선순위를 설정하여 최대한 문맥이 유지되는 지점에서 자른다. 예를 들어 문단 단위로 먼저 자르고 그 안에서 문장 단위로 자르는 식이다. 현재 업계에서 가장 널리 쓰이는 표준적인 방식이다.
- **의미론적 분할 (Semantic Chunking)**: 문장들 간의 임베딩 유사도를 실시간으로 계산하여 의미가 급격히 변화하는 지점을 분할점으로 삼는다. 가장 정교하지만 청킹 과정 자체에 모델 호출 비용이 발생하므로 대규모 문서 처리 시에는 예산 계획을 잘 세워야 한다.

### 3.2 Overlap(중첩)의 필요성

청크를 나눌 때 앞 조각의 마지막 부분과 뒷 조각의 시작 부분이 서로 겹치도록(Overlap) 설계해야 한다. 예를 들어 800 tokens 크기의 청크를 생성할 때 마지막 150 tokens 정도를 다음 청크의 시작 부분에 포함시킨다. 이는 정보가 잘리는 경계면에서 발생할 수 있는 정보 단절 현상을 방지하여, 검색된 결과가 질문에 답하기 위한 충분한 전후 사정을 포함할 수 있게 한다. 중첩이 없으면 모델은 맥락의 절반만 보고 답변을 추측해야 하는 위험한 상황에 처할 수 있다.

⚠️ **함정**: 청킹의 크기가 너무 크면 임베딩 벡터가 너무 많은 정보를 담으려다 검색 정확도가 떨어지고, 반대로 너무 작으면 필요한 주변 문맥(Context)을 소실하여 답변 생성이 불가능해진다. 실무에서는 보통 500자에서 1000자 사이에서 실험을 통해 최적의 값을 찾는다. 또한 문서의 종류(사용자 매뉴얼, 사내 규정집, 기획서)에 따라 최적의 청킹 크기는 모두 다를 수 있으므로 데이터 특성을 파악해야 한다.

💡 **팁**: 표(Table)나 수식이 포함된 문서는 텍스트로만 청킹하면 데이터 구조가 완전히 파괴된다. 이런 경우 Microsoft Document Intelligence와 같은 도구를 병행하여 문서를 Markdown이나 레이아웃 인식 구조로 먼저 변환한 뒤, 구조를 유지하며 분할하는 것이 검색 품질 향상의 지름길이다. 특히 엑셀 데이터는 행과 열의 의미가 보존되도록 텍스트화하는 전처리가 필수적이다.

---

## 4. Azure AI Search 개요: 엔터프라이즈 통합 검색의 완성

Azure AI Search는 단순한 벡터 데이터베이스가 아니라, 키워드 검색 엔진의 강점과 벡터 검색의 유연함을 결합한 완전 관리형 서비스이다. 보안, 확장성, 가용성 면에서 엔터프라이즈 환경에 가장 적합한 검색 인프라이다.

### 4.1 Hybrid Search: 최고의 정확도를 위한 결합

현업에서 가장 높은 만족도를 보이는 방식은 키워드 검색과 벡터 검색을 동시에 수행하고 그 결과를 병합하는 **하이브리드 검색**이다.

1. **키워드 검색 (BM25)**: 'GTX-3080', 'HR-2024-001'과 같은 고유 명사나 제품 번호를 정확히 찾아내는 데 탁월하다. 벡터 검색은 유사한 숫자를 찾으려다 엉뚱한 제품명을 가져올 수 있지만, 키워드 검색은 정확한 문자 매칭을 보장한다.
2. **벡터 검색 (HNSW)**: "성능 좋은 노트북 추천해줘"와 같은 사용자 의도와 맥락을 파악하여 관련 문서를 찾는다. 질문에 포함된 단어가 문서에 직접 없더라도 의미적으로 통하는 내용을 찾아준다.
3. **시맨틱 랭커 (Semantic Ranker)**: 🔴 **2026 변경**: 검색된 결과들을 Microsoft의 튜닝된 딥러닝 모델이 다시 한 번 정독하고, 질문과의 실질적 관련성 순위로 재정렬한다. 이는 마치 전문가가 검색 결과 50개를 직접 읽고 가장 정답에 가까운 것을 맨 위로 올려주는 것과 같은 효과를 낸다.

### 4.2 인덱스 형태와 분석기 (Analyzer)

Azure AI Search에서는 필드별로 분석기(Analyzer)를 설정할 수 있다. 한국어 문서의 경우 Microsoft에서 제공하는 ko.lucene 또는 ko.microsoft 분석기를 사용하면 형태소 분석을 통해 더욱 정교한 검색이 가능하다. 예를 들어 '가방을'과 '가방이'를 동일한 '가방'이라는 명사 키워드로 인식하여 검색해준다.

⚠️ **함정**: 시맨틱 랭커는 매우 강력하지만 유료 옵션이며, Azure Search 서비스의 SKU가 Standard 이상이어야만 활성화가 가능하다. 무료(Free) 티어에서는 사용할 수 없으며 지역별 가용성도 다르므로 배포 전 반드시 확인해야 한다. 시맨틱 랭커를 활성화했을 때와 비활성화했을 때의 검색 품질 차이는 실제 상용 서비스 수준에서 확연하게 나타난다.

---

## 🔧 실습 1 — Index 생성 (Azure CLI)

인덱스는 데이터가 저장될 테이블의 구조를 정의하는 과정이다. Java 코드로 데이터를 넣기 전, CLI를 통해 벡터 검색 프로필과 필드를 구성한다. 인덱스 생성 시에는 검색 가능성(Searchable), 필터 가능성(Filterable), 정렬 가능성(Sortable) 등을 명확히 정의해야 한다.

`ash
# 1. 환경 변수 준비
SERVICE_NAME="my-foundry-search-service"
ADMIN_KEY="your-search-admin-key"

# 2. 인덱스 생성 (REST API 호출)
# 벡터 검색 구성(HNSW)과 시맨틱 설정을 포함한 JSON 전송
curl -X POST "https://.search.windows.net/indexes?api-version=2024-07-01" \
  -H "Content-Type: application/json" \
  -H "api-key: " \
  -d '{
    "name": "enterprise-kb-index",
    "fields": [
      { "name": "id", "type": "Edm.String", "key": true, "searchable": false },
      { "name": "title", "type": "Edm.String", "searchable": true, "retrievable": true },
      { "name": "content", "type": "Edm.String", "searchable": true, "retrievable": true, "analyzer": "ko.microsoft" },
      { "name": "category", "type": "Edm.String", "filterable": true, "facetable": true },
      {
        "name": "content_vector",
        "type": "Collection(Edm.Single)",
        "searchable": true,
        "retrievable": true,
        "dimensions": 1536,
        "vectorSearchProfile": "my-vector-profile"
      }
    ],
    "vectorSearch": {
      "algorithms": [ { "name": "my-hnsw", "kind": "hnsw" } ],
      "profiles": [ { "name": "my-vector-profile", "algorithm": "my-hnsw" } ]
    },
    "semantic": {
      "configurations": [
        {
          "name": "default-semantic-config",
          "prioritizedFields": {
            "titleField": { "fieldName": "title" },
            "contentFields": [ { "fieldName": "content" } ]
          }
        }
      ]
    }
  }'
`

---

## 🔧 실습 2 — Java 인덱싱 파이프라인 개발

문서를 읽고, 임베딩을 생성한 뒤 Azure AI Search 인덱스에 저장하는 전체 파이프라인을 Java로 구현한다. zure-search-documents 12.0.0 라이브러리를 사용하며, 이는 엔터프라이즈 환경에서 안정성이 검증된 버전이다.

### pom.xml 설정 (검색 모듈 추가)

Ch.3 프로젝트의 pom.xml에 다음 의존성을 추가한다. 검색 서비스와 통신하기 위해 필요한 핵심 라이브러리이다.

`xml
<!-- azure-search-documents: Enterprise Search SDK -->
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-search-documents</artifactId>
    <version>12.0.0</version>
</dependency>
`

### 인덱싱 서비스 코드

이 코드는 OpenAI API를 호출하여 임베딩을 생성하고, 이를 Search SDK를 통해 인덱스에 저장한다. 실무에서는 대량의 문서를 처리하기 위해 배치(Batch) 업로드 기능을 주로 사용한다.

`java
// IndexerService.java
package com.example;

import com.azure.ai.openai.OpenAIClient;
import com.azure.ai.openai.OpenAIClientBuilder;
import com.azure.ai.openai.models.Embeddings;
import com.azure.ai.openai.models.EmbeddingsOptions;
import com.azure.core.credential.AzureKeyCredential;
import com.azure.search.documents.SearchClient;
import com.azure.search.documents.SearchClientBuilder;

import java.util.*;

public class IndexerService {
    public static void main(String[] args) {
        // 환경 변수 로드 (Endpoint 및 Key)
        String openaiUrl = System.getenv("AZURE_FOUNDRY_ENDPOINT");
        String openaiKey = System.getenv("AZURE_FOUNDRY_KEY");
        String searchUrl = System.getenv("AZURE_SEARCH_ENDPOINT");
        String searchKey = System.getenv("AZURE_SEARCH_KEY");

        if (openaiUrl == null || searchUrl == null) {
            System.err.println("환경변수 설정을 확인하세요.");
            return;
        }

        // 1. OpenAI 임베딩 클라이언트 설정
        OpenAIClient openAIClient = new OpenAIClientBuilder()
            .endpoint(openaiUrl)
            .credential(new AzureKeyCredential(openaiKey))
            .buildClient();

        // 2. AI Search 클라이언트 설정
        SearchClient searchClient = new SearchClientBuilder()
            .endpoint(searchUrl)
            .credential(new AzureKeyCredential(searchKey))
            .indexName("enterprise-kb-index")
            .buildClient();

        // 가공된 청크 데이터 예시
        String chunkId = UUID.randomUUID().toString();
        String title = "Foundry RAG 연동 가이드 2026";
        String content = "Azure AI Search를 사용하면 대규모 텍스트 데이터를 벡터화하여 하이브리드 검색을 구현할 수 있습니다. 이는 최신 Foundry 아키텍처의 핵심입니다.";
        
        System.out.println("임베딩 벡터를 생성하는 중...");
        
        // 3. 텍스트를 고차원 벡터로 변환 (text-embedding-3-small)
        EmbeddingsOptions embeddingOptions = new EmbeddingsOptions(Collections.singletonList(content));
        Embeddings embeddings = openAIClient.getEmbeddings("text-embedding-3-small", embeddingOptions);
        List<Float> vector = embeddings.getData().get(0).getEmbedding();

        // 4. 검색 엔진에 저장할 문서 맵 생성
        Map<String, Object> document = new HashMap<>();
        document.put("id", chunkId);
        document.put("title", title);
        document.put("content", content);
        document.put("category", "Guide");
        document.put("content_vector", vector);

        // 5. 인덱싱 실행 (Upsert)
        try {
            searchClient.uploadDocuments(Collections.singletonList(document));
            System.out.println("인덱싱이 성공적으로 완료되었습니다. ID: " + chunkId);
        } catch (Exception e) {
            System.err.println("인덱싱 실패: " + e.getMessage());
        }
    }
}
`

⚠️ **함정**: SearchClient를 사용하여 문서를 업로드할 때, 업로드한 데이터의 필드 이름이 인덱스 정의서와 정확히 대소문자까지 일치해야 한다. 또한 content_vector 필드에 들어가는 값은 반드시 List<Float> 형식이어야 하며, Double 형식을 넣으면 런타임 오류가 발생한다. 데이터 업로드 시에는 1,000개 단위의 배치(Batch)를 사용하여 네트워크 오버헤드를 줄이는 것이 권장된다.

---

## 🔧 실습 3 — 하이브리드 검색 쿼리 구현

검색은 질문을 벡터화하여 가장 관련성 높은 조각을 찾아오는 과정이다. 키워드 필터와 시맨틱 정렬을 결합하여 최상의 품질을 끌어낸다. 질문의 임베딩과 검색 인덱스의 벡터 사이의 거리를 계산하여 가장 유사한 결과를 가져온다.

`java
// SearchApp.java
package com.example;

import com.azure.ai.openai.OpenAIClient;
import com.azure.ai.openai.OpenAIClientBuilder;
import com.azure.ai.openai.models.Embeddings;
import com.azure.ai.openai.models.EmbeddingsOptions;
import com.azure.core.credential.AzureKeyCredential;
import com.azure.search.documents.SearchClient;
import com.azure.search.documents.SearchClientBuilder;
import com.azure.search.documents.models.*;
import com.azure.search.documents.util.SearchPagedIterable;

import java.util.*;

public class SearchApp {
    public static void main(String[] args) {
        String userQuestion = "Foundry 검색 엔진의 하이브리드 검색 설정 방법을 구체적으로 알려줘.";

        OpenAIClient openAIClient = new OpenAIClientBuilder()
            .endpoint(System.getenv("AZURE_FOUNDRY_ENDPOINT"))
            .credential(new AzureKeyCredential(System.getenv("AZURE_FOUNDRY_KEY")))
            .buildClient();

        SearchClient searchClient = new SearchClientBuilder()
            .endpoint(System.getenv("AZURE_SEARCH_ENDPOINT"))
            .credential(new AzureKeyCredential(System.getenv("AZURE_SEARCH_KEY")))
            .indexName("enterprise-kb-index")
            .buildClient();

        // 1. 질문을 임베딩 벡터로 변환
        Embeddings embeddings = openAIClient.getEmbeddings("text-embedding-3-small", 
            new EmbeddingsOptions(Collections.singletonList(userQuestion)));
        List<Float> queryVector = embeddings.getData().get(0).getEmbedding();

        // 2. 검색 옵션 구성
        SearchOptions options = new SearchOptions()
            .setVectorQueries(new VectorizedQuery(queryVector)
                .setFields("content_vector")
                .setKNearestNeighborsCount(5)) // 상위 5개 벡터 검색
            .setFilter("category eq 'Guide'") // 카테고리 필터링으로 검색 범위 제한
            .setTop(10); // 결과 상위 10개 반환 (시맨틱 랭커용)
        
        // 🔴 2026 변경: 시맨틱 랭커 적용 (인덱스 생성 시 설정한 이름 사용)
        options.setSemanticSearchOptions(new SemanticSearchOptions()
            .setSemanticConfigurationName("default-semantic-config"));

        // 3. 통합 검색 실행
        System.out.println("검색 쿼리 실행 중...");
        SearchPagedIterable results = searchClient.search(userQuestion, options, null);

        System.out.println("=== 검색 결과 리스트 ===");
        for (SearchResult result : results) {
            Map<String, Object> doc = result.getDocument(Map.class);
            
            // 기본 검색 점수 (RRF 알고리즘 기반 병합 점수)
            System.out.printf("[%f] %s\n", result.getScore(), doc.get("title"));
            System.out.println("요약 내용: " + doc.get("content"));
            
            // 시맨틱 랭커가 동작한 경우 재계산된 점수 출력
            if (result.getSemanticSearch() != null && result.getSemanticSearch().getRerankerScore() != null) {
                System.out.println("  -> 시맨틱 재정렬 점수 (0-4점): " + result.getSemanticSearch().getRerankerScore());
            }
            System.out.println("-----------------------------------");
        }
    }
}
`

💡 **팁**: 검색 결과의 품질이 만족스럽지 않다면 setKNearestNeighborsCount 값을 조정하여 벡터 검색의 비중을 높이거나, 시맨틱 랭커의 설정을 다시 점검하라. 또한 $filter 식을 활용하여 "최근 1년 이내의 문서" 등으로 범위를 좁히는 것이 성능 향상의 핵심이다. 검색 로그를 모니터링하여 사용자가 어떤 검색어에서 실패하는지 추적하는 습관이 중요하다.

---

## 5. RAG 프롬프트 조립: 검색된 지식의 정제와 주입

검색된 문서 조각들을 LLM에게 전달할 때는 모델이 참고 자료와 사용자의 질문을 명확히 구분하여 답변할 수 있도록 정교한 프롬프트를 설계해야 한다. 이를 **프롬프트 그라운딩(Grounding)**이라고 부른다. 모델이 자신의 기억과 눈앞의 자료 중 무엇을 우선시해야 할지 알려주는 과정이다.

### 5.1 시스템 메시지 설계 전략

단순히 텍스트를 붙여넣는 것이 아니라, LLM이 검색 결과에만 의존하여 답변하도록 강제하는 제약 조건을 명시해야 한다. 모델이 멋대로 지식을 지어내는 것을 방지하기 위함이다.

`java
// PromptGenerator.java
String userQuery = "Foundry 검색 엔진 구축 방법을 단계별로 요약해줘.";
List<String> retrievedDocuments = Arrays.asList(
    "1단계: Azure AI Search 인덱스를 생성합니다. 이때 벡터 필드와 하이브리드 설정을 포함합니다.",
    "2단계: Java SDK를 사용해 문서를 임베딩하고 SearchClient로 데이터를 업로드합니다.",
    "3단계: 시맨틱 랭커를 활성화하여 검색 품질을 최종적으로 튜닝합니다."
);

// 검색 결과 리스트를 하나의 컨텍스트 텍스트 블록으로 결합
String contextBlock = String.join("\n", retrievedDocuments);

String finalSystemPrompt = String.format("""
    당신은 기업용 AI 지식 지원 전문가입니다.
    반드시 아래 제공된 [Context] 섹션의 정보만을 절대적인 근거로 사용하여 사용자의 질문에 답변하세요.
    질문에 대한 답을 [Context] 내에서 찾을 수 없는 경우에는 아는 척하지 말고 "죄송하지만 제공된 문서에 해당 내용이 존재하지 않습니다"라고 답변하세요.
    모든 답변은 한국어로 작성하고, 출처가 된 문서의 번호나 요소를 명시하세요.
    답변은 불필요한 서술 없이 간결하고 명확하게 작성하세요.

    [Context]
    %s

    [Question]
    %s
    """, contextBlock, userQuery);
`

⚠️ **함정**: 검색 결과가 너무 많거나 내용이 길어지면 LLM의 최대 입력 토큰 수(Context Window)를 초과하여 API 호출이 실패하거나 비용이 과다 발생할 수 있다. 또한 질문과 관련 없는 정보(Noise)가 섞여 들어가면 모델이 핵심을 놓치고 엉뚱한 답을 하는 **Lost in the Middle** 현상이 발생한다. 실무에서는 상위 3개에서 5개의 핵심 조각만 선별하여 제공하는 것이 가장 높은 답변 품질을 유지하는 비결이다.

---

## 6. Foundry "On Your Data": 관리형 RAG의 활용

모든 인덱싱 파이프라인을 직접 코딩하는 방식은 정교한 제어가 가능하지만 구축 시간이 오래 걸린다. Foundry 포털의 **On Your Data** 기능을 사용하면 코드를 거의 짜지 않고도 고성능 RAG 시스템을 구축할 수 있다. 이는 마이크로소프트가 관리하는 통합 검색 인프라를 사용하는 방식이다.

### 6.1 관리형 서비스 vs 커스텀 파이프라인 전환 시점

어느 방식을 선택할지는 프로젝트의 복잡성과 유지보수 역량에 달려 있다.

| 비교 항목 | Foundry On Your Data (Managed) | Custom Pipeline (Java SDK) |
|---|---|---|
| **구축 난이도** | 매우 낮음 (포털에서 클릭만으로 완료) | 높음 (청킹, 임베딩 로직 직접 구현) |
| **청킹 제어** | 기본 제공 옵션만 사용 가능 (제한적) | 완전한 제어 (비즈니스 로직에 따른 분할 가능) |
| **업데이트 주기** | 데이터 소스(Blob 등) 변경 시 자동 동기화 | 인덱싱 배치를 개발자가 직접 실행해야 함 |
| **추천 시나리오** | 빠른 프로토타이핑, 표준적인 사내 문서 QA 서비스 | 복잡한 데이터 구조, 다단계 검색, 하위 질문 분해 시스템 |

🔴 **2026 변경**: 2026년 기준 On Your Data 서비스는 **자동 동기화(One-Click Sync)** 기능을 통해 SharePoint나 Microsoft Fabric의 데이터 변경 사항을 실시간으로 감지하여 검색 인덱스를 갱신한다. 또한 별도의 인덱싱 인프라 없이도 서버리스 벡터화 기능을 통해 데이터가 업로드되는 즉시 자동으로 임베딩이 완료된다. 포털 UI 역시 개편되어 인덱싱 상태를 실시간 대시보드로 확인할 수 있다.

⚠️ **함정**: 관리형 'On Your Data' 기능을 Java SDK에서 호출할 때는 일반적인 Chat Completions 요청과 형식이 다르다. AzureChatExtensionConfiguration이라는 특수한 확장 구성을 사용해야 하므로, SDK 버전이 바뀔 때마다 파라미터 구조가 변경되었는지 릴리스 노트를 반드시 확인해야 한다. 또한 관리형 방식은 내부 임베딩 모델의 버전을 고정하기 어려울 수 있어, 정교한 버전을 관리해야 하는 엔터프라이즈 환경에서는 커스텀 파이프라인이 여전히 선호된다.

---

## 요약 (Cheat Sheet)

- **RAG**: LLM의 한정된 지식을 외부 검색 엔진(Azure AI Search)으로 보완하여 답변의 신뢰성을 높이는 아키텍처이다.
- **임베딩**: 텍스트를 숫자로 바꾸는 과정이며 	ext-embedding-3-small은 속도와 가성비 면에서 표준으로 사용된다.
- **청킹**: 문서를 의미 있는 단위로 자르는 과정이며 문맥 소실을 막기 위해 10퍼센트에서 20퍼센트의 중첩 구간을 반드시 설정하라.
- **하이브리드 검색**: 키워드(정확성)와 벡터(의미)의 조합은 검색 실패 확률을 최소화하는 현존 최선의 검색 전략이다.
- **시맨틱 랭커**: 검색 결과의 순위를 최종적으로 재조정하는 기능으로, 사용자 만족도를 결정짓는 핵심 유료 옵션이다.

---

## 💡 팁
1. **임베딩 캐싱 전략**: 동일한 문서가 반복적으로 업데이트될 때 매번 임베딩 API를 호출하는 것은 비용 낭비이다. 텍스트의 해시값을 키로 하는 캐시 레이어를 구축하여 이미 생성된 벡터를 재사용하라. 이는 초기 대량 인덱싱 비용을 70퍼센트 이상 줄여준다.
2. **HNSW 튜닝**: 인덱스 생성 시 efConstruction 값을 높이면 인덱싱 시간은 늘어나지만 검색의 리콜(Recall) 성능이 향상된다. 실시간성이 중요하지 않은 대규모 정적 문서 데이터베이스에서 유용한 팁이다.
3. **평가용 쿼리 셋 구축**: 검색 품질은 주관적이다. 사용자가 할 법한 질문 50개를 미리 정의하고, 인덱싱 전략이나 가중치를 변경할 때마다 상위 결과의 변화를 점수화하여 객관적으로 관리하라. 이를 위한 평가 지표로 nDCG나 MRR을 사용하면 더욱 과학적인 접근이 가능하다.

---

## 📚 더 읽기

- [Azure AI Search Java SDK 공식 가이드 및 샘플 코드](https://learn.microsoft.com/en-us/java/api/overview/azure/search-documents-readme)
- [Foundry On Your Data: 외부 데이터 소스 연결 및 설정 상세 방법](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/use-your-data)
- [OpenAI 임베딩 모델의 차원 최적화 기법 및 비용 효율화 분석](https://openai.com/blog/new-embedding-models-and-api-updates)
- [하이브리드 검색과 RRF(Reciprocal Rank Fusion) 알고리즘의 기술적 상세 설명](https://learn.microsoft.com/en-us/azure/search/hybrid-search-ranking)
- [RAG 할루시네이션 방지를 위한 고급 시스템 프롬프트 엔지니어링 전략 가이드](https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/prompt-engineering)

---

[← Ch.5 Function Calling & Tool Use](Ch05_Function_Calling.md) | [Ch.7 Foundry Agent Service →](Ch07_Agent_Service.md)
