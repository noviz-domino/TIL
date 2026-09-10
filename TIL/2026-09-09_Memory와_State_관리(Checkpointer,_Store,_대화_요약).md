---
tags: [langgraph, checkpointer, store, memory]
til: v2 2026-09-10
---

# Memory와 State 관리 (Checkpointer, Store, 대화 요약)
> 작성일: 2026-09-09

## 🔗 관련 글

- [2026-09-09_ReAct_Agent(Tool,_ToolNode,_tools_condition,_MessagesState)](2026-09-09_ReAct_Agent%28Tool,_ToolNode,_tools_condition,_MessagesState%29.md) — MessagesState와 add_messages Reducer의 뒷부분(오늘 그 위에 Checkpointer로 대화를 저장/복원하는 법을 얹음)

## 이 노트북이 푸는 문제

지금까지 만든 그래프는 `graph.invoke(...)`를 한 번 부르면 그 안에서만 State가 이어졌고, 또 부르면 완전히 새로 시작했다. 실제 챗봇이라면 "아까 뭐라고 했는지"를 기억해야 하는데, 지금까지 배운 것만으로는 그게 안 된다.

> Agent가 대화를 이어가려면 State를 저장하고 복원하는 기능이 필요하다. LangGraph는 이를 위해 Checkpointer(Short-term Memory)와 Store(Long-term Memory)를 제공한다.

| | Checkpointer | Store |
|---|---|---|
| 별명 | 단기 기억(Short-term Memory) | 장기 기억(Long-term Memory) |
| 구분 기준 | thread_id | namespace + key |
| 용도 | 같은 대화를 이어가기 | 여러 대화에 걸쳐 사용자 정보 재사용하기 |

강사님이 정리해주신 핵심 문장이 이 노트북 전체를 관통하는 뼈대다.

> 1. llm을 invoke 하는 과정들은 전부 독립적이다.
> 2. 단, ReAct Agent에서 대화가 연속적으로 보였던 건 하나의 그래프 내부에서 같은 State로 관리됐기 때문이다.
> 3. 같은 그래프라도 여러 번 invoke하면 각각은 독립적이다.
> 4. 단, Checkpointer를 쓰면 여러 번의 invoke가 대화처럼 이어질 수 있다.
> 5. 각 대화는 thread_id로 구분되어 독립적이다.
> 6. 단, Store를 쓰면 특정 정보는 공유할 수 있다.

"기본적으로는 다 독립적이다"가 원칙이고, Checkpointer/Store는 그 독립성에 예외를 뚫어주는 도구라는 시각으로 보면 나머지가 다 이 위에 얹히는 각론이다.

## Checkpointer와 Saver

> Checkpointer는 그래프의 State 스냅샷을 저장하고 복원하는 기능이다. 그래프 실행 시작 시와 각 노드 실행 후에 State를 저장하며, 같은 thread_id로 다시 호출하면 저장된 상태를 바탕으로 실행을 이어간다.
> Saver는 Checkpointer가 생성한 스냅샷을 실제로 저장하는 저장소 구현체다. 저장 위치에 따라 메모리, SQLite, PostgreSQL 등의 Saver를 선택할 수 있으며, 어떤 Saver를 사용하더라도 thread_id로 상태 이력을 구분한다. 이번 실습에서는 메모리에 스냅샷을 저장하는 MemorySaver를 사용한다.

```python
def simple_chatbot(state: MessagesState):
    return {"messages": [llm.invoke(state["messages"])]}

simple_builder = StateGraph(MessagesState)
simple_builder.add_node("chatbot", simple_chatbot)
simple_builder.add_edge(START, "chatbot")
simple_builder.add_edge("chatbot", END)

memory = MemorySaver()
graph_with_memory = simple_builder.compile(checkpointer=memory)
```

`compile()`에 `checkpointer=memory`라는 새 매개변수가 붙었다. `checkpointer`는 `compile()`이 정해둔 고정된 매개변수 이름이라 반드시 이 단어 그대로 써야 한다(실제로 `inspect.signature(StateGraph.compile)`로 확인해보면 `checkpointer`, `store`가 정확히 이 이름으로 박혀 있다). 반면 `memory`는 그냥 우리가 지은 변수 이름이라 `my_saver` 같은 다른 이름으로 지어도 된다.

- `checkpointer` — `compile()`의 고정된 매개변수 이름 (바꾸면 안 됨)
- `memory` — `MemorySaver()` 객체를 담은 우리가 지은 변수 이름 (자유롭게 지어도 됨)

```python
config = {"configurable": {"thread_id": "session-1"}}

result = graph_with_memory.invoke(
    {"messages": [("human", "내 이름은 철수야. 반가워!")]},
    config=config,
)
```

`config`도 그냥 우리가 지은 변수 이름이다. 다만 그 안의 키 이름(`"configurable"`, `"thread_id"`)은 LangGraph가 내부적으로 찾아서 읽는 고정된 키다.

### 저장되는 것과 복원되는 것 — 실제 동작 원리

두 번째 invoke가 "기억"이 실제로 일어나는 지점이다.

```python
result = graph_with_memory.invoke(
    {"messages": [("human", "내 이름이 뭐라고 했지?")]},
    config=config,   # 같은 config, 같은 thread_id
)
```

같은 thread_id로 부르면, LangGraph가 실행을 시작하기 전에 "session-1"에 저장해둔 가장 최신 State를 먼저 불러와서 복원한다. 그다음, 새로 넘긴 입력이 복원된 State에 **병합**된다. 여기서 이전에 배운 Reducer(`add_messages`)가 등장한다 — 새 메시지가 기존 메시지를 안 지우고 뒤에 이어붙는다고 했었는데, 그래서 두 번째 invoke가 실제로 노드에 넘기는 `state["messages"]`는 다음 세 개가 다 들어있다.

1. "내 이름은 철수야. 반가워!" (1번째 질문, 복원됨)
2. (1번째 AI 답변, 복원됨)
3. "내 이름이 뭐라고 했지?" (방금 새로 들어온 질문)

`llm.invoke(state["messages"])`가 이 셋을 다 LLM에게 보내니까, LLM이 "아까 철수라고 했었지"를 알 수 있는 것이다.

- 저장 — 대화 끝날 때마다 checkpointer가 State를 thread_id에 붙여 저장
- 복원 — 같은 thread_id로 다시 부르면, 실행 전에 그 State를 먼저 불러와서 새 입력과 합침(add_messages)
- 그 결과 — LLM은 매번 "새 질문 하나"가 아니라 "지금까지의 대화 전체"를 받음

> ♻️ 처음엔 "LLM이 기억을 해서 이전 대화를 안다"고 생각하기 쉬웠으나, 정확히는 LLM 자체는 여전히 아무것도 기억하지 않는다. 매번 대화 기록 전체를 처음부터 다시 통째로 넘겨받아서, 마치 하나도 안 잊은 것처럼 보이는 것뿐이다. Checkpointer가 하는 일은 정확히 "그 통째로 넘길 기록을 어디 저장해뒀다가 다시 꺼내오는 것"뿐이다.

