---
tags: [langgraph, time-travel, checkpointer, replay, fork]
til: v2 2026-09-16
---

# Time Travel (Checkpoint, Replay, Fork)
> 작성일: 2026-09-16

## 🔗 관련 글

- [Memory와 State 관리(Checkpointer, Store, 대화 요약)](2026-09-09_(2)_Memory와_State_관리(Checkpointer,_Store,_대화_요약).md) — Checkpointer/thread_id를 처음 배운 곳(그때는 대화 기억용, 오늘은 과거 회귀용)
- [Human-in-the-Loop(interrupt, Command, Checkpointer)](2026-09-15_Human-in-the-Loop(interrupt,_Command,_Checkpointer).md) — 바로 어제 배운 것, Checkpointer가 "멈췄다 재개"에 쓰이는 용도

## Time Travel이란

Checkpointer에 남은 과거 State를 조회하고, 특정 시점부터 다시 실행하거나 값을 바꿔 새로운 실행 경로를 만드는 기능이다. 21번(HITL)에서 배운 Checkpointer를 그대로 응용한 것이다 — 그때는 "멈췄다가 나중에 이어가는" 용도로 썼는데, 이번엔 Checkpointer가 기록해둔 과거의 모든 시점을 조회하고 원하는 시점으로 돌아갈 수 있다는 걸 배운다.

| 동작 | 설명 |
|---|---|
| 조회 | 과거 checkpoint의 State와 다음 실행 노드를 확인한다 |
| Replay | 과거 값을 그대로 사용해 그 시점 이후를 다시 실행한다 |
| Fork | 과거 값을 수정한 뒤 다른 경로로 실행한다 |

## 실습 시나리오

`주제 정리 → 초안 작성 → 검토` 3단계 그래프를 만들고, 과거로 돌아가서 Replay/Fork를 실습한다.

```python
class PostState(TypedDict, total=False):
    request: str
    topic: str
    draft: str
    review: str

def choose_topic(state: PostState):
    print("[실행] 주제 정리")
    return {"topic": f"{state['request']} 입문"}

def write_draft(state: PostState):
    print("[실행] 초안 작성")
    return {"draft": f"{state['topic']}: 쉽게 설명하는 게시글입니다."}

def review_draft(state: PostState):
    print("[실행] 검토")
    return {"review": f"검토 완료: {state['draft']}"}

builder = StateGraph(PostState)
builder.add_node("choose_topic", choose_topic)
builder.add_node("write_draft", write_draft)
builder.add_node("review_draft", review_draft)
builder.add_edge(START, "choose_topic")
builder.add_edge("choose_topic", "write_draft")
builder.add_edge("write_draft", "review_draft")
builder.add_edge("review_draft", END)

graph = builder.compile(checkpointer=InMemorySaver())
```

## State 조회 — get_state, get_state_history

`get_state(config)`는 최신 `StateSnapshot`을 반환한다. `values`는 저장된 State, `next`는 다음에 실행할 노드, `config`는 `thread_id`와 `checkpoint_id`를 담고 있다.

`get_state_history(config)`는 최신 checkpoint부터 과거 순서로 snapshot을 반환한다.

## thread_id vs checkpoint_id — 구분이 필요하다

- **thread_id**: 하나의 실행 이력 **전체**를 가리키는 이름표 (예: "post-1")
- **checkpoint_id**: 그 이력 안에서 **특정 시점 하나하나**를 가리키는 이름표 (A, B, C, D 각각)

```text
thread_id: post-1
├── checkpoint_id: A  → 주제 정리 전
├── checkpoint_id: B  → 초안 작성 전
├── checkpoint_id: C  → 검토 전
└── checkpoint_id: D  → 실행 완료
```

각 snapshot의 `next`에는 그 시점에서 다음으로 실행할 노드가 담겨 있다. `next`가 `('write_draft',)`인 snapshot을 찾으면 초안을 작성하기 직전 상태로 돌아갈 수 있다.

> ➕ **더 알아두기 — 체크포인트는 Fork할 때만 생기는 게 아니다**
> A, B, C, D는 Fork를 하기 전, 그냥 평범하게 `invoke()`를 한 번 실행했을 때 이미 다 생긴 것들이다. 노드 하나가 실행될 때마다 State 스냅샷이 자동으로 하나씩 저장된다 — 이게 21번에서 배운 Checkpointer가 원래 하던 일이다(멈췄다가 재개하려면 이 기록이 필요하니까). Fork는 "체크포인트를 만드는 특별한 기능"이 아니라, 원래 있던 체크포인트 중 하나를 골라 거기서부터 다시 시작하는 것뿐이다.

