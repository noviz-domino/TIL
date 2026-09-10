---
tags: [langgraph, state, node, edge, reducer]
til: v2 2026-09-10
---

# LangGraph 기초(State, Node, Edge와 조건부 분기, 반복, Reducer)
> 작성일: 2026-09-08

## Chain의 한계와 LangGraph가 필요한 이유

LCEL Chain(`prompt | llm | parser`처럼 파이프로 이어붙이는 방식)은 입력을 정해진 순서로 처리하는 직선형 파이프라인에 잘 맞는다.

```
입력 → 프롬프트 → 모델 → 출력 파서 → 결과
```

하지만 실제 서비스에는 실행 도중 다음 행동을 결정해야 하는 상황이 자주 등장한다.

- 고객 문의를 처리할 때 환불 요청은 환불 담당 흐름으로, 제품 문의는 상담 흐름으로 보내야 한다.
- 문서를 검색했는데 답변에 필요한 내용이 부족하면 질문을 고쳐 다시 검색해야 한다. 검색이 계속 실패하면 정해진 횟수에서 중단해야 한다.
- 결제나 메일 발송처럼 영향이 큰 작업은 실행 직전에 멈추고 사용자의 승인을 기다려야 한다.
- 실행 중 오류가 발생하면 처음부터 다시 시작하지 않고 저장된 상태를 바탕으로 이어서 처리해야 한다.
- Agent가 검색, 계산, 데이터 조회 중 어떤 Tool이 필요한지 판단하고 결과가 충분할 때까지 사용해야 한다.

이 요구사항들도 일반 Python 코드(if/for)와 LangChain을 조합하면 구현할 수는 있다. 문제는 분기, 반복, 상태 저장, 중단과 재개가 늘어날수록 제어 로직이 Chain 밖의 조건문과 반복문으로 흩어진다는 점이다. 실행 흐름을 이해하고 추적하거나, 중간 상태에서 다시 시작하는 일도 점점 어려워진다.

예를 들어 "검색 결과가 부족하면 질문을 바꾸어 다시 검색한다"는 요구사항을 손으로 짜면 이런 형태가 된다.

```python
question = "AI 반도체 시장의 전망은?"
max_retries = 3

for attempt in range(max_retries):
    docs = retriever.invoke(question)
    context = format_docs(docs)

    check = llm.invoke([
        HumanMessage(
            content=f"질문: {question}\n\n검색 결과:\n{context}\n\n"
            "이 검색 결과가 질문에 답변하기에 충분한가? 'yes' 또는 'no'로만 답해."
        )
    ])
    check_text = check.text

    if "yes" in check_text.lower():
        break

    rewrite = llm.invoke([
        HumanMessage(
            content=f"'{question}'이라는 질문으로 문서를 검색했지만 충분한 결과가 없었다. "
            "같은 의도이지만 다른 표현으로 질문을 다시 작성해줘. 질문만 출력해."
        )
    ])
    question = rewrite.text
```

모델 호출은 Chain 안에 있지만, 반복과 종료 조건은 Python `for`문이 제어한다. 요구사항이 늘어날수록 제어 흐름과 상태 관리가 Chain 밖으로 흩어진다.

> LangGraph는 LangChain으로 할 수 없던 일을 가능하게 만드는 도구라기보다, 복잡한 제어 흐름과 상태 변화를 명시적인 그래프로 관리하게 해주는 프레임워크다.

## Workflow vs Agent 스펙트럼

AI 애플리케이션의 제어 방식은 스펙트럼 위에 있다.

```
개발자가 흐름 결정                                    LLM이 흐름 결정
├──────────────────────────────────────────────────────────┤
Chain       Workflow        Agent           Autonomous Agent
(직선)       (분기+루프)     (LLM이 판단)            (완전 자율)
```

| 방식 | 제어 주체 | 예시 |
|------|----------|------|
| Chain | 개발자가 모든 흐름을 고정 | 번역 → 요약 → 출력 |
| Workflow | 개발자가 분기/루프 설계 | 검색 결과 부족하면 재검색 |
| Agent | LLM이 다음 행동을 결정 | 어떤 Tool을 몇 번 쓸지 LLM이 판단 |

Chain은 스펙트럼의 가장 왼쪽이다. LangGraph는 이 스펙트럼 전체를 구현할 수 있는 프레임워크다.

## LangGraph 핵심 개념 3가지

LangGraph는 3가지 개념으로 구성된다.

| 개념 | 역할 | 비유 |
|------|------|------|
| State | 전체 흐름에서 공유하는 데이터 | 칠판 — 모든 노드가 읽고 쓸 수 있음 |
| Node | 각 처리 단계 (Python 함수) | 작업자 — State를 받아서 처리하고 결과를 돌려줌 |
| Edge | 노드 간의 연결 | 화살표 — 다음에 어떤 노드로 갈지 결정 |

### State는 대화 메모리와 다르다

Gemini API나 LangChain에서 말하는 "메모리"는 보통 대화 기록(주고받은 메시지 목록) 하나만 가리킨다. State는 그보다 범위가 넓다. 대화 기록을 담을 수도 있지만, 그 외에 이 워크플로우가 필요로 하는 아무 데이터나 담을 수 있다. 재시도 횟수, 감지한 언어, 검색된 문서 같은 것도 State의 필드가 될 수 있다.

- 메모리(대화 기록) → State 안에 들어갈 수 있는 필드 하나
- State → 그 메모리를 포함해서, 이 그래프가 돌아가는 데 필요한 모든 데이터를 담는 그릇

> ➕ **더 알아두기**
> State에 `{"messages": [...], "retry_count": 2, "language": "ko"}`처럼 대화 기록과 그 외 작업 상태를 같이 넣는 경우가 실무에서 많다. 메모리 기능이 사라지는 게 아니라, State의 한 종류로 흡수되는 것이다.

### State는 TypedDict로 정의한다

`TypedDict`는 Python 표준 라이브러리(`typing`)에 포함된 타입으로, 딕셔너리의 키와 값 타입을 명시할 수 있다.

```python
# 일반 dict — 어떤 키가 있는지, 값이 무슨 타입인지 알 수 없음
state = {"name": "홍길동", "age": 30}

# TypedDict — 키 이름과 타입이 명확
class State(TypedDict):
    name: str
    age: int
```

`dict`라고만 쓰면 "딕셔너리라는 종류"만 표시하고, `TypedDict`(State)는 그 종류 중에서도 구체적으로 어떤 키가 있고 어떤 타입인지까지 표시한다.

**런타임에는 일반 `dict`와 동일하게 동작한다.** `TypedDict`가 주는 건 IDE 자동완성과 정적 타입 검사(mypy, pyright 등)뿐이고, Python이 실행 중에 타입을 강제로 검사해주지는 않는다. 그래서 `graph.invoke({"question": "..."})`처럼 State 필드 중 일부(`answer`)가 빠진 dict를 넣어도 에러가 나지 않는다. 아직 채워지지 않은 필드는 Node가 실행되며 채워진다.

### Node는 State를 받아 일부만 반환하는 함수다

노드는 Python 함수다. State를 받아서 업데이트할 내용을 반환한다.

```python
llm = ChatGoogleGenerativeAI(model=MODEL_NAME)

def chatbot(state: State):
    response = llm.invoke(state["question"])
    return {"answer": response.text}
```

노드 함수의 패턴은 항상 동일하다.

1. `state`를 인자로 받는다
2. state에서 필요한 데이터를 꺼내 처리한다
3. 업데이트할 필드를 dict로 반환한다

**반환된 dict는 State에 병합된다.** `"question"`을 안 돌려줘도 State에서 `"question"`이 사라지지 않고, `"answer"`만 새로 추가/갱신된다.

> ♻️ 처음엔 "`return`으로 돌려준 dict가 State에 병합된다"는 걸 `return` 자체가 하는 일로 이해했으나, 그렇지 않다. `return`은 그냥 평범한 Python 문법으로, 함수가 호출한 쪽에 값을 하나 돌려주는 것뿐이다. 병합은 **LangGraph 엔진**이 그 반환값을 받아서 자기가 들고 있는 State에 갖다 붙이는 별개의 동작이다. `chatbot` 함수 자체는 병합에 대해 전혀 모른다.

이 "부분만 반환하면 병합된다"는 규칙이 LangGraph Node의 가장 중요한 계약(contract)이다. Node가 State 전체를 다 알 필요도, 다 돌려줄 필요도 없다는 뜻이다.

### Edge는 그래프를 완성하는 연결이다

Edge로 경로를 미리 그려놔야, LangGraph가 그래프를 실행하면 어디서 시작해서 어디로 가는지 알 수 있다. `add_node`만 해두고 `add_edge`를 안 하면, 노드는 등록됐어도 언제 실행되는지 연결이 안 돼서 그래프가 완성되지 않는다.

