---
tags: [langgraph, human-in-the-loop, interrupt, checkpointer, structured-output]
til: v2 2026-09-15
---

# Human-in-the-Loop (interrupt, Command, Checkpointer)
> 작성일: 2026-09-15

## 🔗 관련 글

- [Memory와 State 관리(Checkpointer, Store, 대화 요약)](2026-09-09_(2)_Memory와_State_관리(Checkpointer,_Store,_대화_요약).md) — Checkpointer/thread_id를 처음 배운 곳(그때는 대화 기억용, 오늘은 중단/재개용)
- [RAG Agent(Retriever Tool, create_retriever_tool, Checkpointer)](2026-09-10 (1) RAG_Agent(Retriever_Tool,_create_retriever_tool,_Checkpointer).md) — Checkpointer+thread_id 응용 사례
- [Plan-and-Execute(Planner, Executor, Replanner, Union)](2026-09-14_Plan-and-Execute(Planner,_Executor,_Replanner,_Union).md) — 오늘 실습에서 HITL을 추가하는 기반 그래프

## Human-in-the-Loop(HITL)이란

지금까지 만든 Agent는 사용자가 질문하면 끝까지 알아서 처리했다. HITL은 그 중간에 **중요한 판단이 필요한 지점에서 일부러 멈춰서, 사람이 검토·결정한 뒤에 이어가는** 방식이다.

사람의 개입이 필요한 상황:
- 결제, 삭제, 이메일 발송처럼 되돌리기 어려운 작업의 승인
- LLM이 생성한 답변이나 문서의 검토와 수정
- 작업에 필요한 정보가 부족할 때 추가 입력 요청

```
AI 작업 수행 → 사람의 판단이 필요한 지점에서 중단 → 검토·입력 → 작업 재개
```

Agent가 검색만 하던 이전 노트북들과 달리, 이제 Agent가 실제 행동(결제 등)을 할 수 있게 되므로, LLM의 판단을 그대로 믿고 실행해버리면 되돌릴 수 없는 사고가 날 수 있다. LangGraph에서는 그래프 실행을 중단하고, 사람의 입력을 받은 뒤 재개하는 방식으로 HITL을 구현한다.

## interrupt()와 Command(resume=...)

- **`interrupt(value)`**: Tool이나 노드 내부에서 호출한다. 실행이 이 코드에 도달하면 그래프를 중단하고 `value`를 호출자에게 반환한다. Checkpointer가 당시의 그래프 State와 중단된 작업을 기록한다.
- **`Command(resume=결정)`**: 같은 `thread_id`로 이 값을 전달하면, 노드나 Tool을 **처음부터 다시 실행**하지만 `interrupt()` 지점에서는 이번엔 멈추지 않고 전달받은 결정을 반환한 뒤 다음 코드로 진행한다.

```python
@tool(parse_docstring=True)
def send_payment(amount: int, recipient: str) -> str:
    """수신자에게 결제를 실행한다.

    Args:
        amount: 결제 금액
        recipient: 수신자 이름
    """
    # 재개 시 Tool이 처음부터 실행되므로 이 메시지는 다시 출력된다.
    print(f"결제 요청 검토: {recipient}, {amount:,}원")

    decision = interrupt(
        {
            "question": "이 결제를 실행하시겠습니까?",
            "tool": "send_payment",
            "args": {"amount": amount, "recipient": recipient},
        }
    )

    if decision["action"] == "reject":
        reason = decision.get("reason", "사유 없음")
        return f"결제가 취소되었습니다. 사유: {reason}"

    # 결제처럼 부작용이 있는 작업은 interrupt() 뒤에서 실행한다.
    return f"{recipient}에게 {amount:,}원 결제 완료."
```

핵심 동작 원리: **"똑같은 함수를 다시 실행하는데, interrupt() 지점만 이번엔 멈추지 않고 값을 반환한다."**

## 그래프 구성 — Checkpointer 필수

```python
class State(TypedDict):
    messages: Annotated[list, add_messages]

tools = [send_payment]
llm = ChatGoogleGenerativeAI(model=MODEL_NAME).bind_tools(tools)

def agent(state: State):
    return {"messages": [llm.invoke(state["messages"])]}

builder = StateGraph(State)
builder.add_node("agent", agent)
builder.add_node("tools", ToolNode(tools))
builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", tools_condition)
builder.add_edge("tools", "agent")

graph = builder.compile(checkpointer=MemorySaver())
```

