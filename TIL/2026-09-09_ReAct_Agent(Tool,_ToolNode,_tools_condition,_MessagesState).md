---
tags: [langgraph, react, tool, messagesstate]
til: v2 2026-09-10
---

# ReAct Agent (Tool, ToolNode, tools_condition, MessagesState)
> 작성일: 2026-09-09

## 🔗 관련 글

- [2026-09-08_LangGraph_기초(State_Node_Edge와_조건부분기_반복_Reducer)](2026-09-08_LangGraph_기초%28State_Node_Edge와_조건부분기_반복_Reducer%29.md) — State/Node/Edge, 조건부 분기, Reducer의 뒷부분(오늘 그 위에 Tool 호출과 ToolNode/tools_condition을 얹음)

## ReAct 패턴이란

ReAct는 Reasoning(추론) + Acting(행동)의 줄임말이다.

기반 기술은 Function Calling(Tool Use)이다. LLM이 응답으로 텍스트 대신 "이 함수를 이 인자로 호출해줘"라는 tool_calls를 반환하면, 시스템이 해당 함수를 실행하고 결과를 다시 LLM에게 전달한다. ReAct는 이 Function Calling을 루프 안에서 반복하는 패턴이다.

```
Think   → LLM이 현재 메시지를 바탕으로 다음 행동을 결정
Act     → tool_calls 응답 → 시스템이 Tool을 실행
Observe → Tool 결과를 LLM에게 전달
(반복)
Answer  → tool_calls가 없으면 최종 답변
```

핵심은 LLM이 다음 행동을 선택한다는 점이다. 어떤 Tool을 호출할지, 몇 번 반복할지, 언제 멈출지를 매 턴마다 결정한다. 지금까지 배운 그래프(11번 노트북의 감정 분석 라우터, 숫자 맞히기)는 개발자가 갈림길을 전부 미리 정해뒀는데, ReAct는 그 갈림길 자체를 LLM이 그때그때 판단한다 — Workflow-Agent-Autonomous Agent 스펙트럼에서 "Agent" 칸의 실제 구현이다.

이때 모델의 내부 추론 과정 전체가 애플리케이션에 공개되는 것은 아니다. 애플리케이션은 tool_calls와 최종 응답만 관찰할 수 있다.

> ➕ **더 알아두기**
> 지금 이 대화(Claude Code)나 Tool을 쓰는 ChatGPT도 이 ReAct 루프 위에서 돌아간다. 다만 "Claude/GPT라는 모델"과 "Claude Code/ChatGPT라는 제품(하네스)"은 역할이 다르다.
>
> - LLM(모델) — "이 Tool을 이 인자로 불러줘"라고 판단만 한다
> - 하네스(Claude Code, Codex CLI 등) — 그 판단을 받아서 실제로 실행하고 결과를 다시 전달한다
>
> 오늘 배우는 ToolNode가 바로 이 "하네스" 역할을 LangGraph 안에서 대신해주는 부품이다.

## Tool 정의

ReAct Agent가 사용할 Tool을 정의한다. 모델은 함수 구현이 아니라 Tool 이름, docstring, 인자 타입으로 만들어진 schema를 보고 Tool과 인자를 선택한다. 따라서 Tool 설명은 용도와 입력 형식을 구체적으로 작성해야 한다.

```python
# 서버 시간 Tool
@tool(parse_docstring=True)
def get_current_time() -> str:
    """서버의 현재 로컬 날짜와 시간을 반환한다."""
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")


tools = [get_current_time]
print(f"등록된 Function Tool: {[t.name for t in tools]}")
```

`@tool` 데코레이터는 5번 노트북(Tool Agent)에서 배운 것을 그대로 응용한 것이다. 새로 짚을 부분만 정리하면 이렇다.

