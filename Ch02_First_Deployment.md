# Chapter 2. 첫 모델 배포와 Playground

[← 목차로](README.md)

> **학습 목표**
> - Model Catalog의 방대한 모델 라인업을 이해하고 필요한 모델을 필터링하는 능력을 갖춘다.
> - 서비스 요구사항(비용, 지연 시간, 거버넌스)에 맞는 최적의 배포 유형을 선택한다.
> - GPT-5 모델을 포털과 CLI를 통해 배포하고, 실제 환경에서의 트러블슈팅을 경험한다.
> - 배포된 모델의 물리적 정보(Endpoint, Key)를 확보하고 Playground에서 비즈니스 로직을 검증한다.
> - Quota의 동작 원리를 이해하고 프로덕션 런칭을 위한 할당량 확보 전략을 수립한다.

> **전제 조건**
> - Ch.1 개요 완료 (Foundry 리소스 및 프로젝트 생성 완료)
> - Azure Portal 접근 권한 및 로컬 환경에 Azure CLI 설치 완료
> - (권한) 'Cognitive Services OpenAI User' 혹은 'Foundry Project Manager' 이상의 권한 필요

---

## 1. Model Catalog: 1900개 이상의 모델 저장소

Foundry의 핵심은 전 세계의 검증된 모델들을 한곳에 모아둔 **Model Catalog**다. 2026년 7월 기준, 1900개가 넘는 모델이 등록되어 있으며, 매주 수십 개의 신규 모델이 추가된다. 단순한 리스트 제공을 넘어, '모델로서의 서비스(Model-as-a-Service, MaaS)'라는 혁신적인 경험을 제공한다. 이는 모델을 돌리기 위한 GPU 서버를 직접 사고 관리할 필요 없이, 필요한 만큼만 쓰고 API로 비용을 내는 방식이다.

### 1.1 검색과 필터링 (Smart Discovery)
수천 개의 모델 사이에서 최적의 모델을 찾기 위해 Foundry는 고도화된 필터링 시스템을 제공한다.
- **Task (작업 유형)**: 모델이 가장 잘하는 일을 기준으로 분류한다. 
  - **Chat**: 멀티턴 대화 및 추론.
  - **Embeddings**: 문장을 벡터로 변환 (RAG 구축 시 필수).
  - **Image/Video Generation**: 시각 자료 생성.
  - **Audio**: STT(Speech-to-Text) 및 TTS(Text-to-Speech).
- **Provider (제공처)**: 모델의 출처를 명확히 한다. 2026년에는 Microsoft와 OpenAI뿐만 아니라 Anthropic, Meta, Mistral, xAI 등 경쟁사 모델들도 동일한 인터페이스로 제공된다.
- **License (라이선스)**: 오픈 소스(MIT, Apache 2.0 등) 모델은 내부 구축 및 수정이 자유롭고, 상용 모델은 관리형 서비스로 안정적인 지원을 받는다.
- **Deployment Options (배포 방식)**: 
  - **Serverless API**: 인프라 관리 없이 토큰당 비용 지불.
  - **Managed Compute**: 특정 GPU 인스턴스에 직접 모델을 올리는 방식 (특수 보안 환경 혹은 커스텀 모델 사용 시).

### 1.2 주요 모델 컬렉션 (Collections)
- **Azure OpenAI**: GPT-5.5, GPT-5, GPT-4.1 등 최신 플래그십 모델들이 가장 먼저 공급된다. 특히 2025년 하반기 출시된 GPT-5 시리즈는 추론(Reasoning) 능력에서 압도적인 성능을 자랑한다.
- **Anthropic (2026-06 파트너십)**: Claude 4(2026년 신규) 및 Claude 3.5 Sonnet 모델이 공식 합류했다. 이전에는 Anthropic API를 따로 관리해야 했으나, 이제 Foundry 크레딧으로 결제가 통합되고 Azure의 보안 정책이 그대로 적용된다.
- **Meta Llama**: 오픈 소스 진영의 표준이다. Llama 4(2026년 출시) 계열까지 포함되어 있으며, 기업이 자체 데이터를 학습시켜 특화 모델을 만들 때 가장 많이 선택한다.
- **Mistral AI**: 유럽 최고의 AI 기업 모델로, 언어적 효율성이 뛰어나며 한국어를 포함한 다국어 처리 능력이 검증되었다.
- **Microsoft Phi-4**: 2026년형 Microsoft 자체 제작 소형 모델(SLM)이다. 서버 없이 로컬 기기에서도 구동 가능할 정도로 가볍지만, 논리 성능은 과거 대형 모델에 육박한다.
- **DeepSeek-R1**: 최근 급부상한 고효율 추론 모델로, 수학이나 코딩 등 복잡한 논리 문제에서 가성비 끝판왕이라는 평가를 받는다.
- **HuggingFace**: 수백 개의 커뮤니티 오픈 모델을 Foundry 인프라 위에서 즉시 서빙할 수 있도록 파이프라인이 구축되어 있다.

🔴 **2026 변경**: Claude의 공식 합류는 단순한 모델 추가 이상의 의미를 갖는다. 이제 개발자는 GPT-5와 Claude 4를 하나의 프로젝트 내에서 배포하고, 뒤에서 배울 'Model Router'를 통해 상황에 따라 두 모델을 스위칭하며 사용할 수 있게 되었다.

### 1.3 모델 추론 아키텍처 (Deep Dive: Inference Stack)

단순히 "모델을 배포한다"는 행위 뒤에는 복잡한 인프라가 숨어 있다. Foundry의 MaaS(Model-as-a-Service) 아키텍처를 이해하면 왜 특정 모델의 성능이 더 안정적인지 알 수 있다.

