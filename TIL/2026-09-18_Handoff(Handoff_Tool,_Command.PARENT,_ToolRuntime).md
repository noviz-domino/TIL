---
tags: [langgraph, multi-agent, handoff, command, tool-runtime, state]
til: v2 2026-09-18
---

# 2026-09-18_Handoff(Handoff_Tool,_Command.PARENT,_ToolRuntime)
> 작성일: 2026-09-18

## 🔗 관련 글

- [Subgraph와 Supervisor(재사용, MultiAgent 구현)](2026-09-17_Subgraph와_Supervisor(재사용_MultiAgent_구현).md) — Supervisor는 결과가 중앙으로 돌아오지만, Handoff는 담당자 자체가 바뀌어 돌아오지 않는다. 같은 Multi-Agent의 다른 갈래다.
- [Human-in-the-Loop(interrupt, Command, Checkpointer)](2026-09-15_Human-in-the-Loop(interrupt,_Command,_Checkpointer).md) — 거기서는 멈춘 Graph를 재개하려고 `Command(resume=...)`를 썼다. 여기서는 같은 `Command`를 `update`/`goto`로 쓴다.
- [LangGraph 기초(State, Node, Edge와 조건부분기, 반복, Reducer)](2026-09-08_LangGraph_기초(State_Node_Edge와_조건부분기_반복_Reducer).md) — Handoff가 담당자를 기억하는 방식은 결국 State에 값 하나를 쓰고 Reducer로 합치는 것이다.
- [ReAct Agent(Tool, ToolNode, tools_condition, MessagesState)](2026-09-09_(1)_ReAct_Agent(Tool,_ToolNode,_tools_condition,_MessagesState).md) — Handoff Tool도 결국 `@tool`로 만든 평범한 Tool이다. 반환값만 다르다.
- [Memory와 State 관리(Checkpointer, Store, 대화 요약)](2026-09-09_(2)_Memory와_State_관리(Checkpointer,_Store,_대화_요약).md) — 바뀐 담당자가 다음 턴에도 유지되려면 Checkpointer와 `thread_id`가 필요하다.

---

## 주제 1: Handoff란

### 한 줄 정의

**현재 Agent가 다른 Agent에게 대화의 담당자 자리를 통째로 넘기는 것.** 넘겨받은 Agent가 새 담당자가 되어 이후 대화를 계속 맡는다.

### 왜 Agent를 여러 개로 쪼개나

Agent 하나에게 판매·환불·기술지원·배송조회를 전부 시키면

- system_prompt가 길어져 LLM이 헷갈린다
- Tool이 많아져 엉뚱한 것을 고른다
- 정책 하나를 바꾸려 해도 거대한 프롬프트 덩어리를 건드려야 한다

그래서 역할별로 작은 Agent로 나눈다. 그러면 **"누가 대답할지 어떻게 정하나"** 라는 새 문제가 생기고, 그 해법이 아래 세 가지다.

### 세 가지 패턴 비교

| 패턴 | 흐름 | 결과가 중앙으로 돌아오나 | 사용자와 대화하는 주체 |
| --- | --- | --- | --- |
| Router | 요청을 한 번 분류해 담당자에게 배달하고 끝 | 안 돌아옴 | 배달된 담당자 |
| Supervisor | 관리자가 부하에게 일을 시키고 결과를 받아 다음을 판단 | **돌아옴** | 계속 관리자 |
| Handoff | 담당자가 직접 옆 담당자에게 자리를 넘김 | 안 돌아옴 | **바뀐 담당자** |

전화 상담 비유가 가장 정확하다.

- Supervisor: 상담원이 뒤에서 기술팀에 물어본 뒤 **본인이** 나에게 답한다.
- Handoff: "기술팀 연결해드릴게요" 하고 넘긴다. 그 뒤로는 기술팀 직원이 나와 직접 대화한다.

### 이번에 만든 시나리오

```text
사용자 → Sales Agent
             └─ 기술 문의 → Support Agent
                                └─ 구매 문의 → Sales Agent
```