**LLM에게 실제로 보내지는 건 "주소"가 아니라 "진짜 대화 내용"이다.** thread_id, checkpoint_id 같은 "주소" 개념은 LangGraph 내부에서 어느 체크포인트를 꺼내올지 찾는 데만 쓰인다. 일단 그 주소로 원하는 체크포인트를 찾아서 실제 State(진짜 메시지 텍스트들)를 복원하고 나면, 그다음부터는 그 진짜 텍스트 내용이 새 질문과 합쳐져서 `llm.invoke()`로 넘어간다. LLM은 thread_id나 checkpoint_id라는 게 있는지조차 모른다.

- 주소(thread_id, checkpoint_id) — LangGraph가 "어느 저장된 State를 꺼내올지" 찾는 데만 씀 (LLM은 이걸 모름)
- 실제로 LLM에게 가는 것 — 그 주소로 찾아낸, 진짜 대화 텍스트 내용 (복원 + 병합된 것)

### 체크포인트란 정확히 무엇인가

"체크포인트"는 한 시점에 찍힌 State의 스냅샷 그 자체다. "Checkpointer"(그 스냅샷을 찍고 관리하는 기능)와는 다른 개념이다.

- 체크포인트(checkpoint) — 어느 한 순간의 State 내용을 그대로 찍어둔 것 (데이터 자체)
- Checkpointer — 그 체크포인트를 언제, 어떻게 찍고 저장할지 관리하는 기능 (시스템)

체크포인트는 그래프가 실행을 시작할 때 한 번, 그리고 각 노드가 끝날 때마다 한 번씩 찍힌다. 그래서 그래프 하나를 실행하는 동안에도 노드가 여러 번 실행되면 체크포인트가 여러 개 생긴다.

```python
state = graph_with_memory.get_state(config)
state.values  # {"messages": [...]}
```

`get_state(config)`는 그 thread_id에 저장된 **가장 최신 체크포인트**를 가져온다. `state`는 평범한 dict가 아니라 `StateSnapshot`이라는 객체인데, 그 안의 `.values` 속성에 실제 State 데이터가 들어있다.

```python
history = list(graph_with_memory.get_state_history(config))
```

`get_state_history(config)`는 그 thread_id에 쌓인 **모든** 체크포인트를 최신순으로 꺼내준다.

- `get_state()` — 가장 최신 체크포인트 하나만
- `get_state_history()` — 그 thread_id에 쌓인 모든 체크포인트를 최신순으로

### 체크포인트를 실제로 뜯어보기

체크포인트(`StateSnapshot`)가 어떤 요소로 구성돼 있는지 직접 확인했다.

```python
snap = graph.get_state(config)
snap._fields
# ('values', 'next', 'config', 'metadata', 'created_at', 'parent_config', 'tasks', 'interrupts')
```

- **values** — 그 순간의 State 실제 내용물. `{'count': 1}`처럼 우리가 정의한 State의 필드가 담긴다.
- **config** — 이 체크포인트의 주소. thread_id, checkpoint_id가 들어있다.
- **parent_config** — 이 체크포인트 바로 이전 체크포인트의 주소. 체크포인트들이 부모-자식처럼 사슬(체인)로 연결돼 있다는 뜻이다.
- **metadata** — 이 체크포인트의 부가 정보(`step: 1`처럼 몇 번째 단계인지 등).
- **created_at** — 이 체크포인트가 찍힌 시각.
- next / tasks / interrupts — 사람의 개입(Human-in-the-loop)이나 병렬 실행 관련 정보(지금 단계에서는 안 건드림).

> ➕ **더 알아두기**
> `parent_config`가 있다는 게, 체크포인트들이 단순히 thread_id 아래 여러 개 흩어져 있는 게 아니라 하나의 사슬처럼 순서대로 이어져 있다는 걸 보여준다. 마치 git의 커밋들이 각자 "이전 커밋"을 가리키는 것과 똑같은 구조라, `get_state_history()`가 정확히 시간 순서(최신부터)로 되짚어갈 수 있는 것이다.

### 헷갈렸던 것들을 하나씩 정리

> ♻️ **thread_id는 노드가 끝날 때마다 새로 생기는 게 아니다.** 새로 생기는 건 thread_id가 아니라, 각 체크포인트마다 붙는 별도의 고유 번호(checkpoint_id)다. 직접 확인했다.
> ```python
> config = {"configurable": {"thread_id": "thread-A"}}
> graph.invoke({"count": 0}, config)
> graph.invoke({"count": 10}, config)
> for snap in graph.get_state_history(config):
>     print(snap.config["configurable"]["thread_id"], snap.config["configurable"]["checkpoint_id"])
> ```
> 결과: thread_id는 6개 체크포인트 모두 "thread-A"로 동일했고, checkpoint_id는 매번 완전히 달랐다(`1f1ac17f-f534-...`, `1f1ac17f-f533-...` 등). **thread_id가 큰 서랍이고, 그 안에 checkpoint_id가 다른 체크포인트 여러 개가 담기는 구조**다.

> ♻️ **체크포인트는 "메시지 하나"가 아니다.** State 안의 `messages`는 리스트이고, 체크포인트는 그 리스트 전체(그 순간까지 쌓인 메시지 전부)를 담은 스냅샷이다. 대화가 진행될수록 체크포인트 안의 메시지 개수는 계속 늘어난다(1번째 체크포인트엔 2개, 2번째엔 4개, ...). "체크포인트 1개 = 메시지 1개"가 아니라 "체크포인트 1개 = 그 순간까지의 메시지 리스트 전체"다.
>
> 크기 순서로 보면 이렇다: **체크포인트 > State의 필드들(messages, summary 등) > 메시지(messages 리스트 안의 개별 항목)**

> ♻️ **`values` 안의 숫자가 체크포인트의 순서를 나타내는 게 아니다.** 예제 State가 우연히 `count: int` 필드 하나였고 그 값이 1씩 늘어나서 순서처럼 보였을 뿐이다. 진짜로 "몇 번째 체크포인트인지"를 나타내는 건 `metadata`의 `step` 필드다. `values`는 그때그때 State가 어떤 필드를 갖고 있느냐에 따라 완전히 다르게 생긴다(순서 정보가 아니라 실제 데이터).
>
> 마찬가지로, 우리가 만든 `count` 같은 State 필드와 LangGraph 내부의 `step`은 완전히 다른 주체가 관리하는 값이다. `count`는 우리 애플리케이션 로직을 위한 데이터, `step`은 LangGraph 자신이 내부적으로 세는 값이다. 노드가 하나뿐인 단순한 예제에서는 둘이 우연히 비슷하게 늘어나 보일 뿐, 노드가 여러 개거나 count를 다르게 늘리면 두 숫자는 서로 안 맞게 된다.

