---
tags: [langgraph, fault-tolerance, retry-policy, middleware, logging]
til: v2 2026-09-21
---

# 2026-09-21_장애_허용과_Tool_통제_로깅(RetryPolicy,_wrap_tool_call,_logging)
> 작성일: 2026-09-21

강사와 따로 진행한 학습. 27(장애 허용), 28(Agent Tool 실행 검증과 통제), 29(LangGraph 로깅) 세 파일을 다뤘다.
세 주제 모두 "돌아가는 코드"가 아니라 **운영에서 버티는 코드**를 만드는 이야기다.

## 🔗 관련 글

- [Handoff(Handoff Tool, Command.PARENT, ToolRuntime)](2026-09-18_Handoff(Handoff_Tool,_Command.PARENT,_ToolRuntime).md) — 거기서 배운 `Command(update=, goto=)`를 여기서는 `error_handler`가 그대로 반환해 실패 처리 경로로 보낸다.
- [Time Travel(Checkpoint, Replay, Fork)](2026-09-16_Time_Travel(Checkpoint,_Replay,_Fork).md) — 실패한 실행을 재개하는 원리는 checkpoint를 되짚는 것과 같은 구조다.
- [Memory와 State 관리(Checkpointer, Store, 대화 요약)](2026-09-09_(2)_Memory와_State_관리(Checkpointer,_Store,_대화_요약).md) — 재개(`invoke(None, config)`)가 가능한 이유는 checkpointer와 `thread_id`가 State를 저장해두기 때문이다.
- [LangGraph 기초(State, Node, Edge와 조건부분기, 반복, Reducer)](2026-09-08_LangGraph_기초(State_Node_Edge와_조건부분기_반복_Reducer).md) — `RetryPolicy`와 `error_handler`는 `add_node()`에 붙이는 옵션이다.
- [Parallelization](<2026-09-11 (1) 18-parallelization.md>) — super-step 개념은 병렬 실행에서 배운 "같은 차례에 실행되는 노드 묶음"이 그대로 쓰인다.
- [Guardrails(Input Guardrail, Output Guardrail, Fail-Closed)](<2026-09-10 (2) Guardrails(Input_Guardrail,_Output_Guardrail,_Fail-Closed).md>) — Tool 실행 전/후 검사는 Guardrail을 Tool 단위로 옮긴 것이다.

---

## 주제 1: 장애 허용 (Fault Tolerance)

### 정의

**장애 허용**(Fault Tolerance)은 장애가 발생했을 때 오류를 처리하거나 복구하여 **작업을 계속하거나 재개할 수 있게 하는 능력**이다.

- fault = 결함, 고장 / tolerance = 견딤, 내성
- 직역하면 "고장을 견디는 힘". 고장이 **안 나게** 하는 게 아니라, 나더라도 **서비스가 죽지 않게** 하는 것이 목표다

이 주제는 크게 두 갈래로 나뉜다.

| 갈래 | 수단 | 언제 |
| --- | --- | --- |
| **자동 재시도** | `RetryPolicy` | 한 번의 그래프 호출 안에서, 일시적 오류 |
| **재개** | checkpointer + `invoke(None, config)` | 실행이 실패로 끝난 뒤, 원인을 해결하고 이어서 |

### RetryPolicy로 자동 재시도하기

노드에 `RetryPolicy`를 설정하면 지정한 오류가 발생했을 때 **한 번의 그래프 호출 안에서** 해당 노드를 자동으로 재시도한다.

| 설정 | 의미 |
| --- | --- |
| `initial_interval` | 첫 재시도 전 대기 시간 |
| `backoff_factor` | 대기 시간 증가 배수 |
| `max_interval` | 최대 대기 시간 |
| `max_attempts` | **최초 실행을 포함한** 최대 시도 횟수 |
| `jitter` | 무작위 지연 적용 여부 |
| `retry_on` | 재시도할 예외 타입 또는 판별 함수 |

- `backoff` = 물러나다. 재시도 간격을 점점 늘려 상대 서버에 부담을 덜 주는 방식을 `exponential backoff`(지수 백오프)라 한다
- `jitter` = 흔들림. 여러 클라이언트가 **동시에** 재시도해서 서버를 다시 죽이는 현상(thundering herd)을 막으려고 대기 시간에 무작위 값을 섞는 것
- `max_attempts`는 재시도 횟수가 아니라 **총 시도 횟수**다. 3이면 최초 1회 + 재시도 2회

```python
# 외부 서비스 호출이 두 번 실패한 뒤 세 번째에 성공하는 상황을 만든다
class RetryState(TypedDict):
    query: str
    prepared_query: str
    response: str


retry_count = {"prepare": 0, "call_service": 0}    # 노드별 실행 횟수를 세는 카운터


def retry_prepare(state: RetryState):
    retry_count["prepare"] += 1
    print("prepare 실행: 조회 요청 준비")
    return {"prepared_query": state["query"].strip()}   # strip(): 앞뒤 공백 제거


def call_unstable_service(state: RetryState):
    retry_count["call_service"] += 1
    attempt = retry_count["call_service"]
    print(f"call_service 시도 {attempt}")
    # 실습을 위해 강제로 예외를 발생시킨 것. 실제로는 외부 API 호출이라고 보면 된다
    if attempt < 3:
        print(f"call_service 시도 {attempt} 실패: 일시적 연결 오류")
        raise ConnectionError("일시적 연결 오류")
    return {"response": f"조회 성공: {state['prepared_query']}"}


retry_policy = RetryPolicy(
    initial_interval=2.0, backoff_factor=2.0, max_interval=1.0,
    max_attempts=3, jitter=True, retry_on=ConnectionError,   # ConnectionError만 재시도 대상
)

retry_builder = StateGraph(RetryState)
retry_builder.add_node("prepare", retry_prepare)
retry_builder.add_node("call_service", call_unstable_service, retry_policy=retry_policy)
#                                                             ↑ 노드에 정책을 붙인다
retry_builder.add_edge(START, "prepare")
retry_builder.add_edge("prepare", "call_service")
retry_builder.add_edge("call_service", END)
retry_graph = retry_builder.compile()

retry_result = retry_graph.invoke({"query": "배송 현황"})
print(retry_result["response"])
print("노드별 실행 횟수:", retry_count)     # prepare=1, call_service=3
```

**결과가 `prepare=1`, `call_service=3`이라는 점이 핵심이다.** 실패한 `call_service`만 재시도되고, 이미 완료된 `prepare`는 다시 실행되지 않는다.

### 최대 시도 횟수까지 실패하면

재시도 횟수를 소진하면 `invoke()`에서 예외가 그대로 발생한다. 즉 **`RetryPolicy`는 오류를 삼키지 않는다.**

```python
retry_count = {"prepare": 0, "call_service": 0}    # 카운터 초기화

limited_retry_policy = RetryPolicy(
    initial_interval=1.0, backoff_factor=1.0, max_interval=1.0,
    max_attempts=2, jitter=True, retry_on=ConnectionError,   # 총 2회만 시도
)

limited_retry_builder = StateGraph(RetryState)
limited_retry_builder.add_node("prepare", retry_prepare)
limited_retry_builder.add_node(
    "call_service", call_unstable_service, retry_policy=limited_retry_policy
)
limited_retry_builder.add_edge(START, "prepare")
limited_retry_builder.add_edge("prepare", "call_service")
limited_retry_builder.add_edge("call_service", END)
limited_retry_graph = limited_retry_builder.compile()

try:
    limited_retry_graph.invoke({"query": "배송 현황"})
except ConnectionError as error:              # 예외가 호출 코드까지 전달된다
    print("최대 시도 횟수 후 실행 실패:", error)
print("노드별 실행 횟수:", retry_count)         # prepare=1, call_service=2
```

### 재시도하지 않을 오류 구분하기

**모든 오류를 재시도하면 안 된다.** 요청 형식이 잘못된 경우처럼 **같은 요청을 반복해도 절대 성공할 수 없는 오류**는 재시도가 시간 낭비일 뿐이다.

`retry_on`으로 재시도 대상 예외를 지정하면, 그 외 예외는 한 번 만에 밖으로 나간다.

```python
permanent_count = {"attempts": 0}


def reject_invalid_input(state: RetryState):
    permanent_count["attempts"] += 1
    raise ValueError("잘못된 요청 형식")        # ValueError는 retry_on 대상이 아니다


permanent_builder = StateGraph(RetryState)
permanent_builder.add_node("validate", reject_invalid_input, retry_policy=retry_policy)
#                                                            ↑ retry_on=ConnectionError
permanent_builder.add_edge(START, "validate")
permanent_builder.add_edge("validate", END)
permanent_graph = permanent_builder.compile()

try:
    permanent_graph.invoke({"query": "잘못된 요청"})
except ValueError as error:
    print(f"실행 실패: {error}")
print("총 시도 횟수:", permanent_count["attempts"])    # 1 — 재시도하지 않았다
```