## Replay — 과거를 그대로 다시 실행

선택한 checkpoint의 `config`와 함께 `None`을 입력하면 새 State를 넣지 않고 그 시점부터 실행한다. 완료된 이전 노드(`choose_topic`)는 재실행하지 않고 `write_draft`부터 실행한다. **같은 값을 사용해도 이후 노드의 코드는 다시 호출된다.**

```python
replayed = graph.invoke(None, before_draft.config)
```

Replay는 기존 checkpoint를 삭제하거나 덮어쓰지 않는다. 새 checkpoint는 같은 `thread_id`의 이력에 계속 추가되므로 `get_state_history()`의 결과는 최신순 목록으로 보인다. 각 checkpoint는 자신이 이어진 이전 checkpoint를 `parent_config`로 가리킨다. 과거의 B에서 Replay하면 새 checkpoint C'의 부모는 기존 최신 checkpoint D가 아니라 선택한 B가 된다.

```text
저장·조회 순서: A, B, C, D, C', D'
부모 관계:     A → B → C → D
                    └→ C' → D'
```

> ➕ **더 알아두기 — Git 브랜치와 완전히 같은 구조**
> 과거 커밋(체크포인트)으로 체크아웃해서 새 브랜치를 파도, 원래 있던 미래 커밋(C, D)들이 삭제되지 않고 그대로 남아있는 것과 똑같다. checkpoint_id는 Git의 커밋 해시, parent_config는 Git의 "부모 커밋" 관계에 대응한다. State 값이 기존 실행과 같더라도 새 checkpoint에는 새로운 checkpoint_id가 부여된다.

## Fork — 과거 값을 수정한 뒤 다른 경로로 실행

`update_state()`로 과거 State를 바꾸면 기존 checkpoint를 덮어쓰지 않고 새 checkpoint를 만든다. `as_node="choose_topic"`은 해당 값을 그 노드가 작성한 것처럼 처리한다. 따라서 다음 노드인 `write_draft`부터 이어진다. 반환값은 새 checkpoint를 가리키는 config다.

```python
fork_config = graph.update_state(
    before_draft.config,
    {"topic": "LangGraph Time Travel 심화"},
    as_node="choose_topic",
)
forked = graph.invoke(None, fork_config)
```

Fork도 기존 checkpoint를 삭제하거나 덮어쓰지 않는다. 과거의 B를 기준으로 `update_state()`를 호출하면 수정된 State를 가진 checkpoint U가 생성되고, U의 부모는 B가 된다. 이어서 그래프를 실행하면 U의 `next`부터 실행되며 C'과 D'가 추가된다.

```text
저장·조회 순서: A, B, C, D, U, C', D'
부모 관계:     A → B → C → D
                    └→ U → C' → D'
                        ↑
                  update_state()
```

Replay와 달리 Fork에는 변경된 State를 저장하는 checkpoint U가 하나 더 존재한다. Fork 이후에도 원래 결과(`result`)와 과거 checkpoint는 바뀌지 않는다 — `result["topic"]`과 `forked["topic"]`을 비교하면 서로 다른 값이 나온다.

## as_node의 역할

Fork로 값을 바꿀 때 "이 값을 누가 만든 것처럼 취급할지"를 지정해야 한다. State의 `next`(다음에 뭘 실행할지)는 "직전에 어느 노드가 끝났는지"로 정해지기 때문이다. `as_node="choose_topic"`이라고 지정하면, 그래프는 "choose_topic이 방금 끝났다"고 인식해서 자연스럽게 다음(write_draft)으로 이어간다.

## thread_id, checkpoint_id, "갈래 구분"의 관계 — Q&A로 정리한 부분

- **thread_id는 "사용자 계정"이 아니라 "하나의 대화/작업 단위"다.** 사용자 계정 안에는 여러 개의 thread_id(여러 대화)가 있을 수 있고, 그 thread_id 하나하나가 내부에서 여러 갈래(브랜치)를 가질 수 있다.
- **Fork해도 thread_id는 안 바뀐다.** 새로 생기는 건 checkpoint_id뿐이다 — 같은 저장소(thread_id) 안에 새 커밋(checkpoint_id)이 추가되는 것과 같다.
- **"이게 원래 응답, 이게 새로 만든 응답"이라고 구분해서 보여주는 기능은 코드에 없다.** `get_state_history()`는 체크포인트들을 그냥 나열해서 줄 뿐, "이게 1번 갈래, 이게 2번 갈래"라고 미리 그룹핑해주지 않는다. 이 그룹핑은 `parent_config`를 직접 따라가며 우리가 계산해서 만들어야 한다 (Git도 마찬가지 — 브랜치 이름은 커밋 그래프 위에 사람이 붙이는 라벨일 뿐이다).
- **회귀(Replay/Fork)는 항상 같은 thread_id 안에서만 이뤄진다.** `get_state_history(config)`는 그 config에 담긴 thread_id에 속한 체크포인트만 보여주고, 다른 thread_id로는 회귀할 수 없다.