> ♻️ **삭제(RemoveMessage)해도 과거 체크포인트는 안 바뀐다.** 체크포인트는 "그 순간의 완전한 사본"이라, 나중에 뭘 지운다고 과거 스냅샷을 다시 가서 고칠 수는 없다. 요약 노드가 `RemoveMessage`를 반환하면 **새로운 체크포인트가 하나 더 찍히고**, 그 새 체크포인트의 `messages`에는 그 메시지가 빠진 채로 담긴다. 하지만 그 이전 체크포인트들은 전혀 안 바뀐다 — 여전히 그 메시지가 담겨 있다.
> - 과거 체크포인트들 — 그대로 보존됨 (그 메시지가 여전히 담겨있음)
> - 가장 최신 체크포인트 — 그 메시지가 빠진 새 리스트로 새로 찍힘
>
> 그리고 이게 "삭제 이력이 남으니 토큰을 더 쓰는 것 아니냐"는 질문으로 이어졌는데, 답은 아니다. **저장 공간 비용**과 **LLM 토큰 비용**은 완전히 다른 자원이다. 과거 체크포인트가 저장소(RAM/디스크)에 남아있는 건 저장 공간을 쓰는 것뿐이고, 토큰이 소모되는 순간은 오직 `llm.invoke(...)`를 실제로 호출할 때뿐이다. 요약 이후로는 앞으로 매번 가장 최신(요약+최근 메시지만 남은 짧은) 체크포인트만 LLM에게 넘어가므로, 토큰 비용은 그대로 줄어든다. 다만 저장 공간은 계속 쌓이므로, 실무에서는 오래된 체크포인트를 주기적으로 정리하는 정책이 별도로 필요하다.

## Long-term Memory (Store)

> Checkpointer는 thread_id 단위로 대화 State를 저장한다. 반면 Store는 스레드를 넘어 사용자 정보를 저장하는 Long-term Memory다.
>
> 여기서 Long-term은 데이터가 반드시 영구 보존된다는 뜻이 아니라, 하나의 대화 스레드를 넘어 공유된다는 의미다.

- **namespace**: 사용자별 데이터를 묶는 저장 경로
- **key**: namespace 안에서 프로필을 식별하는 값
- **value**: 실제로 저장할 사용자 이름과 관심 분야

> ♻️ Store가 곧 Long-term Memory다 — 별개의 세 번째 개념이 아니라, Store라는 기능에 "장기 기억"이라는 별명을 붙인 것뿐이다.
> - Checkpointer = Short-term Memory (별명)
> - Store = Long-term Memory (별명)

### Store는 언제 사용하는가

> 정보가 현재 대화에만 필요한지, 다른 대화에서도 다시 필요할지를 기준으로 저장 위치를 선택한다.

| 저장할 정보 | 적합한 위치 | 예시 |
|---|---|---|
| 현재 대화의 흐름과 중간 결과 | State와 Checkpointer | 메시지 기록, 현재 작업 단계, 도구 실행 결과 |
| 여러 대화에서 재사용할 정보 | Store | 사용자 프로필, 선호 언어, 관심 분야, 장기 설정 |
| 모든 사용자가 함께 사용하는 정보 | Store | 공통 정책, 애플리케이션 설정 |

> Store에는 이후 대화에서 다시 활용할 가치가 있는 정보만 저장한다. 매번 달라지는 임시 값이나 전체 대화 기록을 무분별하게 저장하면 오래된 정보가 사용되거나 불필요한 데이터가 쌓일 수 있다.

이 판단 기준을 한 문장으로 요약하면, "이 정보가 이 대화방을 나가도 다시 쓸 가치가 있는가"다. 대화 흐름 자체(메시지)는 Checkpointer, 사람에 대한 지속적인 정보(이름, 취향)는 Store로 나뉜다.

### Store는 키-값(key-value) 저장소다

Store는 데이터베이스의 아주 단순한 형태로 볼 수 있다.

- namespace — 데이터베이스의 "테이블" 또는 "폴더"에 가까움
- key — 그 테이블 안에서 한 행(row)을 찾는 "기본 키(primary key)"
- value — 그 행에 실제로 담긴 내용물

메서드도 데이터베이스 연산이랑 대응된다.

```python
store.put(namespace, key, value)   # INSERT/UPDATE
item = store.get(namespace, key)     # WHERE key = ... (하나만 정확히 찾기)
items = store.search(namespace)       # SELECT * FROM ... (그 폴더 전체 훑기)
```

다만 진짜 데이터베이스에 비하면 훨씬 단순하다 — 복잡한 조건 검색이나 여러 테이블 조인 같은 건 못 하고, 딱 "이 namespace의 이 key" 또는 "이 namespace 전체"만 찾을 수 있다.

`InMemoryStore()`는 이 저장소를 파이썬 프로세스의 메모리(컴퓨터의 실제 RAM) 안에만 두는 버전이다. 프로그램이 꺼지면 다 사라진다. `Checkpointer`의 `MemorySaver`/`SqliteSaver` 관계와 똑같은 구조로, 실무에서는 이 자리에 디스크나 데이터베이스 기반 Store를 붙여서 영구 저장하게 만든다.

- `InMemoryStore` / `MemorySaver` — RAM에 저장 (빠르지만 휘발성, 실습·테스트용)
- 디스크/DB 기반 Store·Saver — 파일이나 데이터베이스에 저장 (느리지만 영구적, 실서비스용)

### namespace가 필요한 이유

namespace가 필요한 이유는 "같은 이름의 데이터를 여러 개 저장해야 하는데, 서로 안 섞이게 하고 싶을 때"다.

```text
("users", "user-001") / profile → 철수의 프로필
("users", "user-002") / profile → 영희의 프로필
```

namespace 없이 그냥 key만으로 저장한다면, 모든 사용자의 데이터가 "profile"이라는 이름 하나에 겹쳐 써져서 마지막에 저장한 사람 것만 남는다. namespace는 "어느 그룹(누구)의 데이터인지"를 미리 나눠주는 역할을 한다.

- namespace — "이건 누구(또는 어느 그룹)의 데이터인가"를 구분
- key — 그 그룹 안에서 "무슨 항목인가"를 구분

**"저장할 데이터의 종류(key 이름)는 같은데, 그걸 소유한 대상(사용자, 그룹 등)이 여러 개일 때" namespace를 쓴다.**

> ➕ **더 알아두기**
> 실무에서는 사용자별 분리뿐 아니라, "이 정보는 어느 팀/어느 프로젝트/어느 환경(운영·개발)의 것인가"를 나눌 때도 namespace를 이런 식으로 쓴다. 폴더 구조를 만드는 것과 똑같은 발상이다 — 파일 이름(key)이 겹쳐도, 폴더(namespace)가 다르면 안전하다.

### Store를 사용하는 노드

> 기본 챗봇 노드는 State만 사용했지만, 다음 노드는 실행 설정과 Store도 함께 사용한다. LangGraph는 노드를 실행할 때 함수의 매개변수에 맞춰 이 값들을 전달한다.
>
> - state: MessagesState — 현재 스레드의 메시지가 들어 있는 State
> - config: RunnableConfig — 호출 시 전달한 실행 설정으로, 이 예제에서는 configurable 안의 user_id를 읽는다
> - store: BaseStore — compile(store=store)로 연결한 Store이며 LangGraph가 노드에 전달한다