- Sales Agent: 제품, 가격, 구매 방법 담당
- Support Agent: 설치, 오류, 고장 담당
- **한 번 넘어가면 그 담당자가 계속 응대해야 한다** (이것이 핵심 요구사항)

---

## 주제 2: Command — State 변경과 Node 이동을 한 번에

### 기존 방식의 어색함

지금까지 Node와 Edge는 역할이 나뉘어 있었다. Node는 State만 고치고, 다음 목적지는 Edge가 정한다.

```python
# [방식 A] Conditional Edge — 조립부터 실행까지
from typing import TypedDict
from langgraph.graph import StateGraph, START, END


class RoutingState(TypedDict):    # 공용 메모장(State) 양식
    query: str                    # 사용자 질문
    category: str                 # 분류 결과가 담길 칸
    result: str                   # 최종 처리 결과가 담길 칸


def classify_query(state: RoutingState) -> dict:
    """질문을 분류해서 State에 적기만 한다."""
    if "오류" in state["query"] or "고장" in state["query"]:
        return {"category": "support"}       # 여기서 판단이 이미 끝났다
    return {"category": "sales"}


def route(state: RoutingState) -> str:
    """안내원: State를 다시 읽어서 목적지를 지목한다."""
    return state["category"]                 # 방금 적힌 값을 또 꺼내 읽음


def handle_sales(state: RoutingState) -> dict:
    return {"result": "판매 문의로 처리했습니다."}

def handle_support(state: RoutingState) -> dict:
    return {"result": "기술 지원 문의로 처리했습니다."}


builder = StateGraph(RoutingState)              # ① 설계도
builder.add_node("classify", classify_query)    # ② 부품 등록
builder.add_node("sales", handle_sales)
builder.add_node("support", handle_support)

builder.add_edge(START, "classify")             # ③ 시작하면 classify부터
builder.add_conditional_edges(                  # ③' 갈림길
    "classify", route, ["sales", "support"],
)
builder.add_edge("sales", END)
builder.add_edge("support", END)

graph = builder.compile()                       # ④ 완성

result = graph.invoke({                         # ⑤ 실행
    "query": "제품에서 오류가 발생했어요.",
    "category": "",
    "result": "",
})
# {'query': '...', 'category': 'support', 'result': '기술 지원 문의로 처리했습니다.'}
```

`classify_query`에서 이미 "support로 가야 한다"는 판단이 끝났는데, 그 결과를 State에 적고 → `route`가 다시 꺼내 읽어 → 목적지를 정한다. **한 번 한 판단을 두 군데로 쪼갠 셈**이다.

### Command를 쓰면

Node가 딕셔너리 대신 **명령서(Command)** 를 반환한다.

```python
return Command(
    update={"category": "support"},   # ① State에 적을 내용 (원래 반환하던 dict 그대로)
    goto="support",                   # ② 다음에 실행할 Node 이름 (Edge 역할)
)
```

- `update` — 갱신. State에 반영할 값
- `goto` — go to, "~로 가라". 옛 언어의 `GOTO` 문에서 온 이름
- 둘 다 선택 사항이다. `update`만 쓰면 사실상 기존 dict 반환과 같고, `goto`만 쓰면 이동만 한다

```python
# [방식 B] Command — 방식 A와 완전히 같은 동작, 조립부터 실행까지
from typing import Literal, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command          # Command는 langgraph.types에 있다


class RoutingState(TypedDict):    # 방식 A와 동일
    query: str
    category: str
    result: str


def classify_query(
    state: RoutingState,
) -> Command[Literal["sales", "support"]]:   # 반환 타입으로 이동 후보를 명시
    """판단하고, State도 고치고, 목적지도 직접 정한다."""
    if "오류" in state["query"] or "고장" in state["query"]:
        return Command(
            update={"category": "support"},  # State 고치기
            goto="support",                  # 목적지도 여기서 지정
        )
    return Command(
        update={"category": "sales"},
        goto="sales",
    )

# route 함수가 통째로 사라졌다

def handle_sales(state: RoutingState) -> dict:
    return {"result": "판매 문의로 처리했습니다."}

def handle_support(state: RoutingState) -> dict:
    return {"result": "기술 지원 문의로 처리했습니다."}


builder = StateGraph(RoutingState)              # ① 설계도
builder.add_node("classify", classify_query)    # ② 부품 등록
builder.add_node("sales", handle_sales)
builder.add_node("support", handle_support)

builder.add_edge(START, "classify")             # ③ 시작하면 classify부터
# classify에서 나가는 Edge를 등록하지 않는다. goto가 대신하기 때문
builder.add_edge("sales", END)
builder.add_edge("support", END)

graph = builder.compile()                       # ④ 완성

result = graph.invoke({                         # ⑤ 실행 — 결과는 방식 A와 동일
    "query": "제품에서 오류가 발생했어요.",
    "category": "",
    "result": "",
})
```

