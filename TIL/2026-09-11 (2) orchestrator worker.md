---
tags: [langgraph, orchestrator-worker, send-api, dynamic-parallelization, structured-output]
til: v2 2026-09-14
---

# (2) LangGraph Orchestrator-Worker 패턴 & Send API로 동적 병렬 처리
> 작성일: 2026-09-11

## 🔗 관련 글

- [(1) LangGraph 병렬 처리(Parallelization) & Voting 패턴](2026-09-11%20(1)%2018-parallelization.md) — 같은 날 먼저 배운 "고정 개수" 병렬 처리, 이 글은 그걸 "동적 개수"로 확장한 것
- [(1) ReAct Agent (Tool, ToolNode, tools_condition, MessagesState)](2026-09-09_(1)_ReAct_Agent(Tool,_ToolNode,_tools_condition,_MessagesState).md) — `add_conditional_edges`와 라우팅 함수를 처음 다룬 글
- [(2) Memory와 State 관리 (Checkpointer, Store, 대화 요약)](2026-09-09_(2)_Memory와_State_관리(Checkpointer,_Store,_대화_요약).md) — State/TypedDict, `state["키"]` 접근 방식이 이어짐

## 1. 18번(고정 병렬)과 뭐가 다른가

18번 노트북에서는 병렬로 실행할 노드의 **종류와 개수가 그래프를 짜는 시점에 이미 정해져 있었다** (`legal_analysis`, `financial_analysis`, `technical_analysis` 셋).

19번(Orchestrator-Worker)은 다르다. **실행해봐야(런타임에) 몇 개의 하위 작업이 필요한지 알 수 있다.** 예를 들어 "파이썬 학습 자료를 만들어줘"라는 요청에 대해 LLM이 목차를 3개로 짤 수도, 5개로 짤 수도 있다. 이 "몇 개일지 미리 모르는" 병렬 작업을 다루는 게 이 패턴의 핵심이다.

패턴 구조:

> 사용자 요청 → **Orchestrator**(지휘자 LLM)가 계획을 세워 하위 작업으로 분할 → 여러 **Worker**(일꾼)가 각자 담당 작업을 동적으로 병렬 실행 → 결과를 하나로 모아 종합

## 2. 두 개의 State: CourseState와 WorkerState

이 패턴은 State를 두 종류로 나눠서 쓴다.

```python
class CourseState(TypedDict):
    request: str                              # 사용자의 전체 학습 요청
    items: list[CurriculumItem]               # Orchestrator가 생성한 목차
    results: Annotated[list, operator.add]    # 여러 Worker의 결과를 누적
    final_material: str                       # 최종적으로 완성된 학습 자료


class WorkerState(TypedDict):
    request: str          # 전체 학습 요청 (Worker도 큰 그림을 참고하려고 복사해서 가짐)
    task_id: int           # 목차의 순서를 기억하기 위한 번호
    title: str             # Worker가 담당하는 목차 제목
    description: str       # Worker가 담당하는 목차 설명
```

- `CourseState` = 그래프 전체가 공유하는 큰 작업판.
- `WorkerState` = Worker 한 명이 자기 몫만 처리하는 데 필요한 값만 모아둔, 작은 "작업 지시서".

`request`가 두 State에 똑같이 들어있는 건 우연이 아니다. Worker가 자기 목차(예: "조건문")만 알아도 되지만, 전체적으로 뭘 만들려는지도 참고해야 좋은 글이 나오기 때문에 복사해서 넘겨준다.

`task_id`는 서로 다른 Worker를 구분하는 값이 아니라, **병렬 실행이 끝난 뒤 결과를 원래 목차 순서대로 다시 정렬하기 위한 번호표**다 (병렬 실행은 완료 순서가 보장되지 않으므로).

## 3. Orchestrator — 구조화된 출력으로 목차 짜기

```python
class CurriculumItem(BaseModel):
    title: str = Field(description="학습 목차 제목")
    description: str = Field(description="해당 목차에서 다룰 세부 학습 내용 요약")

class CurriculumPlan(BaseModel):
    items: list[CurriculumItem] = Field(description="생성된 학습 목차 리스트 (3~5개)")


def create_curriculum(state: CourseState):
    planner = llm.with_structured_output(CurriculumPlan)   # 구조화된 출력 강제
    plan = planner.invoke(f"...\n\n요청:\n{state['request']}")
    return {"items": plan.items}
```

