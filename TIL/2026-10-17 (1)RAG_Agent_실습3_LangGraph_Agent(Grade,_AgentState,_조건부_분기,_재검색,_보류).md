---
tags: [rag, agent, langgraph, structured-output, practice]
til: v2 2026-10-17
---

# 2026-10-17 (1)RAG Agent 실습3 LangGraph Agent(Grade, AgentState, 조건부 분기, 재검색, 보류)
> 작성일: 2026-10-17

## 🔗 관련 글

- [RAG Agent 실습2 임베딩, 검색, 기본 RAG](2026-10-16%20(1)RAG_Agent_실습2_임베딩_검색_기본_RAG(Chroma,_build_index,_search,_LCEL).md) — 이 글의 Agent가 그대로 재사용하는 `search`, `format_context`, `answer_chain`, `NO_ANSWER`가 들어 있다. 기본 RAG는 이 글의 Agent와 비교하는 기준선이다.
- [LangGraph 기초(State, Node, Edge와 조건부 분기, 반복, Reducer)](2026-09-08_LangGraph_기초(State_Node_Edge와_조건부분기_반복_Reducer).md) — `StateGraph`, 노드, 엣지, 조건부 엣지의 기본 개념이다. 이 글은 그 개념을 RAG Agent에 적용한다.
- [Agentic RAG(Hybrid Retrieval, grade_documents, HyDE)](2026-09-16_Agentic_RAG(Hybrid_Retrieval,_grade_documents,_HyDE).md) — 검색 결과를 평가(grade)하고 다시 검색하는 Agentic RAG를 다룬 글이다. 이 글의 `grade` → `rewrite` → `retrieve` 반복과 같은 구조다.
- [대화 메모리와 Structured Output](2026-09-01%20(3)%20대화%20메모리와%20Structured%20Output.md) — `llm.with_structured_output(Grade)`로 모델의 응답 형식을 강제하는 기법의 바탕이다.

## ❓ 아직 모르겠는 것

대화에서 사용자가 질문하거나 막힌 부분은 따로 기록되어 있지 않다. 이 단계의 코드는 실습을 마친 뒤 모두 이해한 상태다. 복습하면서 막힌 곳이 생기면 이 자리에 추가한다.

- 

## 주제 1: Agent의 전체 흐름

기본 RAG(10-16)는 검색을 한 번만 하고 답한다. 검색이 빗나가면 그대로 틀린 답(또는 보류)이 나온다. 이 실습의 Agent는 **검색 → 근거 판단 → 재검색 / 답변 / 보류**를 반복하는 LangGraph 그래프다.

```
START → retrieve → grade ─┬─ 근거 충분 ───────────────→ generate → END
                          ├─ 부족 & 검색 3회 미만 → rewrite → retrieve (반복)
                          └─ 부족 & 3회 도달 ───→ abstain(보류) → END
```

| 노드 | 하는 일 |
|---|---|
| `retrieve` | 검색어로 상위 k개 청크를 가져온다. 결과를 누적 근거에 합친다 |
| `grade` | 지금까지 모은 근거로 질문에 답할 수 있는지 모델이 판단한다 |
| `rewrite` | 근거가 부족할 때 검색어를 바꾼다 |
| `generate` | 모은 근거로 답변한다 |
| `abstain` | 최대 검색 횟수까지 찾았는데도 부족하면 답변을 보류한다 |

설계 문서(`RAG_Agent_실습_전체흐름.docx`)에는 노드 이름이 `retrieve`, `grade_evidence`, `generate_answer`, `rewrite_query`, `abstain`으로 적혀 있다. 코드에서는 `grade`, `generate`, `rewrite`로 줄여 썼다. 같은 노드다.

문서의 중요한 원칙: **기본 RAG의 검색(`retrieve`)과 답변 생성(`generate_answer`)을 Agent에서 그대로 재사용해야 두 방식의 비교가 공정하다.** 코드에서 `retrieve_node`는 같은 `search`를, `generate_node`는 같은 `answer_chain`을 쓴다. 다른 점은 "근거를 판단하고 다시 찾는 과정"뿐이다.

## 주제 2: 판단·재작성 체인과 Structured Output