사라진 코드는 두 덩어리다. `route` 함수 전체와 `add_conditional_edges` 호출 전체.

### Command[Literal[...]]의 정체

`Literal`(리터럴 — 값 자체를 타입으로 쓰는 문법)은 "이 값은 이것들 중 하나"라는 뜻이다.

방식 B에서는 `add_conditional_edges`의 후보 목록 `["sales", "support"]`가 사라졌으므로, LangGraph 입장에서는 classify가 어디로 가는지 알 방법이 없다. `goto` 값은 실행해봐야 나오기 때문이다. **그래서 타입 힌트로 후보를 대신 알려준다.**

- 생략해도 **실행은 정상**이다
- 다만 `draw_mermaid_png()`로 그림을 그릴 때 classify 뒤가 텅 비어 보인다

> 파이썬 타입 힌트가 실제로 런타임에 읽혀 기능하는 드문 사례. `@tool`이 함수 시그니처를 읽어 LLM용 스키마를 만드는 것과 같은 패턴이다.

### Edge와 Command 중 무엇을 쓸까

| 상황 | 사용 방식 |
| --- | --- |
| 다음 단계가 항상 동일 | `add_edge()` |
| 작업과 Routing 판단을 분리하고 싶음 | `add_conditional_edges()` |
| State 변경과 이동이 하나의 판단 | `Command(update=..., goto=...)` |
| 내부 Graph에서 부모 Graph로 탈출 | `Command(graph=Command.PARENT)` — **유일한 방법** |

**기본값은 여전히 Edge다.** Edge로 짜면 조립 코드만 읽어도 전체 흐름이 보이지만, `goto`는 함수 본문을 열어봐야 흐름을 알 수 있다. 짧다는 이유만으로 Command를 쓰면 리뷰에서 지적받는다.

---

## 주제 3: Handoff Tool

### 왜 Tool로 만드나

LLM은 글자만 생성할 뿐 프로그램을 조종할 수 없다. LLM이 무언가를 **실행**하게 하는 유일한 통로가 Tool이다. 그래서 **담당자를 바꾸는 행위 자체를 Tool로 만든다.** 이것을 Handoff Tool이라 한다.

- 일반 Tool: 실행 결과를 반환 (`"서울은 맑음"`)
- Handoff Tool: **`Command`를 반환**하여 담당 Agent를 바꾸고 다른 Agent Node로 이동

### 부품 1 — ToolRuntime

Tool의 매개변수는 원래 **LLM이 채운다.** `search_weather(city: str)`의 `city`에 "서울"을 넣는 것은 LLM이다. 함수 시그니처가 스키마로 번역되어 LLM에게 전달되기 때문이다.

그런데 `ToolRuntime`은 예외다.

| 매개변수 | LLM에게 보이는 스키마에 포함? | 실제로 채우는 주체 |
| --- | --- | --- |
| `city: str` | 포함됨 | **LLM** |
| `runtime: ToolRuntime` | **빠짐** | **LangChain(시스템)** |

LLM 입장에서 Handoff Tool은 **인자가 하나도 없는 Tool**이다. `runtime`은 실행 순간에 시스템이 끼워 넣는다. 이런 방식을 `dependency injection`(의존성 주입 — 필요한 것을 바깥에서 넣어주기)이라 한다.