> ⚠️ **꼭 기억할 함정:** 노드가 오류를 **발생시키지 않고** `{"status": "failed"}`를 **반환**하면, LangGraph 입장에서는 정상 반환이므로 `RetryPolicy`가 전혀 동작하지 않는다. 재시도를 원하면 반드시 `raise`를 써야 한다.

### 재시도는 어느 단위에 거는가

| 방식 | 오류가 발생했을 때의 동작 |
| --- | --- |
| `RetryPolicy` | **노드** 단위로 다시 실행한다 |
| `with_retry()` | **Runnable** 단위로 다시 실행한다 |
| `with_fallbacks()` | 지정한 **다른 모델이나 체인**으로 바꾸어 실행한다 |

재시도는 **실패할 가능성이 있고, 다시 실행할 필요가 있는 가장 작은 단위**에 적용한다.

- 노드 전체를 다시 실행해야 하면 → `RetryPolicy`
- 노드 안의 모델 호출만 다시 하면 되면 → `model.with_retry()`

> 단위를 크게 잡으면 불필요한 작업이 반복되고, 부작용(side effect)이 있는 코드가 여러 번 실행될 위험이 커진다.

### super-step — 재개를 이해하기 위한 선행 개념

**super-step**은 그래프에서 **같은 차례에 실행하도록 예정된 노드들의 묶음**이다. checkpointer를 설정하면 **각 super-step이 완료될 때** 노드들의 반환값을 반영한 State가 checkpoint로 저장된다.

```text
순차 실행: prepare → call_service → finish
병렬 실행: prepare → [stable_branch, unstable_branch] → summarize
```

- **순차 실행**에서는 각 노드가 서로 다른 super-step에 속한다 → 노드마다 완료 후 checkpoint가 만들어진다
- **병렬 실행**에서는 대괄호 안의 두 노드가 같은 super-step에 속한다 → 두 노드가 **모두** 완료된 뒤 checkpoint가 만들어진다

super = 위의, 상위의. 개별 노드 실행보다 **한 단계 위의 단위**라서 붙은 이름이다.

### 노드 실행 실패와 재개

자동 재시도로도 해결되지 않아 실행이 실패로 끝난 뒤, **원인을 해결하고 이어서 처리**하는 방법이다.

실패 원인을 해결한 뒤 **같은 `thread_id`로 `invoke(None, config)`** 를 호출하면 저장된 실행을 이어서 진행한다.

- `None`은 **새 입력이 없다**는 뜻이다. 처음부터 다시 하는 게 아니라 멈춘 지점부터 잇는 것
- 재개할 노드는 checkpoint의 `next`에서 확인한다

#### 순차 실행 재개

```python
class SequentialState(TypedDict):
    request: str
    prepared: str
    response: str


sequential_count = {"prepare": 0, "call_service": 0}   # 실행 횟수 카운터
sequential_gate = {"allow_service": False}             # 연결 성공 여부를 조작하는 스위치


def sequential_prepare(state: SequentialState):
    sequential_count["prepare"] += 1
    return {"prepared": state["request"].strip()}


def sequential_call_service(state: SequentialState):
    sequential_count["call_service"] += 1
    if not sequential_gate["allow_service"]:           # gate가 닫혀 있으면 실패
        raise ConnectionError("서비스 연결 실패")
    return {"response": f"처리 완료: {state['prepared']}"}


sequential_builder = StateGraph(SequentialState)
sequential_builder.add_node("prepare", sequential_prepare)
sequential_builder.add_node("call_service", sequential_call_service)
sequential_builder.add_edge(START, "prepare")
sequential_builder.add_edge("prepare", "call_service")
sequential_builder.add_edge("call_service", END)
sequential_graph = sequential_builder.compile(checkpointer=InMemorySaver())
#                                             ↑ 재개하려면 checkpointer가 반드시 필요
sequential_config = {"configurable": {"thread_id": "fault-sequential-1"}}


# ── 1단계: 첫 호출에서 실패시키기 ──
try:
    sequential_graph.invoke({"request": "주문 확인"}, sequential_config)
except ConnectionError as error:
    print("실행 실패:", error)


# ── 2단계: 어디서 멈췄는지 확인하기 ──
sequential_snapshot = sequential_graph.get_state(sequential_config)
print("저장된 State:", sequential_snapshot.values)   # prepared는 이미 저장되어 있다
print("다음 노드:", sequential_snapshot.next)         # ('call_service',)


# ── 3단계: 원인을 해결하고 같은 thread에서 재개하기 ──
sequential_gate["allow_service"] = True              # 문제 해결 (연결 복구를 가정)
sequential_result = sequential_graph.invoke(None, sequential_config)
#                                           ↑ None = 새 입력 없음, 이어서 진행
print("재개 결과:", sequential_result["response"])
print("호출 횟수:", sequential_count)                 # prepare=1, call_service=2
```

`prepare=1`이 핵심이다. **완료된 준비 노드는 반복하지 않고, 실패한 서비스 호출 노드만 처음부터 다시 실행**한 것이다.

#### 재개는 누가 요청하는가

재개 요청은 사용자나 프로그램이 보낼 수 있다.

- **사용자 요청**: '다시 시도' 버튼을 누르면 서버가 해당 작업을 재개한다
- **자동 재개**: 실패한 작업을 기록해 두고, 일정 시간이 지난 뒤 **worker**가 재개하도록 구성한다. Worker는 백그라운드에서 작업을 실행하는 프로그램이다

두 경우 모두 실제 재개는 같은 `thread_id`로 `invoke(None, config)`를 호출해서 시작한다. 자동 재개에서는 **재개할 오류 종류, 대기 시간, 최대 횟수**를 정하고 **같은 작업이 동시에 실행되지 않도록** 관리해야 한다.

```python
# 예외 처리 코드에서 직접 재개하는 형태
try:
    auto_result = auto_graph.invoke({"request": "주문 확인"}, auto_config)
except ConnectionError as error:
    # 실제 서비스에서는 worker가 일정 시간 뒤 재개하도록 구성할 수 있다
    print("실행 실패:", error)
    print("2초 후 재개합니다.")
    time.sleep(5)
    auto_resume_gate["allow_service"] = True          # 서비스 연결 복구를 가정
    auto_result = auto_graph.invoke(None, auto_config)   # 같은 config로 재개

print("결과:", auto_result["response"])
print("호출 횟수:", auto_resume_count)                  # prepare=1, call_service=2
```

#### 병렬 실행에서 일부 노드만 실패하면

같은 super-step에서 일부 노드가 실패해도 **성공한 노드의 결과는 보존된다.** 재개할 때는 이 결과를 재사용하고 **실패한 노드만** 다시 실행한다.

```text
prepare
   ├─ stable_branch   ─ 성공 → 결과 보존
   └─ unstable_branch ─ 실패 → 재개 시 이 노드만 다시 실행
```

```python
class ParallelState(TypedDict):
    request: str
    prepared: str
    stable_result: str
    unstable_result: str
    summary: str


call_count = {"prepare": 0, "stable": 0, "unstable": 0}
failure_gate = {"allow_unstable": False}


def prepare(state: ParallelState):
    call_count["prepare"] += 1
    return {"prepared": state["request"].strip()}


def stable_branch(state: ParallelState):               # 항상 성공하는 분기
    call_count["stable"] += 1
    return {"stable_result": f"검증 완료: {state['prepared']}"}


def unstable_branch(state: ParallelState):             # gate에 따라 실패하는 분기
    call_count["unstable"] += 1
    if not failure_gate["allow_unstable"]:
        raise ConnectionError("외부 서비스에 일시적으로 연결할 수 없습니다.")
    return {"unstable_result": f"조회 완료: {state['prepared']}"}


def summarize(state: ParallelState):                   # 두 분기 결과를 합치는 노드
    return {"summary": f"{state['stable_result']} | {state['unstable_result']}"}


builder = StateGraph(ParallelState)
builder.add_node("prepare", prepare)
builder.add_node("stable_branch", stable_branch)
builder.add_node("unstable_branch", unstable_branch)
builder.add_node("summarize", summarize)

builder.add_edge(START, "prepare")
builder.add_edge("prepare", "stable_branch")           # prepare에서 두 갈래로 뻗는다
builder.add_edge("prepare", "unstable_branch")         # = 두 노드가 같은 super-step
builder.add_edge("stable_branch", "summarize")
builder.add_edge("unstable_branch", "summarize")
builder.add_edge("summarize", END)

parallel_graph = builder.compile(checkpointer=InMemorySaver())
config = {"configurable": {"thread_id": "fault-parallel-1"}}


# ── 실패 상태 확인 ──
try:
    parallel_graph.invoke({"request": "주문 상태 확인"}, config)
except ConnectionError as error:
    print(f"실행 실패: {error}")

failed_snapshot = parallel_graph.get_state(config)
print("저장된 State:", failed_snapshot.values)          # stable_result는 남아 있다
print("다음 노드:", failed_snapshot.next)                # ('unstable_branch',)
for task in failed_snapshot.tasks:                      # tasks로 분기별 성공/실패 확인
    print({"name": task.name, "error": task.error, "result": task.result})


# ── 재개 ──
failure_gate["allow_unstable"] = True
result = parallel_graph.invoke(None, config)
print(result["summary"])
print("호출 횟수:", call_count)                          # prepare=1, stable=1, unstable=2
print("다음 노드:", parallel_graph.get_state(config).next)   # () = 실행할 노드 없음
```

