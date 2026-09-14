# Plan-and-Execute (Planner, Executor, Replanner, Union)

## Plan-and-Execute란

지금까지 배운 ReAct는 매 순간 "다음에 뭘 할지"만 판단했다. Plan-and-Execute는 그 앞에 **전체 계획을 먼저 세우는 단계**를 추가한 패턴이다.

```
입력 → Planner(전체 계획 수립) → Executor(한 단계 실행) → Replanner(판단) → Executor → ... → 최종 응답
```

| 구분 | ReAct | Plan-and-Execute |
|---|---|---|
| 계획 범위 | 매 순간 다음 행동 결정 | 전체 계획을 먼저 수립 |
| 장점 | 단순하고 빠름 | 복잡한 목표와 제약 조건 처리에 유리 |
| 단점 | 전체 흐름을 놓칠 수 있음 | 계획과 재계획 비용 발생 |
| 적합한 작업 | 단일 조회, 간단한 질의 | 조사, 비교, 일정 설계, 보고서 작성 |

세 가지 역할이 핵심이다.
- **Planner**: 사용자 요청을 보고 실행할 단계 목록을 만듦 (LLM이 수립, 딱 한 번만 실행됨)
- **Executor**: 그 목록의 **첫 번째 단계만** 실제로 실행함
- **Replanner**: 실행 결과를 보고 "정보가 부족하니 계획을 다시 짜자" 또는 "이제 충분하니 최종 답을 만들자"를 판단

Executor ↔ Replanner가 "충분해질 때까지" 반복되는 구조이며, 이 반복은 17번(Reflection/Evaluator-Optimizer)의 "생성 → 검토 → 다시 생성" 루프와 뼈대가 같다.

## State 설계

```python
class PlanExecuteState(TypedDict):
    input: str
    plan: list[str]
    past_steps: Annotated[list[tuple[str, str]], operator.add]   # 계속 쌓여야 하니 Reducer
    response: str
```

- **past_steps**: 이미 끝낸 일은 계속 쌓여야 하는 기록이라 `operator.add`(리스트끼리 이어붙이기)로 누적시킨다. 17번의 `history: Annotated[list, add]`와 같은 이유다.
- **plan**: Replanner가 남은 계획을 통째로 교체할 수 있어야 하므로 Reducer 없이 그냥 덮어쓴다.

원칙: **"계속 쌓여야 하는 것"엔 Reducer를 붙이고, "완전히 새로 교체돼야 하는 것"엔 안 붙인다.**

## TypedDict(State) vs BaseModel(LLM 출력) — 왜 구분하나

State(`PlanExecuteState`)는 계속 `TypedDict`를 썼는데, `Plan`/`Response`/`ReplanDecision`은 `BaseModel`(Pydantic)을 썼다. 처음엔 "둘 다 BaseModel로 하면 더 안전하지 않나?" 싶었지만, 두 가지 이유로 그렇게 하지 않는다.

1. **노드는 State 전체가 아니라 "바뀐 부분만" 반환한다** (`return {"plan": ...}`). BaseModel이었다면 매번 완전한 객체를 만들어야 해서 번거롭다. TypedDict는 그냥 dict라 부분 반환이 자연스럽다.
2. **검증은 "믿을 수 없는 곳"(LLM 출력)에서만 하면 충분하다.** State에 들어가는 값은 이미 `Plan`/`Response` 같은 BaseModel을 통해 한 번 검증된 뒤이므로, State 자체를 또 검증하는 건 낭비다. 16번(Guardrails)에서 배운 "측정 가능한 건 필요한 곳에서만 검사한다"는 원칙과 같다.

정리: **State(내부 데이터 흐름) = TypedDict(가벼움), LLM 출력(신뢰 못 하는 입력) = BaseModel(검증)**.

## Planner

```python
class Plan(BaseModel):
    """앞으로 수행할 계획"""
    steps: list[str] = Field(description="순서대로 수행할 단계")

planner_prompt = ChatPromptTemplate.from_messages([...])
planner = planner_prompt | llm.with_structured_output(Plan)   # 프롬프트 -> LLM -> Plan 객체로 나오는 체인

def plan_step(state: PlanExecuteState):
    result = planner.invoke({"input": state["input"]})
    return {"plan": result.steps}
```

- `steps: list[str]`은 **모양(타입)만 강제**하고 내용은 자유다. 16번(Guardrails)의 `Literal["safe", "unsafe", ...]`는 **값까지** 강제하는 것과 대비된다 — Structured Output이 강제하는 게 "모양"과 "값" 두 층위로 나뉜다는 걸 여기서 확인했다.
- `planner`(체인, 파이프라인)과 `Plan`(그 파이프라인이 만들어낼 데이터의 틀)은 서로 다른 물건이다. `planner.invoke(...)`를 실행하면 파이프라인 자체가 아니라, 그 파이프라인을 통과한 **최종 결과물**(Plan 인스턴스)이 나온다.