```python
# 그래프 생성
graph_builder = StateGraph(State)

# 노드 추가 (이름, 함수)
graph_builder.add_node("chatbot", chatbot)

# 엣지 추가
graph_builder.add_edge(START, "chatbot")  # 시작 → chatbot
graph_builder.add_edge("chatbot", END)    # chatbot → 종료

# 컴파일
graph = graph_builder.compile()
```

순서를 정리하면 이렇다.

1. `StateGraph(State)` — State 스키마로 그래프 생성
2. `add_node(이름, 함수)` — 노드 등록. 첫 번째 인자(문자열)는 그래프 안에서 이 노드를 부를 이름표, 두 번째 인자(함수)는 실제로 실행될 코드다. **이름과 함수는 다를 수 있다.** `add_node("검색", search_documents)`처럼 함수명과 다른 노드 이름을 붙일 수 있다.
3. `add_edge(출발, 도착)` — 노드 간 연결
4. `compile()` — 실행 가능한 그래프로 변환

`START`, `END`는 LangGraph가 미리 정의해둔 특수 노드로, 모든 그래프에 항상 존재하는 시작 지점과 종료 지점을 나타내는 상수다.

### compile()은 실행이 아니라 "실행 가능한 객체 생성"이다

`compile()` 전후로 객체의 종류(class)가 다르다는 걸 실제로 확인했다.

```python
print("compile 전 (builder):", type(builder))
# <class 'langgraph.graph.state.StateGraph'>
#   invoke 있음? False   stream 있음? False   add_node 있음? True

graph = builder.compile()

print("compile 후 (graph):", type(graph))
# <class 'langgraph.graph.state.CompiledStateGraph'>
#   invoke 있음? True    stream 있음? True    add_node 있음? False
```

`compile()` 전 객체(`StateGraph`)는 `add_node`는 있지만 `invoke`, `stream`이 아예 없다 — 애초에 실행할 방법이 없는 객체다. `compile()`을 부르면 `CompiledStateGraph`라는 완전히 다른 클래스의 새 객체가 만들어지고, 이 객체는 반대로 `invoke`, `stream`은 있는데 `add_node`가 없다 — 더 이상 구조를 수정할 방법이 없는, 실행 전용 객체다.

> ♻️ 처음엔 `compile()`이 "실행 가능한 형태로 조립"한다는 표현 때문에 `compile()` 자체가 실행하는 것으로 오해했으나, 그렇지 않다. `compile()`은 실행할 수 있는 객체를 만들기만 하고(설계도 → 완성된 자동차), 실제로 노드를 하나씩 돌리는(운전하는) 건 그 뒤에 부르는 `invoke()`나 `stream()`이다.

- `graph_builder`(조립 중) — `add_node`, `add_edge`로 계속 수정 가능
- `graph`(compile 결과) — 실행만 할 수 있는 상태. 이 시점 이후로는 노드/엣지를 더 추가할 방법이 없다(구조가 고정됨)

이 "구조 고정"과 "실행마다 다른 경로로 가는 것"은 다른 얘기다. 그래프 구조 자체는 `compile()` 때 고정되지만, 그 구조 안에 미리 만들어둔 조건부 분기가 있으면 실행마다 조건에 따라 다른 노드로 갈 수 있다. 다만 그 갈림길 자체는 `compile()` 전에 미리 다 정의돼 있어야 한다.

### 실행: invoke()

```python
result = graph.invoke({"question": "LangGraph가 뭐야?"})
print(result["answer"])
```

`graph.invoke(초기 State)`는 그래프를 실행하고 모든 노드의 처리가 끝난 뒤 최종 State를 반환한다. `result`는 특수한 결과 객체가 아니라 State 그대로인 평범한 dict다.

**`invoke()`는 매번 새로 시작한다.** `graph.invoke()`를 두 번 부르면 두 결과는 완전히 독립적이다. 두 번째 호출할 때 첫 번째 질문·답변이 State 안에 남아있지 않다 — 이 기본 그래프는 아무것도 기억 안 하는 상태다.

## 그래프: 조건부 분기

시나리오: 사용자 입력의 언어를 감지해서 한국어면 한국어로, 영어면 영어로 답변하는 그래프.

```
START → detect_language → 한국어? → answer_ko → END
                          영어?  → answer_en → END
```

```python
class RouterState(TypedDict):
    question: str
    language: str
    answer: str
```

이 필드가 왜 필요한지: 이 그래프가 진행되면서 나중 단계가 참고해야 할 중간 결과가 있는가가 필드 추가 기준이다. `language`가 없었다면 라우팅 함수가 판단할 근거 자체가 없다.

```python
def detect_language(state: RouterState):
    question = state["question"]
    result = llm.invoke(f"다음 문장의 언어가 한국어면 'ko', 영어면 'en'으로만 답해: {question}")
    return {"language": result.text.strip().lower()}

def answer_ko(state: RouterState):
    result = llm.invoke(f"한국어로 답변해줘: {state['question']}")
    return {"answer": result.text}

def answer_en(state: RouterState):
    result = llm.invoke(f"Answer in English: {state['question']}")
    return {"answer": result.text}
```

`.strip()`은 문자열 앞뒤의 공백(줄바꿈 포함)을 제거하고, `.lower()`는 알파벳을 전부 소문자로 바꾼다. LLM이 `"Ko"`나 `" ko\n"`처럼 살짝 다르게 답할 수 있으니, 나중에 `"ko"`와 정확히 비교하려고 형태를 통일시키는 것이다.

### add_edge vs add_conditional_edges

`add_edge`는 다음 노드가 고정되어 있지만, `add_conditional_edges`는 함수의 반환값에 따라 다음 노드가 결정된다.

```python
# 고정 연결
add_edge("A", "B")  # A 다음은 항상 B

# 조건부 연결
add_conditional_edges(
    "A",              # 출발 노드
    routing_function,  # State를 받아 문자열을 반환하는 함수
    {                  # 반환값 → 도착 노드 매핑
        "go_b": "B",
        "go_c": "C",
    },
)
```

```python
def route_by_language(state: RouterState):
    """language에 따라 다음 노드를 결정하는 라우팅 함수"""
    if "ko" in state["language"]:
        return "ko"
    else:
        return "en"

graph_builder = StateGraph(RouterState)
graph_builder.add_node("detect_language", detect_language)
graph_builder.add_node("answer_ko", answer_ko)
graph_builder.add_node("answer_en", answer_en)

graph_builder.add_edge(START, "detect_language")
graph_builder.add_conditional_edges(
    "detect_language",
    route_by_language,
    {
        "ko": "answer_ko",
        "en": "answer_en",
    },
)
graph_builder.add_edge("answer_ko", END)
graph_builder.add_edge("answer_en", END)

router_graph = graph_builder.compile()
```

실행 순서: `detect_language` 실행 끝남 → LangGraph가 자동으로 `route_by_language(state)`를 호출 → 반환된 문자열(`"ko"` 또는 `"en"`)을 매핑 dict에서 찾아 → 매칭되는 이름의 실제 노드로 이동.

### Node 함수와 라우팅 함수의 반환값은 다르다

- Node 함수(`detect_language`, `answer_ko` 등) — dict를 반환. State를 갱신함
- 라우팅 함수(`route_by_language`) — 문자열 하나만 반환. State를 안 건드림(읽기만 함)

이 둘이 State를 받는다는 점에서 겉모습이 비슷해서 헷갈리기 쉬운데, 반환값의 용도가 완전히 다르다. 하나는 "State에 뭘 쓸지"(데이터), 다른 하나는 "다음에 어디로 갈지"(경로 선택)다.

`detect_language`가 LLM을 호출해서 언어를 판단하고 State에 써 넣는 노드라면, `route_by_language`는 이미 State에 써진 값을 읽어서 다음 노드 이름을 반환만 하는 라우팅 함수다.

왜 이 둘을 하나로 합치지 않을까: `detect_language`는 LLM 호출이 있어서 Node로 그래프에 등록해야 실행 이력이 남고 State가 갱신된다. 반면 `route_by_language`는 LLM을 안 부르고 이미 있는 값을 보고 판단만 하는 가벼운 로직이라, Node가 아니라 `add_conditional_edges`에 끼워 넣는 판단 전용 함수로 분리한다. "이 함수가 State를 갱신하는가"가 Node와 라우팅 함수를 가르는 기준이다.

매핑 dict(`{"ko": "answer_ko", "en": "answer_en"}`)를 한 단계 더 감싼 이유: 판단 함수는 "의미"만 반환하게 하고(`"ko"`), 그 의미를 실제 어느 노드로 보낼지는 그래프 조립 코드에서 따로 정하게 하면, 나중에 노드 이름을 바꾸거나 같은 판단으로 다른 노드를 연결하고 싶을 때 판단 함수는 안 건드리고 이 dict만 고치면 된다.

## graph.stream() — 실행 과정을 실시간으로 보기