`stable=1`이 핵심이다. 성공한 분기는 반복하지 않는다. `next=()`는 **실행할 다음 노드가 없다**는 뜻, 즉 정상 완료다.

### 재시도와 재개 비교

| 구분 | 자동 재시도 | checkpoint 재개 |
| --- | --- | --- |
| 실행 범위 | 한 노드를 다시 시도 | 저장된 그래프 실행을 이어서 진행 |
| 호출 코드에서 그래프를 다시 호출해야 하는가 | 아니요 | 예: `invoke(None, config)` |
| 설정 | `RetryPolicy` | checkpointer와 `thread_id` |

**둘은 경쟁 관계가 아니라 계층 관계다.** 짧은 일시적 오류는 제한된 횟수만 자동 재시도하고, 끝까지 실패하면 나중에 checkpoint에서 재개하는 방식으로 **함께** 사용한다.

### 노드 실행 시간 제한하기 (TimeoutPolicy)

외부 서비스의 응답이 늦어 후속 작업이 계속 기다리는 상황을 막는다.

- `run_timeout`은 **노드 한 번의 실행 시간**을 제한한다
- **비동기(async) 노드에만 적용된다** — 동기 함수는 중간에 끊을 수단이 없기 때문
- `NodeTimeoutError`를 재시도 대상으로 지정하면 새 시도에서 시간 제한이 다시 시작된다

```python
class TimeoutState(TypedDict):
    response: str


timeout_count = {"attempts": 0}


async def slow_service(state: TimeoutState):           # async 함수여야 timeout이 걸린다
    timeout_count["attempts"] += 1
    print(f"시도 {timeout_count['attempts']} 시작: 작업은 3초, 제한은 2초")
    await asyncio.sleep(3)                             # 3초 걸리는 작업을 흉내
    return {"response": "완료"}


timeout_builder = StateGraph(TimeoutState)
timeout_builder.add_node(
    "slow_service",
    slow_service,
    timeout=TimeoutPolicy(run_timeout=2.0),            # 2초 넘으면 끊는다
    retry_policy=RetryPolicy(
        initial_interval=1.0, backoff_factor=1.0, max_interval=1.0,
        max_attempts=2, jitter=False,
        retry_on=NodeTimeoutError,                     # 시간 초과도 재시도 대상으로
    ),
)
timeout_builder.add_edge(START, "slow_service")
timeout_builder.add_edge("slow_service", END)
timeout_graph = timeout_builder.compile()

try:
    await timeout_graph.ainvoke({})                    # async 노드라 ainvoke를 쓴다
except NodeTimeoutError as error:
    print("두 번 모두 시간 초과로 실패했습니다.")
    print("시간 초과 노드:", error.node)
    print("시간 초과 종류:", error.kind)
print("총 시도 횟수:", timeout_count["attempts"])
```

재시도해도 시간 초과가 계속되면 작업 성격에 따라 대응한다.

- **실패 안내**: "응답 시간이 초과되었습니다. 잠시 후 다시 시도해 주세요"라고 안내하고 재시도 버튼 제공
- **대체 처리**: 다른 모델로 전환해 응답
- **백그라운드 처리**: 오래 걸려도 완료해야 하는 작업은 worker가 처리하고, 완료되면 사용자에게 알림

### 재시도 소진 후 오류 처리하기 (error_handler)

오류가 났을 때의 처리 순서는 이렇다.

```text
노드 오류 발생
   → RetryPolicy가 재시도 여부를 판단
   → 재시도 대상이 아니거나 횟수를 소진하면
   → error_handler 실행
```

`error_handler`는 `NodeError`에서 노드 이름과 예외를 읽고, **`Command`로 State를 갱신하거나 다음 노드로 이동**할 수 있다.

> 여기서 [Handoff 때 배운 `Command`](2026-09-18_Handoff(Handoff_Tool,_Command.PARENT,_ToolRuntime).md)가 그대로 재등장한다. 새 개념이 아니라, `update`로 실패 상태를 적고 `goto`로 마무리 노드로 보내는 응용이다.

```python
class RecoveryState(TypedDict):
    status: str
    error_message: str


recovery_count = {"attempts": 0, "handled": 0}


def failing_service(state: RecoveryState):             # 계속 실패하는 노드
    recovery_count["attempts"] += 1
    print(f"call_service 시도 {recovery_count['attempts']}: 연결 실패")
    raise ConnectionError("서비스 연결 실패")


def service_error_handler(state: RecoveryState, error: NodeError) -> Command:
    #                                            ↑ NodeError에 노드 이름과 예외가 담겨 온다
    recovery_count["handled"] += 1
    print("error_handler 실행: 실패 상태를 기록하고 finalize로 이동")
    return Command(
        update={"status": "failed", "error_message": f"{error.node}: {error.error}"},
        goto="finalize",                               # 실패 처리 경로로 보낸다
    )


def finalize(state: RecoveryState):                    # 성공/실패 공통 마무리 노드
    print(f"finalize 실행: status={state['status']}")
    return {"status": state["status"]}


recovery_builder = StateGraph(RecoveryState)
recovery_builder.add_node(
    "call_service",
    failing_service,
    retry_policy=RetryPolicy(
        initial_interval=2.0, backoff_factor=2, max_interval=1.0,
        max_attempts=2, jitter=False, retry_on=ConnectionError,
    ),
    error_handler=service_error_handler,               # 재시도 소진 후 여기로
)
recovery_builder.add_node("finalize", finalize)
recovery_builder.add_edge(START, "call_service")
recovery_builder.add_edge("finalize", END)
recovery_graph = recovery_builder.compile()

recovery_result = recovery_graph.invoke({})
print("결과:", recovery_result)
print("호출 횟수:", recovery_count)
```

**`error_handler`가 있으면 예외가 호출 코드까지 올라가지 않는다.** 그래프 안에서 실패를 흡수해 정상 흐름으로 마무리하는 것이다.

### 공통 노드 정책 설정하기

여러 노드에 같은 정책을 적용하되 일부만 다르게 하려면 `set_node_defaults()`를 쓴다. **노드에 직접 지정한 값이 공통 설정보다 우선한다.**

```python
builder.set_node_defaults(                             # 모든 노드의 기본값
    retry_policy=RetryPolicy(max_attempts=3),
    error_handler=service_error_handler,
)
builder.add_node("call_api", call_api)                 # 기본값 적용 → 최대 3번
builder.add_node("validate", validate, retry_policy=RetryPolicy(max_attempts=1))
#                                      ↑ 개별 지정이 우선 → 1번만
```

### 중복 저장 방지와 멱등성

**이 절이 27번 파일에서 실무적으로 가장 중요한 부분이다.**

DB 저장이나 메시지 전송은 그래프 State **밖에도** 변화를 남기는 작업이다. 이런 것을 **side effect**(부작용, 외부 효과)라 한다.

문제는 이것이다.

> **checkpoint는 이미 수행한 외부 작업을 되돌리지 않는다.**
> 실패한 노드는 재개 시 **처음부터** 다시 실행되므로, 오류 발생 전에 완료한 저장 코드도 **다시 실행되어 중복 기록**이 생길 수 있다.

```text
[1차 실행]  DB에 주문 저장 성공 → 응답 받기 직전 연결 끊김 → 노드 실패
[재개]      노드를 처음부터 다시 실행 → DB에 주문을 또 저장?!  ← 중복
```

**멱등성**(idempotency)은 **같은 요청을 여러 번 처리해도 한 번 처리했을 때와 같은 결과를 유지하는 성질**이다. 같은 주문의 저장 요청을 두 번 받아도 주문 기록이 한 건만 남도록 하는 것.

요청을 구별하는 **멱등성 키**(idempotency key)로 이미 처리한 요청인지 확인해 중복 저장을 막는다.

```python
class SaveState(TypedDict):
    request_id: str        # ← 이것이 멱등성 키 역할을 한다
    content: str
    saved: bool


external_rows: list = []                  # 외부 DB를 단순화한 것
processed_request_ids: set = set()        # 이미 처리한 요청 ID 목록
save_gate = {"allow_completion": False}


def idempotent_save(state: SaveState):
    request_id = state["request_id"]
    # 이전에 진행한 요청인지 확인한다
    # 즉, 노드가 2번 실행되어도 외부 서비스는 1번만 실행된다
    if request_id not in processed_request_ids:
        # 외부 서비스와 소통 (DB 저장, 이메일 발송 등)
        external_rows.append({"request_id": request_id, "content": state["content"]})
        processed_request_ids.add(request_id)
    if not save_gate["allow_completion"]:
        raise ConnectionError("저장 후 응답을 받지 못했습니다.")   # 저장은 됐는데 응답 실패
    return {"saved": True}


save_builder = StateGraph(SaveState)
save_builder.add_node("save", idempotent_save)
save_builder.add_edge(START, "save")
save_builder.add_edge("save", END)
save_graph = save_builder.compile(checkpointer=InMemorySaver())
save_config = {"configurable": {"thread_id": "save-1"}}

try:
    save_graph.invoke({"request_id": "req-2026-001", "content": "주문 기록"}, save_config)
except ConnectionError as error:
    print(f"실행 실패: {error}")
    print("실패 직후 기록 수:", len(external_rows))       # 1

    save_gate["allow_completion"] = True                 # 완료 응답을 받을 수 있는 상황
    save_result = save_graph.invoke(None, save_config)   # 재개
    print("재개 결과:", save_result)                      # saved=True
    print("재개 후 기록 수:", len(external_rows))         # 1 — 중복되지 않았다!
```

