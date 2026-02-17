---
title: '[낚시없는 우리집] 시나리오 자동 생성 파이프라인과 RAG 품질 평가 방법'
tags:
  - Project
  - LLM
  - RAG
date: '2025-08-31'
thumbnail: '/assets/img/devlog/09/dev09_0.png'
---

이번 포스팅에서는 **RAG 기반 낚시없는 우리집의 시나리오 자동 생성 파이프라인**과 **품질 평가 방법**에 대해 알아보겠습니다.

## 🏠 시나리오 자동 생성 파이프라인

---

<img src="/assets/img/devlog/11/dev11_1.png" alt="velog 404" style="display: block; margin: 0 auto; max-width: 900px; max-height: 600px;">

### 1️⃣ 뉴스 입력 받기 / 요약 생성

- **역할**
  - 크롤링 + 전처리 이후의 **정제된 뉴스 데이터**를 불러오고, 핵심 요약문을 생성.
- 실제 구현에서는
  - 크롤링/전처리 파이프라인에서 전달된 JSON/DB에서 불러오기
  - 요약은 단순 문자열 가공이 아니라 **요약 모델**이나 LLM을 이용 가능

👉 결과적으로 `이 뉴스가 어떤 사기 수법을 다루는지` 요약문이 만들어짐.

```Python
# data_loader.py
def load_news():
    """정제된 뉴스 데이터 입력"""
    return {
        "title": "보이스피싱 조직, 검찰 사칭해 5천만원 편취",
        "content": "사기범들은 검찰청 직원이라고 속이며 피해자에게 전화를 걸어 은행 계좌 정보를 요구..."
    }

def summarize_news(news: dict) -> str:
    """뉴스 요약문 생성 (실제로는 요약 모델 적용 가능)"""
    return "검찰청 사칭하여 피해자에게 계좌 정보를 요구"
```

### 2️⃣ RAG 검색 (중복 방지 + 컨텍스트 제공)

- **역할**
  - 뉴스 요약문을 **임베딩 벡터**로 변환
  - 벡터DB(예: Pinecone, Weaviate, ChromaDB)에서 **기존 시나리오 검색**
- 목적
  - **중복 방지**: 이미 같은 유형의 시나리오가 있으면 새로운 생성 막음
  - **참고 자료 제공**: LLM이 기존 시나리오를 보고 더 자연스럽고 다양한 대화를 생성하도록 도움

👉 여기서는 `MockVectorDB`를 만들었지만 실제 운영에서는 **클라우드 벡터 DB**로 교체.

```Python
# rag_search.py
from sentence_transformers import SentenceTransformer
import torch

embedder = SentenceTransformer("all-MiniLM-L6-v2")

class MockVectorDB:
    """실제 환경에서는 Pinecone / Weaviate / ChromaDB 사용"""
    def __init__(self):
        self.scenarios = [
            "검찰을 사칭하여 계좌번호와 비밀번호를 요구하는 보이스피싱",
            "대출을 빌미로 개인정보를 요구하는 사기",
            "메신저 피싱으로 가족을 사칭하며 송금을 유도"
        ]
        self.embeddings = embedder.encode(self.scenarios, convert_to_tensor=True)

    def search(self, query_emb, top_k=3):
        cos_scores = torch.nn.functional.cosine_similarity(query_emb, self.embeddings)
        top_results = cos_scores.topk(top_k)
        return [self.scenarios[idx] for idx in top_results.indices.tolist()]

def search_similar_scenarios(news_summary: str, vector_db: MockVectorDB):
    query_emb = embedder.encode(news_summary, convert_to_tensor=True).unsqueeze(0)
    cos_scores = torch.nn.functional.cosine_similarity(query_emb, vector_db.embeddings)
    top_results = cos_scores.topk(3)
    return [(vector_db.scenarios[idx], cos_scores[idx].item()) for idx in top_results.indices.tolist()]
```

👉 유사한 시나리오 리스트, 유사도 점수 같이 리턴

```Python
[("검찰을 사칭하여 계좌번호 요구", 0.92), ("대출 사기", 0.65)]
```

### 3️⃣ 실제 대화 시나리오 생성

- **역할**
  - 유사도 점수에 따라 생성 방식 분기:
    1. 유사도 높음(예: 0.85 이상) → 기존 시나리오를 그대로 쓰지 않고, **랜덤 변형 전략**을 적용해 새로운 버전 생성
    2. 유사도 낮음 → 기존 시나리오에 의존하지 않고, 완전히 새로운 시나리오 생성, 생성된 시나리오 벡터화해서 벡터 DB에 저장
  - 뉴스 요약문과, RAG 검색 결과(기존 유사 시나리오 + 유사도 점수)를 받아 **LLM에게 전달**