`invoke()`가 그래프 실행이 끝난 뒤 최종 상태를 반환한다면, `graph.stream()`은 각 노드가 실행된 뒤의 상태 업데이트를 순서대로 반환한다. 이를 통해 조건에 따라 어떤 노드가 실행되었는지 확인할 수 있다.

`llm.stream()`이 모델의 응답을 토큰 또는 콘텐츠 조각 단위로 전달하는 것과 달리, `graph.stream()`은 기본적으로 상태 업데이트를 노드 단위로 반환한다.

```python
for event in router_graph.stream({"question": "파이썬의 장점이 뭐야?"}):
    for node_name, value in event.items():
        print(f"[{node_name}] {value}")
        print()
```

`stream()`은 노드가 실행될 때마다 값을 하나씩 내보내는 제너레이터를 반환하는 함수라서, `for`로 하나씩 받아야 한다. `event`는 그때그때 dict 하나로, 키는 방금 실행된 노드 이름, 값은 그 노드가 반환한 dict다.

**`stream()`도 새 실행이다.** `for event in router_graph.stream(...)`는 `invoke()`와 마찬가지로 그래프를 지금 처음부터 새로 돌리는 호출이다. 과거 기록을 조회하는 게 아니다. 다만 결과를 "다 끝나고 한 번에" 주지 않고, "노드 하나 끝날 때마다 하나씩" 준다는 차이뿐이다.

실제 실행 예시(질문: "파이썬의 장점이 뭐야?"):

```
[detect_language] {'language': 'ko'}
[answer_ko] {'answer': '파이썬(Python)은...'}
```

이벤트가 딱 두 번 나온 건, 이 그래프에서 실행된 노드가 `detect_language` → `answer_ko` 두 개뿐이라서다. `route_by_language`는 이벤트로 안 찍히는데, Node가 아니라 라우팅 함수라서 State를 반환하지 않기 때문이다(`stream()`은 State를 갱신한 Node만 이벤트로 내보낸다).

### stream_mode="values"

`graph.stream()`은 기본적으로 각 노드가 반환한 State 업데이트를 전달한다. 각 단계의 전체 State를 확인하려면 `stream_mode="values"`를 지정한다.

```python
for state in router_graph.stream(
    {"question": "파이썬의 장점이 뭐야?"},
    stream_mode="values",
):
    print(state)
    print()
```

결과는 이렇게 필드가 누적된다.

```
{'question': '파이썬의 장점이 뭐야?'}
{'question': '파이썬의 장점이 뭐야?', 'language': 'ko'}
{'question': '파이썬의 장점이 뭐야?', 'language': 'ko', 'answer': '...'}
```

- 기본 `stream()` — 이번에 끝난 노드가 새로 반환한 필드만 (예: `{"language": "ko"}`)
- `stream(..., stream_mode="values")` — 그 시점까지 누적된 State 전체 (예: `{"question": ..., "language": "ko"}`)

첫 번째 출력이 `{'question': ...}`뿐인 이유는, 그래프가 노드를 하나도 안 돌린 시작 시점의 State부터 한 번 보여주기 때문이다. 그래서 총 이벤트 수는 "노드 개수"가 아니라 "노드 개수 + 1"(시작 시점 포함)이다.

## 그래프: 루프

LangGraph의 조건부 엣지를 사용해 반복과 종료 조건을 구현할 수 있다. State의 `count`를 1씩 증가시키고, 5가 되면 반복을 종료하는 예시.

```
START → increment → count < 5 → increment로 돌아감
                   count ≥ 5 → END
```

```python
class CountState(TypedDict):
    count: int

def increment(state: CountState):
    return {"count": state["count"] + 1}

def route_by_count(state: CountState):
    if state["count"] >= 5:
        return "end"
    return "continue"

count_builder = StateGraph(CountState)
count_builder.add_node("increment", increment)

count_builder.add_edge(START, "increment")
count_builder.add_conditional_edges(
    "increment",
    route_by_count,
    {
        "continue": "increment",   # 핵심: increment 다음이 다시 increment
        "end": END,
    },
)

count_graph = count_builder.compile()
```

이건 State/Node/조건부 Edge를 그대로 응용한 것이다. 유일하게 새로운 건 매핑 dict의 `"continue": "increment"`다 — 아까는 `"ko": "answer_ko"`처럼 다른 노드로 갔는데, 여기는 자기 자신("increment")으로 다시 연결된다.

실행 결과(`{"count": 0}`으로 시작):

```
{'increment': {'count': 1}}
{'increment': {'count': 2}}
{'increment': {'count': 3}}
{'increment': {'count': 4}}
{'increment': {'count': 5}}
```

`increment`가 정확히 5번 실행됐다. `count`가 5가 된 시점에 `route_by_count`가 `state["count"] >= 5`를 만족해서 `"end"`를 반환 → `END`로 이동해서 멈췄다.

**핵심 포인트**

- `increment` 노드가 실행될 때마다 State의 `count`가 1씩 증가한다.
- `route_by_count`는 State를 변경하지 않고 다음 경로만 결정한다.
- `"continue": "increment"` 연결을 통해 이전 노드로 돌아갈 수 있다.
- `count`가 5 이상이면 `END`로 이동해 반복을 종료한다.

**퀴즈로 확인한 것**: `{"count": 10}`처럼 이미 조건을 넘긴 채로 시작하면 `increment`가 몇 번 실행될까? 답은 **1번**이다. `START → increment`는 조건 없이 무조건 타는 고정 Edge라서, `count`가 이미 얼마든 일단 `increment`가 한 번은 실행된다(11이 됨). 그 직후 `route_by_count`가 `11 >= 5`를 만족해서 바로 `END`로 간다. 실제로 실행해서 `{'increment': {'count': 11}}` 한 줄만 나오는 걸로 확인했다.

> ➕ **더 알아두기**
> 이건 실무에서 "이 루프가 최소 몇 번은 무조건 실행된다"는 걸 알아야 안전한 설계를 할 수 있다는 뜻이다. 예를 들어 이 그래프가 "Tool 호출 → 결과 확인" 루프였다면, 초기 State를 아무리 잘 세팅해도 Tool은 최소 1번은 반드시 호출된다. 만약 그 Tool 호출이 비용이 큰 작업(결제 API 등)이었다면, "조건을 만족해도 최소 1번은 실행된다"는 걸 모르고 설계하면 예상 못 한 호출이 나갈 수 있다.

## 언제 Node를 나눌까?

LangGraph에서는 보통 다음 기준으로 Node를 나눈다.

- 책임이 분리되는가? 서로 다른 역할을 수행하는 작업은 별도의 Node로 나눈다.
- 상태가 의미 있게 바뀌는가? 중간 결과를 State에 저장하고 다음 단계에서 사용해야 한다면 Node를 나누는 것이 좋다.
- 분기나 재시도가 필요한가? 결과에 따라 경로가 달라지거나 특정 작업만 다시 실행해야 한다면 독립된 Node로 구성한다.
- 관찰이 필요한가? 실행 결과, 소요 시간, 오류 등을 단계별로 확인해야 하는 작업은 별도의 Node로 나눈다.

조건부 분기 예제는 언어 감지와 답변 생성을, 반복 예제는 숫자 추측과 결과 확인을 각각 별도의 Node로 나누었다. 반대로 항상 함께 실행되고 개별적인 분기, 재시도, 관찰이 필요 없는 작은 작업은 하나의 Node 안에서 처리해도 된다.

## Reducer

노드가 반환한 값은 기본적으로 기존 State 값을 덮어쓴다. State 필드에 Reducer를 지정하면 기존 값과 새 값을 어떻게 합칠지 정의할 수 있다.

```python
name: str                              # reducer 없음 → 새 값으로 덮어쓰기
messages: Annotated[list, add_messages] # 대화 메시지 누적
logs: Annotated[list, operator.add]     # 기존 리스트와 새 리스트 연결
```

### Annotated는 "포장지"다

`Annotated`는 Python `typing` 모듈에 있는 문법으로, 타입 하나 + 그 옆에 아무 값이나 하나를 같이 묶어서 들고 다니는 포장지를 만든다. 실제로 뜯어서 확인했다.

```python
from typing import Annotated, get_args, get_origin
import operator

thing = Annotated[list, operator.add]
print(get_origin(thing))  # typing.Annotated
print(get_args(thing))    # (<class 'list'>, <built-in function add>)
```

`get_args`로 열어보니 `Annotated[list, operator.add]`는 딱 두 개짜리 튜플 `(list, operator.add)`를 포장해둔 상자였다. `Annotated` 자체는 아무 계산도 안 하고 그냥 두 개를 묶어서 보관만 한다. 실제로 뭔가를 하는 건 LangGraph 쪽이다 — LangGraph는 State 클래스의 필드마다 이 상자를 열어봐서, `Annotated` 상자면 두 번째 값을 꺼내서 "이 필드는 값이 갱신될 때마다 이 함수를 호출해서 합쳐라"고 등록한다.

