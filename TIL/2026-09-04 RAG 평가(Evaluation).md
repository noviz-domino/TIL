# 2026-09-04 TIL — RAG 평가 (10-rag-evaluation)

오늘은 10번 노트북 "RAG 평가"를 진행했다. 검색이 잘 됐는지와 답변이 잘 나왔는지를 나눠서 평가하는 방법을 배웠다.

## 왜 검색 평가와 답변 평가를 나누는가

최종 답변이 이상할 때, 원인이 "검색이 애초에 엉뚱한 문서를 가져와서"인지 "검색은 맞았는데 LLM이 답변을 잘못 만들어서"인지 구분해야 어디를 고칠지 알 수 있다. 그래서 두 단계를 따로 평가한다.

```
질문 → 검색기가 문서를 찾음 → (검색 평가) → LLM이 답변을 만듦 → (답변 평가)
```

정답(기대 정답, 정답 문서)은 AI가 정하는 게 아니라 **사람이 미리 정리해둔 기준 데이터**(`eval_qa_50.json`)다. 검색기가 실제로 뽑아온 결과를 이 사람이 정한 정답과 비교하는 구조다. 관련도 1등으로 뽑힌 문서가 실제 정답이라는 보장은 없기 때문에 이런 비교가 필요하다.

## 검색 평가 지표 4가지

**Hit@k** — 상위 k개 안에 정답 문서가 하나라도 있으면 1점, 없으면 0점.

```python
def hit_at_k(ranked, relevant, k):
    return int(bool(set(ranked[:k]) & set(relevant)))
```

`ranked[:k]`로 상위 k개만 자르고, `set()` 두 개를 `&`로 겹치는 부분만 뽑은 뒤, `bool()`로 있다/없다 판단하고, `int()`로 1/0으로 바꾼다. `int(bool(...))`처럼 명시적으로 안 바꿔도, 파이썬은 `sum()` 같은 계산 안에서는 `True`/`False`를 자동으로 1/0 취급한다(`bool`이 `int`의 하위 타입이라서).

**Precision@k** — 상위 k개 중 정답이 차지하는 비율. 뽑아온 것 중 쓸데없는 게 얼마나 섞였는지를 본다.

```python
def precision_at_k(ranked, relevant, k):
    selected = ranked[:k]
    hits = sum(source in relevant for source in selected)
    return hits / len(selected) if selected else 0.0
```

`source in relevant for source in selected`는 generator 표현식이고(값을 즉시 흘려보내기만 함), `sum()`이 그 True/False들을 다 더해서 개수를 센다. `A if 조건 else B`는 조건에 따라 값을 고르는 문법으로, `selected`가 비어서 0으로 나누는 걸 막아준다.

**Recall@k** — 전체 정답 중 몇 개를 찾았나. 놓친 정답이 있는지를 본다.

```python
def recall_at_k(ranked, relevant, k):
    hits = len(set(ranked[:k]) & set(relevant))
    return hits / len(set(relevant))
```

Precision과 핵심 차이는 **분모**다. Precision은 "내가 뽑아온 개수"로 나누고, Recall은 "전체 정답 개수"로 나눈다. 그리고 Recall은 `set()`을 거쳐서 중복이 제거되기 때문에, 같은 문서에서 나온 청크가 여러 개 뽑혀도 한 번만 센다 — 반면 Precision은 청크 단위 리스트를 그대로 세기 때문에, 청크가 많은(글자 수가 긴) 문서가 상위권을 여러 자리 차지하면 점수가 왜곡될 수 있다는 한계가 있다.

**RR(Reciprocal Rank) / MRR** — 정답이 순위상 몇 등에서 처음 나왔는지의 역수.

```python
def reciprocal_rank(ranked, relevant):
    relevant = set(relevant)
    for rank, source in enumerate(ranked, start=1):
        if source in relevant:
            return 1 / rank
    return 0.0
```

`enumerate(ranked, start=1)`은 원소와 함께 순번(1등부터)을 같이 꺼내준다. `for` 안에서 정답을 찾는 순간 `return`으로 함수를 즉시 끝내버려서, 그 이후 등수는 아예 확인하지 않는다. 정답을 하나도 못 찾으면 `for`문이 끝까지 다 돌고 마지막 `return 0.0`에 도달한다. MRR은 이 RR을 50개 질문 전체에 대해 계산한 평균이다.

이 네 지표를 `evaluate_strategy()` 함수로 Similarity/MMR/BM25/Hybrid 네 전략 각각에 적용해서 표로 비교했다.

## 규칙 기반 인용 검사

검색된 문서로 실제 답변을 만든 뒤(`GroundedAnswer` 구조화 출력으로 `answer`, `citations` 받음), 인용이 제대로 됐는지 코드로 확인한다.

