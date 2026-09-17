---
tags: [langgraph, subgraph, multi-agent, supervisor, state-schema]
til: v2 2026-09-17
---

# 2026-09-17_Subgraph와_Supervisor(재사용_MultiAgent_구현)
> 작성일: 2026-09-17

## 🔗 관련 글

- [LangGraph 기초(State, Node, Edge와 조건부분기, 반복, Reducer)](2026-09-08_LangGraph_기초(State_Node_Edge와_조건부분기_반복_Reducer).md) — Subgraph도 결국 StateGraph/Node/Edge로 만들어진 것을 그대로 다시 조립해서 쓰는 것이다.
- [ReAct Agent(Tool, ToolNode, tools_condition, MessagesState)](2026-09-09_(1)_ReAct_Agent(Tool,_ToolNode,_tools_condition,_MessagesState).md) — Supervisor도 내부적으로는 이 ReAct 방식(Tool 호출 → 관찰 → 다음 행동 결정)을 그대로 사용한다.

## 주제 1: Subgraph

### Subgraph란

LangGraph에서는 별도로 만든 Graph를 부모 Graph의 Node로 등록할 수 있다. 이렇게 다른 Graph에 포함된 Graph를 Subgraph라 한다.

### Subgraph를 쓰는 이유

- 재사용: 미리 만든 그래프를 여러 워크플로우의 노드로 활용할 수 있다.
- 개발·관리 용이: 복잡한 흐름을 역할별 그래프로 나누어 구현하고 테스트할 수 있다. 부모 그래프에서는 전체 작업 순서에 집중한다.
- State 분리: Subgraph가 내부 작업 데이터를 별도로 관리하고, 부모와 필요한 입력·출력만 주고받도록 구성할 수 있다.

단순한 작업은 일반 노드로도 충분하다. Subgraph는 여러 단계로 구성된 작업 흐름을 하나의 단위로 묶어 관리하거나 재사용할 때 장점이 크다.

### 케이스 1 — 부모와 Subgraph가 같은 State 스키마를 공유하는 경우

부모와 Subgraph의 State 스키마가 같으면, 컴파일된 그래프를 그대로 `add_node`에 전달하면 된다. 노드가 반환하는 값은 State의 변경분이며, 반환하지 않은 필드는 기존 값을 유지한다.

```python
# 공통 State: 부모와 두 Subgraph가 모두 이 스키마를 그대로 쓴다
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]   # 사용자 요청과 최종 보고서
    research: str                              # 리서치 결과 (리서치 Subgraph가 채우고, 작성 Subgraph가 읽음)


# 리서치 Subgraph
def research_start_node(state: AgentState):
    print("리서치 작업을 시작합니다.")
    return {}


def research_node(state: AgentState):
    system = SystemMessage(content="너는 리서치 전문가다. 주어진 주제의 핵심 사실 3개를 정리해.")
    messages = [system] + state["messages"]
    response = llm.invoke(messages)
    return {"research": response.text}

research_builder = StateGraph(AgentState)
research_builder.add_node("research_start", research_start_node)
research_builder.add_node("research", research_node)
research_builder.add_edge(START, "research_start")
research_builder.add_edge("research_start", "research")
research_builder.add_edge("research", END)
research_graph = research_builder.compile()


# 작성 Subgraph
def writing_node(state: AgentState):
    system = SystemMessage(content="너는 보고서 작성 전문가다. 사용자 요청과 조사 결과를 바탕으로 간결한 보고서를 작성해.")
    request = HumanMessage(content=f"다음 조사 결과로 보고서를 작성해.\n\n{state['research']}")
    messages = [system] + state["messages"] + [request]
    response = llm.invoke(messages)
    return {"messages": [response]}

writing_builder = StateGraph(AgentState)
writing_builder.add_node("write", writing_node)
writing_builder.add_edge(START, "write")
writing_builder.add_edge("write", END)
writing_graph = writing_builder.compile()


# 부모 그래프: 컴파일된 Subgraph를 그대로 노드로 등록한다
def prepare_node(state: AgentState):
    print("요청 처리를 시작합니다.")
    return {}

parent_builder = StateGraph(AgentState)
parent_builder.add_node("prepare", prepare_node)
parent_builder.add_node("researcher", research_graph)   # 컴파일된 그래프 자체를 Node로 등록
parent_builder.add_node("writer", writing_graph)
parent_builder.add_edge(START, "prepare")
parent_builder.add_edge("prepare", "researcher")
parent_builder.add_edge("researcher", "writer")
parent_builder.add_edge("writer", END)
parent_graph = parent_builder.compile()
```