> ♻️ 처음엔 `Annotated[list, operator.add]`가 docstring처럼 그냥 사람이 읽는 설명(주석 같은 것)이라고 생각했으나, 그렇지 않다. docstring은 Python이 실행 중 관여 안 하는 텍스트지만, `Annotated`의 두 번째 자리(`operator.add`)는 State가 갱신될 때마다 LangGraph 엔진이 실제로 호출하는 진짜 함수다. 겉보기엔 타입 힌트처럼 조용히 붙어있어서 오해하기 쉽지만, 실행 결과를 직접 바꾸는 코드다.

### Reducer는 "함수 이름"이 아니라 "역할 이름"이다

`Reducer`라는 이름의 클래스나 메서드가 코드 어딘가에 있는 게 아니다. `Node`가 `add_node`로 등록한 평범한 함수를 부르는 개념적인 이름이었던 것처럼, `Reducer`도 "기존 값과 새 값, 두 개를 매개변수로 받아서 합친 값 하나를 반환하는 함수"라는 모양을 가진 함수들을 통틀어 부르는 이름이다.

`operator.add`의 실제 정의를 확인했다.

```
add(a, b, /)
    Same as a + b.
```

매개변수 두 개(`a`, `b`)를 받아서 `a + b`를 반환한다. LangGraph는 이 함수를 `operator.add(기존값, 새값)`처럼 호출한다.

**`Annotated`의 첫 번째 자리(타입)도, 두 번째 자리(Reducer)도 정해진 게 아니다.** `list`가 아니어도 되고, 이름이 `add`가 아니어도 된다. 조건은 딱 하나, "매개변수 2개(기존값, 새값)를 받아서 값 하나를 반환하는 형태"면 된다. 실제로 직접 만든 함수로 검증했다.

```python
def keep_max(a: int, b: int) -> int:
    return max(a, b)

class ScoreState(TypedDict):
    high_score: Annotated[int, keep_max]   # 타입 = int(list 아님), Reducer = keep_max(add 아님)
```

`round1`(70점) → `round2`(55점) 순서로 반환해도 최종 `high_score`는 70(더 큰 값)이 남는 걸로 확인했다. 참고로 `max`(파이썬 내장 함수)를 그대로 Reducer로 쓰려고 하면 `ValueError: no signature found for builtin`이 난다. LangGraph가 Reducer의 매개변수 개수를 `inspect.signature`로 검사하는데, `max`는 가변 인자를 받는 내장 함수라 시그니처 확인이 안 되기 때문이다. 직접 만든 함수(`keep_max`)처럼 시그니처가 명확해야 한다.

### 리스트가 아닌 타입(dict)은 Reducer가 다르다

리스트는 `+`(이어붙이기)가 정의돼 있어서 `operator.add`가 바로 되지만, dict는 `+` 자체가 정의돼 있지 않다.

```python
operator.add({"a": 1}, {"b": 2})
# TypeError: unsupported operand type(s) for +: 'dict' and 'dict'
```

`add_messages`도 시그니처를 확인하면 `(left: Messages, right: Messages) -> Messages`처럼 메시지 전용이다. dict를 합치려면 직접 함수를 만들어야 한다.

```python
def merge_dict(old: dict, new: dict) -> dict:
    return {**old, **new}   # 딕셔너리 언패킹(**)으로 두 dict를 합침(같은 키면 new가 덮어씀)
```

**Reducer 선택 가이드**

| 상황 | reducer | 동작 |
|------|---------|------|
| 최신 값만 필요 (이름, 최종 판정 등) | 없음 | 덮어쓰기 |
| 대화 메시지 누적 | `add_messages` | 메시지 형식으로 처리하며 누적 |
| 로그, 결과 목록 누적 | `operator.add` | 단순 리스트 이어붙이기 (`+`) |
| dict 등 그 외 | 직접 정의 | 합치는 규칙을 직접 함수로 작성 |

### Reducer 있는 필드 vs 없는 필드 비교

두 노드가 값을 반환할 때, 필드에 Reducer가 있는지 없는지에 따라 결과가 완전히 다르다는 걸 실제로 확인했다.

```python
class DemoState(TypedDict):
    plain: str                                # reducer 없음
    accumulated: Annotated[list, operator.add]  # reducer 있음

def step1(state):
    return {"plain": "1번 노드가 씀", "accumulated": ["1번 로그"]}

def step2(state):
    return {"plain": "2번 노드가 씀", "accumulated": ["2번 로그"]}
```

`step1 → step2` 순서로 실행한 결과:

```
plain       : 2번 노드가 씀
accumulated : ['1번 로그', '2번 로그']
```

`plain`은 `step2`가 반환한 값만 남았다(덮어쓰기). `accumulated`는 둘 다 남았다(`operator.add`가 `["1번 로그"] + ["2번 로그"]`를 계산). 같은 방식으로 노드 두 개가 값을 반환했는데, 필드에 Reducer가 있냐 없냐에 따라 결과가 완전히 다르게 나온 것이다.

## Chain은 State 같은 공용 그릇이 없다

State도 사실 "강제"하는 건 아니다(`TypedDict`는 실행 중 검사 안 됨). 진짜 차이는 "여러 단계가 공유하는 하나의 그릇이 있느냐 없느냐"다.

- LangGraph — State라는 공용 그릇을 하나 정의하고, 모든 Node가 그걸 같이 씀
- Chain(LCEL) — 그런 공용 그릇 없이, 한 단계의 출력이 그대로 다음 단계의 입력이 됨

Chain 안에서도 Pydantic 모델이나 dict로 데이터를 구조화할 수 있지만, LangGraph의 State처럼 "전체 흐름이 공유하는 하나의 정해진 틀"은 아니고, 그때그때 필요한 곳에서만 개별적으로 쓰는 것이다. Chain 대신 클래스를 만들어서 데이터를 공유할 수도 있지 않을까 하는 질문에는, LangGraph가 dict(TypedDict) 기반 State를 쓰는 이유가 있다.

1. **저장·복원이 쉬워야 한다.** LangGraph는 State를 디스크에 저장했다가 나중에 다시 불러오는 Checkpoint 기능이 있다. dict는 JSON으로 그대로 저장/복원되지만, 직접 만든 클래스는 저장·복원 로직을 따로 다 짜야 한다.
2. **"병합" 모델이 dict라서 단순하다.** dict는 키 하나씩 비교해서 합치기 쉽지만, 클래스 객체 두 개를 합치는 건 애매하다.
3. **LangGraph가 State 구조를 코드로 뜯어봐야 한다.** `Annotated`를 `get_origin`/`get_args`로 열어봤듯이, LangGraph는 State 정의를 표준적인 방법으로 읽어서 필드와 Reducer를 자동으로 파악한다.

## Chain vs LangGraph 요약

| | Chain | LangGraph |
|---|---|---|
| 흐름 | 직선 (A → B → C) | 분기 + 루프 가능 |
| 상태 | 이전 단계 출력만 전달 | State로 전체 공유 |
| 제어 | 고정된 순서 | 조건에 따라 동적 결정 |
| 디버깅 | 각 단계 출력 확인 | 그래프 시각화 + 노드별 트레이싱 |
| 적합한 경우 | 단순 파이프라인 | 분기, 반복, 상태 관리가 필요한 경우 |

## 실습 1: 감정 분석 라우터 — 강사님 코드와 비교

문제: 사용자의 문장을 받아 감정을 분석하고, 긍정이면 긍정 응답, 부정이면 위로 응답을 생성하는 그래프.

```
START → analyze → 긍정? → respond_positive → END
                   부정? → respond_negative → END
```

내가 만든 버전은 `llm.invoke(f"...")`를 직접 썼고, 정제(`.strip().lower()`)를 `analyze` 노드 안에서 했다. 강사님 버전은 두 가지가 달랐다.

```python
def analyze(state: State):
    text = state['text']
    prompt = PromptTemplate.from_template(
        "다음 <text>에 대해서 감정을 분석해줘. 긍정이면 'positive' 부정이면 'negative'로 응답해줘. <text>{text}</text>"
    )
    chain = prompt | llm | StrOutputParser()
    response = chain.invoke({'text' : text})
    return {'sentiment' : response}

def route_by_sentiment(state: State):
    sentiment = state["sentiment"]
    sentiment = sentiment.strip().lower()   # 정제를 여기(판단 함수)에서 함
    if sentiment == "positive":
        return "positive"
    else:
        return "negative"
```

**차이 1 — 정제(`.strip().lower()`)를 어디서 하는가.** 나는 `analyze` 노드 안에서 정제해서 State에 깨끗한 값을 저장했는데, 강사님은 `analyze`에는 원본 그대로 저장하고 `route_by_sentiment`(판단 함수) 안에서 정제한다. State에 "가공 안 된 원본"을 남기느냐 "가공된 값"을 남기느냐의 철학 차이다. 강사님 방식은 디버깅할 때 LLM이 실제로 뭐라고 답했는지 State에서 그대로 볼 수 있다는 장점이 있다.