```python
def memory_chatbot(
    state: MessagesState,
    config: RunnableConfig,
    store: BaseStore,
):
    user_id = config["configurable"]["user_id"]
    namespace = ("users", user_id)
    profile = store.get(namespace, "profile")

    if profile:
        system = (
            "다음 사용자 프로필을 참고하여 답변하세요.\n"
            f"이름: {profile.value['name']}\n"
            f"관심 분야: {profile.value['interest']}"
        )
    else:
        system = "저장된 사용자 프로필이 없습니다."
    response = llm.invoke([SystemMessage(content=system), *state["messages"]])

    return {"messages": [response]}
```

Node 함수가 매개변수를 하나가 아니라 **세 개** 받는다는 게 새로운 부분이다. LangGraph가 노드 함수를 호출할 때 매개변수 이름을 보고 알아서 맞는 값을 채워준다.

`profile.value['name']`도 짚어야 한다 — `store.get()`이 돌려주는 `profile`은 dict가 아니라 **Item이라는 객체**다. 그 객체 안의 `.value`(속성)가 실제로 저장했던 값(dict)이고, 그 안에서 `['name']`(딕셔너리 키)으로 이름을 꺼낸다. **속성 접근(`.value`)과 딕셔너리 인덱싱(`['name']`)이 한 줄에 같이 쓰인 것**이다.

`[SystemMessage(content=system), *state["messages"]]`에서 Store로 조회한 정보(`system`, 문자열)를 `SystemMessage`로 감싸서 맨 앞에 두고, 그 뒤에 실제 대화 기록을 이어붙인다. **LLM은 항상 메시지 객체 리스트만 받으므로**, Store에서 조회한 내용도 메시지 형태로 변환해서 넘겨야 한다. State(딕셔너리 그대로) 자체는 LLM에게 절대 안 간다.

- 틀린 표현 — "summary/profile 같은 건 안 가고 messages만 간다"
- 맞는 표현 — "LLM에게는 항상 메시지 객체 리스트만 간다. Store의 내용은 그 안에 들어가려면 먼저 메시지 모양으로 변환돼야 한다"

### Node의 매개변수는 어떻게 매칭되는가 — 직접 검증

이 부분이 궁금해서 실제로 실험해봤다. **완전히 순서대로도 아니고, 완전히 이름으로만도 아니고, 이 둘이 섞인 규칙**이었다.

```python
# 실험 1: state, config, store 순서를 완전히 뒤섞음 (실패)
def weird_order_node(store: BaseStore, config: RunnableConfig, state: State):
    ...
# TypeError: weird_order_node() got multiple values for argument 'store'

# 실험 2: state는 그대로 맨 앞에 두고, store/config만 순서 바꿈 (성공)
def swapped_node(state: State, store: BaseStore, config: RunnableConfig):
    ...
# 정상 실행됨
```

결론:

- **`state`는 반드시 첫 번째 자리(위치)에 있어야 한다.** LangGraph가 이걸 "이름 상관없이 첫 번째 매개변수"로 인식해서 넘긴다. (실제로 이름을 `state`가 아닌 다른 걸로 지어도 첫 번째 자리면 동작한다 — 이름 자체가 아니라 위치가 기준)
- **`config`, `store`는 이름으로 인식된다.** 둘 사이의 순서는 바뀌어도 상관없지만, 매개변수 이름이 정확히 `config`, `store`여야 한다.

| 매개변수 | 언제/어떻게 연결되는가 |
|---|---|
| `state` | 첫 번째 위치 인자로, `invoke()`를 부를 때마다 새로 전달 |
| `config` | 이름(`config`)으로, `invoke()`를 부를 때마다 새로 전달 |
| `store` | 이름(`store`)으로, `compile(store=...)`에서 **한 번만** 연결되면 이후 계속 재사용 |

> ♻️ `store`가 `compile()`에서 한 번만 연결되는 이유는, `store`가 "이 그래프 전체가 공유하는 하나의 창고"라서다. `state`/`config`는 "이번 호출마다 다른 값"이라서 매번 새로 넘겨야 한다.

`invoke()`의 실제 시그니처를 확인해보면, 첫 번째 인자의 진짜 이름은 `state`가 아니라 `input`이다.

```python
(self, input: 'InputT | Command | None', config: 'RunnableConfig | None' = None, ...)
```

그래서 `graph.invoke(state={...})`처럼 쓰면 에러가 난다 — `invoke()`는 `state`라는 이름의 매개변수를 아예 모른다. `graph.invoke(input={...})` 또는 이름 없이 `graph.invoke({...})`로 넘겨야 한다.

- 노드 함수 정의 — 첫 번째 매개변수 이름을 관례상 `state`라고 지음 (실제로는 아무 이름이나 가능, 위치만 첫 번째면 됨)
- `graph.invoke()` 메서드 — 그 첫 번째 인자의 진짜 이름은 `input`

`state`라는 이름 자체는 문법적으로 필요한 게 아니라, 이 값이 "State를 담고 있다"는 걸 코드만 봐도 알 수 있게 하려는 **관례**다.

### 두 가지 MessagesState — 타입(설계도)과 값(내용물)

> ♻️ `StateGraph(MessagesState)`의 `MessagesState`와 노드 함수의 `state: MessagesState`는 같은 단어를 쓰지만 역할이 다르다.
>
> ```python
> practice_builder = StateGraph(MessagesState)   # 실제로 동작에 영향을 주는 인자
>
> def practice_chatbot(
>     state: MessagesState,       # 타입 힌트 (실행에 영향 없음)
>     ...
> ):
> ```
>
> - `StateGraph(MessagesState)`의 `MessagesState` — 함수 호출의 **진짜 인자**. `StateGraph`가 이 값을 실제로 읽어서 "이 그래프는 messages 필드가 있고, add_messages Reducer가 붙어있다"는 걸 진짜로 동작에 반영한다.
> - `state: MessagesState`(노드 함수 쪽) — 콜론(`:`) 뒤에 붙는 진짜 타입 힌트. 실행에 영향 없이, 사람/편집기가 참고하는 표시일 뿐이다.
>
> `MessagesState`는 데이터의 **틀(설계도)**로서 그래프를 만들 때 딱 한 번 정해지고, 실제 `state` 값(내용물)은 `invoke()`를 부를 때마다 새로 채워진다.

### thread_id와 namespace는 나란히 있는 독립된 축이다

> ♻️ 처음엔 "namespace가 체크포인트다" 또는 "체크포인트가 메시지 하나다" 같은 오해가 있었지만, 정리하면 Checkpointer와 Store는 서로 포함하는 관계가 아니라 완전히 별개의 두 시스템이다.
>
> ```text
> Checkpointer
>  └─ thread_id (큰 것)
>      └─ 체크포인트 여러 개 (작은 것, 그 안에 담김)
>
> Store  (Checkpointer와는 완전히 별개!)
>  └─ namespace (큰 것)
>      └─ key (작은 것, 그 안에 담김)
> ```
>
> "Store 안에 Checkpointer가 있다"거나 "Checkpointer 안에 Store가 있다"가 아니라, 각자 독립된 저장 장치이고, 하나의 그래프(`compile(checkpointer=..., store=...)`)에 둘 다 나란히 연결해서 같이 쓸 수 있는 것뿐이다.