- `@tool(parse_docstring=True)` — 평범한 함수를 LangChain이 이해하는 Tool 객체로 변환한다. `parse_docstring=True`는 아래 docstring을 이 Tool의 description으로 자동으로 가져오라는 뜻이다.
- `def get_current_time() -> str:` — 매개변수가 없는 함수다(빈 괄호 = 입력값이 필요 없다는 뜻). `-> str`은 반환 타입 힌트로, 매개변수 옆에 `: 타입`을 붙이는 것과 위치만 다르고 원리는 같다. 실행에는 영향 없고, Tool의 schema(입력·출력 형식)에 포함되는 정보다.
- `datetime.now().strftime("%Y-%m-%d %H:%M:%S")` — `datetime.now()`로 현재 시각 객체를 만들고, `.strftime(...)`(string format time)으로 원하는 형식의 문자열로 바꾼다. `%Y`(연도)/`%m`(월)/`%d`(일)/`%H:%M:%S`(시:분:초) 형식 지정자를 조합한다.
- `tools = [get_current_time]` — 이 Agent가 쓸 수 있는 Tool들을 리스트로 모아둔다. 나중에 Tool을 추가하면 이 리스트에 계속 추가한다.
- `[t.name for t in tools]` — 리스트 컴프리헨션. `tools`를 하나씩 꺼내 `.name`(그 Tool의 이름)만 뽑아 새 리스트를 만든다.

Tool이 실제로 선택되는 원리: LLM은 `datetime.now().strftime(...)` 같은 실제 코드는 전혀 못 보고, 오직 이름(`get_current_time`)과 설명(`"서버의 현재 로컬 날짜와 시간을 반환한다"`)만 보고 판단한다. 사용자가 "지금 몇 시야?"라고 물으면, LLM이 그 문장의 의미와 Tool 설명을 의미적으로 비교해서 "이 도구를 쓰면 되겠다"고 스스로 추론한다. 이건 코드로 짜인 규칙(`if "시간" in question: ...`)이 아니라 LLM 내부의 판단이다.

- 코드가 하는 일 — 도구 목록과 설명(schema)을 LLM에게 넘겨주는 것뿐
- LLM이 하는 일 — 그 설명들을 읽고, 지금 상황에 어떤 도구가 맞는지 스스로 추론

> ➕ **더 알아두기**
> Tool이 여러 개일수록 각 도구의 설명(docstring)이 서로 겹치지 않고 명확해야 한다. `get_current_time`과 `get_current_date`처럼 설명이 애매하게 겹치는 도구를 만들면, LLM이 어느 걸 써야 할지 헷갈려서 엉뚱한 도구를 호출하거나 아예 안 부르는 문제가 생긴다.

## bind_tools — Tool을 쓸 수 있는 새 llm 만들기

```python
llm = ChatGoogleGenerativeAI(model=MODEL_NAME)

# 직접 정의한 Function Tool 바인딩
llm_with_tools = llm.bind_tools(tools)
```

`llm.bind_tools(tools)`는 11번 노트북에서 배운 `llm.with_structured_output(ReviewResult)`와 똑같은 패턴이다. 원래 `llm`을 바꾸는 게 아니라, "도구를 쓸 수 있게 된 새 llm 객체"를 따로 만들어서 `llm_with_tools`라는 새 변수에 담는다.

- `llm` — 그냥 텍스트만 생성하는 원래 모델
- `llm_with_tools` — `tools` 리스트에 있는 도구들의 schema를 알고 있어서, 필요하면 텍스트 대신 tool_calls를 응답으로 낼 수 있는 모델

다만 `with_structured_output`은 "답변 형식을 강제"했던 반면, `bind_tools`는 강제하는 게 아니라 "이런 도구들이 있으니 필요하면 써도 된다"고 선택지를 열어주는 것이다. LLM은 그 상황에서 도구를 쓸지 말지, 쓴다면 어떤 도구를 쓸지를 스스로 판단한다.

## MessagesState — 대화 메시지를 담는 미리 만들어진 State

```python
class MessagesState(TypedDict):
    messages: Annotated[list, add_messages]
```