1.  **Shared Multi-Tenant Infrastructure**: Global Standard 배포 시 사용되는 방식이다. 수천 명의 사용자가 거대한 GPU 팜을 공유한다. 각 요청은 로드밸런서에 의해 최적의 유휴 노드로 전달된다.
2.  **Isolated Inference Nodes**: Managed Compute나 Provisioned SKU를 사용할 때 활성화된다. 특정 모델 인스턴스가 사용자만을 위해 격리된 컨테이너 환경에서 구동된다. 이는 외부 간섭(Noisy Neighbor)을 차단하여 일관된 지연 시간을 보장한다.
3.  **Optimized Serving Engines**: Foundry는 NVIDIA Triton Inference Server나 ONNX Runtime 같은 고성능 엔진을 사용하여 모델을 서빙한다. 특히 Microsoft 전용 모델(Phi-4 등)은 하드웨어 수준의 최적화가 적용되어 타 플랫폼 대비 20~30% 빠른 토큰 생성 속도를 보여준다.

---

## 2. 주요 Provider 별 대표 모델 상세 분석 및 벤치마크

단순히 이름만 아는 수준을 넘어, 각 Provider가 지향하는 가치와 모델의 내부 설계를 이해해야 최적의 선택이 가능하다.

### 2.1 OpenAI 컬렉션: 추론의 표준 (The Gold Standard)

-   **GPT-5.5 (Pro/Max)**: 2026년 4월 업데이트된 'Project Olympus' 아키텍처 기반 모델이다. 텍스트뿐만 아니라 비디오와 오디오를 동시에 처리하는 **Native Multimodal** 기능을 지원한다. 자바 개발자 입장에서 가장 놀라운 점은 '코드 인터프리터(Code Interpreter)' 기능의 비약적인 향상이다. 복잡한 알고리즘을 설계하고, 이를 직접 실행하여 결과까지 검증한 뒤 최종 코드를 제안한다.
-   **GPT-5-mini**: "Small but Mighty"의 정석이다. GPT-4 수준의 지능을 유지하면서 비용은 95% 저렴하다. 실시간 스트리밍 답변이 필요한 일반 고객 지원 챗봇에 가장 많이 쓰인다.

### 2.2 Anthropic: 안전과 문학성 (Safety & Nuance)

-   **Claude 4 (Opus/Sonnet)**: 2026년 6월 Foundry에 공식 탑재되었다. Anthropic의 독자적인 **Constitutional AI** 기법 덕분에 모델의 거절(Refusal) 기준이 매우 정교하다. 무조건 안 된다고 하는 것이 아니라, 왜 안 되는지 설명하고 대안을 제시한다. 또한, 인간적인 문체와 긴 문맥에서의 기억력(Recall)이 뛰어나 보고서 작성이나 법률 검토용으로 선호된다.
-   **Artifacts 기능 연동**: Foundry 포털 내 Claude Playground에서는 모델이 생성한 코드나 문서를 우측 렌더링 창에서 즉시 확인할 수 있는 Artifacts 기능이 통합되어 있다.

### 2.3 Meta: 오픈 소스의 혁신 (The Open Standard)

-   **Llama 4 (405B/70B/8B)**: 메타의 최신 모델은 '오픈 소스는 성능이 낮다'는 편견을 완전히 깼다. 특히 405B 모델은 GPT-5급 성능을 보여주며, 기업이 모델 가중치(Weights)를 직접 다운로드하여 폐쇄망(Air-gapped) 환경에 배포할 수 있는 유일한 하이엔드 대안이다. Foundry에서는 이를 'Managed Online Endpoint' 형식으로 배포하여 관리 부담을 줄일 수 있다.

### 2.4 DeepSeek: 고효율 연산의 선두주자 (Computational Efficiency)

-   **DeepSeek-R1 (Reasoning)**: 최근 중국 AI 기술의 저력을 보여준 모델이다. **Mixture-of-Experts (MoE)** 구조를 극도로 고도화하여, 실제 연산에 참여하는 파라미터 수를 줄이면서도 지능은 높였다. 복잡한 수학 증명이나 논리 퍼즐 분야에서 GPT-5.1과 대등한 점수를 기록하며, 토큰당 단가는 시장 최저 수준이다.

---

## 3. 배포 유형 및 SKU 심층 분석 (Strategic Selection)

모델을 선택했다면, 그다지 친절하지 않은 SKU(Stock Keeping Unit) 리스트를 마주하게 된다. 이 선택은 비용, 성능 안정성, 데이터 거주성(Data Residency)에 결정적인 영향을 미친다.

### 3.1 배포 옵션 상세 비교 (2026 기준)

| 배포 유형 | SKU 코드 | 과금 체계 | 특징 및 권장 시나리오 |
|---|---|---|---|
| **Global Standard** | `GlobalStandard` | 토큰당 과금 (Pay-as-you-go) | **기본 추천.** 트래픽 예측이 어렵거나 개발 초기 단계. 가장 높은 Quota를 제공한다. |
| **Global Provisioned** | `GlobalProvisionedManaged` | 시간당 예약 (PTU) | **상용 서비스.** 지연 시간(Latency)이 일정해야 하고, 트래픽이 많아 토큰 과금이 비싸지는 시점. |
| **Global Batch** | `GlobalBatch` | 50% 할인 (비동기) | 실시간 응답이 필요 없는 대용량 작업. 24시간 내 응답 보장. 로그 분석, 요약 등에 최적. |
| **Data Zone Standard** | `DataZoneStandard` | 토큰당 과금 | 미국(US) 혹은 유럽(EU) 등 특정 구역 내 데이터 거주성을 보장해야 하는 보안 요구 시. |
| **Standard (Regional)** | `Standard` | 토큰당 과금 | 특정 단일 리전(예: Korea Central) 내에서만 데이터가 처리되길 원할 때. 할당량이 적을 수 있다. |
| **Developer** | `DeveloperTier` | 토큰당 과금 | 24시간만 유지되는 테스트용. 미세 조정(Fine-tuning) 모델의 품질 평가용으로 적합. |
| **Instant (preview)** | N/A | 토큰당 과금 | **2026 신규.** 배포 과정 없이 카탈로그에서 모델 ID로 즉시 API 호출. 빠른 프로토타이핑용. |