### Subgraph 실행 과정 스트리밍 (subgraphs=True/False)

- 기본값인 `subgraphs=False`에서는 부모 그래프의 노드 결과만 받는다. 이때 Subgraph는 부모 그래프의 노드 하나로만 표시된다.
- `subgraphs=True`를 지정하면 부모 그래프뿐 아니라 내부 Subgraph의 노드에서 발생한 결과도 함께 받는다.
- `subgraphs=False`에서는 `chunk`를 반환하고, `subgraphs=True`에서는 `(namespace, chunk)`를 반환한다.
  - `namespace`: 이벤트가 발생한 그래프의 경로
  - `chunk`: 해당 실행 단계에서 변경된 State (`stream_mode="updates"`에서는 `{노드 이름: 변경된 State}`)

```python
# 내부 실행 제외 (기본값)
for chunk in parent_graph.stream(
    {"messages": [HumanMessage(content="LangGraph의 Subgraph 기능에 대해 정리해줘.")]},
    subgraphs=False,
    stream_mode="updates",
):
    ...

# 내부 실행 포함
for namespace, chunk in parent_graph.stream(
    {"messages": [HumanMessage(content="LangGraph의 Subgraph 기능에 대해 정리해줘.")]},
    subgraphs=True,
    stream_mode="updates",
):
    ...
```

> **Q. 노드끼리 어차피 같은 State를 쓰는데, Subgraph를 하나 더 돌리면 모니터링만 더 힘들어지는 거 아닌가?**
> 이 토이 예제 자체는 Subgraph가 굳이 필요하지 않다는 지적은 맞다 (노드 3개짜리 단순한 흐름이라 일반 Node로만 짜도 충분함). 다만 "Subgraph를 쓰면 모니터링이 무조건 더 어려워진다"는 부분은 정정이 필요하다.
>
> `subgraphs=True` / `subgraphs=False`는 **내부 실행 과정을 보여줄지 말지 선택하는 옵션**일 뿐이다. 즉 Subgraph 내부가 원래 안 보이는 게 아니라, 보고 싶으면 `subgraphs=True`로 언제든 펼쳐볼 수 있다. 모니터링이 본질적으로 더 어려워지는 게 아니라, 평소엔 접어두고(부모 노드 하나로만 보이게) 필요할 때만 펼쳐보는 **선택적 가시성** 문제에 가깝다.

### 케이스 2 — 부모와 Subgraph의 State가 다른 경우

부모와 Subgraph가 주고받을 데이터의 필드 이름이나 형식이 다르면, 컴파일된 Subgraph를 바로 `add_node`에 넣을 수 없다. 이때는 **래퍼(wrapper) 함수**를 만들어서, 그 함수가 부모 State를 Subgraph 입력 형태로 변환해 Subgraph를 직접 `invoke()`하고, 결과를 다시 부모 State 형태로 변환해서 반환한다.

- 필드 이름이 다른 경우: 부모의 `analysis_result`와 Subgraph의 `analysis`처럼 같은 데이터를 다른 이름으로 저장할 때
- 데이터 형식이 다른 경우: 부모는 `messages`에 메시지 목록을 저장하지만, Subgraph는 `request`에 요청 문자열만 받을 때