이건 완전히 새로운 개념이 아니라, 어제 배운 Reducer(`Annotated[list, add_messages]`)를 그대로 응용한 것이다. 저희가 직접 `class MyState(TypedDict): messages: Annotated[list, add_messages]`라고 만들 수도 있었는데, 이 패턴이 대화형 Agent에서 너무 흔히 쓰여서 LangGraph가 아예 이름까지 붙여서 미리 만들어둔 것뿐이다.

`add_messages`가 Reducer로 붙어 있어서, 각 노드가 반환한 메시지가 기존 State를 덮어쓰지 않고 누적된다. 대부분의 챗봇과 Agent는 이 State 하나만으로 충분하다 — 사용자 메시지, AI 메시지, Tool 메시지가 다 이 `messages` 리스트 안에 순서대로 쌓인다.

## ToolNode — Tool을 실제로 실행해주는 미리 만들어진 Node

ToolNode는 LLM의 tool_calls를 받아서 실제 함수를 실행해주는 노드다. "LLM은 판단만 하고, 실제 실행은 다른 것이 한다"는 구분이 여기서도 적용된다 — ToolNode가 바로 그 "실제로 실행하는" 역할을 하는 노드다.

내부적으로 이런 일을 자동으로 처리한다.

1. LLM 응답에서 tool_calls를 파싱한다(함수 이름 + 인자)
2. 해당 함수를 실행한다
3. 결과를 ToolMessage로 감싸서 State에 추가한다

직접 손으로 짜면 이런 함수가 된다.

```python
def my_tool_node(state: MessagesState):
    response = state["messages"][-1]      # 마지막 메시지(AI의 tool_calls) 꺼내기
    results = []
    for tool_call in response.tool_calls:
        tool_fn = tool_map[tool_call["name"]]
        result = tool_fn.invoke(tool_call["args"])
        results.append(ToolMessage(content=result, tool_call_id=tool_call["id"]))
    return {"messages": results}
```

- `state["messages"][-1]` — 메시지 리스트에서 가장 마지막(최신) 메시지를 꺼낸다. `[-1]`은 "뒤에서 첫 번째"를 뜻하는 인덱싱이다.
- `tool_map` — Tool 이름(문자열) → 실제 Tool 객체를 매핑해둔 딕셔너리. `tool_map = {t.name: t for t in tools}`처럼 `tools` 리스트로부터 미리 만들어둔다. 예를 들어 `tools = [get_current_time]`이면 `tool_map = {"get_current_time": get_current_time}`이 된다.
- `tool_call["name"]` — 우리가 코드에 직접 적은 문자열이 아니라, LLM이 응답할 때 실어 보낸 데이터다. LLM이 "get_current_time을 불러줘"라고 판단하면, 그 응답 안에 문자열 `"get_current_time"`이 실려서 온다.
- `tool_fn.invoke(tool_call["args"])` — `tool_map`에서 찾은 실제 도구 함수를 진짜로 실행하는 줄이다. `my_tool_node`(바깥 함수)가 `tool_fn`(안쪽 함수, 실제 도구)을 호출하는 구조다 — "다른 함수를 실행하는 함수"인 셈이다.

이 코드는 도구 이름을 코드에 직접 하드코딩하지 않아서, `tools` 리스트에 도구가 몇 개든 이 함수 하나로 다 처리된다. 새 도구를 추가해도 이 함수는 안 바뀐다.

```python
graph_builder.add_node("tools", ToolNode(tools))
```

`ToolNode(tools)`는 위 `my_tool_node` 함수 전체를 대신해준다. `ToolNode`는 클래스이고, `ToolNode(tools)`는 그 클래스의 인스턴스(객체)를 만드는 것이다. 우리가 `def`로 함수를 안 만들어도, LangGraph가 이미 만들어둔 "도구 실행 전용 노드"를 그대로 가져다 쓴다.