⚠️ **함정: Global 배포와 데이터 주권 이슈**.
Global 배포 유형은 Microsoft의 전 세계 Azure 데이터 센터 중 유휴 자원이 있는 곳을 실시간으로 찾아 처리한다. 이는 성능과 할당량 확보에는 유리하지만, 데이터가 물리적으로 국경을 넘을 수 있음을 의미한다. "개인정보나 금융 데이터가 반드시 한국 리전 내에서 처리되어야 한다"는 규정이 있다면, 반드시 **Regional**이나 **Data Zone** 옵션을 선택해야 한다.

### 3.2 PTU(Provisioned Throughput Unit)의 경제학

PTU는 모델의 '전용 차선'을 확보하는 개념이다.
- **Pay-as-you-go (Standard)**: 고속도로 톨게이트 비용을 지나갈 때마다 내는 방식이다. 차가 없으면 비용이 0원이지만, 명절(트래픽 폭주)에는 모델 응답이 늦어지거나 거부될 수 있다.
- **PTU (Provisioned)**: 한 달 치 정기권을 미리 사고 전용 차선을 확보하는 방식이다. 차가 한 대도 없어도 비용은 발생하지만, 어떤 상황에서도 일정한 속도를 보장한다.

**계산 가이드**: 만약 우리 서비스의 분당 토큰 사용량(TPM)이 꾸준히 10만 개 이상 유지된다면, PTU로 전환하는 것이 토큰 과금보다 약 20% 이상 저렴해지기 시작한다. 상용 런칭 전 반드시 이 지점을 계산해보자.

---

## 🔧 실습: GPT-5 모델 배포하기 (Portal & CLI)

사용자가 이미 현실에서 GPT-5를 배포해 보았더라도, 2026년형 정석 워크플로우를 숙지하는 것은 협업과 자동화를 위해 필수적이다.