> ⚠️ 위 목록과 집합은 **한 프로세스 안에서만** 중복을 막는다. 실제 저장소에서는
> ① 저장 결과와 처리한 요청 ID가 **둘 다 기록되거나 둘 다 기록되지 않도록** 트랜잭션으로 함께 처리하거나,
> ② 외부 서비스가 제공하는 **멱등성 키 기능**을 사용한다.

### 실습: 재시도 후 실패 결과 남기기

> 외부 조회 서비스에 연결 오류가 계속 발생하고 있다. 잠시 기다렸다 다시 요청하되, **세 번 시도해도 실패하면 실패 사유를 남기고 작업을 마무리**해야 한다.

```python
class State(TypedDict):
    status: str
    error_message: str
    is_success: bool


def practice_call_service(state: State):
    if state['is_success']:
        return {'state': 'success'}
    else:
        print(f"조회 시도 - 연결 실패")
        raise ConnectionError("외부 조회 서비스 연결 실패")   # raise해야 재시도가 걸린다


def service_error_handler(state: State, error: NodeError) -> Command:
    print("error_handler 실행: 실패 상태를 기록하고 finalize로 이동")
    return Command(
        update={"status": "failed", "error_message": f"{error.node}: {error.error}"},
        goto="finalize",
    )


def finalize(state: State):
    print('마무리 작업')
    return {}


builder = StateGraph(State)
builder.add_node(
    'practice_call_service',
    practice_call_service,
    retry_policy=RetryPolicy(
        initial_interval=2.0, backoff_factor=2.0, max_interval=1.0,
        max_attempts=3, jitter=True, retry_on=ConnectionError,   # 총 3번 시도
    ),
    error_handler=service_error_handler,                          # 소진 후 실패 기록
)
builder.add_node('finalize', finalize)

builder.add_edge(START, 'practice_call_service')
builder.add_edge('practice_call_service', 'finalize')
builder.add_edge('finalize', END)

graph = builder.compile()
result = graph.invoke({'is_success': False})
print(result)
```

> 💡 `add_edge('practice_call_service', 'finalize')`와 `error_handler`의 `goto="finalize"`가 **둘 다 finalize를 가리킨다.** 성공하면 Edge를 타고, 실패하면 `error_handler`가 보낸다. 성공/실패가 같은 마무리 노드로 모이는 구조다.

---

## 주제 2: Agent Tool 실행 검증과 통제

### 문제의식

Agent는 사용자 요청에 따라 **실행할 Tool과 인자를 스스로 선택한다.** 그런데 그 선택은 LLM이 한 것이다.

> **Tool을 실행하는 프로그램은 인자, 사용자 권한, 승인 여부를 확인하고 허용된 작업만 실행해야 한다.**

계좌 조회와 송금을 예로 들면 위험이 분명해진다. LLM이 "acct-999를 조회하자", "150만 원을 송금하자"고 제안했다고 해서 그대로 실행하면 사고다. **LLM의 제안은 제안일 뿐, 실행 권한은 프로그램에 있어야 한다.**

### Middleware

**Middleware**는 모델이나 Tool 호출 **전후에 공통 처리를 적용**하는 기능이다. `create_agent`에 등록하면 Agent가 실행 과정에서 해당 Middleware를 호출한다.

- middle(중간) + ware(제품, 소프트웨어) → **"중간에 끼어드는 소프트웨어"**
- 요청과 실제 처리 **사이에 끼어** 검사·변형·차단을 하는 계층을 뜻하는 일반적인 용어다 (웹 프레임워크에도 동일 개념이 있다)

#### 실행 흐름

```text
agent 시작
   │
   ▼
before_agent
   │
   ▼
before_model
   │
   ▼
┌─────────────────┐
│   model call    │  ← wrap_model_call
└─────────────────┘
   │
   ▼
after_model
   │
   ├── tool call 있음
   │       │
   │       ▼
   │   tool execution  ← wrap_tool_call
   │       │
   │       └──────────→ 다시 before_model
   │
   └── tool call 없음
           │
           ▼
      after_agent
```

- `before_agent` / `after_agent`: Agent 실행의 **시작과 종료** 시점
- 모델·Tool 관련 Middleware: **반복 과정**에서 호출됨
- `wrap_model_call` / `wrap_tool_call`: 해당 호출을 **감싸서** 전후를 처리하거나 **실행을 차단**할 수 있다
- Tool 호출이 여러 개면 `wrap_tool_call`은 **각 호출마다** 적용된다

#### request와 handler

`@wrap_tool_call`을 붙인 함수에 LangChain이 두 가지를 전달한다.

| 인자 | 정체 |
| --- | --- |
| `request` | 현재 호출 정보를 담은 객체. Tool의 이름과 인자는 `request.tool_call`에서 확인 |
| `handler` | 전달받은 요청으로 **다음 처리를 진행하는 함수** |

`handler(request)`를 호출하면 다음 Middleware 또는 **실제 Tool 실행**으로 이어진다.

```python
@wrap_tool_call
def my_middleware(request, handler):
    # ── 여기가 "실행 전" ──  인자·권한·승인을 검사
    message = handler(request)      # ← 이 줄이 실제 Tool 실행
    # ── 여기가 "실행 후" ──  반환값을 검사
    return message
```

**실행을 차단하려면 `handler`를 호출하지 않고 사유를 담은 `ToolMessage`를 반환한다.**

> ⚠️ 실행 후 검사는 **이미 수행된 작업을 되돌리지 못한다.** 송금처럼 돌이킬 수 없는 작업은 반드시 **실행 전**에 막아야 한다.

### Tool 인자 검증하기 (Pydantic)

`Pydantic`은 타입과 제약 조건을 선언하면 **값이 그 조건에 맞는지 자동으로 검사**해주는 라이브러리다.

```python
class LookupAccountArgs(BaseModel):                    # 계좌 조회에 필요한 인자
    account_id: str = Field(min_length=3, max_length=40)


class TransferArgs(BaseModel):                         # 송금에 필요한 인자
    account_id: str = Field(min_length=3, max_length=40)
    amount: int = Field(gt=0, le=1000000)              # gt: greater than, le: less or equal
    request_id: str = Field(min_length=8, max_length=80)
```

- `gt=0` — 0보다 커야 함 (음수 송금 차단)
- `le=1000000` — 100만 원 이하 (형식 수준의 상한)

### State와 Context 구분 (중요)

**이 구분이 28번의 핵심 개념이다.**

| 구분 | 전달하는 주체 | 예 |
| --- | --- | --- |
| **State** | Agent가 작업하면서 갱신 | 대화 기록 |
| **Context** | **프로그램이 Agent 실행 시 전달** | 사용자 권한, 승인된 요청 ID |

| 구분 | 만드는 주체 | 예 |
| --- | --- | --- |
| **Tool 인자** | **LLM**이 호출을 제안할 때 생성 | 조회할 계좌 ID, 송금할 금액 |
| **실행 Context** | **프로그램**이 Agent 실행 시 전달 | 접근 가능한 계좌, 승인된 요청 ID |

> **LLM이 어떤 계좌를 조회하자고 제안하더라도, 실행 여부는 프로그램이 전달한 권한 정보로 판단한다.**

```text
agent.invoke(..., context=execution_context)   → 실행할 때 프로그램이 실제 값 전달
request.runtime.context                        → Middleware에서 전달받은 값 사용
```

**이것이 보안의 핵심이다.** 권한 정보가 LLM이 만질 수 있는 곳(Tool 인자)에 있으면 위조될 수 있다. 프로그램만 쓰는 통로(Context)에 두어야 안전하다.

### 실행 권한과 승인 확인하기

`validate_proposal`은 Tool 이름, 인자 형식, 계좌 접근 권한, 송금 한도, 승인 여부를 **순서대로** 검사한다.

- 통과 → **검증된 인자 객체**를 반환
- 거절 → 처리 상태와 사유를 담은 **딕셔너리**를 반환

반환 타입이 다르다는 점이 중요하다. 호출한 쪽에서 `isinstance(result, dict)`로 통과/거절을 구분한다.

