---
tags: [langgraph, parallelization, fan-out-fan-in, reducer, voting]
til: v2 2026-09-14
---

# (1) LangGraph 병렬 처리(Parallelization) & Voting 패턴
> 작성일: 2026-09-11

## 🔗 관련 글

- [(1) ReAct Agent (Tool, ToolNode, tools_condition, MessagesState)](2026-09-09_(1)_ReAct_Agent(Tool,_ToolNode,_tools_condition,_MessagesState).md) — `add_conditional_edges`를 처음 다룬 글, 이 글에서 `add_edge`의 리스트 인자와 비교됨
- [(2) LangGraph Orchestrator-Worker 패턴 & Send API로 동적 병렬 처리](2026-09-11%20(2)%20orchestrator%20worker.md) — 같은 날 이어서 배운, 병렬 개수가 동적으로 정해지는 심화 패턴

## 1. 병렬 처리가 왜 필요한가

하나의 문서를 세 가지 관점(법률, 재무, 기술)으로 분석한다고 하면, 각 분석은 서로의 결과를 필요로 하지 않는다. 즉 서로 **독립적**이다.

**순차 처리** (하나씩 차례로):
```
법률 분석 (3초) → 재무 분석 (3초) → 기술 분석 (3초) = 총 9초
```

**병렬 처리** (동시에):
```
법률 분석 (3초) ─┐
재무 분석 (3초) ─┼→ 종합 = 총 3초 + α
기술 분석 (3초) ─┘
```

판단 기준은 딱 하나다: **"이 작업이 다른 작업의 결과 없이 시작할 수 있는가?"** 그렇다면 병렬화할 수 있다. B 작업이 A 작업의 결과를 입력으로 써야 한다면 `A → B` 순서를 유지해야 한다.

주의할 점: 병렬로 돌려도 **전체 소요 시간이 가장 오래 걸리는 작업(병목)보다 짧아지지는 않는다.** 3초짜리 3개를 병렬로 돌리면 3초+α이지, 1초가 되는 게 아니다.

## 2. 기본 구조 — Fan-out / Fan-in

하나의 시작점 뒤에 여러 노드를 나란히 연결하면, LangGraph가 자동으로 **동시에 실행**한다. 이렇게 갈래가 나뉘는 것을 **Fan-out**(분기), 다시 하나로 모이는 것을 **Fan-in**(합류)이라고 부른다.

```
START ─┬→ legal_analysis     ─┐
       ├→ financial_analysis ─┼→ summarize → END
       └→ technical_analysis ─┘
```

실제 코드는 이렇게 생겼다.

```python
class AnalysisState(TypedDict):
    document: str
    analyses: Annotated[list, operator.add]  # 각 분석 결과가 누적됨
    summary: str


def legal_analysis(state: AnalysisState):
    result = llm.invoke(
        f"다음 문서를 법률 관점에서 핵심 리스크 2가지를 분석해줘. 간결하게.\n\n{state['document']}"
    )
    return {"analyses": [f"[법률] {result.text}"]}

# financial_analysis, technical_analysis 도 프롬프트만 다르고 구조는 동일

def summarize(state: AnalysisState):
    all_analyses = "\n\n".join(state["analyses"])
    result = llm.invoke(
        f"다음 3가지 관점의 분석을 종합해서 3줄로 요약해줘:\n\n{all_analyses}"
    )
    return {"summary": result.text}
```

그래프 조립에서 Fan-out과 Fan-in이 각각 어떻게 코드로 표현되는지가 이 노트북의 핵심이다.

```python
# Fan-out: START에서 3개 노드로 각각 add_edge를 세 번 그음
graph_builder.add_edge(START, "legal_analysis")
graph_builder.add_edge(START, "financial_analysis")
graph_builder.add_edge(START, "technical_analysis")

# Fan-in: add_edge의 첫 번째 자리에 "리스트"를 넣으면,
# 그 리스트 안의 노드들이 전부 끝나야만 다음 노드로 넘어간다
graph_builder.add_edge(
    ["legal_analysis", "financial_analysis", "technical_analysis"],
    "summarize",
)
```

`add_edge`의 첫 번째 자리에 노드 이름을 리스트로 넣을 수 있다는 게 이번에 새로 나온 문법이다. 평소에는 `add_edge("A", "B")`처럼 문자열 하나씩만 썼는데, 여기서는 "이 셋이 전부 끝나야 다음으로 간다"는 **합류 조건**을 표현하려고 리스트를 쓴다.