- `BaseModel`은 Pydantic이 제공하는 클래스로, "LLM이 반환해야 할 데이터의 모양"을 정의한다. `TypedDict`(State용)와는 목적이 다르다 — `BaseModel`은 **검증까지 해주는** 모델이고, `TypedDict`는 그냥 "이런 모양이어야 한다"는 타입 힌트만 주고 검증은 안 한다.
- `Field(description=...)`은 필수는 아니지만, LLM에게 "이 칸에 뭘 채워야 하는지" 힌트를 더 정확히 주는 역할을 한다.
- `with_structured_output(CurriculumPlan)`은 원래 `llm`을 바꾸는 게 아니라 **새 llm 객체(`planner`)를 만들어서** 돌려준다(`bind_tools`와 같은 패턴). 실제 API 호출은 이 줄이 아니라 `.invoke()`가 실행되는 순간 일어난다.
- `with_structured_output`이 받을 수 있는 건 아무 클래스나가 아니라, **`BaseModel`, `TypedDict`, JSON 스키마 딕셔너리처럼 "자기 모양을 스키마로 변환하는 기능"을 가진 것들뿐**이다. 평범한 파이썬 클래스는 이 기능이 없어서 못 쓴다.

노드의 반환값은 항상 딕셔너리다. `{"items": plan.items}`처럼 **"어느 State 필드(칸)에, 어떤 값을 넣을지"를 키-값 쌍으로 알려줘야** LangGraph가 어디에 병합할지 알 수 있기 때문이다. 그리고 Orchestrator가 `llm`이 아니라 `planner`(구조화 출력)를 쓴 것과 달리, 뒤에 나올 Worker는 자유 서술형 글을 써야 하므로 그냥 `llm`을 그대로 쓴다 — **"노드가 LLM을 쓴다"고 해서 반환값이 항상 텍스트인 건 아니고, 어떤 llm을 쓰고 결과를 어떻게 가공해서 반환할지는 노드 코드가 직접 정한다.**

## 4. Send API — 동적으로 몇 벌을 복제할지 정하기

```python
def assign_learning_workers(state: CourseState):
    return [
        Send(
            "generate_section",
            {
                "request": state["request"],
                "task_id": task_id,
                "title": item.title,
                "description": item.description,
            },
        )
        for task_id, item in enumerate(state["items"])
    ]
```

이 함수는 노드가 아니라 **라우팅 함수**(그래프 그림에서는 `add_node`로 등록되지 않는다)다. `enumerate(state["items"])`로 목차를 하나씩 돌면서, 목차 개수만큼 `Send` 객체를 만들어 리스트로 반환한다.

`Send("generate_section", {...})`는 "이 내용물을, `generate_section`이라는 노드한테 보낸다"는 택배 같은 것이다. 두 번째 인자(딕셔너리)는 `WorkerState`의 4개 키와 정확히 일치해야 한다 — Worker가 받는 `state`가 바로 이 딕셔너리 자체이기 때문이다.

그래프에 연결할 때는 이렇게 쓴다.

```python
builder.add_conditional_edges(
    "create_curriculum",
    assign_learning_workers,
    ["generate_section"],   # path_map: 이 라우터가 갈 수 있는 목적지 목록
)
```

- 12번 노트북의 `tools_condition`처럼 반환값 자체가 노드 이름과 같은 문자열이면 세 번째 인자(공식 이름 `path_map`)를 생략할 수 있다. 하지만 `assign_learning_workers`는 문자열이 아니라 **`Send` 객체 리스트**를 반환하므로, LangGraph가 코드만 보고 "이게 어디로 갈지" 미리 알 수 없다. 그래서 `["generate_section"]`으로 "그래프를 짜는 시점"에 미리 목적지를 알려줘야 한다. (이건 어디까지나 그래프 구조를 그리기 위한 힌트이고, 실제 실행 결과와는 무관하다.)
- `add_conditional_edges`에는 `then`이라는 네 번째 매개변수도 있다. "이 갈림길에서 어디로 가든, 그다음엔 무조건 이 노드로 가라"는 뜻이며, 이 노트북에서는 `then` 대신 별도의 `add_edge`로 같은 효과를 냈다.

> ➕ **"몇 번 갈지"를 판단하는 함수인 이유**
> 목적지 노드는 항상 `generate_section` 하나로 고정이다. 판단하는 건 "어디로 갈지"가 아니라 **"같은 노드를 몇 벌 복제해서, 각자 다른 내용을 들려 보낼지"**다. `add_edge`는 "A가 끝나면 무조건 B로 한 번 간다"만 표현할 수 있어서, "몇 벌을 만들지"라는 개념 자체가 없다. 그래서 실행되는 그 순간 `state["items"]`를 실제로 세어보고 몇 벌을 만들지 계산해주는 함수가 필요했다.