한 가지 참고할 점: AI 메시지 하나에 tool_calls가 여러 개 들어있을 수도 있다(예: "지금 몇 시야? 그리고 오늘 날씨는?"처럼 한 번에 두 도구가 필요한 경우). 이런 경우들끼리 서로 영향을 주지 않도록, Tool은 전역 상태를 함부로 바꾸지 않게 설계해야 한다.

## tools_condition — 도구 필요 여부로 분기하는 미리 만들어진 판단 함수

tools_condition은 LLM 응답에 tool_calls가 있으면 "tools" 노드로, 없으면 END로 보내는 라우팅(판단) 함수다. 이건 완전히 새로운 개념이 아니라, 11번 노트북에서 만든 `route_by_language`, `route_by_sentiment`, `route_by_count`와 같은 역할의 함수를 그대로 응용한 것이다 — State를 읽어서 문자열만 반환하고 State는 안 건드린다. 다른 점은 저희가 직접 `def`로 만들지 않고, `from langgraph.prebuilt import ToolNode, tools_condition`으로 LangGraph가 미리 만들어둔 걸 가져다 쓴다는 것뿐이다.

직접 손으로 짜면 이런 모양이다.

```python
def should_continue(state: MessagesState):
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tools"
    return END
```

지금까지 배운 걸 표로 정리하면 이렇다.

| 수동 구현 | LangGraph |
|---|---|
| `if response.tool_calls` 분기 | `tools_condition` 조건부 엣지 |
| Tool 실행 + `ToolMessage` 추가 | `ToolNode`가 자동 처리 |
| `messages` 리스트 직접 관리 | `MessagesState`가 자동 관리 |

## 그래프 조립

```python
def chatbot(state: MessagesState):
    """LLM을 호출하는 노드. Tool 호출이 필요하면 tool_calls가 포함된 메시지를 반환한다."""
    return {"messages": [llm_with_tools.invoke(state["messages"])]}


# 그래프 구성
graph_builder = StateGraph(MessagesState)

# 노드 등록
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", ToolNode(tools))

# 엣지 연결
graph_builder.add_edge(START, "chatbot")
graph_builder.add_conditional_edges("chatbot", tools_condition)
graph_builder.add_edge("tools", "chatbot")

graph = graph_builder.compile()
```

`chatbot` 함수는 지금까지 배운 Node 패턴 그대로다(기존 개념의 응용) — `state`를 받아서, 처리하고, dict를 반환한다. 다른 점은 두 가지다.

- `llm_with_tools.invoke(state["messages"])` — 예전엔 `llm.invoke(state["question"])`처럼 문자열 하나를 넘겼는데, 이번엔 `state["messages"]`(지금까지 쌓인 메시지 리스트 전체)를 통째로 넘긴다. 대화형 Agent는 이전 대화 맥락을 다 알아야 하기 때문이다.
- `{"messages": [llm_with_tools.invoke(...)]}` — 반환값이 리스트 안에 감싸져 있다. `messages` 필드가 Reducer(`add_messages`)를 가지고 있어서, 새 메시지 하나를 리스트에 담아 반환하면 기존 리스트 뒤에 자동으로 이어붙는다.

`llm_with_tools.invoke(...)`의 결과가 뭐가 될지는 그때그때 다르다 — LLM이 도구가 필요하다고 판단하면 tool_calls가 담긴 메시지가, 필요 없다고 판단하면 평범한 텍스트 답변이 나온다. 어느 쪽이든 이 함수는 그 결과를 그대로 `messages`에 추가만 한다.

### `add_node("tools", ToolNode(tools))`도 함수를 등록만 하는 것이다

`add_node("이름표", 함수)`에서 함수 이름 뒤에 괄호가 없으면(`함수()`가 아니라 `함수`), "지금 당장 실행해라"가 아니라 "이 함수 자체를 나중에 쓸 수 있게 넘겨준다"는 뜻이다.

- `add_node("tools", ToolNode(tools))` — 함수를 등록만 함(아직 실행 안 됨)
- 나중에 그래프 실행 중 LangGraph가 `ToolNode(tools)`를 `state`와 함께 호출 — 이때 실제로 실행됨