> ➕ **더 알아두기 — Claude/GPT 같은 실제 제품과의 유추 (확인된 사실 아님)**
> "새 대화를 시작하면 새 thread_id, 같은 대화 안에서 메시지를 수정/재생성하면 같은 thread_id 유지 + 갈래 생성"이라는 패턴은 오늘 배운 원리로 보면 논리적으로 자연스럽다. 다만 실제 Claude나 ChatGPT가 내부적으로 이 방식(또는 LangGraph)을 그대로 쓰는지는 공개된 정보가 아니므로, 어디까지나 "원리상 이렇게 만드는 게 합리적이다"는 추론이지 확인된 사실은 아니다.

## 실제 서비스에서 이 개념이 어떻게 보이나

사용자는 "체크포인트"나 "Fork" 같은 용어를 직접 보지 않고, 익숙한 버튼 이름으로 접한다.

| UI 기능 | 내부 동작 |
|---|---|
| 다시 생성 | 선택한 checkpoint에서 Replay |
| 이 단계부터 다시 실행 | 선택한 checkpoint에서 Replay |
| 수정 후 계속 | 중간 결과의 수정값을 State 필드에 반영한 뒤 Fork 실행 |
| 이 버전에서 다시 작성 | 선택한 버전의 checkpoint에서 Fork |

## 활용 사례

Replay:
- 장애가 발생한 지점부터 다시 실행하며 Agent의 판단 과정 분석
- prompt, model, logic 수정 후 같은 과거 State에서 결과 비교
- 앞부분의 비싼 검색이나 Tool 호출 결과를 유지한 채 이후 단계 테스트

Fork:
- 사용자의 수정 요청이나 검토자의 피드백을 반영해 이후 작업 계속
- 동일한 중간 State에 서로 다른 조건을 적용해 결과 비교
- 긴 조사·분석 결과를 재사용해 여러 전략이나 결과 생성

## 외부 작업 주의사항

State는 과거 checkpoint에서 다시 실행할 수 있지만, **이미 보낸 메일이나 처리한 결제는 되돌아가지 않는다.** 돌아간 지점 이후의 노드는 완전히 다시 실행되므로, 그 안에 이메일 발송·결제 같은 되돌릴 수 없는 부작용이 있다면 재실행 시 중복 실행될 위험이 있다. 21번(HITL)에서 배운 "부작용은 interrupt() 뒤에서 실행한다"는 원칙과 같은 맥락의 주의사항이다.

## ✅ 확인 질문

1. Time Travel의 조회/Replay/Fork 세 가지 동작은 각각 무엇을 하는가?
2. `get_state(config)`가 반환하는 StateSnapshot의 `values`, `next`, `config`는 각각 무엇을 담고 있는가?
3. `get_state_history(config)`는 체크포인트를 어떤 순서로 반환하는가?
4. thread_id와 checkpoint_id는 각각 무엇을 구분하는 이름표인가?
5. 체크포인트는 Fork를 할 때만 생기는가, 아니면 평범한 실행 중에도 생기는가?
6. Replay 시 `graph.invoke(None, before_draft.config)`에서 `None`을 넣는 이유는 무엇인가?
7. Replay나 Fork를 해도 기존 checkpoint(예: C, D)는 어떻게 되는가?
8. `parent_config`는 어떤 역할을 하며, Git의 어떤 개념과 대응되는가?
9. Fork와 Replay의 결정적 차이는 무엇인가?
10. Fork에서 `as_node`를 지정하는 이유는 무엇인가?
11. `as_node`를 지정하지 않으면 그래프가 다음에 뭘 실행해야 할지 어떻게 판단하기 어려워지는가?
12. "thread_id는 사용자 계정과 같다"는 비유가 왜 정확하지 않은가?
13. get_state_history()가 반환하는 체크포인트 목록에서, "어느 것이 원래 경로고 어느 것이 갈라진 경로인지"는 자동으로 구분되는가?
14. Replay/Fork로 다른 thread_id의 체크포인트로 회귀할 수 있는가?
15. 서비스 화면의 "다시 생성"과 "수정 후 계속" 버튼은 각각 Replay와 Fork 중 무엇에 대응하는가?
16. Time Travel로 돌아가서 재실행할 때, 왜 외부 작업(이메일, 결제)에 주의해야 하는가?