> ➕ **19번(Orchestrator-Worker)과 비교**
> 19번 노트북에서는 `add_edge("generate_section", "assemble_course_material")`처럼 노드 이름이 하나뿐이었다. Worker가 몇 벌이 동시에 도는지는 `Send` 객체 개수로 정해지지만, 그 노드들은 전부 **이름이 하나(`generate_section`)뿐이라서** 리스트로 안 묶어도 됐다. 반면 18번은 **이름이 서로 다른 노드 셋(`legal_analysis`, `financial_analysis`, `technical_analysis`)**을 합류시켜야 해서 리스트로 명시한 것이다. 즉 "같은 노드를 몇 벌 복제"(19번, Send)와 "서로 다른 이름의 노드 여러 개를 합류"(18번, `add_edge`의 리스트)는 다른 상황이다.

## 3. State 충돌과 Reducer

세 분석 노드가 전부 같은 State 필드(`analyses`)에 결과를 쓰려고 하면 기본 동작은 **덮어쓰기**다. 즉 마지막에 끝난 노드의 결과만 남고 나머지는 사라진다.

```python
analyses: Annotated[list, operator.add]
```

이렇게 `operator.add` reducer를 지정하면, 여러 노드가 각각 `{"analyses": [값]}`을 반환해도 덮어쓰지 않고 **리스트끼리 이어붙인다** (`[1,2] + [3] = [1,2,3]`과 같은 원리). 그래서 3개 노드의 결과가 전부 살아남아 하나의 리스트에 쌓인다.

> ♻️ Reducer는 State의 **필드 선언부**(`Annotated[...]`)에만 붙이는 것이지, 각 노드의 `return` 문 안에서 따로 "합치기" 코드를 써야 하는 게 아니다. 노드는 그냥 `{"analyses": [자기 결과]}`만 반환하면, 합치는 규칙은 State 정의에 미리 박아둔 reducer가 알아서 처리한다.

## 4. 결과 순서가 필요할 때

병렬로 실행된 노드는 **어느 게 먼저 끝날지 순서가 보장되지 않는다.** 네트워크 상태나 LLM 응답 속도에 따라 매번 달라진다(비결정적, non-deterministic).

순서가 중요하면, 결과 안에 **순서를 표시하는 식별자**를 같이 담아서 반환하고, 나중에 그 식별자로 정렬하면 된다.

```python
def ordered_legal(state: OrderedAnalysisState):
    result = llm.invoke(...)
    return {"analyses": [(0, f"[법률] {result.text}")]}   # (순서, 내용) 튜플

def ordered_summarize(state: OrderedAnalysisState):
    sorted_analyses = sorted(state["analyses"], key=lambda x: x[0])
    ...
```

여기서는 딕셔너리가 아니라 **튜플**(`(0, "...")`)로 순서를 표시했다. 19번(Orchestrator-Worker) 노트북에서는 딕셔너리 안에 `task_id` 키로 순서를 담았던 것과 비교하면, 이번엔 그냥 튜플의 첫 번째 자리를 순서 값으로 쓴 것뿐, 원리는 똑같다.

## 5. Voting 패턴

하나의 질문에 대해 **서로 다른 스타일의 답변 후보를 병렬로 여러 개 생성**하고, 그중 가장 좋은 걸 심판(judge) LLM이 골라내는 패턴이다.

```
START ─┬→ generate_concise ─┐
       ├→ generate_example ─┼→ vote → END
       └→ generate_analogy ─┘
```

```python
def generate_candidate(question: str, candidate_id: int, approach: str):
    result = creative_llm.invoke(
        f"다음 질문에 {approach}으로 유용하게 답변해줘:\n\n{question}"
    )
    return {"candidates": [{"id": candidate_id, "answer": result.text}]}

def generate_concise(state: VotingState):
    return generate_candidate(state["question"], 0, "핵심 중심")

def generate_example(state: VotingState):
    return generate_candidate(state["question"], 1, "구체적인 예시 중심")

def generate_analogy(state: VotingState):
    return generate_candidate(state["question"], 2, "비유와 쉬운 설명 중심")
```

세 노드가 전부 **같은 질문**을 받지만, 프롬프트에 넣는 "접근 방식"(`approach`)만 다르게 줘서 서로 다른 스타일의 답을 만들어낸다. (다른 방법으로, 서로 다른 모델을 동시에 호출해서 후보를 만들 수도 있다.)

그리고 심판 역할은 구조화된 출력으로 처리한다.

```python
class VoteResult(BaseModel):
    """투표 결과"""
    best_id: int = Field(description="가장 좋은 답변의 ID (0부터 시작)")
    reason: str = Field(description="선택 이유")

def vote(state: VotingState):
    candidates = state["candidates"]
    judge = judge_llm.with_structured_output(VoteResult)
    result = judge.invoke(
        f"다음 질문에 대한 여러 답변 후보가 있다. 가장 정확하고 유용한 답변을 골라줘.\n\n..."
    )
    best = next(c for c in candidates if c["id"] == result.best_id)
    return {"best_answer": best["answer"]}
```