```python
# Subgraph 전용 State (부모와 다른 스키마)
class AnalysisState(TypedDict):
    request: str
    analysis: str

def analysis_node(state: AnalysisState):
    system = SystemMessage(content="사용자가 요청한 주제를 분석하여 핵심 쟁점 3개와 각각의 장단점을 정리해.")
    response = llm.invoke([system, HumanMessage(content=state["request"])])
    return {"analysis": response.text}

analysis_builder = StateGraph(AnalysisState)
analysis_builder.add_node("analyze", analysis_node)
analysis_builder.add_edge(START, "analyze")
analysis_builder.add_edge("analyze", END)
analysis_graph = analysis_builder.compile()


# 부모 State
class ParentState(TypedDict):
    messages: Annotated[list, add_messages]
    analysis_result: str

# 래퍼 함수: 부모 State -> Subgraph State로 변환해서 invoke -> 결과를 부모 State로 변환
def analysis_wrapper(state: ParentState):
    sub_input = {"request": state["messages"][-1].text}   # 부모의 마지막 메시지를 Subgraph 입력 형식으로 변환
    sub_result = analysis_graph.invoke(sub_input)           # Subgraph를 직접 invoke (add_node에 그래프 자체를 넣지 않음)
    return {"analysis_result": sub_result["analysis"]}      # Subgraph 결과를 부모 필드 이름으로 변환

def report_node(state: ParentState):
    system = SystemMessage(content="분석 결과를 바탕으로 간결한 보고서를 작성해.")
    response = llm.invoke([system, HumanMessage(content=state["analysis_result"])])
    return {"messages": [response]}

parent_builder = StateGraph(ParentState)
parent_builder.add_node("analysis", analysis_wrapper)   # 컴파일된 그래프가 아니라 래퍼 함수를 노드로 등록
parent_builder.add_node("report", report_node)
parent_builder.add_edge(START, "analysis")
parent_builder.add_edge("analysis", "report")
parent_builder.add_edge("report", END)
diff_state_graph = parent_builder.compile()
```

> **Q. import해서 Subgraph를 다시 불러온다는 게, 여러 파일에서 재사용한다는 뜻이야?**
> 그렇다. `research_graph`, `writing_graph`처럼 컴파일까지 끝난 Graph 객체는 일반 Python 객체이므로, 다른 파일에서 `from ... import research_graph`처럼 불러와서 다른 부모 Graph의 Node로 재사용할 수 있다.
>
> **Q. 그럼 같은 부모 그래프에서는?**
> 같은 파일 안에서도 위 코드처럼 `parent_builder.add_node("researcher", research_graph)`, `add_node("writer", writing_graph)`처럼 여러 개의 컴파일된 Subgraph를 같은 부모 Graph에 등록해서 함께 쓸 수 있다.
>
> **Q. 같은 부모라는 게 클래스 상속 같은 건가, 아니면 같은 파일 내에서 State를 공유한다는 뜻인가?**
> 클래스 상속이 아니다. 상속은 "부모 클래스의 속성/메서드를 자식이 그대로 물려받는 것"이지만, Subgraph 관계는 **포함(Composition)** 관계에 가깝다 — 부모 Graph가 자식 Graph를 하나의 부품(Node)으로 갖고 있는 것이다.
>
> "같은 부모 그래프에서 State를 공유한다"는 건 두 가지 경우로 나뉜다.
> 1. 케이스 1처럼 **부모와 Subgraph가 같은 TypedDict 스키마(AgentState)를 그대로 쓰는 경우** — 이때는 정말로 같은 키를 공유해서 값을 주고받는다.
> 2. 케이스 2처럼 **부모와 Subgraph의 State 스키마 자체가 다른 경우** — 이땐 State를 공유하지 않고, 래퍼 함수가 값을 변환해서 전달해준다.