**차이 2 — `analyze`를 만드는 방법.** 강사님은 `PromptTemplate` + `chain = prompt | llm | StrOutputParser()`라는 LCEL 체인을 새로 만들어 썼고, 나는 `llm.invoke(f"...")`로 바로 불렀다. 결과는 같지만, 강사님 쪽은 프롬프트를 재사용 가능한 템플릿 객체로 분리했다.

### PromptTemplate과 f-string의 차이

`PromptTemplate.from_template("...{text}...")`은 문자열 안의 `{text}`를 나중에 채울 빈칸으로 등록하는 "틀"을 만드는 것뿐이다. 이 시점엔 아직 아무 값도 안 채워진다.

```python
chain.invoke({'text' : text})
```

여기서 `{'text' : text}`의 왼쪽 `'text'`(따옴표 있음)는 템플릿 빈칸 이름과 매칭되는 문자열 키이고, 오른쪽 `text`(따옴표 없음)는 이전 줄 `text = state['text']`에서 만들어진 Python 변수다. 이름이 우연히 같아서 헷갈리기 쉽지만, 이 dict는 "`text`라는 이름의 빈칸에, 변수 `text`에 담긴 실제 값(사용자 문장)을 넣어라"는 뜻이다.

> ♻️ 처음엔 "text라는 빈칸에 text라는 문자열(글자 그대로)을 넣는다"고 오해했으나, 그렇지 않다. 변수의 이름과 그 변수 안에 든 값은 다르다. 변수 `text`의 이름은 `"text"`라는 글자지만, 안에 들어있는 값은 사용자가 입력한 실제 문장(`"오늘 승진했어! 너무 기뻐!"` 등)이다. 채워진 후 최종 프롬프트는 `"...<text>오늘 승진했어! 너무 기뻐!</text>"`처럼 된다.

f-string(`f"{text}"`)은 그 자리에서 즉시 값을 채워 넣는 문법이고, `PromptTemplate`은 나중에 채울 수 있도록 틀을 저장해두는 것이다. 그래서 같은 틀을 여러 번 재사용할 수 있다.

프롬프트 문자열 안의 `<text>...</text>`(꺾쇠괄호)는 `{text}`(중괄호)와 완전히 다르다. `<text>`는 Python/LangChain 문법이 아니라 그냥 프롬프트 안에 손으로 써넣은 평범한 글자다. LLM이 "어디서부터 어디까지가 분석해야 할 대상 텍스트인지"를 명확히 구분하도록, 지시문과 데이터를 시각적으로 표시해주는 관례적인 기법이다.

- `{text}` — Python/LangChain이 인식하는 빈칸 문법(실행 전에 실제 값으로 치환됨)
- `<text>...</text>` — 그냥 프롬프트 안의 평범한 글자(LLM에게 "여기가 대상 텍스트다"라고 알려주는 관례적 표시)

### 체인(chain.invoke)과 llm.invoke의 차이

최종적으로 LLM에게 뭘 보내고 뭘 받는지(내용)는 같지만, 코드 구조와 반환 타입이 다르다.

- `llm.invoke(f"...")` — LLM을 바로 한 번 부름. 응답 객체가 나와서 `.text`로 텍스트를 꺼내야 함
- `chain = prompt | llm | StrOutputParser()` 다음 `chain.invoke(...)` — "프롬프트 조립 → LLM 호출 → 텍스트 추출"을 하나로 묶은 객체. 결과가 이미 문자열이라 `.text`가 필요 없음

`|`(파이프)는 원래 Python에서 비트 OR 연산 기호인데, LangChain이 각 객체에 이 기호를 재정의해서 "다음 단계로 연결한다"는 뜻으로 쓴다. `StrOutputParser()`의 빈 괄호는 함수 호출이 아니라 클래스의 인스턴스(객체)를 만드는 것이다.

> ♻️ 처음엔 `StrOutputParser`가 LLM의 답변 형식을 강제하는 거라고 오해했으나(→ `with_structured_output`이랑 헷갈림), 그렇지 않다. `StrOutputParser`는 LLM의 답변 형식에 전혀 관여 안 하고, 이미 자유 형식으로 만들어진 텍스트에서 응답 객체를 열어 순수 문자열만 꺼내는 역할일 뿐이다(봉투를 열어 편지지만 꺼내는 것과 비슷). 답변 형식을 진짜로 강제하는 건 실습 3에서 쓴 `llm.with_structured_output(모델)`이다.

지금까지 나온 세 가지 반환 방식.

- `llm.invoke(...)` — 응답 객체 반환, `.text`로 텍스트 꺼냄
- `chain.invoke(...)` (StrOutputParser 있음) — 이미 문자열 그 자체
- `구조화된 llm.invoke(...)` (with_structured_output) — Pydantic 객체 반환, `.필드이름`으로 값 꺼냄

### route_by_sentiment의 죽은 코드(`# return sentiment`)가 알려준 것

```python
def route_by_sentiment(state: State):
    sentiment = state["sentiment"]
    sentiment = sentiment.strip().lower()
    if sentiment == "positive":
        return "positive"
    else:
        return "negative"
    # return sentiment
```

`add_conditional_edges`의 매핑 dict는 키가 딱 두 개(`"positive"`, `"negative"`)뿐이라, 라우팅 함수가 반환하는 문자열이 이 두 키 중 하나와 정확히 일치해야 한다. `# return sentiment`처럼 정제된 값을 그대로 반환했다면, LLM이 예상 못한 단어로 답할 경우 dict에 없는 키가 반환돼 에러가 날 위험이 있다. `if/else`로 명시적으로 쓰면 항상 dict의 키와 정확히 일치하는 값만 반환하도록 강제할 수 있다.

## 실습 2: 숫자 맞히기 — Node를 나누는 원칙이 가장 잘 드러난 예제

문제: LLM이 1~100 사이의 숫자를 추측하고, 정답이 아니면 범위를 좁혀 다시 시도.

```
START → guess → check → 정답? → END
                  ↑      오답? → guess로 돌아감
```

`check` 노드는 추측값이 정답보다 작으면 `low`를 높이고, 크면 `high`를 낮춘다.

내가 만든 버전은 "정답 확인 + low/high 갱신"을 `check` 노드 하나에 합쳤다. 강사님 버전은 셋으로 나눴다.

```python
MAX_ATTEMPT = 5

def guess(state: GuessState):
    low = state['low']
    high = state['high']
    attempt = state.get('attempt', 0)
    response = llm.invoke(f"{low} ~ {high}까지의 숫자를 골라줘. 단, 정수만 답변해.")
    return {'guess' : int(response.text), 'attempt' : attempt + 1}

def route_success(state: GuessState):
    # 판단만 함 (State 안 건드림) -> Node가 아니라 라우팅 함수
    if state['attempt'] >= MAX_ATTEMPT:
        return 'end'
    if state['target'] == state['guess']:
        return 'end'
    return 'continue'

def change_high_low(state: GuessState):
    # State 갱신만 함 (판단 없음) -> 여기가 진짜 Node
    if state['target'] > state['guess']:
        return {'low' : state['guess'] + 1}
    if state['target'] < state['guess']:
        return {'high' : state['guess'] - 1}

guess_graph_builder.add_edge('guess', ??? )  # 실제로는 아래처럼 조립
```

실제 조립은 이랬다.

```python
guess_graph_builder = StateGraph(GuessState)
guess_graph_builder.add_node('guess', guess)
guess_graph_builder.add_node('change_high_low', change_high_low)

guess_graph_builder.add_edge(START, 'guess')
guess_graph_builder.add_conditional_edges(
    'guess',
    route_success,
    {
        'end' : END,
        'continue' : 'change_high_low'
    }
)
guess_graph_builder.add_edge('change_high_low', 'guess')
```

**가장 중요한 차이는 노드를 어디서 나눴는가다.** 나는 "정답 확인 + low/high 갱신"을 `check`라는 노드 하나로 합쳤다. 강사님은 이걸 셋으로 쪼갰다 — `route_success`(판단만, State 안 건드림), `change_high_low`(State 갱신만, 판단 없음), 그리고 반복을 만드는 자리도 다르다.

- 내 그래프: `guess → check(고정) → [continue: guess, end: END](조건부)`
- 강사님 그래프: `guess → [continue: change_high_low, end: END](조건부) → change_high_low → guess(고정)`

강사님 쪽이 "판단 함수는 State를 안 건드리고 문자열만 반환, Node는 State를 갱신한다"는 원칙을 더 엄격하게 지킨 것이다.

> ➕ **더 알아두기**
> 또 하나, 강사님은 `result`라는 필드를 아예 안 만들었다. `target == guess`인지를 `route_success` 안에서 그때그때 바로 비교하고, State에 저장 안 한다. 반면 나는 `"correct"/"too_low"/"too_high"`라는 문자열을 State에 저장했는데, 사실 그 값은 `check`가 실행되는 순간에만 쓰이고 나중에 아무도 다시 안 읽는다 — 있으나 마나 한 필드였다. 판단 결과를 나중에 다른 곳에서도 써야 할 게 아니라면, 굳이 State에 저장할 필요가 없다.

