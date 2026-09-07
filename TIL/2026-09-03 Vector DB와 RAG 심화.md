# 2026-09-03 TIL — Vector DB, RAG Pipeline, RAG Search 심화

오늘은 07(Vector DB), 08(RAG Pipeline), 09(RAG Search Advanced) 세 개 노트북을 진행했다.

## 07. Vector DB (Chroma)

임베딩(embedding)된 벡터를 그냥 리스트에 저장하는 게 아니라, 검색에 최적화된 저장소인 **Vector DB**에 저장한다.

```python
from langchain_chroma import Chroma
from langchain_google_genai import GoogleGenerativeAIEmbeddings

embeddings = GoogleGenerativeAIEmbeddings(model="models/text-embedding-004")

# 문서 리스트를 임베딩해서 Chroma에 저장하고, persist_directory에 파일로 남긴다
vectorstore = Chroma.from_documents(
    documents=split_docs,
    embedding=embeddings,
    persist_directory="./chroma_db",
)

# 유사한 문서 3개를 검색
results = vectorstore.similarity_search("질문 내용", k=3)

# 점수(거리)까지 같이 받기 — 기본은 Euclidean(L2) 거리라서 값이 작을수록 유사함
results_with_score = vectorstore.similarity_search_with_score("질문 내용", k=3)
```

- `persist_directory`가 있으면 임베딩 결과가 디스크에 저장돼서, 다음에 또 임베딩 API를 부를 필요가 없다.
- 벡터가 몇만 개 이상으로 많아지면 전체를 다 비교(brute-force)하는 대신 **ANN**(Approximate Nearest Neighbor, 근사 최근접 이웃) 알고리즘(HNSW, IVF 등)으로 빠르게 후보를 좁힌다.
- `similarity_search_with_score`의 점수는 코사인 유사도가 아니라 기본적으로 **L2 거리**라서, "점수가 높을수록 좋다"가 아니라 "거리가 작을수록 좋다"로 해석해야 한다.
- 메타데이터로 필터링도 가능하다: `vectorstore.similarity_search(query, filter={"category": "보안"})`

## 08. RAG Pipeline — 검색과 답변을 하나의 체인으로

검색(retrieve)과 답변 생성(generate)을 각각 따로 `.invoke()`하면 동작은 하지만, LangSmith에 트레이스가 나란히 흩어져서(형제 관계) 하나의 요청으로 묶이지 않는다.

그래서 `RunnablePassthrough.assign()`을 체인으로 이어 붙여서, 검색부터 답변까지 **하나의 Runnable**로 만들었다.

```python
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

def retrieve_docs(input_dict):
    return retriever.invoke(input_dict["question"])

def build_context(input_dict):
    # 검색된 문서들을 출처 표시(citation)까지 포함한 문자열로 합친다
    return "\n\n".join(f"[{d.metadata['source']}] {d.page_content}" for d in input_dict["docs"])

rule_rag_chain = (
    RunnablePassthrough.assign(docs=retrieve_docs)
    | RunnablePassthrough.assign(context=build_context)
    | RunnablePassthrough.assign(answer=(citation_prompt | llm | StrOutputParser()))
)

result = rule_rag_chain.invoke({"question": "연차는 며칠이야?"})
print(result["answer"])   # 최종 답변
print(result["docs"])     # 답변에 쓰인 원본 문서(출처 확인용)
```

`RunnablePassthrough.assign(key=함수)`는 현재까지의 dict 전체를 그 함수에 넣어서 실행하고, 결과를 새 key로 **같은 레벨에** 추가해서 반환한다. 그래서 `docs` → `context` → `answer` 순서로 dict가 점점 커지면서, 마지막에도 처음 넣은 `question`과 중간에 만든 `docs`까지 전부 꺼내 쓸 수 있다.

이렇게 하나의 체인으로 묶으면 LangSmith에서 검색과 생성이 부모-자식 관계인 트리 구조로 보인다.

## 09. RAG Search Advanced — 검색 품질 높이기

기본 유사도 검색(similarity search)만으로는 한계가 있어서, 상황에 맞는 검색 방식을 추가로 배웠다.

**MMR (Max Marginal Relevance)** — 그냥 유사도 순으로 뽑으면 비슷한 내용만 여러 개 뽑힐 수 있다. MMR은 "질문과 관련 있으면서도, 이미 뽑은 문서들과는 겹치지 않는" 문서를 우선한다.