- `runtime` = run(실행) + time(시점) → "실행되는 그 순간의 상황 정보"
- `runtime.state["messages"]` — 현재 Agent(내부 Graph)의 대화 기록
- `runtime.tool_call_id` — 이번 Tool 호출에 붙은 고유 번호

> 식당 주문서 비유: 손님(LLM)에게 주는 주문서에는 메뉴·사이즈 칸만 인쇄되어 있고, 주문번호·접수시각은 포스기(시스템)가 찍는다. 손님이 알 수 없는 정보라서 아는 쪽이 대신 채운다.

### 부품 2 — graph=Command.PARENT

`create_agent`로 만든 Agent는 그 자체가 하나의 Graph다. 이것을 부모 Graph의 Node로 꽂으면 2층 구조(Subgraph)가 된다.

```text
┌─ 부모 Graph ──────────────────────────────────┐
│  Node 이름표: "sales_agent", "support_agent"  │
│                                               │
│  ┌─ sales_agent (내부 Graph) ─┐               │
│  │ Node 이름표: "model","tools"│              │
│  └────────────────────────────┘               │
└───────────────────────────────────────────────┘
```

**Node 이름표는 각 Graph 안에서만 통한다.** Handoff Tool은 내부 Graph 안에서 실행되므로, 그냥 `goto="support_agent"`라고 쓰면 내부 명단(`"model"`, `"tools"`)에서 찾다가 실패한다.

```python
graph=Command.PARENT     # "내부 명단 말고 바깥(부모) 명단에서 찾아라"
```

회의실 안에서 "3번 방으로 가자"고 하면 같은 층의 3번 방을 찾지만, `Command.PARENT`는 "건물 밖으로 나가 단지 기준 3번 동으로 가자"는 뜻이다. 같은 번호라도 **어느 명단에서 찾느냐**가 다르다.

### 부품 3 — AIMessage + ToolMessage를 손으로 챙기기

**LLM API 규칙:** Tool 호출은 항상 메시지 두 개가 한 쌍으로 다닌다.

```text
AIMessage   : "transfer_to_support 불러줘"    [tool_call_id: call_abc]
ToolMessage : "Support Agent로 전환했습니다"   [tool_call_id: call_abc]
```

`tool_call_id`가 주문번호처럼 둘을 묶는다. **짝이 깨지면 다음 LLM 호출이 API 에러로 터진다.**

평소에는 내부 Graph가 정상 종료되면 내부 대화 기록이 자동으로 부모 State에 합쳐진다. 그런데 `Command.PARENT`는 **정상 종료가 아니라 중간 탈출**이라 이 자동 합치기가 일어나지 않는다.

> 회의 도중 문을 박차고 나가면, 회의록을 챙겨 나오지 않는 한 바깥 사람들은 아무것도 모른다.

그래서 두 장을 직접 챙겨 `update`에 실어 보낸다. 직접 만드는 `ToolMessage`는 실제 업무 결과가 아니라 **짝을 맞추기 위한 형식적 메시지**다.

### 구현

```python
# 대화 기록에서 가장 최근 AIMessage를 찾는 도우미
def find_last_ai_message(messages: list) -> AIMessage:
    for message in reversed(messages):        # reversed: 뒤에서부터 순회 (최신 것부터)
        if isinstance(message, AIMessage):    # isinstance(값, 타입): 타입 검사
            return message
    raise ValueError("AIMessage를 찾을 수 없습니다.")   # 정상이면 올 일 없는 방어 코드


@tool                                                      # 일반 함수 → LLM이 부를 수 있는 Tool로
def transfer_to_support(runtime: ToolRuntime) -> Command:  # runtime은 시스템이 주입
    """설치, 오류, 고장 등 기술 문의를 Support Agent에게 넘긴다."""
    # ↑ 이 docstring이 LLM이 읽는 description. 언제 호출할지 판단하는 근거가 된다

    last_ai_message = find_last_ai_message(runtime.state["messages"])   # 주문서
    transfer_message = ToolMessage(                                     # 영수증
        content="Support Agent로 전환했습니다.",
        tool_call_id=runtime.tool_call_id,      # 주문서와 같은 번호로 짝을 맞춤
    )
    return Command(
        goto="support_agent",                   # 부모 Graph의 이 Node로
        update={
            "active_agent": "support_agent",    # 담당자 교체 (덮어쓰기)
            "messages": [last_ai_message, transfer_message],   # 회의록 두 장 (이어붙이기)
        },
        graph=Command.PARENT,                   # 부모 Graph 명단에서 찾아라
    )


@tool                                          # 위와 거울 이미지
def transfer_to_sales(runtime: ToolRuntime) -> Command:
    """제품, 가격, 구매 문의를 Sales Agent에게 넘긴다."""
    last_ai_message = find_last_ai_message(runtime.state["messages"])
    transfer_message = ToolMessage(
        content="Sales Agent로 전환했습니다.",
        tool_call_id=runtime.tool_call_id,
    )
    return Command(
        goto="sales_agent",
        update={
            "active_agent": "sales_agent",
            "messages": [last_ai_message, transfer_message],
        },
        graph=Command.PARENT,
    )
```