### add_conditional_edges에서 매핑 dict가 생략된 이유

```python
graph_builder.add_conditional_edges("chatbot", tools_condition)
```

`add_conditional_edges(출발, 판단함수, 매핑dict)`에서 매핑dict의 역할은 "판단 함수가 반환한 값을 실제 노드 이름으로 바꿔주는 것"이었다. 매핑dict가 있는 경우와 없는 경우를 나란히 비교하면 이렇다.

```python
# 매핑dict 있음 (route_by_sentiment)
add_conditional_edges("analyze", route_by_sentiment, {"positive": "respond_positive", "negative": "respond_negative"})
#                                                      ^반환값       ^실제 노드 이름 → 서로 다름 → dict로 연결해줘야 함

# 매핑dict 없음 (tools_condition)
add_conditional_edges("chatbot", tools_condition)
#                                 ^tools_condition은 "tools" 또는 END를 반환하는데,
#                                  이 값 자체가 이미 실제 노드 이름이라 연결해줄 게 없음
```

즉 매핑dict가 필요한 이유는 "판단함수가 반환한 값"과 "실제 노드 이름"이 다를 때 그 둘을 이어주기 위해서다. `tools_condition`은 반환값 자체가 이미 노드 이름과 똑같아서(`"tools"`를 반환하면 그게 곧 `"tools"`라는 노드 이름) 이어줄 게 없어서 생략된다.

- 매핑dict 있는 경우 — 판단함수의 반환값 ≠ 실제 노드 이름
- 매핑dict 없는 경우 — 판단함수의 반환값 = 실제 노드 이름 (`tools_condition`이 이 경우)

명시적으로 쓰면 이런 모양이 된다.

```python
graph_builder.add_conditional_edges("chatbot", tools_condition, {
    "tools": "tools",
    END: END,
})
```

### `add_edge`와 `add_conditional_edges`가 각각 몇 번 불리는지는 "출발 노드 개수"로 정해진다

어제(11번 노트북) 만든 반복 그래프를 다시 보면 이렇다.

```python
count_builder.add_conditional_edges(
    "increment",
    route_by_count,
    {"continue": "increment", "end": END},   # 조건부 Edge로 자기 자신에게 되돌아감
)
```

노드가 `increment` 하나뿐이라, "increment 다음"이라는 갈림길 하나(그 안에 도착지가 두 곳: `increment` 자신 또는 `END`)로 전부 처리됐다.

오늘은 노드가 두 개(`chatbot`, `tools`)라서 "다음에 어디로 갈지"를 각 노드마다 따로 정의해야 한다.

```python
graph_builder.add_conditional_edges("chatbot", tools_condition)   # "chatbot 다음"이라는 갈림길
graph_builder.add_edge("tools", "chatbot")                          # "tools 다음"이라는 별개의 연결
```

"갈림길(조건 판단)"은 `chatbot` 쪽에서만 일어난다. `tools` 쪽은 조건 없이 무조건 `chatbot`으로 돌아가는 고정 Edge다 — 도구를 실행했으면 그 결과를 무조건 다시 LLM에게 보고해야 하니, 여기엔 판단할 게 없다.

- 어제 — 노드 하나가 조건부로 자기 자신에게 돌아감
- 오늘 — 노드 두 개가 번갈아 돌되, 판단은 한쪽(`chatbot`)에서만, 복귀는 고정 Edge로

> ➕ **더 알아두기**
> 이 구조 덕분에 반복 횟수 제한 없이도 자연스럽게 끝난다. `chatbot`이 "더 이상 도구 필요 없다"고 판단하는 순간 `tools_condition`이 `END`를 반환해서 멈추기 때문에, 어제처럼 `attempts >= N` 같은 안전장치를 따로 안 넣어도 LLM 스스로 멈출 지점을 판단하는 구조다(다만 LLM이 무한히 도구를 요청하는 극단적 상황엔 여전히 안전장치가 필요할 수 있다).