```python
class Grade(BaseModel):
    """근거 판단 결과. 모델이 이 형식으로 대답하도록 강제한다 (Structured Output)."""
    sufficient: bool = Field(description="검색된 근거만으로 질문에 답할 수 있으면 true, 아니면 false")
    reason: str = Field(description="판단 이유를 한 문장으로")


grade_chain = ChatPromptTemplate.from_messages([
    ("system",
     "검색된 근거로 질문에 답할 수 있는지 판단합니다. 답에 필요한 핵심 내용이 근거에 있으면 sufficient=true, "
     "관련은 있지만 핵심이 빠졌거나 무관하면 false 입니다."),
    ("human", "질문: {question}\n\n근거:\n{context}"),
]) | llm.with_structured_output(Grade)

rewrite_chain = ChatPromptTemplate.from_messages([
    ("system",
     "문서 검색에 쓸 검색어를 다시 만듭니다. 이전에 쓴 검색어와 다른 표현(동의어, 핵심 용어 중심)으로, "
     "검색어 한 줄만 출력하세요. 설명은 쓰지 마세요."),
    ("human", "원래 질문: {question}\n이전 검색어: {queries}\n근거가 부족했던 이유: {reason}"),
]) | llm | StrOutputParser()
```

### `Grade`와 Structured Output

- `Grade`는 Pydantic의 `BaseModel`을 상속한 클래스다. **모델이 돌려줄 응답의 형식**을 정한다: `sufficient`(참/거짓)와 `reason`(한 문장 이유).
- `Field(description=...)`의 설명은 모델에게 **각 필드가 무엇인지 알려 주는 안내문**이다.
- `llm.with_structured_output(Grade)`: 모델이 자유로운 글 대신 **`Grade` 형식에 맞춰** 대답하게 한다. 결과로 `Grade` 객체가 나오므로 `grade.sufficient`, `grade.reason`처럼 **속성으로 바로 꺼내 쓸 수 있다.** "충분한 것 같아요" 같은 글을 코드에서 해석하는 일이 필요 없다. 판단 결과를 `if`문의 조건으로 직접 쓸 수 있다는 점이 핵심이다.
- 체인의 끝이 `StrOutputParser()`가 아니라 `with_structured_output(Grade)`다. 문자열이 아니라 `Grade` 객체를 돌려준다.

### `rewrite_chain`

- 검색어를 **새로 만드는** 체인이다. 시스템 규칙: 이전과 **다른 표현**(동의어, 핵심 용어 중심)으로, 검색어 **한 줄만** 출력하고 설명은 쓰지 않는다.
- 입력은 세 가지: 원래 질문, 이전에 쓴 검색어들, **근거가 부족했던 이유**(`grade`의 `reason`). 부족했던 이유를 알려 줘야 "무엇이 빠졌는지"를 고려한 새 검색어를 만들 수 있다.
- 출력은 문자열(`StrOutputParser`)이다. 검색어 한 줄이면 충분하다.

## 주제 3: 상태(`AgentState`)와 노드

### 상태 정의

```python
class AgentState(TypedDict):
    """그래프를 흐르는 상태. 노드는 바꾸고 싶은 항목만 딕셔너리로 돌려주면 그 항목이 갱신된다."""
    question: str        # 사용자 질문 (바뀌지 않는다)
    query: str           # 이번에 검색할 검색어
    queries: list        # 지금까지 쓴 검색어들
    search_count: int    # 검색한 횟수
    docs: list           # 이번 검색 결과
    all_docs: list       # 지금까지 모은 근거 (청크 ID 기준 중복 제거)
    sufficient: bool     # 근거가 충분한지
    reason: str          # 판단 이유
    answer: str          # 최종 답변
    abstained: bool      # 답변을 보류했는지
```

- **State**는 그래프의 모든 노드가 함께 읽고 쓰는 공용 작업판이다. `TypedDict`는 딕셔너리의 키와 값의 타입을 적어 두는 문법이다.
- 노드는 **바꾸고 싶은 항목만** 딕셔너리로 돌려주면 된다. 나머지 항목은 그대로 유지된다. 별도의 Reducer를 지정하지 않은 항목은 돌려준 값으로 **덮어쓴다.** 그래서 리스트(`all_docs`, `queries`)를 쌓고 싶을 때는 노드 안에서 **합친 새 리스트를 만들어 돌려줘야** 한다(아래 `retrieve_node`).
- `question`은 끝까지 바뀌지 않고, `query`는 재작성될 때마다 바뀐다. 이 둘을 나눈 것이 중요하다. 판단·답변·재작성은 **원래 질문**(`question`)을 기준으로 하고, 검색만 현재 검색어(`query`)로 한다.
- `docs`(이번 검색 결과)와 `all_docs`(누적 근거)를 따로 둔다.

### 노드 다섯 개