---

## 주제 4: 전체 구현

### State 설계부터

State 설계가 곧 Graph 설계다. "무엇을 기억해야 하는가"를 먼저 정해야 Node가 따라온다.

```python
class MultiAgentState(MessagesState):    # MessagesState 상속 → messages 칸이 딸려온다
    active_agent: str                    # "현재 담당자" 칸 추가
```

`MessagesState`의 실제 정의는 messages 한 칸짜리다.

```python
# LangGraph 라이브러리 내부 정의
class MessagesState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    #                                     ↑ Reducer: 덮어쓰지 말고 이어붙여라
```

**두 칸의 합치는 방식이 다르다는 점이 중요하다.**

| 칸 | Reducer | 동작 |
| --- | --- | --- |
| `messages` | `add_messages` | 이어붙이기 (대화가 쌓임) + ID 중복 제거 |
| `active_agent` | 없음(기본) | 덮어쓰기 (최신 담당자만 남음) |

같은 `update` 딕셔너리를 줘도 칸마다 다르게 처리되는 이유가 이것이다.

### 전체 코드

```python
# 판매/기술지원 두 담당자가 서로 대화를 넘겨주는 Handoff 멀티 에이전트

from typing import Literal

from dotenv import load_dotenv
from langchain.agents import create_agent                            # LangChain: 에이전트
from langchain.messages import AIMessage, HumanMessage, ToolMessage  # LangChain: 메시지
from langchain.tools import ToolRuntime, tool                        # LangChain: Tool
from langchain_google_genai import ChatGoogleGenerativeAI
from langgraph.checkpoint.memory import InMemorySaver                # LangGraph: 저장 장치
from langgraph.graph import END, START, MessagesState, StateGraph    # LangGraph: 조립
from langgraph.types import Command                                  # LangGraph: 명령서

load_dotenv()
llm = ChatGoogleGenerativeAI(model="gemini-3.5-flash-lite")   # 무료 한도 넉넉한 모델


# ══ ① State ═════════════════════════════════════════════
class MultiAgentState(MessagesState):
    active_agent: str


# ══ ② Handoff Tool ══════════════════════════════════════
# find_last_ai_message / transfer_to_support / transfer_to_sales
# (주제 3의 구현 코드 그대로)


# ══ ③ Agent 두 개 ═══════════════════════════════════════
sales_agent = create_agent(
    model=llm,
    tools=[transfer_to_support],         # 상대방 Tool만 준다 (자기 자신 Tool은 없음)
    system_prompt=(
        "너는 제품 판매 상담을 담당하는 Sales Agent다. "
        "대화 기록에서 최신 사용자 요청의 주된 의도를 확인해. "      # 과거 대화에 휘둘리지 말라
        "제품 정보, 가격, 구매 방법에 관한 문의라면 사용자에게 직접 답해. "
        "설치, 오류, 고장 등 기술 지원이 필요한 문의라면 직접 답하지 말고 "
        "transfer_to_support만 호출해. Handoff Tool은 한 번에 하나만 호출해."
    ),
    name="sales_agent",                  # 부모 Graph의 Node 이름표와 일치해야 함
)

support_agent = create_agent(
    model=llm,
    tools=[transfer_to_sales],
    system_prompt=(
        "너는 기술 지원을 담당하는 Support Agent다. "
        "대화 기록에서 최신 사용자 요청의 주된 의도를 확인해. "
        "설치, 오류, 고장 등 기술 문의라면 사용자에게 직접 답해. "
        "제품 정보, 가격, 구매 방법에 관한 문의라면 직접 답하지 말고 "
        "transfer_to_sales만 호출해. Handoff Tool은 한 번에 하나만 호출해."
    ),
    name="support_agent",
)


# ══ ④ 부모 Graph 조립 ═══════════════════════════════════
def route_to_active_agent(state: MultiAgentState) -> Literal["sales_agent", "support_agent"]:
    """시작할 때 State의 담당자를 읽어 그 Node로 보낸다."""
    return state["active_agent"]         # 이 문자열이 곧 Node 이름표


def route_after_agent(
    state: MultiAgentState,
) -> Literal["sales_agent", "support_agent", "__end__"]:
    """Agent가 최종 답변을 냈으면 종료, 아니면 현재 담당자를 한 번 더."""
    last_message = state["messages"][-1]                    # [-1] = 가장 최근 메시지
    if isinstance(last_message, AIMessage) and not last_message.tool_calls:
        return "__end__"     # AIMessage인데 tool_calls가 비었다 = 할 말을 다 했다
    return state["active_agent"]


builder = StateGraph(MultiAgentState)                       # ① 설계도
builder.add_node("sales_agent", sales_agent)                # ② 함수가 아니라 Agent(Graph)를 꽂는다
builder.add_node("support_agent", support_agent)

builder.add_conditional_edges(                              # ③ 시작 갈림길
    START, route_to_active_agent, ["sales_agent", "support_agent"],
)
builder.add_conditional_edges(                              # ③ sales 종료 후 갈림길
    "sales_agent", route_after_agent, ["sales_agent", "support_agent", END],
)
builder.add_conditional_edges(                              # ③ support 종료 후 갈림길
    "support_agent", route_after_agent, ["sales_agent", "support_agent", END],
)

graph = builder.compile(checkpointer=InMemorySaver())       # ④ 저장 장치를 달고 완성


# ══ ⑤ 실행 ══════════════════════════════════════════════
config = {"configurable": {"thread_id": "customer-1"}}      # 이 대화방의 State를 쓴다

state1 = graph.invoke(                                      # 1턴 — 첫 담당자를 지정해 시작
    {
        "messages": [HumanMessage(content="AI 스피커 가격 알려줘.")],
        "active_agent": "sales_agent",       # 첫 턴에만 초기값을 넣는다
    },
    config=config,
)
print("담당자:", state1["active_agent"])      # → sales_agent

state2 = graph.invoke(                                      # 2턴 — active_agent를 안 넣는다
    {"messages": [HumanMessage(content="산 제품이 Wi-Fi가 안 잡혀.")]},
    config=config,                           # 같은 thread_id → 저장된 State를 이어받음
)
print("담당자:", state2["active_agent"])      # → support_agent (Handoff 발생)

state3 = graph.invoke(                                      # 3턴 — 담당자 유지 확인
    {"messages": [HumanMessage(content="재부팅해도 안 되는데?")]},
    config=config,
)
print("담당자:", state3["active_agent"])      # → support_agent (그대로 유지)
```