- **조건 (프롬프트에 명시)**
  - 최소 6턴 이상
  - 전화 대화 형식
  - 기존 시나리오와 중복되지 않도록 변형
- **변형 전략 (랜덤 적용)**
  - 피해자 특성을 바꾸기 (예: 대학생 → 직장인, 자영업자)
  - 사기 금액·조건 변경 (예: 5000만 원 → 300만 원, 소액 대출 등)
  - 접근 방식 바꾸기 (예: 전화 → 문자, 메신저, 이메일)
  - 사칭 대상 변경 (예: 검찰 → 경찰, 은행 직원, 건강보험공단)
  - 대화 전개 순서 바꾸기 (예: 협박 → 보안 안내 → 송금 요구 순서 조정)

👉 즉, RAG 결과를 활용해 **중복 방지 + 다양성 확보**

```Python
# scenario_generator.py
import random
from openai import OpenAI

client = OpenAI()

VARIATION_PATTERNS = [
    "피해자의 연령대, 직업, 배경을 변경하세요 (예: 대학생, 직장인, 자영업자 등).",
    "사기 금액이나 요구 조건을 바꾸세요 (예: 5천만원 → 300만원, 소액 대출 등).",
    "접근 수단을 바꾸세요 (예: 전화 대신 문자/메신저/이메일).",
    "사칭 대상을 변경하세요 (예: 검찰 → 경찰, 은행 직원, 건강보험공단).",
    "대화의 전개 순서를 바꿔 보세요 (예: 먼저 협박 → 보안 안내 → 송금 요구)."
]

def generate_scenario(news_summary: str, similar_scenarios: list, threshold=0.85) -> str:
    if similar_scenarios and similar_scenarios[0][1] > threshold:
        # 유사한 시나리오 → 변형 모드
        base_scenario = similar_scenarios[0][0]
        variation_instruction = random.choice(VARIATION_PATTERNS)

        prompt = f"""
아래 기존 시나리오와 유사한 뉴스가 들어왔습니다.
완전히 같지 않도록 **새로운 변형 시나리오**를 작성하세요.

기존 시나리오:
{base_scenario}

뉴스 요약:
{news_summary}

변형 조건:
- {variation_instruction}

공통 조건:
1. 최소 6턴 이상의 대화
2. 기존 시나리오와 중복되지 않도록 세부 내용을 변경
3. 사기 수법의 본질은 유지
4. 실제 보이스피싱 대화처럼 자연스럽게 표현
"""
    else:
        # 새로운 시나리오 모드
        prompt = f"""
다음 뉴스를 바탕으로 보이스피싱 상황을 재현한 '대화형 시나리오'를 작성해 주세요.
조건:
1. 최소 6턴 이상의 대화 (사기범 ↔ 피해자)
2. 사기 수법이 명확히 드러나야 함
3. 실제 전화 대화처럼 자연스럽게 표현
뉴스 요약:
{news_summary}

참고할 기존 시나리오(중복 방지 참고용):
{[s[0] for s in similar_scenarios]}
"""

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content

```

### 4️⃣ 품질 검증 (룰 기반 + LLM 평가)

- **역할**
  - 생성된 시나리오를 검증/평가하는 단계
- 2가지 레이어로 구성
  1. **룰 기반 필터**
     - 시나리오 길이가 너무 짧은지
     - 욕설, 불필요한 표현 포함 여부
  2. **LLM 평가**
     - 품질 점수(1~5점) 부여
     - 평가 기준: 사실성 / 자연스러움 / 사기 수법 전달력

👉 최종적으로 **3점 이상**이어야 DB에 저장 가능.

```Python
# evaluator.py
from openai import OpenAI

client = OpenAI()

def rule_based_check(scenario: str) -> bool:
    """룰 기반 필터: 너무 짧거나 욕설 포함 여부"""
    if len(scenario.splitlines()) < 6:  # 최소 6턴
        return False
    if any(bad_word in scenario for bad_word in ["욕설1", "욕설2"]):
        return False
    return True

def evaluate_scenario(scenario: str) -> int:
    """LLM 기반 품질 평가 (1~5점)"""
    eval_prompt = f"""
다음 시나리오를 1~5점으로 평가하세요.
기준: (1) 사실성 (2) 대화 자연스러움 (3) 사기 수법 전달력
시나리오:
{scenario}
"""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": eval_prompt}]
    )
    try:
        return int(response.choices[0].message.content.strip())
    except:
        return 1
```

### 5️⃣ DB 저장

- **역할**
  - 합격한 시나리오를 **데이터베이스에 저장**