## Executor

```python
search_llm = llm.bind_tools([{"google_search": {}}])   # Gemini 내장 Google Search 기능을 켬 (직접 만든 Tool 아님)

def execute_step(state: PlanExecuteState):
    current_step = state["plan"][0]        # plan의 첫 단계만 꺼냄
    ...
    result = search_llm.invoke(task)
    return {"past_steps": [(current_step, result.text)]}   # Reducer 덕에 기존 리스트 뒤에 누적됨
```

`{"google_search": {}}`는 12번에서 직접 만든 함수를 `@tool`로 감싸던 것과 달리, **Gemini가 이미 내장하고 있는 검색 기능을 그대로 켜는 것**이다.

## Replanner — Union과 isinstance

Replanner는 매번 "새 계획(Plan)" 또는 "최종 응답(Response)" 둘 중 하나만 반환한다. 이를 표현하려고 `Union` 타입이 등장한다.

```python
class Response(BaseModel):
    """사용자에게 전달할 최종 응답"""
    response: str = Field(description="검색 결과를 반영한 최종 여행 계획")

class ReplanDecision(BaseModel):
    """Replanner가 선택한 다음 결정"""
    decision: Union[Plan, Response]   # decision은 Plan이거나 Response, 둘 중 하나
```

`Union[A, B]`는 "이 값의 타입이 A 또는 B 둘 중 하나다"라는 선언일 뿐이고, 실제로 어느 쪽이 왔는지는 **실행 중에 직접 확인**해야 한다. 그 확인 도구가 파이썬 내장 함수 `isinstance`다.

```python
def replan_step(state: PlanExecuteState):
    result = replanner.invoke({...})

    if isinstance(result.decision, Response):   # "지금 온 게 Response 타입이야?"를 실행 중에 확인
        return {"response": result.decision.response}

    return {"plan": result.decision.steps}
```

`isinstance(값, 타입)`은 LangGraph 전용 문법이 아니라 파이썬 어디서나 쓰는 범용 내장 함수다. `Union`(선언)과 `isinstance`(실행 중 구분)는 사실상 세트로 쓰인다 — Union 없이 isinstance만 있으면 애초에 여러 타입이 올 상황이 아니고, isinstance 없이 Union만 있으면 코드가 어느 필드(`.steps`/`.response`)를 꺼내야 할지 알 수 없다.

```python
def should_continue(state: PlanExecuteState):
    return "end" if state.get("response") else "continue"
```

should_continue는 새로 판단하는 게 아니라, **직전에 replan_step이 response를 채워놨는지만 다시 확인**하는 것이다. 진짜 판단(계속할지 말지)은 이미 replan_step 단계에서 LLM이 내렸다.

## 그래프 조립 — LangChain과의 구조적 차이

```python
graph_builder = StateGraph(PlanExecuteState)
graph_builder.add_node("planner", plan_step)
graph_builder.add_node("executor", execute_step)
graph_builder.add_node("replanner", replan_step)

graph_builder.add_edge(START, "planner")
graph_builder.add_edge("planner", "executor")
graph_builder.add_edge("executor", "replanner")
graph_builder.add_conditional_edges(
    "replanner", should_continue,
    {"continue": "executor", "end": END},
)
```

- LangChain의 `|`(LCEL)는 A → B → C처럼 **한 방향으로만 흐르는 연결 리스트**에 가깝다. 되돌아가거나 여러 갈래로 나뉠 수 없다.
- LangGraph의 `add_edge`/`add_conditional_edges`는 **일반 그래프 자료구조**다 — 한 노드가 여러 갈래로 연결될 수 있고, 무엇보다 **다시 이전 노드로 되돌아가는 화살표**도 그릴 수 있다. `"continue": "executor"`가 바로 그 되돌아가는 화살표이며, 이 덕분에 executor ↔ replanner 반복(사이클)이 가능해진다.

## recursion_limit — 무한 반복 방지

```python
config = {"recursion_limit": 20}
```

노드 실행의 **총 횟수 상한**이다. 같은 노드를 반복 실행하는 것도 매번 횟수에 포함되며, 초과하면 `GraphRecursionError`가 발생한다. 17번의 `MAX_ITERATIONS`(State 안에 직접 만든 카운터)와 목적은 같지만, 이번엔 LangGraph가 그래프 실행 전체에 제공하는 별도 안전장치(config)를 쓴다는 차이가 있다.

이 코드엔 `try`/`except`가 없어서, 실제로 20번을 넘기면 에러가 그대로 튀어나오며 멈춘다. 실무라면 `GraphRecursionError`를 잡아서 "지금까지 조사한 내용을 참고해달라"는 식으로 부드럽게 처리하는 게 맞다.