> ➕ **더 알아두기 — Subgraph 상태 매핑, 사실은 세 번째 경우도 있다**
> 수업에서는 "같은 스키마" / "다른 스키마(래퍼 필요)" 두 경우만 다뤘는데, LangGraph 공식 문서 기준으로는 이렇게 세 가지로 나뉜다.
> 1. 부모와 Subgraph가 **완전히 같은 State 스키마**를 쓰는 경우 → 컴파일된 Subgraph를 그대로 `add_node`에 전달.
> 2. 부모와 Subgraph의 State 스키마가 다르지만 **일부 키를 공유**하는 경우 → 공유된 키에 한해 변환 없이 그대로 주고받을 수 있다.
> 3. 부모와 Subgraph가 **공유하는 키가 하나도 없는 경우** → 수업에서 다룬 것처럼 반드시 래퍼 함수(Node 함수)에서 부모 State → Subgraph State로 변환한 뒤 Subgraph를 `invoke()`하고, 결과를 다시 부모 State로 변환해서 반환해야 한다. 공유 키가 없는 Subgraph를 변환 없이 바로 `add_node`에 넣으면 LangGraph가 오류를 낸다.
>
> State를 공유하면 구현이 간단하고 값이 자동으로 오가지만, 다른 Agent에 노출되면 안 되는 내부 정보가 있다면(예: 특정 Agent만 봐야 하는 중간 추론 과정) State를 완전히 분리하고 경계에서 명시적으로 변환하는 쪽이 더 안전하다 — [LangChain 공식 Subgraph 문서](https://docs.langchain.com/oss/python/langgraph/use-subgraphs) 기준.

## 주제 2: Supervisor (Multi-Agent)

### Multi-Agent란

Multi-Agent 시스템은 하나의 Agent가 모든 작업을 처리하는 대신, 서로 다른 역할을 맡은 여러 Agent가 협력하여 하나의 목표를 달성하는 구조이다. 각 Agent는 역할에 맞는 Prompt, Tool, Context를 독립적으로 가질 수 있으며, Agent 사이의 작업 전달과 결과 공유를 통해 복잡한 작업을 나누어 처리한다.

### Multi-Agent를 고려하는 이유

하나의 Agent가 너무 많은 역할, Tool, Context를 담당하면 지시 충돌과 Tool 선택 오류가 늘어날 수 있다. 역할별 Agent로 분리하면 각 Agent가 자신의 작업에 필요한 지시와 정보에 집중할 수 있다.

| 이점 | 설명 |
|------|------|
| 역할과 Context 분리 | 각 Agent가 담당 작업에 필요한 정보에 집중 |
| Tool과 권한 분리 | Agent별로 필요한 Tool만 제공 |
| 독립적인 설계와 평가 | 역할마다 Prompt, 모델, 평가 기준을 다르게 설정 |
| 동적인 작업 위임 | 요청과 중간 결과에 따라 적절한 Agent를 선택 |

역할이 단순하고 실행 순서가 고정되어 있다면 단일 Agent나 일반 Workflow가 더 적합하다. Multi-Agent는 모델 호출 수, 지연 시간, 비용과 실패 지점을 증가시키므로 역할 분리와 동적 위임의 이점이 충분할 때 사용한다.

### Supervisor 패턴

Supervisor 패턴은 중앙의 Supervisor가 작업을 분석하여 적절한 Subagent에 분배하고, 각 결과를 종합해 최종 응답을 만드는 구조이다.

```text
사용자 → Supervisor Agent
             ├─→ Research Agent
             ├─→ Writer Agent
             └─→ Reviewer Agent
```

### Supervisor 구현

보고서 작성 과정을 자료 조사, 작성, 검토로 나눈다. `create_agent`로 만든 각 Subagent를 `@tool` 함수로 감싸 Supervisor의 Tool로 등록한다. 각 Tool은 작업 설명을 Subagent에 전달하고 마지막 응답만 Supervisor에 반환한다. Checkpointer는 전체 대화를 관리하는 최상위 Supervisor에만 설정한다.

```python
# Subagent 세 개를 각자 역할에 맞는 system_prompt로 만든다
researcher_agent = create_agent(
    model=search_llm,   # Google Search Tool이 bind된 모델
    tools=[],
    system_prompt=(
        "너는 자료 조사 전문가다. 요청받은 주제를 Google Search로 조사해. "
        "보고서 작성에 필요한 핵심 사실과 근거를 정리하고, 각 내용의 출처를 함께 제시해."
    ),
    name="researcher",
)

writer_agent = create_agent(
    model=llm,
    tools=[],
    system_prompt=(
        "너는 보고서 작성 전문가다. 전달받은 조사 결과만을 근거로 보고서를 작성해. "
        "보고서는 제목, 개요, 본문, 결론 순서로 구성하고 출처를 유지해."
    ),
    name="writer",
)

reviewer_agent = create_agent(
    model=llm,
    tools=[],
    system_prompt=(
        "너는 보고서 검토 전문가다. 전달받은 보고서의 구조, 가독성, 내용의 일관성, "
        "출처 표기를 검토해. 수정이 필요하면 구체적인 수정 사항을 제시하고, "
        "문제가 없으면 '승인'이라고 명시해."
    ),
    name="reviewer",
)


# 각 Subagent를 Supervisor가 부를 수 있는 Tool로 감싼다
@tool
def research(query: str) -> str:
    """주제에 관한 최신 자료와 출처를 조사한다."""
    result = researcher_agent.invoke({"messages": [{"role": "user", "content": query}]})
    return result["messages"][-1].text


@tool
def write_report(query: str) -> str:
    """조사 결과를 바탕으로 보고서를 작성한다."""
    result = writer_agent.invoke({"messages": [{"role": "user", "content": query}]})
    return result["messages"][-1].text


@tool
def review_report(query: str) -> str:
    """보고서를 검토하고 수정 사항이나 승인 여부를 반환한다."""
    result = reviewer_agent.invoke({"messages": [{"role": "user", "content": query}]})
    return result["messages"][-1].text


# Supervisor: 세 Tool을 갖고 있고, Checkpointer는 여기(최상위)에만 설정한다
supervisor_agent = create_agent(
    model=llm,
    tools=[research, write_report, review_report],
    system_prompt=(
        "너는 보고서 작성 팀의 Supervisor다. "
        "사용자의 요청을 분석하고 필요한 작업을 적절한 전문가에게 위임해. "
        "작업을 위임할 때는 해당 전문가에게 필요한 맥락을 충분히 전달해. "
        "전문가의 결과를 검토하고 필요한 경우 추가 작업을 요청해. "
        "근거와 출처가 포함된 완성도 높은 보고서를 최종 응답으로 제공해. "
        "중간 작업 과정은 사용자에게 설명하지 마."
    ),
    checkpointer=InMemorySaver(),
)
```

### 실행

```python
config = {"configurable": {"thread_id": "report-session-1"}}

for event in supervisor_agent.stream(
    {"messages": [HumanMessage(content="AI Agent 기술 동향에 대한 보고서를 작성해줘.")]},
    config=config,
    stream_mode="updates",
):
    for node_name, state in event.items():
        message = state["messages"][-1]
        print("=" * 60)
        print(f"Node: {node_name}")
        if getattr(message, "tool_calls", None):
            print(f"Tool calls: {[call['name'] for call in message.tool_calls]}")
        if message.text:
            print(message.text[:500])
```

### Supervisor의 동작 방식

이 예제의 Supervisor도 필요한 작업을 판단하고, Tool을 실행하고, 결과를 관찰해 다음 행동을 결정하는 ReAct 방식으로 동작한다. 여기서 Tool은 독립적인 지시와 Context를 가진 Subagent를 호출한다. 예를 들어 자료 조사가 필요하면 `research`를 호출하고, 반환된 결과를 바탕으로 보고서 작성이나 검토를 위임한 뒤 최종 응답을 만든다.

> **Q. 이 Supervisor는 새로운 실행 방식인가?**
> 아니다. ReAct(9월 9일에 배운 Tool 호출 → 관찰 → 다음 행동 결정 반복)를 그대로 쓴다. 다른 점은 Tool이 평범한 함수가 아니라, **독립적인 System Prompt와 Context를 가진 또 다른 Agent(`create_agent`로 만든 Subagent)를 호출한다는 것**뿐이다.

> ➕ **더 알아두기 — Supervisor 외에 다른 Multi-Agent 구조도 있다**
> 수업에서는 Router(15번)와 Supervisor(오늘)만 다뤘는데, LangGraph가 소개하는 대표적인 Multi-Agent 구조는 이 둘 외에도 있다.
> - **Network(네트워크형)**: 중앙 판단자 없이 여러 Agent가 서로 자유롭게 호출하며 대화를 이어간다. Agent 수가 적을 때는 유연하지만, Agent가 늘어날수록 누가 누구를 부르는지 파악하기 어려워져 혼란스러워지기 쉬운 구조로 꼽힌다.
> - **Supervisor(계층형의 한 종류)**: 오늘 배운 것처럼 중앙 Supervisor가 모든 통신과 작업 위임을 통제한다.
> - **Hierarchical(다단계 계층형)**: Supervisor를 여러 명 두고, 그 위에 또 다른 Supervisor를 두는 식으로 계층을 여러 단계로 쌓는 구조. Supervisor 하나로 감당하기엔 하위 팀·역할이 너무 많아졌을 때 쓴다.
>
> 실무 가이드에서는 "관리할 전문가가 4명 정도"라면 Network는 통제가 안 되고 Hierarchical은 과설계이므로, 그 중간인 Supervisor 패턴이 적당하다고 설명한다 — Supervisor는 Network와 Hierarchical 사이의 절충안 정도로 이해하면 된다. ([참고 글](https://towardsai.net/p/l/a-complete-guide-to-multi-agent-systems-in-langgraph-network-to-supervisor-and-hierarchical-models), [참고 글2](https://callsphere.ai/blog/langgraph-supervisor-multi-agent-orchestration-2026))

## ✅ 확인 질문

1. Subgraph를 쓰지 않고 그냥 하나의 큰 StateGraph 안에 모든 Node를 다 넣으면 안 되는 이유는 무엇인가?
2. 부모와 Subgraph가 같은 State 스키마를 쓸 때, 컴파일된 Subgraph를 `add_node`에 바로 넣을 수 있는 이유는 무엇인가?
3. `stream()`에서 `subgraphs=False`일 때와 `subgraphs=True`일 때 반환되는 값의 형태(`chunk` vs `(namespace, chunk)`)는 각각 어떻게 다른가?
4. Subgraph 내부가 부모의 스트리밍 결과에 기본적으로 보이지 않는다는 것이, Subgraph를 쓰면 모니터링이 항상 더 어려워진다는 뜻이 아닌 이유는 무엇인가?
5. 부모와 Subgraph의 State 스키마가 완전히 다를 때, 컴파일된 Subgraph를 바로 `add_node`에 넣지 못하고 래퍼 함수를 만들어야 하는 이유는 무엇인가?
6. Subgraph와 부모 Graph의 관계를 "클래스 상속"이 아니라 "포함(Composition)"이라고 설명하는 이유는 무엇인가?
7. Multi-Agent 시스템을 도입하는 것이 항상 이득은 아닌 이유는 무엇이며, 어떤 경우엔 단일 Agent가 더 나은가?
8. Multi-Agent에서 역할별로 Agent를 분리했을 때 얻는 이점을 표에 나온 4가지 중 2가지를 골라 설명하면?
9. Supervisor 패턴에서 작업 결과가 각 Subagent에서 끝나지 않고 항상 Supervisor에게 돌아오는 구조는 어떤 장점이 있는가?
10. Supervisor의 `research`, `write_report`, `review_report` Tool은 왜 일반 함수가 아니라 각각 내부에서 `create_agent`로 만든 Subagent를 호출하도록 만들었는가?
11. Checkpointer를 Subagent들이 아니라 최상위 Supervisor에만 설정한 이유는 무엇인가?
12. Supervisor의 동작 방식이 "새로운 실행 방식"이 아니라 ReAct의 응용이라고 말할 수 있는 근거는 무엇인가?
13. Multi-Agent 구조 중 Network, Supervisor, Hierarchical의 차이를 "누가 통신을 통제하는가" 기준으로 설명하면?
14. 관리할 전문가(Subagent) 수가 매우 많아진다면 Supervisor 패턴 대신 어떤 구조를 고려해야 하는가?