실무에서 논리적으로 그리면 "사용자(namespace) 안에 여러 대화 세션(thread_id), 그 안에 체크포인트들"이라는 계층 구조로 설계하는 게 자연스럽지만, **이 계층을 LangGraph가 코드로 알아서 연결해주는 건 아니다.** `namespace`는 `user_id`로 만들고, `thread_id`는 그냥 별도로 마음대로 지어서 넘긴다. LangGraph 입장에서는 이 둘이 "우연히 같은 사람 것"이라는 걸 모른다 — 그저 우리가 코드에서 `user_id`를 넘길 때마다 항상 그 사람이 실제로 쓴 `thread_id`들과 짝지어서 관리하는 걸 우리 애플리케이션(또는 별도 DB)이 책임져야 한다.

> ➕ **더 알아두기**
> 실무에서는 보통 "이 user_id가 가진 thread_id 목록"을 별도의 테이블(진짜 데이터베이스)에 저장해둔다. 예를 들어 `conversations` 테이블에 `user_id`, `thread_id`, `생성일`을 기록해두면, "이 사용자의 대화 목록을 보여줘"라는 화면을 만들 수 있다. LangGraph 자체는 이 "사용자가 어떤 대화들을 했는지 목록화"하는 기능까지는 안 준다.

### thread_id는 미리 만들 필요가 없지만, 프로필 데이터는 미리 만들어야 한다

> ♻️ user_id(프로필 데이터)와 thread_id(대화 기록)는 "미리 준비해둬야 하는가"가 다르다.
>
> - `user_id`의 프로필 데이터 — **미리 `store.put()`으로 만들어둬야** 함. 안 하면 조회 시 `None`(`store.get()`이 아무것도 못 찾음).
> - `thread_id`(대화 기록) — 미리 만들 필요 없음. `invoke()`를 처음 부르는 순간 Checkpointer가 "이 thread_id로 저장된 게 없네, 그럼 이번이 첫 시작이구나"라고 **자동으로 새 대화로 시작**한다.

## State/Store 조회는 Tool이 아니다

> ♻️ Store에서 데이터를 조회하는 코드가 "필요하면 부르고 아니면 마는" Tool처럼 동작하는 것 아니냐는 질문이 있었는데, 그렇지 않다. Tool과 State/Store 조회는 "누가 판단해서 부르는가"가 다르다.
>
> - Tool — LLM이 스스로 "지금 이 도구가 필요하다"고 판단해서 부름(tool_calls). 안 필요하다고 판단하면 아예 안 부름.
> - State/Store 조회 — LLM의 판단이 전혀 개입하지 않는다. 노드 함수가 실행되면 매번 무조건, 자동으로 실행되는 평범한 Python 코드다.
>
> 마찬가지로, 요약(summarize) 여부를 결정하는 것도 LLM에게 물어보는 게 아니라, `route_after_chatbot` 같은 라우팅 함수(Python 코드)가 State를 읽어서 조건을 확인하고 판단한다.
>
> - 그래프의 흐름 제어(분기, 반복, 언제 요약할지, 언제 멈출지) — 개발자가 짜둔 Python 코드가 결정
> - 그 흐름 안에서 텍스트를 생성하는 것(답변 만들기, Tool 호출 여부 판단) — 그 노드가 실행되는 순간 LLM이 판단
>
> 다만 Store 조회 자체를 LLM이 스스로 판단해서 부르게 만들 수도 있다 — 그 조회 함수를 아예 Tool로 등록해서 LLM에게 넘기면, 그건 진짜 Tool처럼 "LLM이 필요하다고 판단할 때만" 불린다. 지금 노트북 방식은 그게 아니라, 개발자가 "이 노드에선 항상 프로필을 참고하게 하겠다"고 미리 정해둔 방식이다.

## 대화 요약 관리

> 대화가 길어지면 메시지가 계속 누적되어 토큰 비용이 증가한다. 전체 토큰 수가 임계값을 넘으면 요약 노드가 실행되어, trim_messages로 최근 메시지만 남기고 나머지를 요약으로 대체하는 패턴을 구현한다.

```
START → chatbot → 토큰 초과 → summarize → END
                → 토큰 이내 →             END
```

핵심 도구:
- trim_messages: 토큰 기준으로 최근 메시지만 유지한다
- RemoveMessage: State에서 특정 메시지를 ID 기반으로 삭제한다. add_messages reducer가 RemoveMessage를 만나면 해당 ID의 메시지를 제거한다

### 자르기(trim_messages)와 삭제하기(RemoveMessage)의 차이

**trim_messages(자르기)**는, 지금까지 쌓인 메시지 전체 중에서 "토큰 한도 안에 들어오는 최근 것들만 골라내는" 역할을 한다. 오래된 것부터 하나씩 세다가, 정해둔 토큰 기준을 넘기기 직전까지만 남기고 그 앞의 것들은 "요약해야 할 대상"으로 따로 분류한다. 이 자체가 State를 바꾸는 건 아니고, "이 중에 최근 것과 오래된 것을 나눠줘"라는 분류 작업이다.

**RemoveMessage(삭제하기)**는 Reducer(`add_messages`)가 평소엔 "새 메시지를 뒤에 이어붙이는" 역할을 하는데, 새 메시지가 `RemoveMessage`라는 특수한 종류라면 이어붙이는 대신 "그 ID와 일치하는 기존 메시지를 찾아서 지워버리는" 정반대의 동작을 한다는 걸 이용한다.

즉 "메시지를 추가하는 것"과 "메시지를 지우는 것"이 겉보기엔 정반대인데, 사실 **같은 Reducer(`add_messages`) 하나가 상황에 따라 다르게 처리하는 것**뿐이다. RemoveMessage는 그 Reducer에게 "이번엔 추가가 아니라 삭제해줘"라고 알리는 신호 같은 존재다.

- trim_messages — 메시지 목록을 "최근 것 / 오래된 것"으로 **분류**만 함 (State를 직접 안 바꿈)
- RemoveMessage — 오래된 것으로 분류된 메시지를 State에서 **실제로 삭제**시키는 신호

또한 삭제되는 단위는 "Node"가 아니라 "Message"다. Node는 그래프의 실행 단계(chatbot, tools 등)를 가리키는 완전히 다른 개념이고, 여기서 지워지는 건 대화 기록 안의 메시지 하나하나다. 기준도 "토큰 단위"가 아니라 "메시지 ID 단위"다 — 각 메시지는 고유한 ID를 갖고 있어서, 그 ID를 콕 집어 메시지 하나를 통째로 지운다(부분적으로 일부만 지우는 게 아니다).

### 요약 State 정의