```python
def retrieve_node(state):
    """검색한다. 결과를 누적 근거(all_docs)에도 합치고, 검색 횟수와 검색어 기록을 갱신한다."""
    docs = search(state["query"], TOP_K)
    seen = {d["chunk_id"] for d in state["all_docs"]}                       # 이미 모은 청크 ID
    merged = state["all_docs"] + [d for d in docs if d["chunk_id"] not in seen]
    return {
        "docs": docs,
        "all_docs": merged,
        "search_count": state["search_count"] + 1,
        "queries": state["queries"] + [state["query"]],
    }


def grade_node(state):
    """지금까지 모은 근거로 답할 수 있는지 모델이 판단한다."""
    grade = grade_chain.invoke({"question": state["question"], "context": format_context(state["all_docs"])})
    return {"sufficient": grade.sufficient, "reason": grade.reason}


def rewrite_node(state):
    """근거가 부족할 때 검색어를 바꾼다."""
    new_query = rewrite_chain.invoke({
        "question": state["question"],
        "queries": ", ".join(state["queries"]),
        "reason": state["reason"],
    }).strip()
    return {"query": new_query}


def generate_node(state):
    """모은 근거로 답변한다."""
    answer = answer_chain.invoke({"question": state["question"], "context": format_context(state["all_docs"])})
    return {"answer": answer.strip(), "abstained": False}


def abstain_node(state):
    """최대 검색 횟수까지 찾았는데도 근거가 부족하면 답변을 보류한다."""
    return {"answer": NO_ANSWER, "abstained": True}
```

**`retrieve_node`**

- `search(state["query"], TOP_K)`: 현재 검색어로 검색한다(10-16의 `search` 그대로).
- `seen = {d["chunk_id"] for d in state["all_docs"]}`: 이미 모은 청크 ID의 **집합(set)**이다. 집합은 "있는지" 확인이 빠르다.
- `merged = state["all_docs"] + [d for d in docs if d["chunk_id"] not in seen]`: 기존 근거에 **이번에 새로 찾은 청크만** 덧붙인다. **청크 ID 기준으로 중복을 제거**하는 줄이다. 검색어를 바꿔도 같은 청크가 또 나올 수 있으므로 이 처리가 없으면 같은 근거가 반복해서 쌓인다.
- 돌려주는 네 가지: 이번 결과(`docs`), 누적 근거(`all_docs`), 검색 횟수 +1, 검색어 기록에 현재 검색어 추가.

**`grade_node`**: `all_docs`(지금까지 모은 **전체 근거**)를 `format_context`로 덩어리로 만들어 판단 체인에 넣는다. 이번 검색 결과(`docs`)만이 아니라 **누적 근거**로 판단하는 것이 핵심이다. 첫 검색에서 부족했던 내용을 두 번째 검색이 채울 수 있기 때문이다.

**`rewrite_node`**: 검색어 기록(`queries`)을 `", ".join(...)`으로 쉼표로 이어 붙여 넣고, 판단 이유(`reason`)도 넣는다. 돌려주는 것은 새 `query` 하나뿐이다.

**`generate_node`**: 기본 RAG와 **같은 `answer_chain`**에 `all_docs`를 넣어 답한다. `abstained`는 `False`.

**`abstain_node`**: 모델을 부르지 않고 `NO_ANSWER`를 **고정 문구로** 돌려주며 `abstained`를 `True`로 둔다.

## 주제 4: 분기 함수와 그래프 조립

```python
def route_after_grade(state):
    """근거 판단 뒤에 어디로 갈지 정한다. 반환값은 아래 add_conditional_edges 의 키와 같다."""
    if state["sufficient"]:
        return "generate"                          # 충분 -> 답변
    if state["search_count"] < MAX_SEARCH:
        return "rewrite"                           # 부족하고 횟수가 남음 -> 검색어를 바꿔 다시 검색
    return "abstain"                               # 부족하고 횟수를 다 씀 -> 보류


graph = StateGraph(AgentState)
graph.add_node("retrieve", retrieve_node)
graph.add_node("grade", grade_node)
graph.add_node("rewrite", rewrite_node)
graph.add_node("generate", generate_node)
graph.add_node("abstain", abstain_node)
graph.add_edge(START, "retrieve")
graph.add_edge("retrieve", "grade")
graph.add_conditional_edges("grade", route_after_grade,
                            {"generate": "generate", "rewrite": "rewrite", "abstain": "abstain"})
graph.add_edge("rewrite", "retrieve")              # 검색어를 바꾼 뒤에는 다시 검색 (반복)
graph.add_edge("generate", END)
graph.add_edge("abstain", END)
agent = graph.compile()
```

### 분기 함수 `route_after_grade`

- 상태를 보고 **다음에 갈 노드의 이름(키)을 문자열로** 돌려준다. 판단 순서가 중요하다.
  1. 근거가 충분하면 → `"generate"`
  2. 부족한데 검색 횟수가 `MAX_SEARCH`(3) **미만**이면 → `"rewrite"`
  3. 부족하고 횟수를 다 썼으면 → `"abstain"`