```python
TOOL_SCHEMAS = {'lookup_account': LookupAccountArgs, 'transfer': TransferArgs}

# 실제 송금에 적용할 업무 한도는 10만 원이다
MAX_TRANSFER = 100000


def failure(code: str, message: str) -> dict:          # 거절 사유를 만드는 도우미
    return {'status': 'failed', 'code': code, 'message': message, 'data': {}}


def validate_proposal(proposal: dict, context: dict):
    # 사용자가 취소한 요청은 검사와 실행을 중단한다
    if context['cancelled']:
        return {'status': 'cancelled', 'code': 'USER_CANCELLED',
                'message': '요청이 취소되었습니다.', 'data': {}}

    # Tool 이름에 맞는 검증 모델을 찾고, 등록되지 않은 이름은 거절한다
    schema = TOOL_SCHEMAS.get(proposal['name'])
    if schema is None:
        return failure('UNKNOWN_TOOL', '허용되지 않은 Tool입니다.')

    # 선택한 Tool의 모델로 필수 인자와 값의 범위를 검사한다
    try:
        args = schema.model_validate(proposal['args'])
    except ValidationError:
        return failure('INVALID_ARGUMENT', 'Tool 인자가 올바르지 않습니다.')

    # 사용자에게 접근 권한이 있는 계좌인지 확인한다 (Context 사용)
    if args.account_id not in context['allowed_accounts']:
        return failure('FORBIDDEN', '이 계좌에 접근할 권한이 없습니다.')

    # 송금에는 업무 한도와 승인 검사를 추가로 적용한다
    if proposal['name'] == 'transfer':
        # 인자 형식(100만 원)을 통과해도 업무 한도(10만 원)를 넘으면 거절한다
        if args.amount > MAX_TRANSFER:
            return failure('LIMIT_EXCEEDED', '송금 한도를 초과했습니다.')

        # 승인이 거절된 요청은 취소 상태로 종료한다
        if args.request_id in context['rejected_request_ids']:
            return {'status': 'cancelled', 'code': 'APPROVAL_REJECTED',
                    'message': '승인이 거절되었습니다.', 'data': {}}

        # 승인 기록이 없으면 실행하지 않고 승인을 기다린다
        if args.request_id not in context['approved_request_ids']:
            return {'status': 'waiting_for_human', 'code': 'APPROVAL_REQUIRED',
                    'message': '송금 승인을 기다립니다.',
                    'data': {'request_id': args.request_id, 'amount': args.amount}}

    # 모든 검사를 통과한 Pydantic 인자 객체를 반환한다
    return args
```

> 💡 **형식 한도와 업무 한도가 다르다는 점을 눈여겨볼 것.** Pydantic의 `le=1000000`(100만)은 **형식상 말이 되는 범위**이고, `MAX_TRANSFER=100000`(10만)은 **업무 규칙**이다. 형식 검증과 비즈니스 규칙은 층이 다르므로 분리해서 검사한다.

### Tool과 모델 준비하기

```python
account_responses = {
    "acct-001": {"account_id": "acct-001", "balance": 80000, "currency": "KRW"},
}


@tool("lookup_account", args_schema=LookupAccountArgs)   # args_schema로 Pydantic 모델 연결
def lookup_account_tool(account_id: str):
    """계좌 ID로 잔액을 조회한다."""
    return account_responses[account_id].copy()


@tool("transfer", args_schema=TransferArgs)
def transfer_tool(account_id: str, amount: int, request_id: str):
    """계좌 ID, 금액, 요청 ID로 송금을 요청한다."""
    pass          # 본문은 비워두고 승인에 따른 실행 여부만 확인한다


tools = [lookup_account_tool, transfer_tool]
```

### 모델 호출 통제하기 (wrap_model_call)

`@wrap_model_call`은 **모델 호출**을 감싸는 Middleware를 만든다. `request.override(...)`로 호출 설정을 바꾼 요청을 만들어 `handler`에 전달한다.

```python
@wrap_model_call
def force_tool_call(request, handler):
    # 첫 요청에서 Tool 호출을 강제하고, 결과를 받은 뒤에는 안내만 생성한다
    if isinstance(request.messages[-1], ToolMessage):   # 마지막이 Tool 결과라면
        tool_choice = "none"                            # 추가 호출 금지, 안내만
    else:
        tool_choice = "any"                             # 반드시 Tool을 호출하게 강제

    return handler(request.override(tool_choice=tool_choice))
```

`tool_choice`는 LLM에게 Tool 호출을 어떻게 시킬지 정하는 옵션이다: `"any"`(무조건 하나 호출), `"none"`(호출 금지), 기본값(알아서 판단).

### 실행 전 검증 Middleware

```python
execution_context = {                       # 프로그램이 준비하는 실행 기준
    "user_id": "user-1",
    "allowed_accounts": {"acct-001"},       # 이 사용자가 접근 가능한 계좌
    "approved_request_ids": set(),          # 승인된 송금 요청 ID
    "rejected_request_ids": set(),          # 거절된 송금 요청 ID
    "cancelled": False,
}


@wrap_tool_call
def validate_tool_call(request: ToolCallRequest, handler):
    print("실행 전 검사:", request.tool_call["name"])
    result = validate_proposal(request.tool_call, request.runtime.context)
    #                          ↑ LLM이 만든 제안      ↑ 프로그램이 준 권한 정보

    # 거절·취소·승인 대기이면 Tool 실행 없이 사유를 반환한다
    if isinstance(result, dict):            # dict면 거절 (통과하면 Pydantic 객체)
        return ToolMessage(                 # handler를 부르지 않으므로 Tool은 실행 안 됨
            content=json.dumps(result, ensure_ascii=False),   # 한글이 깨지지 않게
            tool_call_id=request.tool_call["id"],
            name=request.tool_call["name"],
            status="error",
        )

    print("Tool 실행:", request.tool_call["name"])
    return handler(request)                 # 통과한 경우에만 실제 실행


agent = create_agent(
    model=llm,
    tools=tools,
    system_prompt=(
        "사용자가 지정한 계좌 ID와 금액으로 Tool 호출을 제안하세요. "
        "Tool 결과에 근거해 한국어로 안내하세요. "
        "오류가 있으면 사유를 안내하고 잔액이나 성공 여부를 추측하지 마세요."
    ),
    middleware=[force_tool_call, validate_tool_call],   # 등록 순서가 적용 순서
)

result = agent.invoke(
    {"messages": [HumanMessage(content="acct-991 계좌의 잔액을 조회해 줘.")]},
    context=execution_context,              # ★ 여기서 Context를 전달한다
)
```

권한 없는 `acct-991`을 요청하면 실행 전 검사에서 `FORBIDDEN`이 반환되고, `Tool 실행`은 출력되지 않는다. 마지막에 LLM이 거절 사유를 안내한다.

#### 거절 조건 모아보기

```python
context = {'user_id': 'user-1', 'allowed_accounts': {'acct-001'},
           'approved_request_ids': set(), 'rejected_request_ids': set(), 'cancelled': False}

cases = [
    # 미등록 Tool: 허용 목록에 없는 Tool을 요청한다
    {'name': 'delete_account', 'args': {'account_id': 'acct-001'}},
    # 잘못된 인자: 계좌 ID가 최소 길이(3)보다 짧다
    {'name': 'lookup_account', 'args': {'account_id': 'x'}},
    # 권한 없음: 접근할 수 없는 계좌를 조회한다
    {'name': 'lookup_account', 'args': {'account_id': 'acct-999'}},
    # 한도 초과: 업무 한도인 10만 원을 넘겨 송금한다
    {'name': 'transfer', 'args': {'account_id': 'acct-001', 'amount': 150000,
                                  'request_id': 'request-01'}},
    # 승인 정보 위조: 인자에 approved=True를 넣어도 실제 승인으로 인정하지 않는다
    {'name': 'transfer', 'args': {'account_id': 'acct-001', 'amount': 10000,
                                  'request_id': 'request-02', 'approved': True}},
]
for proposal in cases:
    print(proposal['name'], validate_proposal(proposal, context))
```

> 🔐 **마지막 케이스가 이 절의 하이라이트다.** LLM이 인자에 `approved=True`를 넣어도 **승인으로 인정되지 않는다.** 승인 여부는 Tool 인자가 아니라 **Context의 `approved_request_ids`** 로만 판단하기 때문이다. 권한 정보를 LLM이 만질 수 있는 곳에 두면 안 되는 이유가 이것이다.

### 실행 후 결과 검증

결과 검증은 **외부 Tool이나 API처럼 반환값을 보장하기 어려운 경우**에 유용하다. 직접 만든 Tool도 외부 데이터를 사용하면 검증이 필요할 수 있다.

| 검사 | 대상 |
| --- | --- |
| 실행 **전** 검사 | LLM이 **제안한 호출** |
| 실행 **후** 검사 | Tool이 **반환한 데이터** |