> MessagesState를 상속하고, 오래된 대화의 요약을 저장할 summary 필드를 추가한다.

```python
MAX_TOKENS = 200

class SummaryState(MessagesState):
    """최근 메시지와 오래된 대화의 요약을 관리하는 State."""
    summary: str
```

- `messages` — `MessagesState`에서 그대로 물려받은, 최근 메시지들이 담기는 리스트
- `summary` — 새로 추가된, "오래된 대화를 압축한 요약문 한 편"을 담는 필드

### 답변 생성과 요약 조건 — 요약문을 임시로 메시지에 끼워 넣기

```python
def chatbot_with_summary(state: SummaryState):
    """기존 요약과 최근 메시지를 사용하여 답변한다."""
    messages = [SystemMessage(content="모든 답변을 2문장 이내로 짧게 해줘.")]

    # 요약을 합성된 HumanMessage로 전달하되 State에는 저장하지 않는다.
    if state.get("summary"):
        messages.append(
            HumanMessage(
                content=f"이전 대화 요약:\n{state['summary']}",
                additional_kwargs={"lc_source": "summarization"},
            )
        )

    messages.extend(state["messages"])
    response = llm.invoke(messages)

    return {"messages": [response]}


def route_after_chatbot(state: SummaryState):
    """토큰 수에 따라 요약 실행 여부를 결정한다."""
    token_count = llm.get_num_tokens_from_messages(state["messages"])
    if token_count > MAX_TOKENS:
        return "summarize"
    return END
```

핵심 아이디어는 "요약문을 LLM에게 보여주긴 하지만, 대화 기록(State)에는 안 남긴다"는 것이다. State에는 `summary` 필드가 따로 있지만, LLM은 `messages`만 보고 답하기 때문에, 저장된 요약문을 LLM이 실제로 참고하게 하려면 그 요약을 일시적으로 메시지 하나로 만들어서 `messages` 앞에 끼워 넣어야 한다. "합성된 HumanMessage"라는 게 그거다 — 원래 사람이 한 말이 아니라, 코드로 만들어낸 가짜 메시지다.

**이 합성 메시지는 이번 한 번의 LLM 호출에만 쓰이고, State에는 영구히 저장 안 된다.** `return`에 이 조립된 `messages` 리스트를 안 넣기 때문에, Node가 반환한 것만 State에 병합된다는 규칙에 따라 자연스럽게 버려진다. 따로 "버리는 코드"가 필요한 게 아니라, "저장하는 코드를 안 쓴다"는 게 곧 파기다.

> ♻️ **"버려지는 것"과 "영구 저장되는 것"을 정확히 나눠야 한다.**
> - 영구 저장되는 것 — State의 `summary` 필드(문자열) 자체. 안 사라지고 계속 남아있다.
> - 한 번 쓰이고 버려지는 것 — 그 `summary` 문자열을 가지고 그때그때 새로 만든 `HumanMessage` 객체. 진짜 저장된 게 아니라, LLM 호출 한 번을 위해 임시로 만든 사본이다.
>
> 비유하면, `summary`는 서랍 속에 안전하게 보관된 원본 문서이고, LLM에게 보여줘야 할 때마다 그 원본을 서랍에서 꺼내는 게 아니라 복사본(HumanMessage)을 한 장 만들어서 보여주고 버리는 것이다. 원본(`summary` 필드)은 서랍 안에 그대로 남아있다.

**왜 굳이 HumanMessage로 포장해서 끼워 넣는가?** `llm.invoke(state["messages"])`처럼 메시지 객체들이 담긴 리스트를 통째로 넘기는 방식을 그대로 쓰려면, 새로 끼워 넣는 요약문도 같은 리스트 안에 들어갈 수 있는 형태(메시지 객체)여야 한다. 그냥 문자열 하나만 따로 넘기면, 리스트 안의 다른 메시지 객체들이랑 형태가 안 맞아서 합칠 수가 없다. 즉 "불편한 방법"이 아니라, 원래 쓰던 `invoke(메시지_리스트)` 방식을 그대로 재사용하기 위한 최소한의 형태 맞추기다.

### 오래된 대화 요약

> trim_messages로 최근 메시지와 요약할 메시지를 나눈다. 오래된 메시지는 summary에 반영하고, RemoveMessage로 현재 messages에서 제거한다.

```python
def summarize_conversation(state: SummaryState):
    """오래된 메시지를 요약하고 최근 메시지만 유지한다."""
    messages = state["messages"]
    existing_summary = state.get("summary", "")

    # 최근 100토큰 이내의 메시지는 원문 그대로 유지한다.
    kept_messages = trim_messages(
        messages,
        max_tokens=100,
        strategy="last",
        token_counter=llm,
        start_on="human",
    )

    # 유지할 메시지를 제외한 나머지를 요약 대상으로 분리한다.
    kept_ids = {message.id for message in kept_messages}
    messages_to_summarize = [
        message for message in messages if message.id not in kept_ids
    ]

    summary_instruction = (
        "대화 맥락에 필요한 핵심 정보만 요약해줘.\n"
        "- 2문장, 150자 이내\n"
        "- 제목, 불릿, 서론 없이 요약문만 출력\n"
        "- 중복된 내용은 제거\n"
        "- 사용자 이름과 선호처럼 이후 대화에 필요한 정보는 반드시 유지"
    )

    # 기존 요약이 있으면 새 대화 내용을 더해 요약을 갱신한다.
    if existing_summary:
        prompt = (
            f"{summary_instruction}\n\n"
            f"기존 요약:\n{existing_summary}\n\n"
            "새 대화:\n"
        )
    else:
        prompt = f"{summary_instruction}\n\n대화:\n"

    for message in messages_to_summarize:
        if message.type in ("human", "ai") and message.content:
            prompt += f"{message.type}: {message.content}\n"

    summary_response = llm.invoke(prompt)

    # 요약한 원본 메시지는 RemoveMessage로 현재 messages에서 제거한다.
    return {
        "summary": summary_response.text,
        "messages": [
            RemoveMessage(id=message.id) for message in messages_to_summarize
        ],
    }
```

### summary도 무한정 커지지 않는가?

> ♻️ **"기존 요약 + 새로 오래된 메시지들"을 합쳐서 다시 요약하는 건 진짜로 LLM을 또 부르는 것이 맞다.** `summary_response = llm.invoke(prompt)` — 이건 진짜 API 호출이라 토큰이 든다. 그리고 그 프롬프트 안에 `existing_summary`(기존 요약)가 그대로 들어가니까, 매번 "이전 요약 + 새 대화"를 합쳐서 다시 압축하는 과정이 반복된다.
>
> 다만 무한정 커지지는 않는다. `summary_instruction`에 "2문장, 150자 이내"로 압축하라고 매번 강제하기 때문에, 요약본은 매번 비슷한 작은 크기로 다시 만들어진다. 기존 요약이 아무리 길어도, 새로 만들어지는 요약은 다시 150자 이내로 눌린다 — "쌓이면서 커지는" 게 아니라 "매번 다시 압축되는" 것이다.
>
> - 원본 메시지 그대로 뒀다면 — 대화할수록 무한정 커짐
> - summary — 매번 다시 150자 이내로 압축되니 대략 일정한 크기 유지
>
> 그래도 토큰이 드는 건 사실이지만, 이 비용은 매 턴마다 나가는 게 아니라 `MAX_TOKENS`를 넘을 때만 가끔 한 번씩 나간다. 그것도 "지금까지 쌓인 전체 대화"가 아니라 "작은 요약본 + 이번에 새로 오래된 것으로 분류된 몇 개 메시지"만 넣어서 부르니, 매번 전체를 다시 보내는 것보다 훨씬 저렴하다.

