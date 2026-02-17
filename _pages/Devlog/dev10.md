---
title: '[낚시없는 우리집] LLM과 LLM의 한계를 극복하기 위한 기술 RAG'
tags:
  - Project
  - LLM
  - RAG
date: '2025-08-30'
thumbnail: '/assets/img/devlog/09/dev09_0.png'
---

지난 포스팅에서는 **시나리오 자동 생성을 위한 LLM 활용 방식**을 살펴보았습니다. 이번 포스팅에서는 **LLM과 LLM의 한계를 극복하기 위한 기술인 RAG**에 대해 알아보겠습니다.

## 🏠 LLM의 한계 및 RAG 소개

---

RAG를 이해하려면, 우선 생성형 AI와 LLM에 대해 알아야 합니다.

### 💡 생성형 AI란

생성형 AI란 새로운 **텍스트, 이미지, 음성** 등을 만들 수 있는 모델을 말하며, 많은 양의 데이터를 **사전 학습**한 대규모의 모델입니다.

> **대형 언어 모델(LLM)** : 자연어 이해와 생성에 특화된 모델로, 글 작성, 요약, 번역, 대화 등 텍스트 기반 작업 수행
> &nbsp; ex) ChatGPT, Claude, GPT-4, LLaMA
> **이미지 / 영상 생성 모델** : 텍스트 또는 이미지 입력을 바탕으로 새로운 시각 콘텐츠 생성
> &nbsp; ex) DALL·E, Midjourney, Stable Diffusion
> **음성 / 오디오 생성 모델** : 텍스트를 음성으로 변환하거나 음악/효과음을 생성
> &nbsp; ex) TTS(Text-to-Speech) 모델, MusicLM, Bark
> **멀티모달 생성 모델** : 텍스트, 이미지, 음성 등 여러 형태의 데이터를 동시에 이해하고 생성
> &nbsp; ex) GPT-4V, Gemini, Flamingo

### 💡 LLM (Large Language Model)

#### 1. LLM이란

**LLM**은 자연어 처리(NLP)에서 사용되는 AI 기술의 한 종류입니다. **대규모 텍스트 데이터를 학습**하여 언어의 구조와 의미를 이해하고, 이를 바탕으로 텍스트 생성, 번역, 요약, 질의응답 등 다양한 작업을 수행할 수 있습니다. 대규모라는 이름은 학습에 사용되는 데이터와 모델의 크기를 의미합니다.

#### 2. LLM의 단점 및 한계점

LLM은 강력한 자연어 생성 능력을 가지고 있지만, 다음과 같은 단점과 한계가 있습니다.

- **최신 정보 반영 어려움** : 학습된 데이터가 특정 시점까지의 정보만 포함되므로, 실시간 뉴스나 최신 사건을 반영하기 어렵습니다.
- **할루시네이션 발생** : LLM이 생성한 텍스트는 모델이 학습한 패턴에 기반한 추론 결과이므로, 실제 사실과 다를 수 있습니다.
- **도메인 특화 어려움** : 일반 LLM은 광범위한 텍스트를 학습했기 때문에, 특정 분야 (예: 보이스피싱 시나리오)에 최적화된 답변을 만들기 어렵습니다.
- **연산 비용과 리소스 부담** : 대형 모델일수록 추론 속도와 비용이 높으며, 서버 환경에 부담을 줄 수 있습니다.

실제로 GPT를 사용하면서 엉뚱한 이야기를 하거나, 잘못된 답을 마치 정답인 것처럼 제공받았던 경험이 한 번쯤 경험해봤을 것입니다. 이처럼 LLM이 가지는 단점과 한계를 보완하기 위해 고안된 것이 바로 RAG(검색 증강 생성)입니다.

> **✅ 할루시네이션 (Hallucination)**
> 편향되거나 불충분한 학습 데이터, 모델의 과적합 등으로 인해 LLM이 부정확한 정보를 생성하는 현상