```python
class LookupResponse(BaseModel):            # 조회 결과가 갖춰야 할 형태
    account_id: str
    balance: int
    currency: Literal["KRW"]                # KRW만 허용


@wrap_tool_call
def validate_tool_result(request: ToolCallRequest, handler):
    message = handler(request)              # ★ 먼저 Tool을 실행한다

    # 조회 결과만 검사하고, 이미 실패한 결과는 그대로 전달한다
    if request.tool_call["name"] != "lookup_account" or message.status == "error":
        return message

    print("실행 후 검사:", request.tool_call["name"])
    error = None
    try:
        response = LookupResponse.model_validate_json(message.content)
    except ValidationError:
        error = failure("INVALID_TOOL_RESULT", "조회 결과의 필드나 형식이 올바르지 않습니다.")
    else:                                   # try가 성공했을 때만 실행되는 블록
        if response.account_id != request.tool_call["args"]["account_id"]:
            error = failure("RESULT_MISMATCH", "요청한 계좌와 반환된 계좌가 다릅니다.")

    # 잘못된 결과 대신 오류 사유를 LLM에 전달한다
    if error is not None:
        return ToolMessage(
            content=json.dumps(error, ensure_ascii=False),
            tool_call_id=request.tool_call["id"],
            name=request.tool_call["name"],
            status="error",
        )

    return message


result_agent = create_agent(
    model=llm, tools=tools, system_prompt=system_prompt,
    middleware=[force_tool_call, validate_tool_call, validate_tool_result],
)
```

호출은 `validate_tool_call → validate_tool_result → Tool` 순서로 **들어가고**, Tool 결과가 **돌아올 때** `validate_tool_result`가 반환값을 검사한다. 양파 껍질처럼 감싸는 구조다.

**실행 전 검사에서 거절하면 `handler`를 호출하지 않으므로 Tool 실행과 결과 검사도 진행되지 않는다.**

### 호출 횟수 제한하기

`ToolCallLimitMiddleware`는 Tool 호출 횟수를 관리하는 **내장 Middleware**다.

```python
limited_agent = create_agent(
    model=llm, tools=tools, system_prompt=system_prompt,
    middleware=[
        ToolCallLimitMiddleware(run_limit=0, exit_behavior="continue"),
        #                       ↑ 한 번의 Agent 실행에서 허용할 호출 수
        #                                    ↑ 한도 초과 시 모델이 결과를 받아 안내
        force_tool_call,
        validate_tool_call,
        validate_tool_result,
    ],
)
```

- `run_limit=0` → Tool을 아예 실행하지 않는다. 한도 초과 안내가 담긴 `ToolMessage`가 반환된다
- `exit_behavior="continue"` → 모델이 그 결과를 받아 사용자에게 안내할 수 있다
- 호출 횟수는 **새 실행마다 초기화**된다

특정 Tool에만 한도를 적용할 수도 있다.

```python
ToolCallLimitMiddleware(
    tool_name="lookup_account",     # 이 Tool에만 적용
    run_limit=2,                    # 한 번의 실행에서 최대 2회
)
```

### 검사 위치 정리

| 검사 | 실행 시점 | 적용 방식 |
| --- | --- | --- |
| 인자·권한·승인 | Tool 실행 **전** | `@wrap_tool_call`에서 `handler(request)` **호출 전**에 검사 |
| 반환값 | Tool 실행 **후**, LLM 전달 전 | `@wrap_tool_call`에서 `handler(request)`가 **반환한 결과** 검사 |
| 호출 횟수 | 허용 횟수를 넘긴 호출의 실행 전 | `ToolCallLimitMiddleware` |

> **검증 함수는 검사 조건을 정의하고, Middleware는 그 검사를 실행할 시점에 연결한다.**
> 이 한 문장이 28번 전체의 요약이다. 조건(what)과 시점(when)을 분리하는 설계다.

### Tool이 많아질 때

현재 예제는 `validate_proposal` 안에서 Tool 이름으로 `if`를 나누지만, Tool이 많아지면 **검증 함수를 각각 정의하고 이름으로 연결**한다.

```python
TOOL_GUARDS = {                             # Tool 이름 → 전용 검증 함수
    "lookup_account": validate_lookup,
    "transfer": validate_transfer,
}


@wrap_tool_call
def validate_by_tool(request: ToolCallRequest, handler):
    tool_call = request.tool_call
    guard = TOOL_GUARDS.get(tool_call["name"])      # 딕셔너리에서 검증 함수를 꺼낸다

    if guard is None:                                # 등록 안 된 Tool은 거절
        result = failure("UNKNOWN_TOOL", "등록되지 않은 Tool입니다.")
    else:
        result = guard(tool_call["args"], request.runtime.context)

    if isinstance(result, dict):
        return ToolMessage(
            content=json.dumps(result, ensure_ascii=False),
            tool_call_id=tool_call["id"], name=tool_call["name"], status="error",
        )

    return handler(request)
```

> 💡 `if/elif` 체인을 딕셔너리로 바꾸는 이 패턴을 `dispatch table`(디스패치 테이블 — 이름으로 처리 함수를 찾는 표)이라 한다. Tool을 추가할 때 딕셔너리에 한 줄만 넣으면 되고, 검증 함수를 개별 테스트할 수 있다.

### 실습: Context로 Tool 실행 허용하기

> Context에 따라 Tool의 실행을 허용하거나 차단해 보자.

```python
@tool
def get_notice() -> str:
    """안내 문구를 조회한다."""
    print("Tool 실행")
    return "오늘 수업은 오후 6시에 시작합니다."


# tool 실행 전에 context에 따라서 tool을 실행할건지, 안할건지 정하는 함수
@wrap_tool_call
def check_tool_permission(request, handler):
    context = request.runtime.context       # 프로그램이 전달한 Context를 읽는다

    allow = context['allow_tool']

    # 허용이 안되어 있으면 가짜 tool message를 전달한다
    if not allow:
        print('툴 실행 불가')
        return ToolMessage(                 # handler를 부르지 않는다 = Tool 실행 안 됨
            content="실행 불가 함수입니다.",
            tool_call_id=request.tool_call["id"],
            name=request.tool_call["name"],
            status="error",
        )
    print('툴 실행')
    return handler(request)                 # 허용된 경우에만 실제 실행


agent = create_agent(model=llm, tools=[get_notice], middleware=[check_tool_permission])

# 허용된 경우
result = agent.invoke(
    {'messages': [HumanMessage(content="function call을 활용해서 안내 문구를 출력해줘.")]},
    context={'allow_tool': True})
print(result['messages'][-1])

# 차단된 경우
result = agent.invoke(
    {'messages': [HumanMessage(content="function call을 활용해서 안내 문구를 출력해줘.")]},
    context={'allow_tool': False})
print(result['messages'][-1])
```

**같은 질문, 같은 Agent인데 Context만 다르면 결과가 달라진다.** 실행 통제권이 LLM이 아니라 프로그램에 있다는 것을 가장 짧게 보여주는 예제다.

---

## 주제 3: LangGraph 로깅

### 로그와 로깅

프로그램 실행 중 발생한 일을 남긴 **기록을 로그**(log), **기록하는 작업을 로깅**(logging)이라고 한다. 로그에 시각, 실행 단계, 처리 결과를 남기면 실행 흐름과 오류 원인을 확인할 수 있다.

- log = 원래 배의 항해 일지(logbook)에서 온 말. 항해 중 일어난 일을 시간순으로 적은 기록
- Python의 `logging`은 **기본 모듈**이므로 별도 설치 없이 사용할 수 있다

### print() 대신 logging을 쓰는 이유

`print()`로도 실행 과정을 확인할 수 있지만, **출력이 많아지면 필요한 기록만 골라 보기 어렵다.**

`logging`을 사용하면 **설정만으로** 다음을 조절할 수 있다.

- 기록할 **중요도** (레벨)
- 출력 **형식**
- 저장 **위치** (화면, 파일, 여러 곳 동시에)

> 실무 관점에서 이건 취향 문제가 아니다. `print()`는 운영 서버에서 끌 수가 없고, 언제 어디서 찍힌 것인지 알 수 없다. **`print()`로 디버깅한 코드를 그대로 배포하면 리뷰에서 반드시 지적받는다.**

### 로거, 핸들러, 포매터

로깅은 부품 세 개로 이루어진다.

| 부품 | 역할 |
| --- | --- |
| **로거**(Logger) | `logger.info()`처럼 **로그를 기록하는 객체** |
| **핸들러**(Handler) | 로그를 화면이나 파일 등 **지정한 위치로 보낸다.** 하나의 로거에 여러 개 연결 가능 |
| **포매터**(Formatter) | 시각, 레벨, 메시지 등 **표시 형식**을 정한다. 핸들러에 연결해 사용 |

```text
logger.info("메시지")
     │
     ▼
  [Logger]  ← 레벨 검사 1차
     │
     ├──→ [Handler: 화면]  ← 레벨 검사 2차 → [Formatter] → 화면 출력
     └──→ [Handler: 파일]  ← 레벨 검사 2차 → [Formatter] → 파일 저장
```

### 설정 코드