강사님 코드에서 발견한 작은 불일치도 있었다. 문제 설명에는 `guess_graph.stream({"target": 37})`만 넣으라고 돼 있는데, 실제 강사님 코드는 `low`/`high`에 `.get()` 기본값이 없어서 `{'target': 37, 'low': 1, 'high': 100}`처럼 셋 다 넣어야 실행된다. 나는 `.get()`으로 기본값을 줘서 문제에 적힌 대로 `{"target": 37}`만 넣어도 돌아가게 만들었다 — 이 부분은 내 쪽이 더 안전했다.

## 실습 3: 글 작성 → 검토 → 재작성 — 구조화된 출력과 안전장치

문제: 주제를 받아 짧은 글을 작성하고, 검토 후 품질이 부족하면 피드백을 반영해 다시 작성.

```
START → write → review → 통과? → END
                         미흡? → write로 돌아감
```

### with_structured_output — 답변 형식을 진짜로 강제하는 방법

```python
from typing import Literal
from pydantic import BaseModel

class WriteState(TypedDict):
    topic: str
    sentense: str
    feedback: str
    result: Literal['pass', 'fail']   # 'pass' 또는 'fail' 두 값만 허용

class ReviewResult(BaseModel):
    result: Literal['pass', 'fail']
    feedback : str

llm_with_review_result = llm.with_structured_output(ReviewResult)
```

`Literal['pass', 'fail']`은 그냥 `str`과 달리 "이 값은 반드시 `'pass'` 아니면 `'fail'` 둘 중 하나여야 한다"고 콕 집어 표시하는 타입 힌트다. `ReviewResult(BaseModel)`은 이전 RAG 노트북에서 `RelevanceScore`, `JudgeScore` 만들 때 썼던 것과 같은 Pydantic 모델 패턴이다.

```python
def review(state: WriteState):
    topic = state['topic']
    text = state['sentense']
    response = llm_with_review_result.invoke(f"""
다음 <text>는 <topic>을 주제로 한 글이야. 다음 글에 대해서 검토해줘.
- result에는 "pass" 또는 "fail"을 담아줘.
- feeback에는 그렇게 생각한 이유를 명시해줘.
<topic>{topic}</topic>
<text>{text}</text>
""")
    return {'result' : response.result, 'feedback' : response.feedback }
```

`llm_with_review_result`는 "답변을 반드시 `ReviewResult` 구조(result/feedback 필드)로만 하도록 제약해둔 llm 객체"다. 이걸로 `.invoke(...)`하면 결과가 `ReviewResult` 객체 자체로 나와서, `.text`가 아니라 필드 이름 그대로 `.result`, `.feedback`처럼 속성 접근으로 값을 꺼낸다.

**`feedback`이라는 필드 이름의 "의미"는 어디서 정해질까?** `ReviewResult`의 `feedback: str`은 형식(문자열이어야 한다)만 강제할 뿐, 그 안에 뭘 넣어야 하는지는 전혀 모른다. 실제로 "검토 이유를 담아라"는 의미를 정의하는 건 프롬프트 문장 `"- feeback에는 그렇게 생각한 이유를 명시해줘."`다. LLM이 이 자연어 지시문을 읽고 그 필드에 뭘 채워야 하는지 이해하는 것이다.

- Pydantic 스키마(`ReviewResult`) — 어떤 필드가 있어야 하는지(형식)를 강제
- 프롬프트 문장 — 그 필드에 어떤 의미의 내용을 넣을지(의미)를 지정

> ➕ **더 알아두기**
> 실무에서는 `feedback: str = Field(description="검토가 fail인 이유")`처럼 Pydantic 필드에 `Field(description=...)`를 추가로 달아서 필드 옆에 바로 의미를 적어두는 경우가 많다. 지금 코드는 이 설명을 프롬프트 본문에 풀어서 적었는데, 둘 다 결국 LLM에게 의미를 알려주는 같은 역할이다.

### 텍스트 파싱보다 구조화된 출력이 안정적인 이유

내가 만든 `review`는 텍스트로 "PASS"/"FAIL: 이유"를 받아서 직접 잘라 파싱했다.

```python
def review(state: WritingState):
    result = llm.invoke(
        "다음 글이 주제에 잘 맞고 품질이 좋으면 'PASS'라고만 답해. "
        f"부족하면 'FAIL: 이유'형태로 답해.\n\n글: {state['draft']}"
    )
    text = result.text.strip()
    if text.upper().startswith("PASS"):
        return {"passed": True, "feedback": ""}
    return {"passed": False, "feedback": text}
```

`text.upper().startswith("PASS")`는 LLM이 정확히 "PASS"로 시작하는 문장을 줄 거라고 믿는 코드다. 만약 LLM이 `"이 글은 PASS 수준입니다."`처럼 답하면, `.startswith("PASS")`는 문자열이 정확히 "PASS"로 시작하는지만 보기 때문에 `False`로 판정돼서 엉뚱하게 FAIL 처리된다. LLM은 지시를 100% 그대로 따른다는 보장이 없어서, 자유 문장을 보고 의미를 추측해서 파싱하는 방식은 프로덕션에서 예측 못한 버그로 이어지기 쉽다.

`with_structured_output(ReviewResult)`을 쓰면 LLM이 자유 문장을 만드는 게 아니라 API 차원에서 정확히 `"pass"` 또는 `"fail"`로만 답하도록 강제되기 때문에, 문구를 추측해서 파싱할 필요가 없다.

### write에서 feedback을 반영하는 부분과 누락됐던 지점

강사님의 기본 버전(cell 52)은 `write`가 `feedback`을 전혀 안 썼다.

```python
def write(state: WriteState):
    topic = state['topic']
    response = llm.invoke(f'{topic}을 주제로 하는 간단한 글을 작성해줘.')
    return {'sentense' : response.text}
```

문제 설명엔 "미흡하면 피드백을 반영해 다시 작성한다"고 돼 있는데, 이 코드는 재작성할 때도 매번 똑같은 프롬프트만 보낸다. 나는 `state.get("feedback", "")`으로 이전 피드백이 있는지 확인해서 프롬프트를 다르게 만들었다 — 이 부분은 내 코드가 문제 요구사항을 더 정확히 지킨 것이다. 강사님도 이 문제를 알고 있었던 것으로 보이는 게, 뒤에 나오는 "fail 유도 버전"(cell 55)에서는 `feedback`을 실제로 프롬프트에 넣도록 고쳐놨다.

### 안전장치(재시도 제한)는 처음부터 필요하다

강사님의 기본 버전(cell 52)은 재시도 최대 횟수 제한이 아예 없었다. `review`가 계속 `'fail'`을 반환하면 이론상 무한 반복할 수 있다. 나는 실습 2, 3 모두 `attempts >= N`이면 강제로 끝내는 안전장치를 처음부터 넣어뒀다. 이건 강사님 쪽도 결국 필요하다고 느꼈는지, "fail 유도 버전"(cell 55)에서 `attempt` 필드와 `attempt >= 3` 체크가 새로 추가됐다.

```python
def route_by_result(state: WriteState):
    result = state['result']
    if state['attempt'] >= 3:
        print('실패')
        return 'pass'   # 3번 넘겨도 강제로 'pass'(=END로) 보냄
    if result == 'pass':
        return 'pass'
    else:
        return 'fail'
```

다만 여기엔 아쉬운 점이 있다. "3번 넘겨서 강제 종료"와 "진짜로 검토를 통과"가 같은 값(`'pass'`)으로 뒤섞인다. 나는 `route_by_review`에서 `state["passed"] or state["attempts"] >= 3`이면 `"end"`(END로 직접 연결된 이름)를 반환해서, State 안에 `passed: False`가 남아있어도 그래프는 종료되게 만들었다. 강사님 방식은 콘솔에 `print('실패')`가 찍히긴 하지만, State 안에는 "진짜 통과인지 포기인지"의 구분이 안 남는다.

## FakeListChatModel — 실제 LLM 없이 재작성 루프 검증하기

**과제**: 실습 3 그래프가 진짜로 재작성 루프를 도는지, 실제 LLM 호출 없이 미리 정해둔 가짜 답변으로 검증하기.

**왜 필요한가**: 실제 Gemini로 돌리면 운이 좋아 한 번에 PASS가 나올 수도 있어서, "재작성(retry)" 경로가 실제로 작동하는지 100% 확신할 수 없었다. `FakeListChatModel`을 쓰면 "1번째는 반드시 FAIL, 2번째는 반드시 PASS"처럼 답을 직접 정해서 루프가 정확히 도는지 확인할 수 있다.