### Step 1: Foundry Portal 방식
1. [Foundry Portal](https://ai.azure.com)에 로그인하고 본인의 **Project**를 선택한다.
2. 왼쪽 메뉴의 **Model Catalog**를 클릭한다.
3. 검색창에 `gpt-5`를 입력하고 `gpt-5-pro` (플래그십 모델)를 선택한다.
4. 모델 상세 페이지에서 **Deploy** 버튼을 누르고 `Global Standard`를 선택한다.
5. 설정창에서 다음 정보를 입력한다.
   - **Deployment name**: `my-gpt5-deployment` (이름은 알파벳, 숫자, 하이픈만 가능하며, 나중에 코드에서 이 이름을 식별자로 쓴다.)
   - **Select SKU**: `GlobalStandard`
   - **Tokens per Minute Rate Limit**: 기본값(예: 10K)으로 시작한다. 나중에 필요하면 증설 가능하다.
6. **Deploy**를 클릭한다. 상태가 `Succeeded`로 바뀌면 배포 완료다.

💡 **팁: Deployment Name 룰 정하기**.
배포 이름은 한 번 정하면 변경이 불가능하다. 팀 내에서 `[환경]-[모델명]-[버전]` 식의 규칙을 정해두면 나중에 관리가 훨씬 편해진다. (예: `prod-gpt5-v1`)

### Step 2: Azure CLI 방식 (자동화)
CI/CD 파이프라인이나 스크립트에서 배포를 자동화할 때 사용한다. 2026년형 CLI 명령어로 GPT-5를 배포해보자.

```bash
# bash
# 1. 로그인 및 구독 확인
az login

# 2. 모델 배포 생성
az cognitiveservices account deployment create \
    --name "my-foundry-resource" \
    --resource-group "my-ai-rg" \
    --deployment-name "my-cli-gpt5" \
    --model-name "gpt-5" \
    --model-version "2025-08-preview" \
    --model-format "OpenAI" \
    --sku-name "GlobalStandard" \
    --sku-capacity 10
```

⚠️ **함정: Deployment Name ≠ Model Name**.
초보자가 가장 많이 하는 실수다. SDK에서 API를 호출할 때 전달하는 식별자는 `gpt-5`(모델명)가 아니라 당신이 지은 `my-cli-gpt5`(배포명)다. CLI 명령어를 칠 때 이 두 파라미터의 차이를 명확히 인지하자.

### 3.3 운영 환경에서의 SKU 선택 전략 (Production SKU Selection)

상용 환경으로 넘어가는 시점에서 SKU 선택은 단순한 '비용' 문제가 아니라 '비즈니스 연속성'의 문제다.

1.  **가용성 확보 (Availability)**: `GlobalStandard`는 고정된 리소스가 아니므로 극심한 트래픽 부하 시 일시적인 429 에러(Too Many Requests)가 발생할 수 있다. 반면 `GlobalProvisioned`는 100% 가용성을 보장하는 전용 인스턴스이므로, SLA(Service Level Agreement)가 중요한 외부 고객 서비스에는 반드시 Provisioned SKU를 고려해야 한다.
2.  **데이터 보관 규정 (Compliance)**: 앞서 언급했듯, 한국 내에서 데이터가 한 발짝도 나가지 않아야 한다면 `Standard (Regional)`을 선택하고, 해당 리전의 쿼터를 미리 확보해야 한다. 이때는 글로벌 배포보다 비용이 다소 높거나 신규 모델 반영이 느릴 수 있다는 점을 사업팀에 미리 고지해야 한다.
3.  **비용 효율 극대화**: `GlobalBatch`를 적극 활용하라. 실시간성이 필요 없는 감정 분석, 대량 문서 요약, 데이터 정제 작업은 배치 API로 던지고 24시간 이내에 결과만 받으면 된다. 이는 토큰 단가를 절반 이하로 낮추는 가장 확실한 방법이다.

### 3.4 모델 버전 관리 및 라이프사이클 (Model Lifecycle)

배포 시 반드시 고려해야 할 것이 **모델 버전**이다.
-   **Stable vs Preview**: `gpt-5-preview` 같은 프리뷰 버전은 성능은 뛰어나지만 언제든 변경되거나 사라질 수 있다. 상용 앱은 가급적 `gpt-5 (stable)` 혹은 `2025-08-01` 처럼 날짜가 명시된 버전을 선택하라.
-   **Auto-update 정책**: Foundry는 모델 자동 업데이트 기능을 제공한다. 하지만 개발자 입장에서는 'Auto-update Off'를 권장한다. 모델이 업데이트되면서 기존에 잘 작동하던 프롬프트가 오동작할 수 있기 때문이다. 버전 업그레이드는 충분한 테스트(Ch.9 Evaluation)를 거친 후 수동으로 진행하는 것이 정석이다.

---

## 4. Quota 및 용량 관리 (Quota & Capacity Management)

"내 돈 내고 쓰겠다는데 왜 안 빌려주는가?" 초보 개발자가 가장 자주 묻는 질문이다. AI 연산 자원(GPU)은 유한하며, 전 세계적인 수요가 공급을 압도하고 있다. 따라서 Foundry는 **Quota(할당량)** 시스템을 통해 자원을 관리한다.

### 4.1 Quota의 세 가지 계층
1.  **Subscription Level**: 해당 Azure 구독이 전체 리전에서 쓸 수 있는 총량.
2.  **Region Level**: 특정 리전(예: East US)에 할당된 양.
3.  **Project Level**: 우리 팀의 특정 프로젝트에 할당된 양.

### 4.2 TPM 증설 전략 (Requesting Quota)
기본 할당량(예: 10,000 TPM)은 실무용으로는 턱없이 부족하다. 증설 요청 시 다음 정보를 포함하면 승인 확률이 높아진다.
-   **비즈니스 케이스**: "사내 AI 비서 1,000명 동시 접속 대비" 등 구체적인 이유.
-   **트래픽 예상치**: 예상되는 피크 타임의 초당 토큰 소모량.
-   **리전 다각화**: "East US가 꽉 찼다면 Sweden Central이라도 달라"는 식의 대안 제시.

💡 **팁: 쿼터 '돌려막기'**.
한 프로젝트에서 사용하지 않는 쿼터는 아까운 자원이다. 'Management Center'에서 사용량이 0인 프로젝트의 쿼터를 회수하여, 현재 바쁜 프로젝트로 즉시 재할당할 수 있다. 이는 추가 비용 없이 성능을 높이는 운영의 묘미다.

---

## 5. 엔드포인트 보안 및 인증 (Endpoint Security)

모델을 배포하면 공개 URL이 생성된다. 이를 보호하는 것은 개발자의 첫 번째 임무다.

### 5.1 API 키 vs Managed Identity
-   **API 키 (Access Key)**: 빠르고 쉽지만 유출 시 치명적이다. 소스 코드나 설정 파일에 절대 직접 적지 마라.
-   **Managed Identity (권장)**: 앱 서비스 자체가 AI 서비스에 로그인하는 방식이다. 키가 없으므로 유출될 위험도 없다. 2026년 기준, 엔터프라이즈 환경에서는 Managed Identity 사용이 의무화되는 추세다.

### 5.2 네트워크 격리 (Private Link)
배포 상세 페이지의 'Networking' 탭에서 **Private Endpoint**를 구성하라. 이렇게 하면 모델 주소가 인터넷망에서 사라지고, 우리 회사의 내부망(VNet) 안에서만 접근 가능한 주소로 바뀐다. 해커가 외부에서 우리 모델 API를 공격하는 경로를 물리적으로 차단하는 가장 강력한 보안 수단이다.

---

## 6. Playground 상세 가이드: 프롬프트 엔지니어링의 실험장


모델 배포 직후 바로 코딩을 시작하는 것은 비효율적이다. **Playground**는 프롬프트와 파라미터를 확정 짓는 '실험실' 역할을 한다.

### 5.1 Chat Playground (기본 검증)
- **System Message (기본 지침)**: 모델의 성격과 제약 조건을 설정한다. "너는 간결하게 답변하는 자바 시니어 엔지니어다"라고 명시하여 답변의 품질을 조절한다.
- **Parameters (매개변수)**:
  - **Temperature**: 답변의 창의성 수준. 0이면 가장 확률 높은 답변(결정론적), 1이면 다채롭고 창의적인 답변을 한다. 보통 0.3~0.7 사이가 적당하다.
  - **Response Format**: `json_object`를 선택하면 모델이 반드시 JSON 형태로만 응답하도록 강제한다. 이는 백엔드 파싱 로직을 짤 때 매우 유용하다.

⚠️ **함정: GPT-5 Reasoning 모델의 특이 파라미터**.
GPT-5.1 등 추론 모델은 일반 모델과 다른 규칙을 따른다.
- `max_tokens` 대신 **`max_completion_tokens`**를 사용해야 한다. (내부 추론 토큰을 포함하기 위함)
- **`temperature`가 1.0으로 고정**되는 경우가 많다. 내부 사고 과정(CoT)의 다양성을 확보하기 위한 조치이므로, 이를 0으로 바꾸려 하면 에러가 날 수 있다.

### 5.2 Assistants Playground (Preview)
Ch.7에서 다룰 'Agent Service'의 맛보기 공간이다. 모델에 문서를 업로드하여 답변하게 하거나(RAG), 파이썬 코드를 실행시켜 그래프를 그리게 하는 등의 고난도 작업을 테스트할 수 있다. 2026년 6월 이후 **Agent Versions** 기능을 통해 여러 버전의 프롬프트를 한눈에 비교하는 기능이 추가되었다.

### 5.3 Prompts Playground (Prompt CMS)
프롬프트만 따로 관리하고 배포하는 공간이다. 여기서 작성한 프롬프트를 저장하면 고유한 ID가 부여되며, 나중에 앱 소스 코드를 수정하지 않고도 포털에서 프롬프트를 업데이트하여 배포 중인 앱의 성능을 개선할 수 있다.

---

## 6. Quota, TPM, RPM 관리 전략

성공적인 배포를 방해하는 가장 큰 벽은 **Quota(할당량)** 부족이다.

- **TPM (Tokens Per Minute)**: 분당 처리할 수 있는 최대 토큰 수.
- **RPM (Requests Per Minute)**: 분당 처리할 수 있는 최대 요청 횟수.

### 6.1 할당량 확보 및 증설
1. 포털 왼쪽 하단의 **Quota** 페이지로 이동한다.
2. 현재 내 구독(Subscription)이 특정 리전에서 가진 총 할당량을 확인한다.
3. 배포된 모델의 슬라이더를 조절하여 TPM을 할당한다. 부족하다면 **Request Quota** 버튼을 눌러 Microsoft에 증설을 요청한다. (보통 24시간 내 승인된다.)

💡 **팁: 런칭 직전 Quota 팁**.
중요한 상용 서비스 런칭 전에는 반드시 필요한 TPM의 1.5배 이상을 미리 확보해두어야 한다. 런칭 당일 트래픽이 몰려 Quota에 걸리면(429 Too Many Requests 에러), 증설 요청을 해도 승인까지 시간이 걸려 장애로 이어질 수 있다.

## 7. 모델 벤치마크 및 성능 지표 해석 (Interpreting Benchmarks)

모델 카탈로그의 각 모델 페이지에는 'Benchmarks' 탭이 있다. 이는 단순히 "이 모델이 좋다"는 광고가 아니라, 특정 작업에서의 수학적 성능을 나타낸다.

### 7.1 주요 지표 상세 설명
- **MMLU (Massive Multitask Language Understanding)**: 일반 상식과 지능을 측정한다. 90점 이상이면 인간 전문가 수준의 지능으로 간주한다.
- **HumanEval**: 파이썬 코딩 능력을 테스트한다. 자바 개발자에게도 이 지표는 매우 중요한데, 코딩 로직이 강한 모델일수록 복잡한 업무 프로세스 설계 능력이 뛰어나기 때문이다.
- **GSM8K**: 초등학교 수준의 수학 문제 해결 능력이다. 이는 모델의 '단계별 사고(Chain of Thought)' 능력을 대변한다.
- **GPQA (Graduate-Level Google-Proof Q&A)**: 매우 어려운 과학/기술 문제를 테스트한다. GPT-5.5나 Claude 4 같은 최상위 모델만이 높은 점수를 낼 수 있는 영역이다.

### 7.2 벤치마크 활용 팁
- **목적에 맞는 지표 우선순위**: 챗봇이라면 MMLU와 텍스트 생성 품질(Coherence)을 보아야 하고, 업무 자동화 에이전트라면 HumanEval과 Reasoning 점수를 최우선으로 고려해야 한다.
- **카탈로그 내 'Compare' 기능**: 두 개 이상의 모델을 선택하고 'Compare' 버튼을 누르면, 이 모든 지표를 한눈에 비교할 수 있는 대시보드가 나타난다. GPT-5와 Claude 4 사이에서 고민될 때 이 기능을 통해 '가성비'가 더 좋은 지점을 찾아낼 수 있다.

---

## 8. 실제 배포 트러블슈팅 케이스스터디 (Real-world Case Studies)

이론과 실제는 다르다. 현업에서 배포 중 마주하게 되는 복잡한 상황들을 해결하는 방법이다.

### 8.1 케이스 1: 배포는 성공했으나 API 호출 시 404 에러 발생
- **상황**: 포털에서는 'Succeeded'인데, 코드에서 호출하면 "Deployment not found" 에러가 뜬다.
- **원인**: 
  1. 엔드포인트 URL과 API 키의 불일치 (다른 리소스의 정보를 가져온 경우).
  2. 코드에서 `deployment-name` 대신 `model-name`을 사용한 경우.
  3. 배포 직후 DNS 전파 대기 시간(보통 1~2분)이 필요한 경우.
- **해결**: 포털의 'Endpoints' 메뉴에서 제공하는 'Sample Code'를 복사해서 그대로 실행해본다. 만약 샘플 코드는 작동한다면 본인의 코드 내 설정값 매핑 로직을 점검해야 한다.

### 8.2 케이스 2: 특정 시간대에만 발생하는 429 (Too Many Requests)
- **상황**: 오전 9시 출근 시간대에만 서비스가 마비된다.
- **원인**: Global Standard 배포의 한계다. 해당 리전의 전체 사용량이 급증하면, 개별 사용자의 쿼터가 남아있더라도 일시적으로 요청이 거부될 수 있다.
- **해결**: 
  1. **Retry Logic**: 지수 백오프(Exponential Backoff)를 적용하여 재시도한다.
  2. **Model Router**: Ch.8에서 다룰 모델 라우터를 사용하여, 429 에러 시 다른 리전이나 다른 모델(예: GPT-5 → Claude 4)로 즉시 우회(Failover)한다.
  3. **Provisioned SKU**: 트래픽이 예측 가능해지면 PTU로 전환하여 전용 대역폭을 확보한다.

### 8.3 케이스 3: 모델의 답변이 갑자기 짧아지거나 품질이 저하됨
- **상황**: 어제까지 잘 나오던 긴 보고서가 오늘부터 요약본으로만 나온다.
- **원인**: 'Auto-update to latest' 옵션이 켜져 있어 모델 버전이 자동으로 올라간 경우다. 새로운 버전에서 시스템 프롬프트에 대한 반응 민감도가 달라졌을 수 있다.
- **해결**: 배포 설정에서 버전을 고정(Pinning)한다. 최신 버전 사용이 목표라면, 새로운 배포를 따로 만들고 Ch.9의 평가 기능을 통해 품질 검증을 마친 뒤 트래픽을 점진적으로 전환(Canary Deployment)해야 한다.

---

## 9. 비용 산정 시뮬레이션: 챗봇 vs 데이터 처리 (Cost Simulation)

배포 전 사업 부서에 보고할 예상 비용을 뽑는 공식이다. (2026년 표준 단가 기준 예시)

### 9.1 시나리오 A: 사내 기술 지원 챗봇
- **조건**: 일일 사용자 1,000명, 1인당 10번 질문, 질문당 입력 500토큰 / 출력 500토큰.
- **계산**: 
  - 총 입력: 1,000 * 10 * 500 = 5,000,000 토큰
  - 총 출력: 1,000 * 10 * 500 = 5,000,000 토큰
  - GPT-5-mini 사용 시 (1M당 입력 $0.15 / 출력 $0.60 가정): (5 * 0.15) + (5 * 0.60) = $3.75 / 일
- **결론**: 월 약 $112 (약 15만원). 매우 경제적이며 Global Standard SKU가 적합하다.

### 9.2 시나리오 B: 대규모 문서 데이터 정제 (Batch Processing)
- **조건**: 100만 건의 고객 상담 로그 요약, 건당 2,000토큰.
- **계산**: 
  - 총 토큰: 1,000,000 * 2,000 = 2,000,000,000 (20억) 토큰
  - Global Standard 사용 시: (2,000 * $0.15) = $300 (입력만 계산 시)
  - **Global Batch 사용 시 (50% 할인)**: $150
- **결론**: 실시간성이 필요 없으므로 반드시 Batch SKU를 사용하여 수백만 원의 비용을 절감해야 한다.

---

## 10. 모델 카탈로그의 '숨겨진 보석' 모델들 (Niche Models)

GPT나 Claude 외에도 특정 상황에서 빛을 발하는 모델들이 있다.

- **Jina Embeddings**: RAG 구축 시 한글과 영어 혼용 검색 성능이 매우 뛰어나다. 8k 이상의 긴 컨텍스트를 지원하는 임베딩 모델을 찾는다면 최선의 선택이다.
- **Grok-1.5 / 2 (xAI)**: 실시간 트렌드나 최신 뉴스 반영 속도가 빠르다. 소셜 미디어 트렌드 분석 앱에 유리하다.
- **Sora-2 (Video Generation)**: 텍스트 한 줄로 1분 분량의 고화질 영상을 생성한다. 사내 교육 자료나 마케팅 영상 초안 작성에 혁신을 가져올 수 있다.
- **Med-Gemini / BioNeMo**: 의료 및 바이오 특화 모델이다. 단백질 구조 분석이나 의학 논문 요약 등 전문 도메인에서는 범용 모델보다 훨씬 정확한 결과를 낸다.

---

## 11. 파인튜닝(Fine-tuning) 모델의 배포와 관리

일반적인 모델(Base Model)로 해결되지 않는 특수한 목적이 있다면, 기업 전용 데이터를 학습시킨 '파인튜닝 모델'을 배포해야 한다. Foundry는 이 과정을 관리형 워크플로로 제공한다.

### 11.1 파인튜닝 모델을 선택해야 하는 경우
- **특수 포맷 강제**: JSON 구조가 극도로 복잡하거나, 특정 업계 표준 서식을 반드시 따라야 할 때.
- **도메인 특화 용어**: 의료, 법률, 사내 약어 등 일반 모델이 알기 어려운 지식을 내재화해야 할 때.
- **비용 최적화**: 거대한 GPT-5.5 대신 작은 Phi-4를 특정 작업에 맞춰 깎아서(Fine-tune) 사용하면 성능은 유지하면서 비용을 크게 낮출 수 있다.

### 11.2 파인튜닝 모델 배포 프로세스
1.  **Dataset Preparation**: JSONL 형식의 학습 데이터를 준비하여 Foundry 프로젝트의 'Data' 섹션에 업로드한다.
2.  **Fine-tuning Job**: 모델 카탈로그에서 베이스 모델을 선택하고 'Fine-tune' 버튼을 눌러 학습 세션을 시작한다. (이 과정에서 GPU 컴퓨팅 비용이 발생한다.)
3.  **Model Validation**: 학습이 완료된 모델은 'Models' 메뉴의 'Custom Models' 탭에 나타난다.
4.  **Deployment**: 커스텀 모델도 일반 모델과 동일하게 `GlobalStandard` 혹은 `Provisioned` SKU로 배포할 수 있다. 이때 배포 이름(Deployment Name)을 지정하여 API에서 호출할 준비를 마친다.

⚠️ **함정: 파인튜닝 모델의 배포 비용**.
커스텀 모델은 베이스 모델보다 토큰당 단가가 비싼 경우가 많다. 또한, 일부 리전에서는 커스텀 모델 배포를 위해 반드시 PTU(Provisioned)를 구매해야 할 수도 있으므로, 배포 전 'Pricing' 탭을 반드시 재확인해야 한다.

---

## 12. 다중 리전 배포 및 고가용성 전략 (Multi-region HA)

엔터프라이즈 서비스에서 특정 리전의 장애나 쿼터 부족은 곧 서비스 중단을 의미한다. 이를 방지하기 위한 배포 전략이 필요하다.

### 12.1 Active-Active 배포 패턴
- **방법**: 서로 다른 두 리전(예: East US와 Sweden Central)에 동일한 모델을 각각 배포한다.
- **장점**: 한쪽 리전이 429 에러(부하 폭주)를 내거나 하드웨어 장애가 발생해도, 다른 쪽 리전으로 즉시 트래픽을 분산할 수 있다.
- **구현**: Ch.8에서 다룰 **Model Router** 기능을 사용하면, 클라이언트 코드 수정 없이 플랫폼 레벨에서 이 로드밸런싱을 수행할 수 있다.

### 12.2 재해 복구(DR)를 위한 지역적 분리
- **규정 준수**: 데이터 거주성(Data Residency) 규정이 있는 경우, 같은 'Data Zone'(예: EU Zone) 내의 두 리전에 배포하여 법적 문제를 피하면서 가용성을 확보한다.
- **지연 시간 최적화**: 한국(Korea Central)과 일본(Japan East)에 동시 배포하여 사용자 위치에 따라 가장 가까운 엔드포인트를 호출하게 함으로써 사용자 경험을 극대화한다.

---

## 13. 배포 자동화와 IaC (Infrastructure as Code)

수동으로 포털에서 클릭하는 것은 실수의 위험이 크다. 실제 운영 환경에서는 테라폼(Terraform)이나 Bicep을 사용하여 배포를 코드로 관리한다.

### 13.1 Bicep을 활용한 모델 배포 예시
```bicep
// model-deployment.bicep
resource foundryResource 'Microsoft.CognitiveServices/accounts@2023-05-01' existing = {
  name: 'my-foundry-res'
}

resource gpt5Deployment 'Microsoft.CognitiveServices/accounts/deployments@2023-05-01' = {
  parent: foundryResource
  name: 'prod-gpt5-v1'
  properties: {
    model: {
      format: 'OpenAI'
      name: 'gpt-5'
      version: '2025-08-01'
    }
    versionUpgradeOption: 'NoAutoUpgrade'
    raiPolicyName: 'Microsoft.Default'
  }
  sku: {
    name: 'GlobalStandard'
    capacity: 50
  }
}
```

이러한 코드를 통해 인프라의 가시성을 확보하고, 개발-테스트-운영 환경에 동일한 설정을 100% 재현할 수 있다.

### 13.2 Terraform (OpenTofu)를 활용한 멀티 모델 배포

테라폼은 클라우드 불문 인프라 자동화의 표준이다. Foundry 리소스는 `azurerm` 프로바이더의 `azurerm_cognitive_deployment` 리소스를 통해 관리한다.

```hcl
# main.tf
resource "azurerm_cognitive_account" "foundry" {
  name                = "foundry-res-prod"
  location            = "eastus"
  resource_group_name = "rg-ai-prod"
  kind                = "AIServices"
  sku_name            = "S0"
}

resource "azurerm_cognitive_deployment" "gpt5" {
  name                 = "prod-gpt5-v1"
  cognitive_account_id = azurerm_cognitive_account.foundry.id
  model {
    format  = "OpenAI"
    name    = "gpt-5"
    version = "2025-08-01"
  }
  sku {
    name     = "GlobalStandard"
    capacity = 100
  }
}

resource "azurerm_cognitive_deployment" "claude4" {
  name                 = "prod-claude4-v1"
  cognitive_account_id = azurerm_cognitive_account.foundry.id
  model {
    format  = "Anthropic"
    name    = "claude-4-sonnet"
    version = "2026-06-01"
  }
  sku {
    name     = "GlobalStandard"
    capacity = 50
  }
}
```

### 13.3 GitHub Actions를 통한 배포 파이프라인 연동

모델 배포를 개발 프로세스에 통합하면, 새로운 모델 버전이 출시될 때마다 자동으로 테스트 환경에 배포하고 검증할 수 있다.

```yaml
# .github/workflows/deploy-models.yml
name: Deploy AI Models
on:
  push:
    paths: ['infra/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Azure Login
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      - name: Deploy Bicep
        uses: azure/arm-deploy@v2
        with:
          resourceGroupName: rg-ai-prod
          template: ./infra/model-deployment.bicep
```

---

## 14. 자주 묻는 질문 (FAQ: Deep Dive)

모델 배포와 관련하여 현업에서 가장 많이 묻는 질문들을 정리했다.

**Q: Global Standard와 Regional 배포의 속도 차이가 실제로 큰가요?**
A: 이론적으로는 Regional 배포가 해당 리전 내에서만 처리되므로 지연 시간(Latency)이 더 안정적이어야 합니다. 하지만 Microsoft의 Global 배포 네트워크는 매우 고도화되어 있어, 일반적인 경우에는 차이를 체감하기 어렵습니다. 다만, 네트워크 홉(Hop)이 늘어날수록 변동성(Jitter)이 생길 수 있으므로, 0.1초가 중요한 리얼타임 서비스라면 Regional을 권장합니다.

**Q: 모델 배포 이름을 나중에 바꿀 수 없나요?**
A: 불가능합니다. 배포 이름은 엔드포인트 URL의 일부로 사용되기 때문입니다. 이름을 바꾸려면 기존 배포를 삭제하고 새로 만들어야 합니다. 이때 기존 엔드포인트를 호출하던 모든 앱의 설정을 변경해야 하므로, 서비스 중단을 피하려면 새로운 배포를 먼저 만들고 트래픽을 옮기는 방식을 택해야 합니다.

**Q: 한 프로젝트에 몇 개까지 모델을 배포할 수 있나요?**
A: 기본적으로 리전 및 구독별로 제한이 있지만, 보통 수십 개의 배포를 생성할 수 있습니다. 하지만 사용하지 않는 배포는 쿼터를 점유하므로 정기적으로 정리하는 것이 좋습니다. 특히 Provisioned SKU는 사용하지 않아도 예약 비용이 발생하므로 주의해야 합니다.

**Q: GPT-5.5 모델이 카탈로그에 안 보입니다.**
A: 리전 확인이 첫 번째입니다. 최신 모델은 대개 `East US`, `Sweden Central`, `West US 3` 등 특정 리전에 먼저 풀립니다. 두 번째는 구독 유형입니다. 체험용(Free) 구독은 일부 최신 모델 접근이 제한될 수 있습니다.

**Q: 모델 배포 시 'RAI Policy'는 무엇인가요?**
A: Responsible AI Policy의 약자입니다. 모델이 생성하는 답변에 대해 Microsoft가 기본적으로 적용하는 유해성 필터링 규칙입니다. 기본값(`Microsoft.Default`)을 사용하면 대부분의 보안 가이드를 준수할 수 있으며, 특정 요구사항이 있다면 'Content Safety' 메뉴에서 커스텀 정책을 만들어 적용할 수 있습니다.

## 15. 모델 성능 비교 및 투자 대비 효과(ROI) 분석

모델 배포 후 비즈니스 관점에서 가장 중요한 질문은 "이 비용을 들여서 이 모델을 쓰는 것이 타당한가?"이다.

### 15.1 성능 대 비용 매트릭스 (Performance-Cost Matrix)
- **High-End (GPT-5.5, Claude 4 Opus)**: 최고의 지능, 가장 비싼 비용. 복잡한 추론, 전략 수립, 멀티모달 분석에 사용.
- **Mid-Range (GPT-5, Claude 4 Sonnet, Llama 4 70B)**: 균형 잡힌 성능과 비용. 대부분의 비즈니스 로직, 문서 요약, 일반 상담에 적합.
- **Low-End (GPT-5-mini, Phi-4, Llama 4 8B)**: 초저비용, 초고속. 단순 분류, 의도 파악, 실시간 스트리밍 대화에 최적.

### 15.2 전환 시점의 ROI 산정
특정 프로젝트가 초기에는 GPT-5.5로 시작했더라도, 데이터가 쌓이고 프롬프트가 최적화되면 GPT-5-mini로 전환할 수 있는 '최적화 지점'이 온다.
- **성능 유지율**: 모델 전환 후 Ch.9의 평가 도구를 통해 답변 품질이 95% 이상 유지되는지 확인한다.
- **비용 절감액**: 모델 전환으로 인해 월간 운영 비용이 80% 이상 절감된다면, 전환에 들어가는 개발 공수(M/M)를 고려하더라도 2~3개월 내에 ROI가 양수로 돌아선다.

### 15.3 하이브리드 모델 전략 (Hybrid Model Strategy)

최근에는 하나의 모델만 쓰는 것이 아니라, 여러 모델을 조합하여 성능과 비용을 모두 잡는 '하이브리드 전략'이 대세다.
- **Router Pattern**: 가벼운 질문은 Phi-4가 처리하고, 어려운 질문만 GPT-5.5로 토스한다.
- **Ensemble Pattern**: 동일한 질문을 GPT-5와 Claude 4에게 동시에 던지고, 세 번째 모델(예: Llama 4)이 두 답변을 비교하여 최상의 결과를 합성한다.
- **Speculative Decoding**: 작은 모델이 먼저 답변 초안을 빠르게 생성하고, 큰 모델이 이를 검증하고 수정하는 방식으로 사용자 체감 속도를 높인다.

---

## 16. 결론 및 실전 체크리스트

2장에서는 모델을 선택하고 배포하며, 실제 운영 환경에서 마주할 다양한 변수들을 관리하는 방법을 배웠다.

### 🚀 실전 배포 체크리스트 (Final Review)
- [ ] 모델의 **Data Zone**이 우리 회사의 보안 정책을 준수하는가?
- [ ] **Deployment Name**에 환경(dev/prod)과 모델 식별자가 포함되었는가?
- [ ] **Managed Identity**를 활성화하여 키 없는 인증 준비를 마쳤는가?
- [ ] 예상 트래픽을 기반으로 초기 **TPM Quota**를 적절히 배분했는가?
- [ ] 실시간성이 불필요한 작업에 **Batch SKU** 적용을 고려했는가?
- [ ] **Playground**에서 시스템 프롬프트와 파라미터(`Temperature` 등)의 최적값을 찾았는가?

이제 여러분은 강력한 AI 모델을 클라우드상에 성공적으로 안착시켰다. 하지만 모델은 배포하는 것보다 '잘 호출하는 것'이 더 중요하다. 다음 장에서는 자바(Java) 코드를 통해 이 모델들과 대화하고, 엔터프라이즈 급 애플리케이션의 뼈대를 세우는 방법을 배울 것이다.

---