```python
import logging
import sys

logger = logging.getLogger("lesson.langgraph")   # 이 이름의 로거를 가져온다
                                                 # 같은 이름으로 부르면 같은 로거를 얻는다
logger.setLevel(logging.INFO)                    # INFO 이상의 중요도를 기록
logger.propagate = False                         # 상위 로거로 다시 전달하지 않는다

# 설정 셀을 다시 실행해도 같은 로그가 중복 출력되지 않게 한다
for handler in logger.handlers[:]:               # [:]는 목록의 복사본
    logger.removeHandler(handler)                # 순회 중 원본을 지우면 꼬이므로 복사본 사용
    handler.close()                              # 파일 등 사용 중인 자원을 닫는다

console_handler = logging.StreamHandler(sys.stdout)   # 노트북 출력 영역으로 보낸다
formatter = logging.Formatter(
    "%(asctime)s | %(levelname)s | %(name)s | %(message)s",
    datefmt="%H:%M:%S",
)
console_handler.setFormatter(formatter)          # 핸들러에 형식을 연결
logger.addHandler(console_handler)               # 로거에 출력 목적지를 연결

logger.info("로깅 준비 완료")
# 14:30:00 | INFO | lesson.langgraph | 로깅 준비 완료
```

포맷 문자열의 의미:

| 표기 | 내용 |
| --- | --- |
| `%(asctime)s` | 시각 |
| `%(levelname)s` | 레벨 |
| `%(name)s` | 로거 이름 |
| `%(message)s` | 메시지 |

> `.py` 파일에서는 보통 `logging.getLogger(__name__)`으로 **모듈 이름**을 사용한다. 그러면 로그만 봐도 어느 파일에서 나온 것인지 알 수 있다.
>
> `propagate`(전파하다)를 `False`로 두는 이유: 로거는 계층 구조라서 기본적으로 상위 로거에도 메시지를 올려보낸다. 상위에도 핸들러가 붙어 있으면 **같은 로그가 두 번 찍힌다.**

### 로그 레벨로 중요도 나누기

| 레벨 | 기록할 상황 |
| --- | --- |
| `DEBUG` | 검색 결과 개수 등 자세한 진단 정보 |
| `INFO` | 노드 시작, 완료, 선택한 경로 |
| `WARNING` | 검색 결과가 없어 안내 답변으로 전환 |
| `ERROR` | 외부 서비스 오류로 요청 처리 실패 |
| `CRITICAL` | 서비스 전체를 계속 운영하기 어려운 심각한 문제 |

중요도 순서: `DEBUG < INFO < WARNING < ERROR < CRITICAL`

- 레벨을 INFO로 정하면 **DEBUG만 걸러진다**
- WARNING으로 정하면 **WARNING, ERROR, CRITICAL이 남는다**
- **설정하지 않은 기본 루트 로거의 기준은 WARNING이다**

```python
logger.setLevel(logging.WARNING)
logger.info("이 INFO는 보이지 않는다")
logger.warning("이 WARNING은 보인다")

logger.setLevel(logging.DEBUG)
logger.debug("이제 DEBUG도 보인다")

logger.setLevel(logging.INFO)
```

### 변수와 함께 기록하기

f-string으로 변수 값을 로그에 포함할 수 있다.

```python
node_name = "search"
document_count = 2
elapsed_ms = 12.345

logger.info(f"node={node_name} documents={document_count} elapsed_ms={elapsed_ms:.2f}")
#                                                                        ↑ 소수점 2자리
```

> 💡 `key=value` 형태로 쓰는 습관을 들이면 나중에 로그를 **검색하고 집계**하기 쉽다. 실무에서는 이걸 더 밀고 나가 JSON으로 남기는 `structured logging`(구조화 로깅)을 쓴다.

### 화면과 파일에 함께 남기기

화면을 닫은 뒤에도 살펴볼 수 있도록 파일 핸들러를 추가한다.

```python
from pathlib import Path

log_dir = Path.cwd() / "logs"                    # Path.cwd(): 현재 작업 폴더
log_dir.mkdir(exist_ok=True)                     # exist_ok: 이미 있어도 에러 안 냄
log_path = log_dir / "29-langgraph.log"

for handler in logger.handlers[:]:               # 파일 핸들러만 골라 교체
    if isinstance(handler, logging.FileHandler):
        logger.removeHandler(handler)
        handler.close()

file_handler = logging.FileHandler(log_path, mode="a", encoding="utf-8")
#                                            ↑ append: 기존 기록 뒤에 추가
#                                                      ↑ 한글 저장에 필요
file_handler.setFormatter(formatter)
logger.addHandler(file_handler)

logger.info("화면과 파일에 함께 기록")
print("로그 파일:", log_path.resolve())
```

### 화면은 간단하게, 파일은 자세하게 (레벨 이중 검사)

**핵심 규칙:**

> 로그는 **로거의 기준을 통과한 뒤, 각 핸들러의 기준도 통과해야** 출력된다.

즉 **관문이 두 개**다. 로거가 INFO라면 파일 핸들러만 DEBUG로 바꿔도 **DEBUG 기록은 남지 않는다.** 로거에서 이미 걸러졌기 때문이다.

```python
logger.setLevel(logging.DEBUG)          # 1차 관문: 전부 통과시킨다
console_handler.setLevel(logging.INFO)  # 2차 관문(화면): DEBUG는 버린다
file_handler.setLevel(logging.DEBUG)    # 2차 관문(파일): 전부 받는다

logger.debug("파일에서만 보이는 상세 기록")
logger.info("화면과 파일에 모두 보이는 진행 기록")

file_handler.flush()                    # 버퍼에 남은 내용을 파일에 밀어넣는다
print(log_path.read_text(encoding="utf-8"))
```

> ⚠️ **자주 하는 실수:** "파일에 DEBUG를 남기고 싶어서 파일 핸들러만 DEBUG로 바꿨는데 안 나와요" — 로거 레벨이 INFO라서 1차에서 걸러진 것이다. **로거를 가장 낮게(DEBUG) 두고, 핸들러별로 조이는 것**이 올바른 순서다.

### 로그 파일 로테이션

로그를 계속 추가하면 파일이 커진다. `RotatingFileHandler`는 **파일 크기를 기준으로 새 파일로 전환**하고, 이전 파일은 정해진 개수만 보관한다.

- `maxBytes`: 파일을 교체할 크기 기준(바이트)
- `backupCount`: 보관할 이전 파일 개수. **현재 기록 중인 파일은 제외**

```python
from logging.handlers import RotatingFileHandler

logger.removeHandler(file_handler)
file_handler.close()

file_handler = RotatingFileHandler(
    log_path,
    maxBytes=1024,        # 확인을 위해 아주 작게 설정 (실무는 보통 MB 단위)
    backupCount=2,        # .1, .2 두 개까지만 보관
    encoding="utf-8",
)
file_handler.setFormatter(formatter)
file_handler.setLevel(logging.DEBUG)
logger.addHandler(file_handler)

for index in range(40):
    logger.debug(f"파일 회전 확인: 반복 기록 {index}")

file_handler.flush()
for path in sorted(log_dir.glob("29-langgraph.log*")):   # glob: 패턴으로 파일 찾기
    print(path.name)
```

- 현재 기록: `29-langgraph.log`
- 이전 기록: `29-langgraph.log.1`, `29-langgraph.log.2`
- **`.1`이 더 최근 기록**이며, 보관 개수를 넘으면 가장 오래된 파일이 삭제된다

> rotation = 회전, 순환. 파일을 돌려가며 쓴다는 뜻. 서버 디스크가 로그로 가득 차서 서비스가 멈추는 사고는 실무에서 흔하다.

### 오류 메시지와 traceback 기록하기

| 함수 | 남기는 내용 |
| --- | --- |
| `logger.error()` | **메시지만** |
| `logger.exception()` | ERROR 레벨 메시지 + **현재 예외의 traceback** |

**traceback**은 어떤 함수와 코드 위치를 거쳐 오류가 발생했는지 보여 주는 추적 기록이다. `logger.exception()`은 **`except` 블록 안에서만** 의미가 있다.

```python
try:
    int("숫자가 아님")
except ValueError:
    logger.error("정수 변환 실패: 메시지만 기록")
    logger.exception("정수 변환 실패: traceback도 기록")
```

> ⚠️ **기록하는 것과 오류를 처리하는 것은 별개다.**
> `logger.exception()`만으로 예외가 다시 발생하거나 작업이 재시도되지 않는다.
> 호출한 곳에 실패를 전달하려면 **`raise`** 를 써야 한다.

이건 27번의 `RetryPolicy`와도 연결된다. 로그만 남기고 `raise`를 안 하면, 그래프 입장에서는 **정상 반환**이라 재시도도 `error_handler`도 동작하지 않는다.

### LangGraph 노드에 로그 남기기

노드도 결국 Python 함수이므로 **앞에서 만든 로거를 그대로 사용**한다. 새로운 문법은 없다.

```text
START → prepare → answer → END
```