강사님이 사진으로 스케치해준 구조: `llm`을 인스턴스 속성으로 주입받는 클래스를 만들어서, 실제 llm도 가짜 llm도 넣어볼 수 있게 한다.

```python
class Graph:
    def __init__(self, llm):
        self.llm = llm                       # 주입받은 llm을 인스턴스 속성으로 저장
        self.compiled = self._build_graph()

    def write(self, state: WritingState):
        feedback = state.get("feedback", "")
        if feedback:
            prompt = f"주제 '{state['topic']}'로 짧은 글(3문장 이내)을 다시 써줘. 이전 피드백을 반영해: {feedback}"
        else:
            prompt = f"주제 '{state['topic']}'로 짧은 글(3문장 이내)을 써줘."
        result = self.llm.invoke(prompt)     # self.llm -> 인스턴스마다 다른 llm을 씀
        attempts = state.get("attempts", 0) + 1
        return {"draft": result.text, "attempts": attempts}

    def review(self, state: WritingState):
        result = self.llm.invoke(
            "다음 글이 주제에 잘 맞고 품질이 좋으면 'PASS'라고만 답해. "
            f"부족하면 'FAIL: 이유'형태로 답해.\n\n글: {state['draft']}"
        )
        text = result.text.strip()
        if text.upper().startswith("PASS"):
            return {"passed": True, "feedback": ""}
        return {"passed": False, "feedback": text}

    def route_by_review(self, state: WritingState):
        if state["passed"] or state["attempts"] >= 3:
            return "end"
        return "continue"

    def _build_graph(self):
        builder = StateGraph(WritingState)
        builder.add_node("write", self.write)      # self.write: 이 인스턴스에 묶인 메서드
        builder.add_node("review", self.review)
        builder.add_edge(START, "write")
        builder.add_edge("write", "review")
        builder.add_conditional_edges(
            "review", self.route_by_review, {"continue": "write", "end": END},
        )
        return builder.compile()
```

`self.llm`이 핵심이다. `Graph(llm=llm)`으로 만들면 그 인스턴스의 `self.llm`은 진짜 Gemini고, `Graph(llm=FakeLmm)`으로 만들면 그 인스턴스의 `self.llm`은 가짜다. 같은 클래스로 만든 두 인스턴스가 서로 다른 llm을 각자 들고 있는 것이다. `add_node("write", self.write)`도 `self.write`(이 인스턴스에 묶인 메서드)를 넘겨야, 나중에 실행될 때 `self.llm`을 정확히 찾아간다.

### FakeListChatModel의 실제 동작 (직접 확인)

```python
from langchain_core.language_models.fake_chat_models import FakeListChatModel

fake = FakeListChatModel(responses=["첫번째 답", "두번째 답", "세번째 답"])
fake.invoke("아무 질문").content  # 첫번째 답
fake.invoke("아무 질문").content  # 두번째 답
fake.invoke("아무 질문").content  # 세번째 답
fake.invoke("아무 질문").content  # 첫번째 답 (4번째 호출은 목록 처음으로 돌아감)
```

`FakeListChatModel(responses=[...])`은 `.invoke()`를 부를 때마다 `responses` 리스트에서 순서대로 하나씩 응답을 내놓는 가짜 LLM이다. `.content`와 `.text` 둘 다 지원해서, 진짜 `llm.invoke()`와 똑같은 방식(`.text`)으로 코드를 짤 수 있다. **`FakeListChatModel`은 프롬프트 내용을 전혀 안 읽는다.** 프롬프트에 뭐라고 자세히 써서 보내든 무시하고, 그냥 호출 순서만 보고 다음 값을 반환한다.

준비한 4개의 가짜 응답과 각각의 역할.

```python
FakeLmm = FakeListChatModel(responses=[
    "AI는 좋다.",                                              # ① write 1번째: 일부러 망친 초안
    "FAIL: 내용이 너무 짧고 구체적인 설명이나 근거가 없습니다.",   # ② review 1번째: 불합격 판정
    "AI는 학습 데이터를 기반으로 패턴을 학습해 새로운 입력에 대한 "
    "답을 예측하는 기술이다. 교육 분야에서는 학생 개개인의 학습 속도에 "
    "맞춰 문제 난이도를 조절하는 데 활용되고 있다.",                # ③ write 2번째: 재작성 성공작
    "PASS",                                                     # ④ review 2번째: 합격 판정
])
```

- ① `"AI는 좋다."` — 일부러 망친 초안. 진짜 LLM이면 첫 시도에 이미 잘 쓸 수도 있어서 재작성 경로를 못 보는데, 이 문장으로 무조건 불합격 나올 부실한 글을 강제로 만든다.
- ② `"FAIL: ..."` — 그 부실한 글에 대한 불합격 판정. `review`가 이 값을 반환하면 `passed: False`가 되고, `route_by_review`가 `"continue"`를 반환해서 `write`로 되돌아가게 만든다.
- ③ `"AI는 학습 데이터를..."` — 재작성이 성공했다고 볼 수 있는 좋은 글.
- ④ `"PASS"` — 재작성본에 대한 합격 판정. 루프가 여기서 끝난다.

이 4개는 "1차 실패 → 재작성 → 2차 성공"이라는 시나리오 하나를 인위적으로 재현하기 위해, 실패할 글 / 실패 판정 / 성공할 글 / 성공 판정을 순서대로 미리 각본처럼 짜둔 것이다.

### 실행 흐름을 순서대로 추적

입력: `{"topic": "AI가 교육에 미치는 영향"}`

1. `write` 1번째 실행 → `self.llm.invoke(...)`가 1번째로 불려서 `"AI는 좋다."` 반환 → `{'write': {'draft': 'AI는 좋다.', 'attempts': 1}}`
2. `review` 1번째 실행 → 2번째로 불려서 `"FAIL: ..."` 반환 → `passed: False` → `{'review': {'passed': False, 'feedback': 'FAIL: ...'}}`
3. `route_by_review` → `passed=False, attempts=1` → `"continue"` → `write`로 되돌아감
4. `write` 2번째 실행(이번엔 `feedback`이 있어서 다른 프롬프트) → 3번째로 불려서 `"AI는 학습 데이터를..."` 반환 → `{'write': {'draft': '...', 'attempts': 2}}`
5. `review` 2번째 실행 → 4번째로 불려서 `"PASS"` 반환 → `passed: True` → `{'review': {'passed': True, 'feedback': ''}}`
6. `route_by_review` → `passed=True` → `"end"` → `END`로 이동, 그래프 종료

실행 결과(가짜 llm):

```
{'write': {'draft': 'AI는 좋다.', 'attempts': 1}}
{'review': {'passed': False, 'feedback': 'FAIL: 내용이 너무 짧고 구체적인 설명이나 근거가 없습니다.'}}
{'write': {'draft': 'AI는 학습 데이터를 기반으로...', 'attempts': 2}}
{'review': {'passed': True, 'feedback': ''}}
```

같은 `Graph` 클래스에 진짜 Gemini(`Graph(llm=llm)`)를 넣어 실행한 결과는 이랬다.

```
{'write': {'draft': 'AI는 개인의 학습 속도와 수준에 맞춘 맞춤형 교육을...', 'attempts': 1}}
{'review': {'passed': True, 'feedback': ''}}
```

진짜 Gemini는 처음부터 잘 써서 재작성 없이 1번 만에 PASS했다. 같은 `Graph` 클래스인데 넣는 `llm`만 바꿨을 뿐인데 완전히 다른 시나리오(재작성 vs 한 번에 통과)를 재현할 수 있었다.

> ♻️ 처음엔 "FakeLmm이 사용자가 입력하는 걸 만든 것"이라고 오해했으나, 그렇지 않다. 사용자 입력은 `{"topic": "..."}`처럼 그래프를 시작할 때 넣는 값이고, 이건 FakeLmm과 무관하다. FakeLmm이 대신하는 건 `self.llm.invoke(prompt)`를 부를 때 원래 진짜 Gemini가 만들어내야 할 답변(LLM의 출력) 쪽이다.

> ♻️ 또한 "재작성 경로를 유도한다"는 게 사람(사용자)을 유도하는 거라고 오해했으나, 그렇지 않다. 이 테스트엔 사람이 개입할 자리가 없다. 유도하는 대상은 `review` 노드의 판정값(`passed`)이고, 그 값에 따라 갈리는 `route_by_review`의 분기 결과다. 가짜 LLM의 답변을 조작해서 그래프 내부의 조건부 분기가 원하는 방향(재작성 쪽)으로 가게 만든 것이다.

### 테스트 더블(test double)이라는 개념

`FakeListChatModel`처럼 진짜 대신 넣는 가짜 객체를 실무에서 "테스트 더블"이라고 부른다. 진짜 LLM은 매번 답이 달라져서 "이 경로가 항상 이렇게 동작한다"를 보장하기 어려운데, 답을 고정해두면 같은 입력엔 항상 같은 결과가 나오는 재현 가능한 테스트를 만들 수 있다. LLM 기반 코드에 자동화된 테스트(pytest 등)를 짤 때 이런 가짜 모델을 필수로 쓴다 — 매번 진짜 API를 불러서 테스트하면 느리고, 비용도 들고, 결과도 매번 달라져서 테스트가 불안정해지기 때문이다.