### 요약 그래프 구성

```python
summary_builder = StateGraph(SummaryState)
summary_builder.add_node("chatbot", chatbot_with_summary)
summary_builder.add_node("summarize", summarize_conversation)

summary_builder.add_edge(START, "chatbot")
summary_builder.add_conditional_edges(
    "chatbot",
    route_after_chatbot,
    {"summarize": "summarize", END: END},
)
summary_builder.add_edge("summarize", END)

summary_memory = MemorySaver()
summary_graph = summary_builder.compile(checkpointer=summary_memory)
```

이건 완전히 새로운 조립 방식이 아니라, 지금까지 봐온 패턴이다(기존 개념의 응용) — `add_node`로 노드 등록, `add_conditional_edges`로 `route_after_chatbot`의 판단에 따라 갈림길 만들기, `summarize` 노드가 끝나면 END로 가는 고정 Edge. "판단 함수 하나 + 그 갈림길의 도착지 두 곳"이라는 익숙한 뼈대 그대로다. 다른 점은 갈림길의 두 도착지가 모두 결국 END로 이어진다는 것 정도다.

### 원본 대화 보관 — 실무 경고

> 이 예제는 오래된 메시지를 요약한 뒤 현재 State에서 제거하며, 원본 대화는 별도로 보관하지 않는다. 실무에서는 원본 대화를 데이터베이스나 로그 저장소에 별도로 저장하고, LangGraph State에는 요약과 최근 메시지만 유지한다. Checkpointer는 그래프 실행 상태를 저장하기 위한 것으로, 원본 대화 저장소를 대체하지 않는다.

지금 만든 구조는 "원본을 진짜로 안전하게 보관하는 장치"가 아니다. `RemoveMessage`로 메시지를 지우면 현재 State에서는 사라지고, `MemorySaver`처럼 메모리에만 저장하는 걸 쓰면 프로그램을 끄는 순간 과거 체크포인트까지 전부 날아간다. 그럼 원본은 영원히 사라진다.

- Checkpointer의 역할 — 그래프 실행을 이어가기 위한 것 (작업용, 요약되면 원본이 사라질 수 있음)
- 원본 보관의 역할 — 별도의 DB/로그 저장소가 맡아야 함 (Checkpointer가 대신해주지 않음)

> ➕ **더 알아두기**
> 실무에서 실제로 자주 나오는 실수다. "LangGraph가 다 저장해주니까 안전하겠지"라고 생각했다가, 나중에 컴플라이언스(법적 요구사항)나 고객 문의 대응 때문에 "6개월 전 대화 원본을 찾아달라"는 요청이 들어오면 그때 이미 요약되고 지워진 뒤라 복구가 안 되는 경우가 있다. Checkpointer는 "대화를 이어가는 기능"이지 "기록 보관 기능"이 아니라는 걸 명확히 구분해야 한다.

## 실습: Checkpointer + Store를 함께 쓰는 챗봇

문제: Checkpointer와 Store를 사용하는 챗봇을 완성한다. 이 노트북에서 배운 것 중 "실습" 코드는 그 셀 하나만 떼어내도 실행 가능하도록, 필요한 import와 `llm` 정의까지 전부 그 셀 안에 다시 포함시켰다.

```python
from dotenv import load_dotenv
from langchain_core.messages import SystemMessage
from langchain_core.runnables import RunnableConfig
from langchain_google_genai import ChatGoogleGenerativeAI
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import END, START, MessagesState, StateGraph
from langgraph.store.base import BaseStore
from langgraph.store.memory import InMemoryStore

load_dotenv()

MODEL_NAME = "gemini-3.5-flash-lite"
llm = ChatGoogleGenerativeAI(model=MODEL_NAME)

practice_store = InMemoryStore()


def practice_chatbot(
    state: MessagesState,
    config: RunnableConfig,
    store: BaseStore,
):
    """State(대화 내용)와 Store(사용자 프로필)를 함께 참고해서 답하는 노드."""
    user_id = config["configurable"]["user_id"]
    namespace = ("users", user_id)
    profile = store.get(namespace, "profile")

    if profile:
        system_text = (
            "다음 사용자 정보를 참고해서 답변해줘.\n"
            f"이름: {profile.value['name']}\n"
            f"관심 분야: {profile.value['interest']}"
        )
    else:
        system_text = "이 사용자에 대해 저장된 정보가 없다."

    prompt_messages = [SystemMessage(content=system_text), *state["messages"]]
    response = llm.invoke(prompt_messages)
    return {"messages": [response]}


practice_builder = StateGraph(MessagesState)
practice_builder.add_node("chatbot", practice_chatbot)
practice_builder.add_edge(START, "chatbot")
practice_builder.add_edge("chatbot", END)

practice_graph = practice_builder.compile(
    checkpointer=MemorySaver(),
    store=practice_store,
)

practice_store.put(
    namespace=("users", "user-001"),
    key="profile",
    value={"name": "민지", "interest": "머신러닝"},
)
```

`practice_store = InMemoryStore()`가 진짜 생성자 호출이다 — 클래스 이름(`InMemoryStore`) 뒤에 괄호를 붙이면 그 클래스의 새 인스턴스가 만들어진다. 반면 `practice_builder.compile()`은 이미 만들어진 객체(`practice_builder`) 위에서 부르는 **메서드**이지 생성자가 아니다.

`compile(checkpointer=MemorySaver(), store=practice_store)`는 "동시에 연결한다"는 게, 두 저장 장치가 하나로 합쳐지는 게 아니라, **각자 따로 자기 역할을 하는 두 시스템이 같은 그래프 하나에 같이 붙어있는 것**이라는 뜻이다.

`practice_store.put(...)`은 `practice_graph`가 아니라, `compile()` 하기 전부터 이미 있던 `practice_store` 객체에 대고 부른다. `compile(store=practice_store)`는 이미 만들어진 `practice_store`를 이 그래프에 "연결"만 하는 것이지, `compile()`이 새로운 Store를 만들어내는 게 아니다. `practice_graph`와 `practice_store`는 서로 다른 객체인데, `practice_graph`가 내부적으로 그 같은 `practice_store`를 참조하는 관계다.

### 검증