## 실행

```python
question = "지금 몇 시야?"

result = graph.invoke({"messages": [("human", question)]})

for msg in result["messages"]:
    print(f"[{msg.type}] {msg.text[:300]}")
    if hasattr(msg, "tool_calls") and msg.tool_calls:
        print(f"  -> Tool 호출: {[tc['name'] for tc in msg.tool_calls]}")
    print()
```

`graph.invoke({"messages": [("human", question)]})`는 어제 배운 `invoke()`를 그대로 쓰는 것이다(기존 개념의 응용). 다만 넣는 값의 모양이 State 정의에 따라 달라진다 — 예전 `State`는 `question`/`answer` 필드를 가졌으니 `{"question": "..."}`을 넣었고, 오늘 `MessagesState`는 `messages` 필드 하나뿐이니 `{"messages": [...]}`을 넣는다. **State가 어떤 필드를 갖고 있느냐에 따라, invoke()에 넣는 dict의 키 이름도 거기 맞춰서 정해진다.**

- `("human", question)` — (역할, 내용) 형태의 튜플. `"human"`은 이 메시지를 보낸 주체가 사람이라는 표시고, `question`은 실제 내용이다. LangChain이 이 튜플을 자동으로 `HumanMessage` 객체로 바꿔준다.
- `result["messages"]` — 그래프 실행이 끝난 뒤(END까지 도달한 뒤) 최종 State에서 `messages` 리스트를 꺼낸다. 이 리스트 안에는 사용자 질문, AI가 도구를 부르겠다고 요청한 메시지(tool_calls 포함), 도구 실행 결과 메시지, 최종 답변까지 순서대로 다 쌓여 있다. `invoke()`는 그래프가 다 끝난 뒤에야 결과를 돌려주므로, 이 시점엔 이미 최종 답변까지 포함돼 있다.
- `msg.text[:300]` — `[:300]`은 슬라이싱이다. 문자열의 처음부터 300번째 글자까지만 잘라낸다. AI 답변이 길 수도 있으니 콘솔에 다 쏟아지지 않게 앞부분만 보여주려는 것이다.

## 오늘 배운 것 요약

| 개념 | 역할 |
|---|---|
| Tool | LLM이 부를 수 있는 함수 (schema로만 판단됨) |
| bind_tools | LLM에 도구 선택지를 열어주는 새 객체 생성 |
| MessagesState | 메시지를 누적 저장하는 미리 만들어진 State |
| ToolNode | tool_calls를 실제로 실행하는 미리 만들어진 Node |
| tools_condition | 도구 필요 여부로 분기하는 미리 만들어진 판단 함수 |

## ✅ 확인 질문

1. ReAct의 Think-Act-Observe-Answer 루프에서, LLM이 담당하는 부분과 하네스(시스템)가 담당하는 부분은 각각 무엇인가?
2. LLM이 어떤 Tool을 호출할지 판단할 때, 실제로 보는 정보는 무엇인가? (함수 코드인가, 다른 것인가)
3. `llm.bind_tools(tools)`는 원래 `llm`을 바꾸는가, 새 객체를 만드는가? 11번 노트북의 어떤 개념과 같은 패턴인가?
4. `MessagesState`가 새로운 개념이 아니라 어떤 기존 개념의 응용인지 설명하라.
5. `ToolNode(tools)`를 직접 손으로 구현하면 어떤 코드가 되는지, `tool_map`은 어디서 오는지 설명하라.
6. `add_conditional_edges`에서 매핑 dict를 생략할 수 있는 조건은 무엇인가?
7. 어제(11번)의 반복 그래프는 노드가 몇 개였고, 오늘(12번)의 ReAct 그래프는 노드가 몇 개인가? 이 차이가 `add_edge`/`add_conditional_edges` 호출 횟수에 어떤 영향을 주는가?
8. `graph.invoke({"messages": [...]})`에서 왜 키가 `"question"`이 아니라 `"messages"`여야 하는가?