- 현재 예시는 SQLite → 운영 환경에서는 MySQL/Postgres/MongoDB 등 확장 가능
- 저장 필드
  - 뉴스 제목 `title`
  - 요약 `summary`
  - 생성된 시나리오 `scenario`
  - 품질 점수 `score`

👉 향후 관리자 검수 시스템과 연결 가능

```Python
# storage.py
import sqlite3

def save_to_db(title: str, summary: str, scenario: str, score: int):
    conn = sqlite3.connect("scenarios.db")
    cur = conn.cursor()
    cur.execute("""CREATE TABLE IF NOT EXISTS scenarios (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT,
        summary TEXT,
        scenario TEXT,
        score INTEGER
    )""")
    cur.execute("INSERT INTO scenarios (title, summary, scenario, score) VALUES (?, ?, ?, ?)",
                (title, summary, scenario, score))
    conn.commit()
    conn.close()
```

지금까지 작성한 전체 파이프라인을 순차 실행하는 엔트리포인트입니다.

```Python
# main.py
from data_loader import load_news, summarize_news
from rag_search import MockVectorDB, search_similar_scenarios
from scenario_generator import generate_scenario
from evaluator import rule_based_check, evaluate_scenario
from storage import save_to_db

if __name__ == "__main__":
    # 1. 입력
    news = load_news()
    summary = summarize_news(news)

    # 2. RAG 검색
    vector_db = MockVectorDB()
    similar = search_similar_scenarios(summary, vector_db)

    # 3. 시나리오 생성
    scenario = generate_scenario(summary, similar, threshold=0.85)

    # 4. 검증
    if rule_based_check(scenario):
        score = evaluate_scenario(scenario)
        if score >= 3:
            save_to_db(news["title"], summary, scenario, score)
            print("✅ 시나리오 저장 완료")
        else:
            print("❌ 품질 미달 (LLM 평가 점수 낮음)")
    else:
        print("❌ 룰 기반 필터에서 탈락")
```

## 🏠 RAG 품질 평가 방법

---

RAG를 활용한 시나리오 생성 파이프라인에서, RAG의 성능은 **검색 정확도와 관련성**이 핵심입니다. 시나리오 생성 단계에서 LLM은 품질을 담당하지만, RAG는 기존 시나리오 검색과 참고 자료 제공에 집중합니다. 따라서 RAG 자체도 주기적으로 품질을 평가하고 모니터링해야 합니다.

### 1️⃣ 평가 지표

- **Top-K 정확도 (Top-K Accuracy)**
  - 검색한 벡터 중 실제 관련 시나리오가 몇 개 포함되어 있는지 확인
  - 예: 뉴스 요약문과 가장 유사한 시나리오가 Top-3 안에 있는지
- **평균 순위 Reciprocal (MRR)**
  - 검색 결과에서 관련 시나리오가 상위 몇 번째로 나타나는지 측정
  - 1순위에 나타날수록 점수 높음
- **Cosine 유사도 평균**
  - 쿼리 요약문과 검색된 시나리오 벡터 간 유사도 평균
  - 유사도가 낮으면 벡터 임베딩 모델 개선 필요
- **중복 방지 효과**
  - 새로운 시나리오 생성 시, 기존 시나리오와 얼마나 겹치지 않는지 평가
  - RAG 검색 결과 활용 여부에 따라 중복 발생률 측정

### 2️⃣ 평가 방법

RAG 품질 평가는 주로 실제 쿼리와 시나리오 간 관련성을 측정하는 방식으로 진행합니다.

- **샘플 기반 테스트 (Manual / Semi-Automated)**
  - 미리 준비된 뉴스 요약문 샘플과, 이와 관련된 정답 시나리오를 정의
  - RAG 검색을 수행하여 상위 K개의 시나리오가 실제 관련 시나리오를 포함하는지 확인
  - 계산 지표: Top-K Accuracy, MRR, Cosine 유사도
- **자동화 평가 (Automated)**
  - Regas와 같은 도구를 활용하여 정기적 평가 및 모니터링
  - 뉴스 요약문을 쿼리로 입력 → RAG 검색 결과와 정답 시나리오 비교
  - 성능 지표 기록, 시각화, 임계치 알람 설정 가능
  - 벡터 임베딩 모델 업데이트, RAG 검색 파라미터 튜닝 여부 결정
- **중복 방지 테스트**
  - 기존 시나리오 벡터와 새로 생성된 시나리오 벡터 간 유사도 측정
  - RAG 검색 결과를 활용했을 때, 얼마나 중복이 줄어드는지 통계화
  - 유사도 기준(threshold) 설정: 예를 들어 0.85 이상이면 변형, 미만이면 완전 신규