## 5. Worker — 자기 몫만 쓰고, 리스트에 담아 반환

```python
def generate_section(state: WorkerState):
    result = llm.invoke(f"...\n\n전체 요청:\n{state['request']}\n담당 목차:\n{state['title']}\n...")
    return {
        "results": [{
            "task_id": state["task_id"],
            "title": state["title"],
            "content": result.text,
        }]
    }
```

- `result.text`는 LangGraph가 아니라 **LangChain**(`langchain_core`)의 `AIMessage`가 가진 속성이다. `.content`가 원래 표준 속성인데, 멀티모달 등으로 복잡한 형태가 올 수도 있어서, "순수 텍스트만 편하게 꺼내는" 편의 속성으로 `.text`가 추가됐다.
- `results`에 `{...}` 하나를 바로 반환하지 않고 `[{...}]`처럼 **리스트로 감싸서** 반환하는 이유는, `CourseState.results`에 `Annotated[list, operator.add]` reducer가 걸려있기 때문이다. 리스트끼리는 `+`로 이어붙일 수 있지만 딕셔너리끼리는 안 되므로, 반드시 리스트로 감싸야 한다.

## 6. 결과 취합 — 정렬 후 다시 다듬기

```python
def assemble_course_material(state: CourseState):
    sorted_results = sorted(state["results"], key=lambda x: x["task_id"])
    full_content = "\n\n".join(
        f"## {item['title']}\n{item['content']}"
        for item in sorted_results
    )
    result = llm.invoke(f"...\n\n개별 학습 단원:\n{full_content}")
    return {"final_material": result.text}
```

- `sorted(..., key=lambda x: x["task_id"])`: Worker는 병렬로 실행되므로 완료 순서가 뒤죽박죽일 수 있다. `task_id` 기준으로 다시 정렬해서 원래 목차 순서를 되살린다. `lambda x: x["task_id"]`는 "정렬 기준을 그 자리에서 즉석으로 만드는 이름 없는 함수"다.
- `"\n\n".join(f"..." for item in sorted_results)`: 괄호 `( )` 없이 `for`가 붙어있으면 **generator expression**이다. 대괄호 `[ ]`를 쓰는 **list comprehension**과 문법 구조(값이 먼저, `for`가 나중 — 수학의 집합 표기법 `{ x² | x ∈ S }`에서 온 방식)는 같지만, list comprehension은 실행 즉시 리스트 전체를 메모리에 만들어두는 반면, generator는 "누가 다음 값을 달라고 요청할 때"(`join()`이 그 역할) 그때그때 하나씩만 계산해서 넘겨준다. `join()`은 어차피 값을 순서대로 한 번씩만 쓰고 버릴 거라 리스트로 안 만들고 generator로 충분하다.
- Worker는 서로 독립적으로 자기 단원만 썼기 때문에 문체가 어색하게 겹칠 수 있다. 그래서 마지막에 `llm.invoke(...)`를 한 번 더 불러 전체를 하나의 자연스러운 글로 다듬는다.

## 7. 그래프 조립과 실행

```python
builder = StateGraph(CourseState)
builder.add_node("create_curriculum", create_curriculum)
builder.add_node("generate_section", generate_section)
builder.add_node("assemble_course_material", assemble_course_material)

builder.add_edge(START, "create_curriculum")
builder.add_conditional_edges("create_curriculum", assign_learning_workers, ["generate_section"])
builder.add_edge("generate_section", "assemble_course_material")   # 병렬로 흩어진 Worker가 다 끝날 때까지 자동으로 기다렸다가 실행 (join point)
builder.add_edge("assemble_course_material", END)

course_graph = builder.compile()   # 아직 실행 아님 — 실행 가능한 상태로 완성된 설계도일 뿐
```

`compile()`까지는 준비 단계고, 실제로 그래프를 실행시키는 건 다음 줄이다.

```python
for chunk in course_graph.stream(
    {"request": course_request},
    stream_mode="updates",
    config={"max_concurrency": 3},
):
    for node_name, update in chunk.items():
        ...
```