### route_after_agent의 판단 기준

| 마지막 메시지 상태 | 의미 | 결과 |
| --- | --- | --- |
| `AIMessage`인데 `tool_calls`가 비어 있음 | 최종 답변을 냈다 | 종료 |
| 그 외 | 아직 진행 중 | 현재 담당자 계속 |

`"__end__"`는 `END`의 실제 문자열 값이다. 타입 힌트 안에는 변수를 못 쓰므로 문자열로 직접 적는다.

**Handoff가 일어날 때는 이 함수를 거치지 않는다.** Handoff Tool의 `Command(goto=...)`가 목적지를 직접 정해버리기 때문이다. 즉 두 번째 분기는 안전장치에 가깝다.

### Checkpointer와 thread_id

Handoff가 **다음 턴에도 유지되려면** State가 저장되어야 한다.

- `checkpointer` — checkpoint(게임의 세이브 포인트) + -er. 실행이 끝난 뒤에도 State를 저장해두는 장치
- `InMemorySaver` — 메모리(RAM)에 저장. 프로그램을 끄면 사라진다. 실무에서는 `PostgresSaver` 등 DB 기반을 쓴다
- `thread_id` — thread(실). 카톡의 대화방 개념. 같은 값을 주면 이전 State를 이어받고, 다른 값을 주면 새 대화가 시작된다