- **검색 횟수의 계산**: 첫 검색 뒤 `search_count`는 1이다. 부족하면 1 < 3이라 `rewrite`로 가고, 두 번째 검색 뒤 2 < 3이라 또 `rewrite`, 세 번째 검색 뒤 3은 3 미만이 아니므로 `abstain`이다. 따라서 **검색은 최대 3번(재작성은 최대 2번)**이다. `MAX_SEARCH`가 "최초 검색 포함"이라고 주석에 적힌 이유다.

### 그래프 조립

- `StateGraph(AgentState)`: 상태의 구조를 알려 주며 그래프를 시작한다.
- `add_node("이름", 함수)`: 노드를 등록한다. 이름은 엣지와 분기에서 쓴다.
- `add_edge(A, B)`: A가 끝나면 **항상** B로 간다. `START`는 시작점, `END`는 끝점이다.
- `add_conditional_edges("grade", route_after_grade, {키: 노드, ...})`: `grade` 노드가 끝난 뒤 **분기 함수를 실행하고, 돌려준 값(키)에 해당하는 노드로** 간다. 세 번째 인자의 딕셔너리가 "분기 함수의 반환값 → 실제 노드 이름"의 대응표다. 여기서는 키와 노드 이름이 같다.
- `add_edge("rewrite", "retrieve")`: 검색어를 바꾼 뒤에는 다시 검색한다. 이 엣지가 **반복(루프)**을 만든다. 반복이 무한히 돌지 않는 것은 `route_after_grade`의 횟수 검사 덕분이다.
- `generate`와 `abstain`은 모두 `END`로 이어져 끝난다.
- `graph.compile()`: 조립한 그래프를 실행 가능한 `agent`로 만든다.

> ➕ **더 알아두기**
> `add_conditional_edges`의 매핑 딕셔너리를 생략하면 분기 함수가 **노드 이름을 직접** 돌려줘야 한다. 이 코드는 반환값("generate" 등)과 노드 이름이 같아서 사실상 같은 효과지만, 매핑을 적어 두면 **그래프에서 갈 수 있는 목적지가 한눈에 보이는** 장점이 있다.

## 주제 5: 실행 — `run_agent`

```python
def run_agent(question):
    """Agent를 실행하고 기본 RAG와 같은 모양의 결과 딕셔너리를 돌려준다."""
    start = time.perf_counter()
    final = agent.invoke({
        "question": question, "query": question, "queries": [], "search_count": 0,
        "docs": [], "all_docs": [], "sufficient": False, "reason": "", "answer": "", "abstained": False,
    })
    return {
        "answer": final["answer"],
        "docs": final["all_docs"],              # 모든 검색에서 모은 근거
        "abstained": final["abstained"],
        "search_count": final["search_count"],
        "queries": final["queries"],
        "seconds": round(time.perf_counter() - start, 2),
    }


result = run_agent("지방은행이 시중은행으로 전환할 때 예비인가를 받아야 하나요?")
print(result["answer"])
print("검색 횟수:", result["search_count"], "| 검색어:", result["queries"], "| 시간:", result["seconds"], "초")
```

- `agent.invoke({...})`: 그래프를 실행한다. **초기 상태를 모든 키에 대해** 채워 넣는다. 첫 검색어(`query`)는 **질문 그대로**(`query: question`)이고, 목록은 빈 리스트, 횟수는 0, `sufficient`는 `False`다.
- 그래프가 끝났을 때의 **최종 상태**(`final`)에서 필요한 값을 꺼내 기본 RAG와 **같은 모양**의 딕셔너리로 돌려준다.
- `"docs": final["all_docs"]`: Agent의 근거는 **모든 검색에서 모은 근거 전체**다. 평가 때 "정답 쪽이 근거에 들어 있었는가"를 볼 때 이 값을 쓴다(10-18).
- 추가로 `queries`(검색어 기록)를 돌려줘 몇 번, 어떤 검색어로 찾았는지 볼 수 있다.

### 확인할 것

- 검색 횟수와 검색어 기록이 나온다. **쉬운 질문은 1회에 끝난다.**
- 문서에 없는 질문(예: `인가 수수료는 얼마인가요?`)으로 `run_agent(...)`를 해 보면 **검색어를 바꿔 3회까지 검색한 뒤 보류**하는 것을 볼 수 있다. (노트북의 확인 항목이다.)