- `.invoke()`는 끝날 때까지 기다렸다가 최종 결과만 한 번에 주고, `.stream()`은 노드가 하나 끝날 때마다 그 즉시 결과를 흘려보내 준다. 노드가 실제로 끝나는 시점 = chunk가 도착하는 시점이라 **진짜 실시간**이지만, "토큰 단위 타이핑 효과"의 실시간은 아니다 — 그건 `stream_mode="messages"`가 하는 일이고, 그러려면 노드 안의 LLM 호출도 `.invoke()`가 아니라 `.stream()`으로 바꿔야 한다.
- `stream_mode` 옵션: `"values"`(매번 State 전체), `"updates"`(방금 바뀐 부분만, 지금 코드가 쓰는 것), `"messages"`(토큰 단위 실시간), `"custom"`, `"debug"`, `"checkpoints"`, `"tasks"` 등이 있다. **어떤 모드를 쓰든 토큰 소모량(비용)은 동일하다** — 모드는 "이미 일어난 호출의 결과를 어떻게 포장해서 보여줄지"만 정하지, LLM 호출 횟수 자체를 바꾸지 않는다.
- `chunk`는 매 반복마다 **덮어씌워지는 임시 변수**다. 13번 노트북에서 배운 Checkpointer/Store 같은 "나중에 다시 꺼내 쓸 memory"가 아니라, 정반대로 "그 순간만 보고 버리는 값"이다. 반면 State 내부의 `results`(reducer가 걸린 필드)는 실제로 계속 누적된다 — 누적은 State 안에서 일어나고, `chunk`는 그 누적 과정 중 "이번에 새로 추가된 조각 하나"만 보여주는 창문이다.
- `config={"max_concurrency": 3}`: 동시에 실행되는 Worker 개수를 최대 3개로 제한한다. 지금 실습(목차 4~5개)은 몇 초 안에 끝나서 체감상 의미가 적어 보일 수 있지만, 이건 "시간"이 아니라 **"한순간에 동시에 열리는 API 연결 개수"**를 제한하는 것이다. 목차가 수십 개로 커지면 아무 제한 없이 전부 동시에 API를 두드릴 경우 `429 Too Many Requests`(rate limit) 에러를 만날 수 있다 — 지금 규모보다는 앞으로 더 큰 규모에 쓰일 걸 대비한 안전장치다.

## 8. (곁가지) Gemini Batch API

병렬 처리와 별개로, "지금 당장 답이 안 급한" 대량 작업이라면 **Batch API**라는 선택지도 있다. 요청을 즉시 처리하지 않고 서버가 한가할 때 몰아서 처리하는 대신, 입력/출력 토큰 가격을 **50% 할인**해준다. 최대 24시간 안에는 처리 완료를 보장하며(대부분 그보다 훨씬 빠름), 24시간이 지나도 못 끝낸 요청은 취소되고 완료된 부분만 과금된다. 실시간 응답이 필요 없는 대량 사전 처리(예: 수백 개 문서 야간 일괄 요약) 같은 작업에 적합하다.

---

## ✅ 확인 질문

1. 18번(고정 병렬)과 19번(Orchestrator-Worker)에서 "몇 개의 하위 작업이 필요한지"를 아는 시점이 어떻게 다른가?
2. `CourseState`와 `WorkerState`를 굳이 둘로 나눈 이유는 무엇인가?
3. `task_id`가 "서로 다른 Worker를 구분하는 값"이 아니라면, 정확히 무슨 역할을 하는가?
4. `with_structured_output`은 왜 아무 파이썬 클래스나 받아줄 수 없는가?
5. 노드가 LLM을 사용했다고 해서, 그 노드의 반환값이 항상 텍스트인 것은 아닌 이유는?
6. `assign_learning_workers`가 `add_node`로 등록되지 않는 이유는?
7. `add_conditional_edges`의 세 번째 인자(`path_map`)가 `tools_condition`을 쓸 때는 생략됐는데, `assign_learning_workers`를 쓸 때는 왜 필요한가?
8. Worker의 반환값을 `{"results": {...}}`가 아니라 `{"results": [{...}]}`처럼 리스트로 감싸는 이유는?
9. `result.text`는 어느 라이브러리 소속이고, `.content` 대신 왜 추가됐는가?
10. `sorted(..., key=lambda x: x["task_id"])`에서 `lambda`는 무슨 역할을 하는가?
11. list comprehension과 generator expression은 겉보기에 비슷한데, 실제로 어떤 시점에 값을 계산하는지가 어떻게 다른가?
12. `.invoke()`와 `.stream()`은 그래프 실행 결과를 각각 언제, 어떤 단위로 돌려주는가?
13. `stream_mode="updates"`와 `"messages"`는 각각 어떤 단위로 "실시간"을 보여주는가?
14. `chunk`가 매번 덮어씌워진다면, Worker 결과가 실제로 누적되는 곳은 어디인가?
15. `max_concurrency`가 "시간 제한"이 아니라 "동시성 제한"이라는 게 왜 중요한가?
16. Gemini Batch API가 실시간 요청보다 저렴한 이유는 무엇인가?