### 같은 문자열이 다섯 곳에서 맞물린다

```python
name="sales_agent"                              # ① Agent를 만들 때 붙인 이름
builder.add_node("sales_agent", sales_agent)    # ② 부모 Graph의 Node 이름표
goto="sales_agent"                              # ③ Handoff Tool의 목적지
"active_agent": "sales_agent"                   # ④ State에 적는 담당자 이름
return state["active_agent"]                    # ⑤ 안내원이 ④를 읽어 Node 이름으로 씀
```

하나라도 오타가 나면 실행 중에 터진다. 실무에서는 `SALES = "sales_agent"` 같은 상수로 빼서 다섯 곳이 같은 변수를 참조하게 한다. 그러면 오타는 `NameError`로 즉시 잡힌다.

---

## 주제 5: 실행 흐름 추적

Handoff가 실제로 일어나는 **2턴**을 따라가면 전체가 한눈에 들어온다.

```text
━━ 2턴 시작 ━━  사용자: "산 제품이 Wi-Fi가 안 잡혀."

[State 현재]
  messages:     [Human("AI 스피커 가격..."), AI("가격은 ...입니다")]
  active_agent: "sales_agent"
        │
        ▼ ① START의 갈림길
        │   route_to_active_agent → "sales_agent" → sales_agent Node로
        │
        ▼ ② sales_agent 내부(Subgraph)로 진입
        │   LLM 판단: "Wi-Fi 고장? 내 담당이 아니다"
        │   → AIMessage 생성 [call_abc, 호출: transfer_to_support]
        │
        ▼ ③ transfer_to_support 실행 (내부에서)
        │   ├ runtime.state["messages"]에서 ②의 AIMessage 확보    ← 주문서
        │   ├ tool_call_id="call_abc"로 ToolMessage 생성          ← 영수증
        │   └ Command(goto="support_agent", graph=PARENT, update={...}) 반환
        │
        ▼ ★ 내부 Graph를 중간 탈출 (정상 종료가 아님)
        │
        ▼ ④ 부모 Graph가 명령서를 실행 — State 갱신
        │   active_agent: "sales_agent" → "support_agent"   (덮어쓰기)
        │   messages: 뒤에 두 장 이어붙이기                  (add_messages)
        │
[State]
  messages:     [Human, AI, AI(call_abc, transfer_to_support), Tool(call_abc)]
  active_agent: "support_agent"
        │
        ▼ ⑤ goto가 가리킨 support_agent Node로 이동
        │   LLM이 대화 기록 전체를 읽고 답변 생성 (tool_calls 없음)
        │
        ▼ ⑥ route_after_agent: AIMessage + tool_calls 비었음 → "__end__"
        │
        ▼ END — 최종 State 반환

[최종 State]
  active_agent: "support_agent"      ← 저장됨. 3턴에서 그대로 쓰인다
```

3턴에서 `active_agent`를 넣지 않았는데도 Support가 응대하는 이유가 이것이다. **이것이 "담당자가 바뀌었다"의 진짜 의미** — 다음 턴에도 유지되는 것.

---

## 정리

| 부품 | 역할 |
| --- | --- |
| `MessagesState` 상속 + `active_agent` | 담당자를 기억하는 State |
| Handoff Tool (`@tool` + `Command`) | 담당자 교체를 Tool 호출로 만든 것 |
| `graph=Command.PARENT` | 내부 Agent를 탈출해 부모 Graph로 점프 |
| `ToolRuntime` | Tool이 대화 기록·tool_call_id를 꺼내 보는 창구 |
| `AIMessage` + `ToolMessage` 한 쌍 | 탈출하며 잃을 회의록을 손으로 챙기기 |
| `route_to_active_agent` | State의 담당자를 읽어 누가 응대할지 결정 |
| `checkpointer` + `thread_id` | 다음 턴에도 State를 기억 |