`with_structured_output(VoteResult)`을 쓴 이유는, 심판 LLM이 "3번이 좋아요!"처럼 자유 텍스트로 답하면 프로그램이 파싱하기 번거롭기 때문이다. `best_id`, `reason`이라는 정해진 필드로 답을 받으면, `result.best_id`로 바로 어떤 후보를 골랐는지 꺼내 쓸 수 있다.

이 패턴은 결과의 품질 편차가 큰 작업(마케팅 카피, 코드 생성, 번역 등)에서 여러 후보를 만들어 비교할 때 유용하다. 하나만 생성해서 바로 쓰는 것보다, 여러 접근을 병렬로 뻗어보고 그중 최선을 고르는 방식이라 **Self-Consistency**(같은 문제를 여러 번 풀어보고 가장 일관된 답을 채택하는 방식)나 **LLM-as-a-Judge**(LLM이 다른 LLM의 결과를 평가하게 하는 방식) 같은 개념과도 연결된다.

## 6. 언제 병렬 처리를 쓰는가

병렬 처리가 적합한 예:

- 하나의 문서를 여러 관점(법률/재무/기술)에서 각각 분석한 뒤 종합할 때
- 하나의 문장을 여러 언어로 각각 번역할 때
- 하나의 질문에 대한 여러 후보 답변을 생성한 뒤 최선을 고를 때

반면 작업 종류와 개수가 미리 정해져 있고, 그냥 "같은 모델에 여러 입력을 보내서 결과만 한꺼번에 받으면 되는" 상황이라면, 굳이 노드를 여러 개 만들지 않고 `batch()`(하나의 LLM 호출로 여러 입력을 한 번에 처리하는 방법)를 쓰는 게 더 간단하다.

주의할 점 세 가지:

- 병렬 요청이 많으면 API의 **Rate Limit**(분당/동시 요청 제한)에 걸릴 수 있다. 이걸 실행 시점에 동적으로 개수를 조절하는 방법(`max_concurrency` 등)은 다음 노트북인 Orchestrator-Worker(19번)에서 다룬다.
- 완료 순서는 보장되지 않으므로, 순서가 중요하면 식별자를 넣고 정렬해야 한다.
- 병렬화해도 가장 오래 걸리는 작업의 시간보다 전체 시간이 짧아지지는 않는다.

## 7. 18번과 19번의 결정적 차이

| | 18번 (이 노트북) | 19번 (Orchestrator-Worker) |
|---|---|---|
| 실행할 노드의 종류·개수 | **그래프를 짜는 시점에 이미 고정** (`legal_analysis`, `financial_analysis`, `technical_analysis`처럼 이름이 다 정해져 있음) | **실행 시점(런타임)에 동적으로 결정** (목차가 3개면 Worker 3개, 5개면 5개) |
| 분기 방법 | `add_edge(START, "노드이름")`을 필요한 만큼 반복 | 라우팅 함수가 `Send` 객체를 필요한 개수만큼 만들어서 반환 |
| 합류 방법 | `add_edge([노드이름들], "다음노드")`로 여러 이름을 리스트로 묶음 | 노드 이름이 하나(`generate_section`)뿐이라 평범한 `add_edge`로 충분 |

한 문장으로 정리하면: **18번은 "이미 몇 개인지 아는 작업들을 병렬로 돌리는 법", 19번은 "몇 개가 될지 그때 가서 정해지는 작업들을 병렬로 돌리는 법"**이다.

---

## ✅ 확인 질문

1. 두 작업을 병렬로 처리할 수 있는지 없는지를 가르는 판단 기준은 무엇인가?
2. 병렬 처리를 적용해도 전체 실행 시간이 절대 줄어들지 않는 경우는 언제인가?
3. `add_edge(START, "노드이름")`을 여러 번 쓰면 LangGraph가 왜 자동으로 병렬 실행하는가?
4. `add_edge`의 첫 번째 자리에 노드 이름 리스트를 넣는 경우는 언제이고, 왜 필요한가?
5. `Annotated[list, operator.add]`를 State 필드에 안 붙이면 여러 노드의 결과가 어떻게 되는가?
6. Reducer는 State 정의와 노드의 `return` 문 중 어디에 선언해야 하는가?
7. 병렬 실행 결과의 완료 순서가 보장되지 않는 이유는 무엇이고, 순서를 되살리려면 어떻게 하는가?
8. Voting 패턴에서 세 후보 답변이 서로 다르게 나오는 이유는 무엇인가?
9. Voting 패턴에서 `with_structured_output`을 심판 LLM에 쓰는 이유는?
10. 작업 종류와 개수가 이미 고정돼 있을 때, 여러 노드를 만드는 대신 `batch()`를 쓰는 게 더 나은 경우는?
11. 병렬 요청이 많을 때 조심해야 하는 API 쪽 문제는 무엇인가?
12. 18번과 19번 노트북에서 "몇 개의 노드가 병렬로 실행되는지"를 아는 시점이 어떻게 다른가?