interrupt()로 멈췄다가 나중에 재개하려면 **그래프 State와 중단된 작업을 기록하는 Checkpointer, 실행을 구분하는 thread_id**가 반드시 있어야 한다. `MemorySaver()`는 이전 노트북(Memory/State 관리)에서 대화를 기억하려고 썼던 것과 같은 도구인데, 이번엔 "멈췄다가 나중에 이어가는" 용도로 쓰인다.

## 중단에 쓰이는 세 가지 값

| 구분 | 역할 |
|------|------|
| 그래프 state | 노드들이 읽고 수정하는 내부 실행 상태 |
| interrupt payload | interrupt()가 호출자(사람)에게 전달하는 질문, Tool 이름과 인자 |
| resume value | 승인자가 그래프에 돌려주는 결정 |

payload와 resume은 방향이 반대다 — payload는 "그래프 → 사람"으로 가는 질문이고, resume은 "사람 → 그래프"로 가는 답이다. 그리고 이 둘 다 State와는 별개다 — State는 그래프 내부에서 노드들끼리 주고받는 데이터고, payload/resume은 그래프 바깥(사람)과 주고받는 데이터다.

```
첫 번째 요청: graph.invoke(작업, config)
             → Tool 내부 interrupt()에서 중단
             → payload 반환 및 그래프 실행 정보 기록

두 번째 요청: graph.invoke(Command(resume=결정), 같은 config)
             → 같은 thread_id의 상태 복원
             → 결정이 interrupt()의 반환값이 되어 실행 재개
```

payload와 resume 값에는 Checkpointer가 저장할 수 있는 문자열, 숫자, 목록, 딕셔너리 같은 **JSON 직렬화 가능한 값**을 사용한다.

> ➕ **더 알아두기 — resume에 Pydantic 객체를 직접 넣는 예제의 함정**
> Gemini에게 물어본 답변 중 `Command(resume=ApprovalResponse(is_approved=True, ...))`처럼 Pydantic 인스턴스를 통째로 resume에 넘기는 예제가 있었다. `MemorySaver`(서버 RAM에 파이썬 객체를 그대로 들고 있음)에서는 당장 잘 작동하지만, 실무에서 쓰는 `SqliteSaver`/`PostgresSaver` 같은 실제 DB 기반 Checkpointer는 보통 JSON으로 직렬화해서 저장하므로, Pydantic 객체를 그대로 넣으면 그 과정에서 깨지거나 예상치 못하게 동작할 수 있다. 더 안전한 방식은 resume에는 평범한 dict를 넘기고, 필요하면 노드 안에서 `ApprovalResponse(**decision)`처럼 그 자리에서 직접 검증하는 것이다 — "저장은 JSON스러운 dict로, 검증은 필요한 시점에 코드로"가 원칙에 더 맞는다.

## 첫 번째 호출 — 중단되는 순간

```python
config = {"configurable": {"thread_id": "payment-1"}}

result = graph.invoke(
    {"messages": [("user", "홍길동에게 50000원 결제해줘")]},
    config,
)

request = result["__interrupt__"][0].value
print(f"질문: {request['question']}")
print(f"Tool: {request['tool']}")
print(f"인자: {request['args']}")
print(f"다음 노드: {graph.get_state(config).next}")
```

- `result["__interrupt__"]`: 그래프가 멈추면 결과 dict 안에 이 특별한 키가 생기고, 여기에 멈춘 지점의 정보가 담긴다.
- `result["__interrupt__"][0]`: 한 실행 중 interrupt가 여러 번 걸릴 수도 있어서 리스트 형태다.
- `.value`: interrupt()에 넣었던 그 dict(질문/Tool이름/인자)를 꺼낸다.
- `graph.get_state(config).next`: 다음에 실행될 노드가 뭔지 알려준다 — 멈춰있으니 아직 실행 안 된 노드가 나온다.

## 재개 — 승인 / 거절

```python
result = graph.invoke(
    Command(resume={"action": "approve"}),
    config,
)
print(result["messages"][-1].text)
```