> ➕ **더 알아두기**
> - **Agent와 단순 반복(루프)의 차이**: 이 그래프의 "에이전트다움"은 모델이 `grade`에서 스스로 "더 찾아야 하는가"를 판단하고, `rewrite`에서 실패 이유를 보고 검색어를 바꾸는 데 있다. 같은 검색어로 같은 검색을 되풀이하지 않는다.
> - **보류가 필요한 이유**: 근거가 없는데도 답을 지어내는 것(환각, hallucination)이 RAG의 가장 큰 위험이다. 최대 횟수까지 찾고도 없으면 "확인할 수 없다"고 답하는 것이 틀린 답보다 낫다. 이 보류가 잘 되는지를 10-18의 답변 불가 문항으로 평가한다.
> - 비용: 질문 하나에 모델 호출이 여러 번(판단 + 재작성 + 답변) 일어나서 기본 RAG보다 **느리고 비싸다.** 그래서 비교 결과에 평균 검색 횟수와 평균 시간이 함께 들어간다.

## 정리

- Agent는 **검색 → 근거 판단 → (충분하면 답변 / 부족하고 횟수가 남으면 검색어 재작성 후 재검색 / 횟수를 다 쓰면 보류)**의 그래프다. 하나의 에이전트로 충분하며, 기본 RAG의 `search`와 `answer_chain`을 그대로 재사용해 비교를 공정하게 한다.
- `Grade`(`sufficient`, `reason`)는 Structured Output으로 모델의 판단을 객체로 받는다. 글을 해석할 필요 없이 `grade.sufficient`를 조건으로 바로 쓴다.
- `rewrite_chain`은 이전 검색어와 근거가 부족했던 이유를 보고 다른 표현의 검색어 한 줄을 만든다.
- `AgentState`는 노드들이 공유하는 상태다. 노드는 바꿀 항목만 딕셔너리로 돌려주고, 리스트를 누적하려면 합친 리스트를 직접 만들어 돌려준다. `retrieve_node`는 청크 ID 기준으로 중복을 제거하며 누적 근거 `all_docs`를 만든다.
- `route_after_grade`가 충분 → `generate`, 부족 & 횟수 남음 → `rewrite`, 부족 & 횟수 소진 → `abstain`을 고른다. `rewrite → retrieve` 엣지가 반복을 만들고 횟수 검사가 반복을 끝낸다. 검색은 최대 3번(재작성은 최대 2번)이다.
- `run_agent`는 모든 키의 초기 상태로 그래프를 실행하고, 기본 RAG와 같은 모양의 결과(`answer`, `docs`=`all_docs`, `abstained`, `search_count`, `seconds` + `queries`)를 돌려준다.

## ✅ 확인 질문

1. 기본 RAG와 비교해 이 Agent가 추가로 하는 일 세 가지는 무엇인가요? 그리고 Agent의 검색·답변 부분을 기본 RAG와 같은 것으로 재사용해야 하는 이유는 무엇인가요?
2. `Grade`처럼 Pydantic 클래스를 만들어 `with_structured_output`에 넘기면 일반 문자열 응답과 비교해 어떤 점이 편한가요?
3. `rewrite_chain`의 입력에 "근거가 부족했던 이유(`reason`)"와 "이전 검색어"를 넣는 이유는 무엇인가요?
4. `AgentState`에서 `question`과 `query`를, 그리고 `docs`와 `all_docs`를 따로 두는 이유는 무엇인가요?
5. `retrieve_node`에서 `seen` 집합과 리스트 컴프리헨션으로 하는 중복 제거는 어떻게 동작하며, 왜 필요한가요?
6. `grade_node`가 이번 검색 결과가 아니라 `all_docs` 전체로 판단하는 이유는 무엇인가요?
7. `route_after_grade`의 세 갈래를 순서대로 설명하고, `MAX_SEARCH = 3`일 때 검색은 최대 몇 번, 검색어 재작성은 최대 몇 번 일어나는지 계산해 보세요.
8. `add_edge`와 `add_conditional_edges`는 어떻게 다르고, `add_conditional_edges`의 세 번째 인자(딕셔너리)는 무엇을 연결하나요?
9. 이 그래프에서 반복이 만들어지는 엣지는 무엇이며, 반복이 무한히 계속되지 않게 하는 장치는 무엇인가요?
10. `run_agent`가 최종 `docs`로 `all_docs`를 돌려주는 이유와, 기본 RAG와 같은 모양으로 결과를 맞추는 이유는 무엇인가요?
11. `abstain_node`가 모델을 부르지 않고 고정 문구를 돌려주는 것과, 기본 RAG에서 모델이 보류 문구를 말하게 하는 것은 어떤 차이가 있나요?