### 💡 RAG (Retrieval-Augmented Generation)

#### 1. RAG란

RAG는 LLM에 외부 지식 베이스를 연결하여, 모델이 **학습된 패턴만으로 텍스트를 생성하는 것을 넘어, 사실 기반 정보와 맥락을 활용하여 답변을 생성**하도록 돕습니다. 예를 들어 GPT에게 질문을 보낼 때, 일반 LLM은 학습한 내용에 기반한 답변을 제공하지만, RAG는 외부 데이터에서 관련 정보를 검색하여 GPT에게 전달함으로써 더 정확하고 최신 정보에 기반한 답변을 만들어낼 수 있습니다. RAG는 위에서 알아본 LLM의 단점 중 **최신 정보 반영의 어려움**과 **할루시네이션 문제**를 주로 개선하며, **도메인 특화**는 지식베이스 설계에 따라 보조적 효과를 기대할 수 있습니다.
다만, RAG의 성능은 연결된 **지식 베이스의 품질과 범위**에 크게 좌우되므로, 고품질의 데이터 구축이 무엇보다 중요합니다.

#### 2. RAG 동작 과정

<img src="/assets/img/devlog/10/dev10_1.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

> 출처: https://aws.amazon.com/ko/what-is/retrieval-augmented-generation

<br>

**1️⃣ 사용자 입력 (Prompt + Query)**

- 사용자가 모델에 전달할 질문이나 명령어 입력
- 이 입력에는 모델에게 수행할 작업(Prompt)과 정보 검색 대상(Query)이 포함됨

```Python
user_input = {
    "prompt": "2023년 기준 대한민국 인구 통계 알려줘",
    "query": "대한민국 인구 2023"
}
```

**2️⃣ 관련 정보 검색**

- 시스템은 Query를 바탕으로 **Knowledge Base**(문서, 데이터베이스, vector DB 등)에서 관련 정보 검색
- 검색 방법: 키워드 매칭, 임베딩 기반 유사도 검색 등등

```Python
# 예시: 간단한 vector DB 검색
from sentence_transformers import SentenceTransformer, util

model = SentenceTransformer('all-MiniLM-L6-v2')
query_emb = model.encode(user_input['query'], convert_to_tensor=True)

# documents_emb: 미리 임베딩된 문서들의 리스트
scores = util.cos_sim(query_emb, documents_emb)
top_idx = scores.argmax()
relevant_doc = documents[top_idx]
```

**3️⃣ Enhanced Context 생성**

- 검색된 정보를 **컨텍스트로 변환**하여 LLM이 이해할 수 있는 형태로 제공
- 기존 Query만 전달하면 모델이 모르는 정보가 있을 수 있으므로, 검색된 정보를 포함시키는 것이 핵심

```Python
enhanced_context = f"검색된 정보: {relevant_doc}"
```

**4️⃣ LLM에 전달 (Prompt + Query + Enhanced Context)**

- 이제 LLM에게 사용자의 원래 입력과 검색된 정보를 함께 전달
- LLM은 Enhanced Context를 참고하여 더 정확하고 사실 기반의 응답 생성

```Python
from openai import OpenAI

client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "system", "content": "당신은 신뢰할 수 있는 정보 제공자입니다."},
        {"role": "user", "content": f"{user_input['prompt']}\n{enhanced_context}"}
    ]
)
```

**5️⃣ LLM 출력 (Generated Text Response)**

- LLM이 Enhanced Context를 바탕으로 답변 생성
- 단순한 일반 생성보다, **검색된 사실 기반 정보**를 포함하기 때문에 정확도가 높아짐

```Python
generated_text = response.choices[0].message['content']
print(generated_text)
```

다음 포스팅에서는 **RAG 기반 시나리오 생성 프로세스**와 **품질 평가 및 모니터링 방법**에 대해 자세히 알아보겠습니다.