`resume={"action": "reject", "reason": "..."}`로 재개하면 Tool 내부의 `if decision["action"] == "reject":` 분기가 실행돼 결제를 수행하지 않고 취소 결과를 반환한다.

**`action`과 `reason`은 역할이 다르다.** `action`은 코드가 `== "reject"`처럼 정확히 비교하는 값이라 정해진 문자열(`"approve"`/`"reject"`)만 들어가야 하고, `reason`은 사람이 읽는 자유 텍스트라 `decision.get("reason", ...)`으로 그냥 꺼내서 보여주기만 한다. 거절 사유를 `action` 자리에 적으면 `== "reject"` 비교가 실패해 거절 분기를 못 탄다.

이 decision dict는 **LLM이 자연어를 해석해서 만드는 게 아니라, 개발자가 짜놓은 평범한 파이썬 코드(if/else, 버튼 클릭 등)가 직접 조립하는 것**이다.

```python
answer = input("결제를 승인하시겠습니까? (y/n): ").strip().lower()

if answer == "y":
    decision = {"action": "approve"}
else:
    reason = input("거절 사유를 입력하세요: ")
    decision = {"action": "reject", "reason": reason}

result = graph.invoke(Command(resume=decision), input_config)
```

## 실제 서비스에서의 요청 흐름

실제 웹 서비스는 승인될 때까지 하나의 API 요청 연결을 계속 붙잡고 있지 않는다. **요청을 두 번에 나눠서** 처리한다.

```
Client                              Server / Graph
  │  POST /chat { message: "..." }
  ├──────────────────────────────────▶
  │                                   │ Graph 실행 → interrupt()에서 중단
  │  response { thread_id, interrupt: {...} }
  ◀───────────────────────────────────┤
  │  (사용자에게 승인/거절 UI 표시)
  │  POST /resume { thread_id, action: "approve" }
  ├──────────────────────────────────▶
  │                                   │ thread_id로 멈춘 Graph 식별 → Command(resume=...) → 실행 재개
  │  response "처리가 완료되었습니다."
  ◀───────────────────────────────────┤
```

### 왜 input()을 웹 서버에서 쓸 수 없는가

Gemini에게 물어본 답변 중 이 표는 정확하고 실무 감각이 잘 담긴 정리였다.

| 구분 | 파이썬 기본 input() | LangGraph interrupt() + Command |
|---|---|---|
| 기억 장소 | 서버 메모리(RAM, 휘발성) | 체크포인터 DB(SQLite, Postgres 등, 비휘발성) |
| 대기 메커니즘 | CPU/RAM을 잡고 동기 블로킹 대기 | 상태 저장 후 프로세스 완전 종료(Stateless) |
| 적용 환경 | CLI / 단일 스크립트 | REST API / 분산 웹 서버 백엔드 |

네 가지 기술적 이유:
1. **프로세스 블로킹**: `input()`은 입력을 받을 때까지 CPU/RAM 자원을 점유하며 무한 대기한다. 웹 서버에서 여러 사용자가 동시에 대기하면 서버 자원이 고갈된다.
2. **HTTP 프로토콜 무상태성**: REST API 요청을 수 분간 열어둘 수 없고, 게이트웨이에서 타임아웃(408/504)이 발생한다.
3. **비휘발성 체크포인팅**: `input()` 대기 중 서버가 재부팅되거나 메모리 부족(OOM)이 발생하면 대기 상태가 즉시 사라진다. LangGraph는 State를 DB에 기록하므로 서버 재시작 후에도 멈춘 시점부터 복구할 수 있다.
4. **수평 확장**: `input()`은 단일 장비의 RAM에 갇힌다. 분산 서버 환경에서 "승인 요청을 받은 서버"와 "승인 응답을 받는 서버"가 달라도, DB에 저장된 체크포인트를 통해 작업을 이어받을 수 있다.

### Checkpointer 종류와 저장 위치

```python
from langgraph.checkpoint.memory import MemorySaver
checkpointer = MemorySaver()
app = workflow.compile(checkpointer=checkpointer)
```