```python
config_a1 = {"configurable": {"thread_id": "practice-thread-1", "user_id": "user-001"}}

result = practice_graph.invoke(
    {"messages": [("human", "내 이름과 관심 분야를 알려줘.")]}, config=config_a1,
)
result = practice_graph.invoke(
    {"messages": [("human", "방금 내가 뭐라고 물어봤지?")]}, config=config_a1,
)

config_a2 = {"configurable": {"thread_id": "practice-thread-2", "user_id": "user-001"}}
result = practice_graph.invoke(
    {"messages": [("human", "방금 내가 뭐라고 물어봤지?")]}, config=config_a2,
)
result = practice_graph.invoke(
    {"messages": [("human", "내 이름과 관심 분야를 알려줘.")]}, config=config_a2,
)

config_b = {"configurable": {"thread_id": "practice-thread-3", "user_id": "user-002"}}
result = practice_graph.invoke(
    {"messages": [("human", "내 이름과 관심 분야를 알려줘.")]}, config=config_b,
)
```

`"practice-thread-1"` 같은 문자열은 시스템이 자동으로 만든 값이 아니라, 코드 작성자가 그냥 임의로 지어낸 이름표다. `thread_id`는 아무 문자열이나 자유롭게 지어도 되고, 유일한 조건은 같은 대화를 이어가려면 그 문자열을 계속 똑같이 써야 한다는 것뿐이다.

- `user_id`의 프로필 데이터 — 미리 `store.put()`으로 만들어둬야 함 (안 하면 조회 시 `None`)
- `thread_id`(대화 기록) — 미리 만들 필요 없음, `invoke()`를 처음 부르는 순간 Checkpointer가 자동으로 새 대화로 시작함

실행 결과:

```
[1차 - user-001] 민지님의 이름은 민지이고, 관심 분야는 머신러닝입니다!
→ 프로필이 있는 사람이라, store.get()이 정보를 찾아서 답변에 반영됨

[2차 - user-001, 같은 thread] 방금 저에게 "내 이름과 관심 분야를 알려줘."라고 물어보셨습니다!
→ 같은 thread_id라서 Checkpointer가 직전 대화를 기억함

[user-001, 다른 thread] 방금 저에게 "방금 내가 뭐라고 물어봤지?"라고 첫 질문을 하셨습니다!
→ thread_id가 바뀌어 Checkpointer 입장에선 완전히 새 대화라, 방금 들어온 이 질문 자체가
  "처음이자 유일한 메시지"였다고 답함 (대화 기록이 없다는 걸 드러낸 것)

[user-001, 다른 thread인데 프로필은 앎] 민지 님의 정보는... 이름: 민지, 관심 분야: 머신러닝
→ thread_id는 다르지만 user_id가 같아서, Store의 프로필은 여전히 조회됨.
  Checkpointer와 Store가 서로 독립된 축이라는 걸 그대로 보여준 결과

[user-002, 프로필 없음] 죄송하지만, 저는 아직 회원님의 이름이나 관심 분야에 대한 정보를 가지고
있지 않습니다.
→ user-002는 store.put()으로 등록한 적이 없어서, profile이 None → "정보 없음" 분기로 감
```

### 왜 굳이 Store가 따로 필요한가 — "각 대화는 어차피 고유하지 않은가"라는 질문

Checkpointer가 `thread_id`로 딱 그 대화 안에서만 기억한다는 건, 뒤집어 말하면 "다른 thread_id로 가면 완전히 처음 보는 사람 취급한다"는 뜻이다. 같은 사람이 오늘 대화를 하고(`thread-1`), 내일 앱을 다시 켜서 새 대화를 시작한다고(`thread-2`) 하면, Checkpointer만 있었다면 `thread-2`는 `thread-1`과 완전히 남남이라서 "당신 이름이 뭐죠?"부터 매번 다시 물어봐야 한다. 사람은 하나인데, 대화방(thread)이 바뀔 때마다 기억을 통째로 잃어버리는 것이다.

실제로 위 검증에서 정확히 이걸 확인했다 — 대화 내용(thread 기준)은 모르는데, 프로필(user_id 기준)은 여전히 알았다. 이것이 Store의 존재 이유다: "대화방은 새로 시작해도, 이 사람이 누구인지는 계속 기억해야 하는" 정보를 위한 것이다.

- Checkpointer만 있으면 — 새 대화방(새 thread)마다 그 사람에 대해 완전히 아무것도 모름
- Store도 있으면 — 새 대화방이어도, 그 사람의 프로필 같은 건 계속 기억함

즉 "각 대화가 고유하다"는 사실 자체가 Checkpointer의 한계이자, Store가 풀어야 하는 문제다.

Store가 다루는 정보 중에서도 "개인 정보(프로필)"와 "진짜 전역(공용) 정보"는 범위가 다르다.

- 개인 정보(프로필) — `namespace = ("users", user_id)`처럼 그 사람 전용 서랍에 저장. "이 사람의 모든 대화방"끼리는 공유되지만, 다른 사람 데이터랑은 안 섞임
- 진짜 전역(공용) 정보 — 모든 사용자가 똑같이 보는 정보(예: `namespace = ("app_settings",)`처럼 사용자 구분 없는 namespace)

둘 다 Store라는 같은 메커니즘(namespace + key)을 쓰지만, namespace를 "사용자별로 나누느냐" "아예 안 나누느냐"에 따라 "개인화된 장기 기억"과 "진짜 전역 공용 데이터"로 갈린다.

## ✅ 확인 질문

1. Checkpointer와 Saver는 각각 무슨 역할이며, 왜 둘이 나뉘어 있는가?
2. thread_id와 checkpoint_id의 차이는? 어느 것이 대화가 진행되는 내내 고정되고, 어느 것이 매번 새로 생기는가?
3. get_state()와 get_state_history()의 차이는?
4. 체크포인트 하나에는 정확히 무엇이 들어있는가? (values, config, parent_config, metadata 등)
5. 체크포인트 안의 어떤 값이 "몇 번째 체크포인트인지"를 나타내는 진짜 순서 정보인가?
6. RemoveMessage로 메시지를 삭제하면 과거 체크포인트에서도 그 메시지가 사라지는가?
7. Store와 Checkpointer는 왜 "포함 관계"가 아니라 "나란히 있는 독립된 두 시스템"인가?
8. namespace와 key는 각각 무엇을 구분하는가? 왜 둘 다 필요한가?
9. 노드 함수의 state, config, store 매개변수는 각각 무엇을 기준으로(위치 vs 이름) LangGraph가 채워주는가?
10. 노드 안에서 Store를 조회하는 코드가 Tool과 다른 점은 무엇인가?
11. trim_messages와 RemoveMessage는 각각 무슨 역할을 하며, 왜 둘 다 필요한가?
12. 요약(summary)에 쓰이는 합성 HumanMessage는 왜 State에 저장되지 않는가?
13. summary는 재요약을 반복해도 왜 무한정 커지지 않는가?
14. thread_id는 사전에 만들어둘 필요가 없는데, 왜 user_id의 프로필은 미리 store.put()으로 만들어둬야 하는가?
15. "각 대화가 어차피 고유한데 왜 Store가 필요한가"라는 질문에 어떻게 답할 수 있는가?