**본질은 한 문장이다.** State에 담당자 이름 한 줄을 적어두고, 매 턴 그것을 읽어 실행한다. 나머지는 전부 그 한 줄을 적기 위한 배관 작업이다.

## 개선 여지 (실무 관점)

- **무한 핑퐁 방지가 없다.** Sales와 Support가 서로 "내 담당 아님" 하며 끝없이 던질 수 있다. State에 `handoff_count: int`를 두고 N회 초과 시 종료하거나 사람에게 넘긴다(escalation). 프롬프트의 "한 번만 호출해"는 보장이 아니라 부탁이다.
- **Node 이름이 문자열로 다섯 곳에 흩어져 있다.** 상수나 Enum으로 빼야 오타를 즉시 잡는다.
- **`InMemorySaver`는 재시작하면 전부 사라진다.** 실무는 DB 기반 저장소를 쓴다. `compile()` 인자만 바꾸면 되도록 설계되어 있어 구조 변경 없이 교체할 수 있다.
- **관측성이 없다.** 담당자 전환마다 구조화된 로그를 남기거나 LangSmith로 추적해야, 운영 중에 "왜 엉뚱한 담당자가 답했지"를 따라갈 수 있다.

---

## ✅ 확인 질문

1. Handoff가 Supervisor와 다른 점을 "작업 결과가 어디로 돌아오는가" 기준으로 설명하면?
2. Router, Supervisor, Handoff 중 "사용자와 계속 대화하는 주체가 바뀌는" 것은 무엇이며 그 이유는?
3. Node가 딕셔너리를 반환할 때와 `Command`를 반환할 때, Node가 결정할 수 있는 범위는 어떻게 달라지는가?
4. `Command`의 `update`와 `goto`는 각각 기존 LangGraph의 어떤 기능을 대신하는가?
5. `Command[Literal["sales", "support"]]`에서 `Literal` 부분을 지우면 무엇이 달라지고, 무엇은 그대로인가?
6. 고정된 흐름과 일반적인 Routing에 `Command`보다 Edge를 권하는 이유는?
7. Tool의 매개변수는 보통 누가 채우며, `ToolRuntime` 타입 매개변수는 왜 예외인가?
8. LLM에게 전달되는 Tool 스키마에 `runtime`이 포함되지 않는 것이 왜 안전 측면에서도 바람직한가?
9. Handoff Tool에서 `graph=Command.PARENT`를 빼면 어떤 에러가 나며, 그 이유는 Node 이름표의 어떤 성질 때문인가?
10. `Command.PARENT`로 내부 Graph를 벗어날 때 `AIMessage`와 `ToolMessage`를 직접 `update`에 담아야 하는 이유는?
11. 직접 만드는 `ToolMessage`의 `tool_call_id`에 `runtime.tool_call_id`를 넣지 않으면 무슨 문제가 생기는가?
12. `MultiAgentState`에서 `messages`와 `active_agent`는 같은 `update`로 갱신되는데 왜 결과가 다르게 합쳐지는가?
13. 각 Agent에게 상대방 Handoff Tool만 주고 자기 자신 Tool은 주지 않는 이유는?
14. `route_after_agent`가 "마지막 메시지가 AIMessage이고 tool_calls가 비어 있음"을 종료 조건으로 삼는 근거는?
15. Handoff가 일어날 때 `route_after_agent`를 거치지 않는 이유는?
16. 3턴에서 `active_agent`를 넘기지 않았는데도 Support Agent가 응대하는 이유를 두 가지 장치로 설명하면?
17. 같은 `thread_id`를 쓰는 것과 다른 `thread_id`를 쓰는 것은 Handoff 동작에 어떤 차이를 만드는가?
18. 무한 핑퐁을 프롬프트가 아니라 코드로 막아야 하는 이유는?