```python
retrieved_sources = {doc.metadata["source"] for doc in docs}  # set comprehension
citations = set(result.citations)

"valid_citation": bool(citations) and citations <= retrieved_sources,
"relevant_citation": bool(citations & set(case["relevant_sources"])),
```

`valid_citation`은 "LLM이 인용한 출처가 실제로 검색된 문서 안에 있나"(hallucination 방지), `relevant_citation`은 "인용한 출처 중 정답 문서가 있나"를 본다. `<=`는 숫자에서는 "작거나 같다"지만 set끼리는 "부분집합인가"로 뜻이 완전히 달라진다 — 이건 LCEL의 `|`가 숫자와 Runnable에서 다르게 동작했던 것과 같은 **연산자 오버로딩**이다.

## RAGAS

LLM을 이용해서 답변의 **의미적 품질**을 채점하는 라이브러리. 규칙 기반 검사는 "형식이 맞나"만 보지만, RAGAS는 "내용이 진짜 맞나"를 본다.

- **Faithfulness(근거성)**: 답변의 주장들이 검색 문서로 뒷받침되는가. 문서엔 없는 내용을 답변이 추가하면 점수가 낮아진다.
- **Answer Relevancy(질문 관련성)**: 답변만 보고 역으로 질문 3개를 만들어, 원래 질문과의 평균 유사도를 잰다. 답변이 질문 핵심에 집중했다면 어느 부분에서 역질문을 뽑아도 원래 질문과 비슷하게 나온다.

Answer Relevancy가 "3개 평균"이라는 게 의미 있으려면, 역질문을 생성하는 단계의 `temperature`가 0이 아니어야 한다(0이면 매번 거의 같은 질문만 나와서 평균 낼 이유가 없어짐) — 이건 RAGAS 라이브러리 내부에 이미 그렇게 설정돼 있다.

이 둘은 각자 좁은 역할만 맡고, 두 점수를 같이 봐야 "근거는 있는데 질문과 무관", "질문엔 답했는데 근거 없음" 같은 걸 구분할 수 있다.

## LLM-as-Judge

RAGAS의 범용 기준 대신, 우리 도메인(사내 규정)에 맞는 채점 기준을 직접 정의한다.

```python
class JudgeScore(BaseModel):
    correctness: int = Field(ge=1, le=10, description="...")       # 정확성
    completeness: int = Field(ge=1, le=10, description="...")      # 완전성
    groundedness: int = Field(ge=1, le=10, description="...")      # 근거성
    citation_accuracy: int = Field(ge=1, le=10, description="...") # 출처 정확성
    reason: str = Field(description="...")
```

`Field`는 pydantic이 제공하는 함수로, 타입 힌트(`: int`) 외에 `description`(LLM에게 주는 작성 지시)과 `ge`/`le`(값 범위 검증, greater-equal/less-equal) 같은 추가 규칙을 필드에 붙인다.

```python
def judge_answer(item):
    contexts = "\n\n".join(item["retrieved_contexts"])  # 문서들을 문자열 하나로 합침
    result = judge_llm.invoke(...)                       # LLM에 채점 요청
    score = result.model_dump()                          # Pydantic 객체 -> dict 변환
    metric_names = ["correctness", "completeness", "groundedness", "citation_accuracy"]
    score["average"] = round(
        sum(score[name] for name in metric_names) / len(metric_names), 2
    )
    return score

judge_results = [judge_answer(item) for item in generation_results]  # 리스트 컴프리헨션
for item, score in zip(generation_results, judge_results):           # 두 리스트를 순서대로 짝지음
    print(...)
```

`.model_dump()`는 Pydantic 객체를 나중에 키를 자유롭게 추가할 수 있는 딕셔너리로 바꿔준다. `[... for ... in ...]`(대괄호)는 generator 표현식과 달리 결과를 실제 **리스트로 저장**해서, 뒤 코드(출력 단계)에서 다시 꺼내 쓸 수 있게 한다. `zip()`은 두 리스트를 같은 순번끼리 짝지어 튜플로 묶어준다(길이가 다르면 짧은 쪽 기준으로 잘림).

## 핵심 한 줄 정리

- 검색 평가(Hit@k, Precision@k, Recall@k, MRR)와 답변 평가(인용 검사, RAGAS, LLM-as-Judge)를 나누면 문제 원인을 정확히 찾을 수 있다.
- Precision과 Recall의 차이는 분모(내가 뽑은 개수 vs 전체 정답 개수)이고, `set()`으로 중복을 제거하느냐가 청크 개수 편향 문제를 좌우한다.
- 파이썬 연산자(`<=`, `|`)는 타입에 따라 다르게 동작하도록 오버로딩될 수 있고, `bool`은 숫자 계산에서 자동으로 1/0 취급된다.