**LangGraph가 왜 노드별 개별 제한이 아니라 총합 하나만 제공하는가**: LangGraph는 범용 라이브러리라 어떤 노드가 몇 번 반복될지, 어떤 노드 이름을 쓸지 미리 알 수 없다. "전체가 무한히 안 돌게 막는다"는 모두에게 공통으로 필요한 위험이라 라이브러리가 총합 하나로 제공하고, "이 노드는 몇 번까지만"처럼 서비스마다 다른 세부 규칙은 State에 카운터를 직접 만들어서(17번 방식) 구현하도록 열어둔다.

## stream_mode=["updates", "values"]

```python
for mode, event in graph.stream(inputs, config=config, stream_mode=["updates", "values"]):
```

- stream_mode를 안 쓰거나 하나만 쓰면(`stream_mode="values"`) → 매번 event 하나만 나온다.
- stream_mode에 **여러 개를 리스트로** 넣으면 → 매번 `(mode, event)` **튜플**로 나온다. 여러 종류가 섞여 나오니, "이게 어떤 종류인지" 이름표(mode)가 필요해서다.

두 모드의 내용물 차이:
- **"updates"**: 방금 그 노드가 "바꾼 부분"만 → `{"planner": {"plan": [...]}}`
- **"values"**: 그 시점의 State **전체 스냅샷** → `{"input": ..., "plan": [...], "past_steps": [...], "response": None}`

```python
for node_name, value in event.items():
    if node_name == "planner":
        for step in value["plan"]:
            print(f"- {step}")
    elif node_name == "executor":
        step, result = value["past_steps"][-1]   # Reducer로 누적되니 방금 추가된 건 항상 리스트 맨 끝
        ...
    elif node_name == "replanner" and value.get("plan"):
        ...   # Union 중 Plan을 반환한 경우
    elif node_name == "replanner" and value.get("response"):
        ...   # Union 중 Response를 반환한 경우
```

replanner 쪽에서 `value.get("plan")`/`value.get("response")`로 다시 분기하는 이유: replan_step 안에서는 `isinstance`로 Plan/Response를 구분했지만, 출력 코드 쪽에서는 그 결과가 이미 dict(`{"plan": ...}` 또는 `{"response": ...}`)로 바뀐 뒤라 어떤 키가 들어있는지로 다시 구분해야 한다 — 같은 목적("Plan이냐 Response냐")을 서로 다른 지점에서 다른 도구(isinstance vs dict 키 확인)로 처리하는 셈이다.

## 실습: 2026 생성형 AI 트렌드 조사 + Replanner 없는 버전

- **주 실습**: 여행 계획 예제와 동일한 Planner/Executor/Replanner 구조를 그대로 재사용하고, 도메인(AI 트렌드)에 맞게 프롬프트만 새로 작성했다. `MODEL_NAME`은 CLAUDE.md 규칙에 따라 `gemini-3.6-flash`(하루 20회 한도) 대신 `gemini-3.5-flash-lite`(하루 500회)로 교체했다.
- **추가 실습(Replanner 없는 버전)**: Replanner가 필수 구성요소가 아니라는 걸 확인했다. Executor가 `{"plan": state["plan"][1:], "past_steps": [...]}`을 반환해 직접 계획을 줄여나가고, `has_remaining_plan`(plan이 남았는가)이라는 단순 분기 함수로 executor를 반복시키거나 finalizer로 보낸다. finalizer는 Structured Output이 필요 없다 — 텍스트 응답 하나만 만들면 되기 때문이다.

## 오늘 배운 것 요약

| 개념 | 역할 |
|---|---|
| Planner/Executor/Replanner | 계획 수립(1회) → 한 단계 실행(반복) → 계속/종료 판단(반복) |
| Union[Plan, Response] | 둘 중 하나가 올 수 있다는 선언 |
| isinstance | Union 중 실제로 어느 타입이 왔는지 실행 중에 확인 |
| TypedDict vs BaseModel | State(내부 흐름)=TypedDict, LLM 출력(검증 필요)=BaseModel |
| recursion_limit | 그래프 전체 노드 실행 총 횟수 상한 (라이브러리 제공, 총합만) |
| stream_mode 리스트 | 여러 모드를 동시에 요청하면 (mode, event) 튜플로 나옴 |

## ✅ 확인 질문

1. ReAct와 Plan-and-Execute의 핵심 차이는 무엇이며, 어떤 작업에 각각 더 적합한가?
2. State의 `plan` 필드와 `past_steps` 필드는 왜 Reducer 적용 여부가 다른가?
3. State는 왜 BaseModel이 아니라 TypedDict를 쓰는가?
4. `Union[Plan, Response]`와 `isinstance`는 왜 항상 세트로 쓰이는가?
5. `recursion_limit`이 노드별이 아니라 그래프 전체에 적용되는 이유는 무엇인가?
6. `stream_mode="values"`(단일)와 `stream_mode=["updates", "values"]`(리스트)의 반환 형태 차이는 무엇인가?
7. Replanner 없는 버전에서 Executor는 어떻게 "다음에 뭘 할지"를 스스로 갱신하는가?