또한 이 방식이, "노드 코드 안에 테스트용 분기를 직접 심는 것"(예: `if attempt == 1: return {부실한 문장}`처럼 LLM 호출 자체를 건너뛰는 하드코딩)과 다르다는 점도 중요하다.

- 하드코딩 방식 — 원본 코드(`write` 함수) 안에 테스트용 `if`문이 영구히 남는다
- 테스트 더블(FakeListChatModel) 방식 — 원본 코드(`llm.invoke(...)`)는 하나도 안 건드리고, 밖에서 `llm`만 갈아끼워서 검증한다. 실제 서비스에 배포할 코드에 테스트용 로직이 섞이지 않는다

## 자체 Agent SDK와 LangGraph의 차이 (심화)

수업 중 궁금해서 찾아본 내용. 대형 서비스도 실제로 LangGraph를 프로덕션에 쓰고 있다. Klarna(고객 지원 챗봇, 전체 문의의 약 2/3 처리), Uber(대규모 코드 마이그레이션 자동화), LinkedIn(채용 자동화 에이전트) 등이 LangGraph 기반으로 운영되고, Elastic·BlackRock·Cisco·JPMorgan 등을 포함해 약 400개 기업이 LangGraph Platform을 쓴다고 한다(다만 이 통계는 LangGraph를 만든 회사 자체 블로그 발 수치라 정확한 비율은 참고만 하고, 특정 회사들의 실사용 사례 자체는 여러 매체에서 반복 인용돼 신뢰도가 있다).

Anthropic의 Claude Agent SDK, OpenAI의 Agents SDK 같은 "자체 SDK"도 LangGraph와 목적은 비슷하지만 설계 철학이 다르다.

- **Claude Agent SDK** — Claude Code 엔진을 라이브러리로 뽑아낸 것에 가깝다. "에이전트가 어떻게 돌아갈지"를 이미 Anthropic이 정해둔 하나의 루프를 그대로 쓰는 방식이다. 그래프 엔진 자체가 없다. 명시적인 분기·반복·병렬 같은 걸 짜고 싶으면 직접 만들거나 LangGraph를 그 위에 얹어야 한다.
- **OpenAI Agents SDK** — 핵심 개념이 Node/Edge가 아니라 "handoff(넘겨주기)"다. 에이전트 A가 대화 맥락을 들고 에이전트 B에게 통제권을 명시적으로 넘긴다.
- **LangGraph** — 노드와 엣지로 이루어진 그래프를 명시적으로 그려서, 흐름·에러 처리·사람 개입 시점을 정확히 지정할 수 있다.

**"그래프 엔진이 없다"는 게 정확히 무슨 뜻인가**: 어떤 제어 흐름이든 사후적으로는 항상 순서도로 그릴 수 있다(순회가 있으니까). "그래프 엔진이 없다"는 그 그림을 못 그린다는 뜻이 아니라, **그 그림의 모양을 사용자가 마음대로 바꿀 수 있는 도구가 없다**는 뜻이다. LangGraph는 `add_node`, `add_edge`, `add_conditional_edges`로 그래프 모양 자체를 직접 설계할 수 있지만, Claude Agent SDK는 그 루프 모양이 이미 Anthropic이 정해서 고정해둔 것이라 사용자가 임의로 갈림길을 추가하거나 다른 모양의 그래프로 바꿀 수 있는 API가 없다.

원문 표현으로는 "누가 이 루프의 주인이냐(who owns the loop)"가 핵심 차이다. LangGraph 쪽은 "You; the graph defines next steps"(당신, 즉 개발자가 그래프로 다음 단계를 정한다), Claude Agent SDK 쪽은 모델(Claude) 스스로가 다음 단계를 판단해서 정한다.

- LangGraph — `add_conditional_edges`에 개발자가 직접 판단 함수를 끼워 넣어서, 다음 노드를 개발자가 지정한 규칙으로 정함
- Claude Agent SDK — 다음에 뭘 할지를 Claude 스스로 판단해서 정함(개발자는 그 판단 결과를 바꿀 그래프 구조 자체를 못 건드림)

> ➕ **더 알아두기**
> OpenAI가 공개한 "Codex 에이전트 루프 풀어보기"라는 글에서, Codex CLI의 에이전트 루프를 이렇게 설명한다. 사용자 입력 → 모델에게 프롬프트 전달(추론) → 모델이 툴 호출을 요청하면 실행하고 그 결과를 다시 프롬프트에 추가 → 모델이 더 이상 툴을 안 부르고 최종 메시지를 낼 때까지 반복. 대화가 길어질수록 이전 기록 전체가 계속 프롬프트에 쌓여서 컨텍스트 윈도우(한 번에 넣을 수 있는 토큰 한도) 관리가 중요한 책임이 된다. 그래서 이전 프롬프트를 정확히 그대로 유지해 "프롬프트 캐싱"을 활용하고, 너무 길어지면 대화를 요약해서 압축(compaction)한다. 이게 바로 "모델이 스스로 다음 단계를 정하는 고정 루프"의 실제 구현 사례다.

**"AGI라면 Claude SDK 방식이 더 맞지 않나?"라는 질문에 대해**: 방향은 맞다. Claude Agent SDK가 "모델이 다음 단계를 스스로 정한다"는 점에서 Workflow-Agent-Autonomous Agent 스펙트럼의 오른쪽 끝(완전 자율)에 더 가깝다. 다만 "흐름을 누가 통제하는가"(구조/architecture)와 "실제로 얼마나 똑똑한가"(AGI 여부, 지능의 수준)는 다른 축이라는 걸 구분해야 한다. 제어권을 모델에 맡긴다고 그 모델이 AGI가 되는 건 아니다. 지금의 Claude나 GPT는 특정 작업에서 강하지만, 사람처럼 완전히 새로운 종류의 문제를 스스로 정의하고 목표까지 세우는 수준(AGI)은 아니다.

## 오늘 확인한 것: 강사 코드와 내 코드를 비교하며 배운 것

- Node를 나누는 기준: "판단만 하는가, State를 바꾸는가"를 실습 2가 가장 잘 보여줬다.
- 구조화된 출력(`with_structured_output`)이 텍스트 파싱보다 안정적이다. 여러 값(result+feedback)을 동시에 받아야 할 때 특히 그렇다.
- 안전장치(재시도 제한)는 나중에 생각하는 게 아니라 처음부터 넣어야 한다. 강사님 코드도 첫 버전엔 빠뜨렸다가 나중에 추가했다.
- 같은 문제를 풀어도 State에 뭘 남길지, Node를 어디서 나눌지, 안전장치를 어떻게 설계할지는 설계자의 판단이다. "왜 이렇게 짰는지" 트레이드오프를 설명할 수 있는 게 중요하다.

## ✅ 확인 질문

1. `TypedDict`로 정의한 State는 왜 런타임에 타입을 강제하지 못하는가?
2. Node 함수가 `return {"answer": ...}`처럼 일부만 반환해도 되는 이유는 무엇인가?
3. `compile()` 전후로 객체의 클래스가 달라진다는 걸 어떻게 확인했는가? 두 클래스가 각각 어떤 메서드를 갖고 있었는가?
4. `add_edge`와 `add_conditional_edges`의 차이는 무엇이며, 후자의 매핑 dict는 왜 필요한가?
5. Node 함수와 라우팅 함수(조건부 판단 함수)를 구분하는 기준은 무엇인가?
6. `graph.stream()`의 기본 모드와 `stream_mode="values"`의 차이는 무엇인가?
7. `CountState`에서 `{"count": 10}`으로 시작하면 `increment`가 몇 번 실행되는지, 그리고 그 이유는?
8. `Annotated[list, operator.add]`를 실제로 뜯어봤을 때(`get_args`) 안에 뭐가 들어있었는가?
9. Reducer로 쓸 함수가 반드시 만족해야 하는 조건은 무엇이며, 왜 `max`는 그대로 못 썼는가?
10. dict 타입 필드에 `operator.add`를 Reducer로 쓸 수 없는 이유는?
11. `StrOutputParser`와 `with_structured_output`의 역할 차이는?
12. `PromptTemplate.from_template("...{text}...")`에서 `{text}`와 f-string의 `{text}`는 어떻게 다른가?
13. 텍스트 파싱("PASS"로 시작하는지 확인)보다 구조화된 출력이 왜 더 안정적인가? 실패 시나리오를 예로 들어 설명하라.
14. `FakeListChatModel`이 대신하는 건 "사용자 입력"인가 "LLM의 출력"인가?
15. "그래프 엔진이 없다"(Claude Agent SDK)는 게 정확히 무슨 뜻인가?