```python
mmr_docs = vectorstore.max_marginal_relevance_search(
    query, k=6, fetch_k=20, lambda_mult=0.5
)
```

`fetch_k`로 일단 후보를 넉넉히 가져온 다음, 그중에서 다양성을 고려해 `k`개를 추린다. `lambda_mult`는 관련성과 다양성 중 뭘 더 우선할지 조절하는 값이다(1에 가까울수록 관련성 우선).

**BM25** — 벡터 유사도는 "의미"가 비슷한 걸 찾지만, 특정 고유명사·용어가 정확히 일치하는지는 잘 못 잡을 때가 있다. BM25는 키워드 빈도 기반의 전통적인 검색 방식이라 정확한 용어 매칭에 강하다. 한국어는 조사 때문에 그냥 띄어쓰기로 쪼개면 안 돼서, 형태소 분석기(Kiwi)로 토큰화해서 넣었다.

```python
from langchain_community.retrievers import BM25Retriever

def kiwi_tokenize(text):
    return [token.form for token in kiwi.tokenize(text)]

bm25_retriever = BM25Retriever.from_documents(
    documents, preprocess_func=kiwi_tokenize, k=5
)
```

**Metadata Filter** — 문서에 미리 붙여둔 메타데이터(카테고리 등)로 검색 범위를 좁힌다. `filter={"category": "보안"}`처럼 넘기면 그 카테고리 안에서만 유사도 검색을 한다.

**Hybrid Search** — 벡터 검색(의미 기반)과 BM25(키워드 기반)를 합쳐서 서로의 약점을 보완한다. `EnsembleRetriever`가 두 검색기의 결과를 **RRF(Reciprocal Rank Fusion)** 방식으로 합친다.

```python
from langchain.retrievers import EnsembleRetriever

hybrid_retriever = EnsembleRetriever(
    retrievers=[vector_retriever, bm25_retriever],
    weights=[0.5, 0.5],
)
```

RRF는 각 검색기가 매긴 원래 점수 크기는 보지 않고, **순위(rank)**만 가지고 합산한다. 두 검색기는 완전히 독립적으로 각자 검색을 실행한 뒤, 마지막에 순위만 합치는 방식이라 계산 자체는 단순하지만, 한쪽이 놓친 문서를 다른 쪽이 잡아주는 효과가 있어서 "하이브리드"라고 부른다.

**Re-ranking** — 벡터/BM25로 빠르게 후보를 추린 다음, 그 후보들만 LLM에 넣어서 관련성을 다시 채점하는 방식이다. 전체 문서를 LLM에 다 넣지 못하는 이유는 비용뿐 아니라 **컨텍스트 윈도우(context window)** 물리적 한계 때문이기도 하다.

```python
from pydantic import BaseModel

class RelevanceScore(BaseModel):
    score: int
    reason: str

def rerank(query, docs):
    scored = []
    for doc in docs:
        result = (rerank_prompt | llm.with_structured_output(RelevanceScore)).invoke(
            {"query": query, "document": doc.page_content}
        )
        scored.append((doc, result.score, result.reason))
    return sorted(scored, key=lambda x: x[1], reverse=True)
```

## 오늘의 실습 — 사내 규정 검색 (integrated-practice-code)

회사 규정 문서(`data/company_rules`)를 대상으로 위에서 배운 걸 다 적용해봤다.

- 여러 규정에 걸친 질문 → Similarity vs MMR 비교 (MMR이 더 다양한 규정을 포함하는지 확인)
- 규정 안의 정확한 용어("코어타임") 찾기 → Similarity vs BM25 비교
- 카테고리 필터링 → 필터 없음 vs `category="보안"` 비교
- Similarity + BM25 → `EnsembleRetriever`로 Hybrid Search
- Hybrid 후보를 `rerank()`로 재정렬해서 최종 우선순위 매기기

## 핵심 한 줄 정리

- Vector DB는 임베딩을 저장하고 빠르게 검색하는 저장소이고, 점수는 기본적으로 거리(작을수록 유사)다.
- RAG 파이프라인은 검색과 생성을 하나의 LCEL 체인으로 묶어야 LangSmith에서 제대로 된 트리 트레이스가 나온다.
- 유사도 검색만으로 부족할 때 MMR(다양성), BM25(정확한 용어), Metadata Filter(범위 좁히기), Hybrid(둘의 결합), Re-ranking(LLM 재채점)을 상황에 맞게 골라 쓴다.