- **MemorySaver**(지금 실습에서 쓰는 것): 체크포인트를 외부 DB가 아니라 현재 실행 중인 서버 RAM(프로세스 메모리)에 저장한다. 설정이 간편해 로컬 테스트·PoC에 좋지만, 파이썬 프로세스가 종료·재시작되면 해당 thread_id의 체크포인트가 모두 사라진다.
- **운영 환경**: 영구 저장이 필요하면 `SqliteSaver`(파일 DB) 또는 `PostgresSaver`(PostgreSQL)로 체크포인터를 교체한다.

## 실습: 게시글 발행 승인 + Plan-and-Execute에 HITL 추가

- **게시글 발행 승인**: `send_payment`와 완전히 같은 패턴으로 `publish_post` Tool을 만들고, 승인/거절 케이스를 각각 다른 `thread_id`로 실행해 결과를 확인했다.
- **Plan-and-Execute + HITL**: Plan-and-Execute(20번) 그래프에 `review_plan`이라는 새 노드를 추가했다. Planner가 계획을 세우면 review_plan이 interrupt()로 멈추고, 승인이면 Executor로, 거절이면 반려 사유(feedback)를 State에 남기고 다시 Planner로 되돌아가 계획을 다시 세운다.
  ```
  Planner → 계획 검토(HITL)
              ├─ 승인 → Executor
              └─ 거절 + 추가 의견 → Planner → 계획 검토(HITL)
  ```
  여기서 "거절 → Planner로 되돌아가기"는 Plan-and-Execute의 Replanner가 하던 "다시 계획 짜기" 역할을, 이번엔 **사람의 결정**이 대신 트리거한다는 점이 다르다 — 판단 주체가 LLM(Replanner)에서 사람(review_plan의 interrupt)으로 바뀐 것뿐, 구조(반려되면 계획을 다시 세운다)는 같다.

## 오늘 배운 것 요약

| 개념 | 역할 |
|---|---|
| interrupt(value) | 그래프를 멈추고 value를 호출자에게 반환 |
| Command(resume=...) | 같은 thread_id로 결정을 전달해 멈춘 지점부터 재개 |
| 그래프 state / payload / resume | 각각 내부 데이터 / 그래프→사람 질문 / 사람→그래프 답 |
| action vs reason | action=코드가 비교하는 정해진 값, reason=사람이 읽는 자유 텍스트 |
| 서버 두 요청 분리 | POST /chat(중단) + POST /resume(재개), 연결을 계속 붙잡지 않음 |
| MemorySaver vs SqliteSaver/PostgresSaver | 휘발성 RAM 저장 vs 영구 DB 저장 |

## ✅ 확인 질문

1. HITL이 필요한 상황 세 가지는 무엇인가?
2. `interrupt()`가 호출되면 그래프에 어떤 일이 일어나며, 그 반환값(payload)은 누구에게 전달되는가?
3. `Command(resume=...)`로 재개할 때, Tool 함수는 처음부터 다시 실행되는가 아니면 멈춘 지점부터 이어서 실행되는가?
4. 그래프 state, interrupt payload, resume value 세 가지는 각각 어떤 방향(그래프↔사람)의 데이터인가?
5. resume에 Pydantic 객체를 직접 넣는 방식이 `MemorySaver`에서는 되는데 `SqliteSaver`/`PostgresSaver`에서는 왜 위험할 수 있는가?
6. `decision["action"]`과 `decision["reason"]`의 역할 차이는 무엇이며, 거절 사유를 `action`에 넣으면 어떤 문제가 생기는가?
7. decision dict를 만드는 주체는 LLM인가 아니면 다른 무엇인가?
8. 실제 웹 서비스는 왜 `input()`처럼 하나의 요청을 계속 붙잡고 기다리지 않고 두 번의 API 요청으로 나누는가?
9. `input()`을 웹 서버에서 쓸 수 없는 네 가지 기술적 이유는 각각 무엇인가?
10. `MemorySaver`와 `SqliteSaver`/`PostgresSaver`의 차이는 무엇이며, 실무에서는 왜 후자로 교체하는가?
11. Plan-and-Execute에 HITL을 추가했을 때, "거절 → 계획 다시 세우기"는 기존 Replanner의 역할과 어떤 점에서 같고 어떤 점에서 다른가?