```python
class LoggingState(TypedDict):
    question: str
    answer: str


def prepare(state: LoggingState):
    logger.info("prepare 시작")                # 노드 진입 기록
    question = state["question"].strip()
    logger.info("prepare 완료")                # 노드 종료 기록
    return {"question": question}


def answer(state: LoggingState):
    logger.info("answer 시작")
    try:
        if not state["question"]:              # 빈 문자열이면 실패 처리
            raise ValueError("질문이 비어 있습니다.")
        response = f"질문을 받았습니다: {state['question']}"
    except ValueError:
        logger.exception("answer 실패")        # traceback까지 기록하고
        raise                                  # ★ 예외를 호출한 곳으로 전달한다
    logger.info("answer 완료")
    return {"answer": response}


builder = StateGraph(LoggingState)
builder.add_node("prepare", prepare)
builder.add_node("answer", answer)
builder.add_edge(START, "prepare")
builder.add_edge("prepare", "answer")
builder.add_edge("answer", END)
graph = builder.compile()


# ── 정상 실행 ──
result = graph.invoke({"question": "  LangGraph가 무엇인가요?  "})
print(result["answer"])
# prepare 시작 → prepare 완료 → answer 시작 → answer 완료 순으로 INFO 로그가 남는다


# ── 오류 위치 확인 ──
try:
    graph.invoke({"question": "   "})          # 공백만 입력
except ValueError as error:
    print("실행 실패:", error)
# 로그에 'answer 시작'과 'answer 실패'는 있지만 'answer 완료'는 없다
```

**"시작"은 있는데 "완료"가 없다** — 이것만으로 어느 노드에서 멈췄는지 알 수 있다. 로그를 시작/완료 쌍으로 남기는 이유가 이것이다.

---

## 세 주제가 어떻게 이어지는가

오늘 배운 셋은 따로 노는 주제가 아니라 **운영 가능한 Agent를 만드는 세 축**이다.

| 축 | 질문 | 수단 |
| --- | --- | --- |
| **장애 허용** | 실패했을 때 어떻게 살아남나? | `RetryPolicy`, 재개, `error_handler`, 멱등성 |
| **Tool 통제** | LLM이 위험한 짓을 하면? | Middleware 실행 전/후 검사, Context |
| **로깅** | 무슨 일이 있었는지 어떻게 아나? | `logging`, 레벨, 파일, `exception()` |

그리고 셋이 한 지점에서 만난다. **`raise`를 하느냐 마느냐.**

- 로그만 남기고 `raise`를 안 하면 → 재시도도, `error_handler`도, 재개도 동작하지 않는다
- `{"status": "failed"}`를 **반환**하면 → 정상 반환이므로 장애 허용 장치가 전부 무력화된다

> **"실패를 기록하는 것"과 "실패를 알리는 것"은 다르다.** 이것이 오늘의 가장 중요한 한 줄이다.

### 개선 여지 (실무 관점)

- **재시도 대상 선정이 가장 어렵다.** `ConnectionError`처럼 명백한 일시적 오류만 넣어야 한다. HTTP 상태 코드로 치면 429/503은 재시도, 400/403은 재시도 금지다. 판단이 애매하면 `retry_on`에 **판별 함수**를 넣어 조건을 세밀하게 쓴다.
- **부작용이 있는 노드는 반드시 멱등하게 짠다.** 재개는 노드를 **처음부터** 다시 실행하므로, 저장·전송·결제 코드가 있으면 멱등성 키 없이는 중복 사고가 난다.
- **실행 후 검사로 막을 수 없는 것이 있다.** 송금처럼 되돌릴 수 없는 작업은 **실행 전**에 막아야 한다. 결과 검증은 "잘못된 데이터를 LLM에 넘기지 않기" 용도이지 방어선이 아니다.
- **승인·권한은 절대 Tool 인자로 받지 않는다.** LLM이 채우는 값은 신뢰할 수 없는 외부 입력이다. Context로만 전달한다.
- **로그에 민감 정보를 남기지 않는다.** 계좌번호, 잔액, 개인정보가 로그 파일에 그대로 쌓이면 그 자체가 보안 사고다. 마스킹하거나 ID만 남긴다.
- **운영에서는 로그만으로 부족하다.** LangSmith 같은 추적 도구를 붙이면 어떤 프롬프트로 어떤 Tool이 불렸는지 한 실행 단위로 묶어 볼 수 있다.

---

## ✅ 확인 질문

#### 장애 허용

1. `RetryPolicy`의 `max_attempts=3`은 재시도를 3번 한다는 뜻인가, 총 3번 시도한다는 뜻인가?
2. `backoff_factor`와 `jitter`는 각각 어떤 문제를 막기 위한 설정인가?
3. 노드가 `raise`하지 않고 `{"status": "failed"}`를 반환하면 `RetryPolicy`가 동작하지 않는 이유는?
4. `retry_on=ConnectionError`인 노드에서 `ValueError`가 나면 총 몇 번 시도되며, 그 이유는?
5. `RetryPolicy`, `with_retry()`, `with_fallbacks()`는 각각 무엇을 다시(또는 다르게) 실행하는가?
6. super-step이란 무엇이며, 순차 실행과 병렬 실행에서 checkpoint가 만들어지는 시점은 어떻게 다른가?
7. 재개할 때 `invoke(None, config)`의 `None`은 무슨 의미인가?
8. 실패 후 재개했을 때 `prepare=1`, `call_service=2`가 나오는 이유를 설명하면?
9. 병렬 분기 중 하나만 실패했을 때, 재개하면 성공한 분기가 다시 실행되지 않는 이유는?
10. `get_state(config)`의 `next`가 빈 튜플 `()`이면 무슨 뜻인가?
11. 자동 재시도와 checkpoint 재개를 함께 쓴다면 각각 어떤 상황을 맡게 되는가?
12. `TimeoutPolicy`의 `run_timeout`이 비동기 노드에만 적용되는 이유는 무엇이라고 생각하는가?
13. `error_handler`가 등록된 노드에서 재시도가 모두 실패하면, 예외는 호출 코드까지 올라가는가?
14. `set_node_defaults()`로 지정한 정책과 개별 노드에 지정한 정책이 충돌하면 어느 쪽이 이기는가?
15. checkpoint가 "이미 수행한 외부 작업을 되돌리지 않는다"는 사실이 왜 중복 저장 문제를 만드는가?
16. 멱등성이란 무엇이며, 멱등성 키는 어떤 역할을 하는가?
17. 예제의 `processed_request_ids`가 실제 서비스에서는 충분하지 않은 이유는?

#### Tool 통제

18. Agent가 고른 Tool과 인자를 프로그램이 다시 검증해야 하는 이유는?
19. `@wrap_tool_call` 함수에서 `handler(request)`를 호출하는 것과 호출하지 않는 것은 각각 어떤 결과를 낳는가?
20. Tool 인자와 실행 Context는 각각 누가 만들며, 권한 정보를 Context에 두어야 하는 이유는?
21. LLM이 Tool 인자에 `approved=True`를 넣어도 승인으로 인정되지 않는 구조적 이유는?
22. Pydantic의 `le=1000000`과 코드의 `MAX_TRANSFER=100000`은 왜 값이 다른가? 각각 무엇을 검사하는가?
23. `validate_proposal`이 통과 시에는 객체를, 거절 시에는 딕셔너리를 반환하도록 설계한 이유는?
24. 실행 전 검사와 실행 후 검사의 대상은 각각 무엇인가?
25. 실행 후 검사로는 막을 수 없는 종류의 문제를 예로 들면?
26. `validate_tool_call`이 거절하면 `validate_tool_result`가 실행되지 않는 이유는?
27. `wrap_model_call`에서 `tool_choice`를 `"any"`와 `"none"`으로 번갈아 설정하는 의도는?
28. `ToolCallLimitMiddleware(run_limit=0, exit_behavior="continue")`를 적용하면 어떤 일이 벌어지는가?
29. Tool이 많아졌을 때 `if/elif` 대신 `TOOL_GUARDS` 딕셔너리를 쓰면 무엇이 좋아지는가?

#### 로깅

30. `print()` 대신 `logging`을 쓰면 구체적으로 무엇을 조절할 수 있게 되는가?
31. 로거, 핸들러, 포매터는 각각 무슨 역할을 하며 어떻게 연결되는가?
32. `logger.propagate = False`를 설정하지 않으면 어떤 문제가 생길 수 있는가?
33. 로거 레벨이 INFO인데 파일 핸들러만 DEBUG로 바꾸면 DEBUG 로그가 파일에 남는가? 왜인가?
34. 로그 레벨을 WARNING으로 두면 어떤 레벨들이 기록되는가?
35. `RotatingFileHandler`의 `maxBytes`와 `backupCount`는 각각 무엇을 정하며, `.log.1`과 `.log.2` 중 어느 것이 더 최근인가?
36. `logger.error()`와 `logger.exception()`의 차이는 무엇이며, 후자는 어디에서 써야 하는가?
37. `logger.exception()`을 호출했는데도 그래프가 재시도되지 않는다면 무엇이 빠진 것인가?
38. 노드 로그를 "시작"과 "완료" 쌍으로 남기면 오류 추적에 어떤 이점이 있는가?
