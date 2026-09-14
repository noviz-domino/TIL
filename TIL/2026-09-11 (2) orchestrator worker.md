


[TIL] LangGraph Orchestrator-Worker 패턴 & 동적 병렬 처리(Send API)
💡 Today I Learned Summary
LangGraph에서 사용자 요청에 따라 실행 시점에 하위 작업 개수를 동적으로 결정하고, 여러 Worker를 병렬로 실행하는 Orchestrator-Worker 패턴과 Send API 활용법, 그리고 API 과부하 방지를 위한 Rate Limit 대응 기법에 대해 학습했다.

1. Orchestrator-Worker 패턴이란?
개념: 입력 요청에 따라 Orchestrator(지휘자 LLM)가 계획을 세워 하위 작업을 분할하고, 여러 Worker(일꾼 LLM)가 각각의 담당 작업을 동적으로 병렬 실행한 뒤, 결과를 하나로 모아 종합하는 패턴.

고정 병렬 처리(18강)와의 차이점:

18강 (고정 병렬 처리): 그래프 작성 시점에 실행할 노드의 종류와 개수가 미리 고정되어 있음.

19강 (Orchestrator-Worker): 실행 시점(Runtime)에 사용자 요청에 따라 필요한 Worker 수(예: 3개 또는 5개)가 동적(Dynamic)으로 결정됨.

2. 핵심 구현 구성 요소
Orchestrator (create_plan):

사용자 요청을 분석하여 실행할 하위 작업 목록을 생성함.

with_structured_output을 사용해 Pydantic 모델 형태의 구조화된 데이터 목록으로 반환받음.

Send API (동적 Fan-out의 핵심):

라우팅 함수(assign_workers)에서 Send("노드이름", worker_state) 객체 리스트를 반환함.

LangGraph는 리스트의 길이만큼 지정된 Worker 노드를 독립된 입력값과 함께 동적으로 병렬 실행시킴.

Worker (analyze_task):

각 Worker는 전달받은 개별 WorkerState를 기반으로 자기 담당 항목만 집중 분석함.

3. 병렬 처리 문제 해결 공식 (18강 복습 & 응용)
State 충돌 방지 (Reducer):

여러 Worker가 동일한 State 필드에 결과를 쓸 때 덮어씌워지는 현상을 방지함.

results: Annotated[list, operator.add] 처럼 Reducer를 지정해 리스트 형태로 결과가 차곡차곡 누적되도록 처리함.

순서 엉킴 방지 (task_id 정렬):

비동기/병렬 실행 특성상 Worker의 완료 순서는 매번 달라짐 (비결정적).

Orchestrator가 부여한 task_id(순서 번호표)를 반환값에 포함시키고, 종합 노드(make_report)에서 sorted()로 정렬하여 원래 계획된 순서대로 보고서를 생성함.

4. API 과부하 방지 (Rate Limit & max_concurrency)
Rate Limit (429 Too Many Requests 에러):

LLM API Provider는 서버 보호를 위해 분당 요청 수(RPM), 토큰 수(TPM) 등을 제한하며, 한계 초과 시 대기열에 넣지 않고 즉시 거절 에러를 보냄.

병렬 Worker가 한 번에 너무 많이 실행되면 Rate Limit에 쉽게 도달함.

해결책 (max_concurrency):

config={"max_concurrency": 3} 옵션을 설정하여 동시에 실행되는 Worker의 최대 개수를 제한함.

전체 작업이 많더라도 정해진 개수만큼 순차적으로 밸브를 조절하여 프로그램이 튕기는 것을 방지함.


================================================================================
📝 TODAY I LEARNED: LangGraph Orchestrator-Worker Architecture
================================================================================

1. 개념 요약 (Overview)
--------------------------------------------------------------------------------
LangGraph의 Orchestrator-Worker 패턴은 복잡하고 거대한 하나의 요청을 
중앙 지휘자(Orchestrator)가 여러 개의 작은 작업 단위로 나누어(Sub-tasks) 계획을 
수립한 뒤, 다수의 일꾼(Worker)에게 작업을 동적으로 분배하여 병렬 처리하는 
구조이다.

이 패턴을 안전하고 예측 가능하게 구현하기 위해서는 
(1) LLM의 출력을 강제하는 Pydantic 스키마, 
(2) 노드 간 데이터 전달 및 병합을 관리하는 State 구조,
(3) 전체 흐름을 제어하는 Orchestrator 노드 로직의 정교한 설계가 필수적이다.


2. Pydantic 스키마를 통한 구조화된 출력 (Structured Output)
--------------------------------------------------------------------------------
LLM은 기본적으로 자유 서술형 텍스트를 반환하므로, 이를 시스템에서 안전하게 
파싱하여 다음 작업으로 넘기기 위해서는 출력을 엄격하게 강제해야 한다.

- BaseModel
  * 파이썬 객체의 데이터 구조와 타입을 검증하는 Pydantic의 기본 클래스.
  * LLM이 반환해야 할 데이터의 전체 틀을 정의할 때 사용함.

- Field(description=...)
  * 단순히 필드 타입을 명시하는 것에 그치지 않고, 해당 필드가 어떤 데이터를 
    담아야 하는지 LLM에게 전달되는 지시문(Prompt) 역할을 겸함.
  * 지시문을 명확히 작성할수록 LLM이 올바른 데이터를 생성할 확률이 높아짐.

- llm.with_structured_output(Schema)
  * LLM 체인에 Pydantic 스키마를 결합하는 메서드.
  * LLM은 반환 형식을 자유 텍스트가 아닌, 정의된 Pydantic 객체 형태(JSON)로 
    출력하도록 동작 방식이 제한됨.

[구조 예시 개념]
CurriculumItem  -> 개별 학습 단위 (task_id, title, description)
CurriculumPlan  -> 개별 단위들을 담는 리스트 구조 (items: list[CurriculumItem])


3. LangGraph State Architecture (상태 관리 구조)
--------------------------------------------------------------------------------
LangGraph에서 State는 각 노드(Node)들이 데이터를 공유하고 주고받기 위한 
"공유 메모장" 역할을 수행한다.

(1) TypedDict와 State 정의
- 단순히 데이터를 담는 딕셔너리(`dict`) 대신 `TypedDict`를 상속받아 정의함.
- 목적:
  * 개발 단계에서 오타 및 잘못된 키 호출을 미연에 방지하는 타입 안전성 확보.
  * 어떤 노드가 어떤 메모장 변수를 참조하고 수정하는지 보여주는 "공식 계약서" 역할.
  * LangGraph 엔진이 상태 변경 규칙(Reducer)을 인식할 수 있는 틀 제공.

(2) BaseModel vs TypedDict (State)의 역할 구분
- BaseModel: LLM 답변의 내용물과 형식 자체를 엄격하게 검증하고 강제하는 도구.
- TypedDict (State): 검증된 결과물 및 노드 간 필요한 변수를 담고 돌려쓰는 메모장.

(3) 이중 State 구조 (CourseState vs WorkerState)
- CourseState (메인 상태 메모장):
  * 전체 그래프 파이프라인 전반에서 유지되는 글로벌 공유 메모장.
  * 사용자 원본 요청(`request`), 생성된 목차(`items`), 작업 결과 집합(`results`), 
    최종 종합 결과물(`final_material`) 등을 관리함.
- WorkerState (개별 일꾼 메모장):
  * 메인 메모장의 모든 데이터를 일꾼에게 전달할 필요가 없으므로 만든 서브 메모장.
  * 개별 Worker가 본인에게 부여된 작업(`task_id`, `title`, `description`)만 
    집중해서 처리하도록 독립적으로 전달되는 컨텍스트.

(4) Reducer (`Annotated[list, operator.add]`)
- 메인 메모장(`CourseState`) 내 병렬 작업 결과가 들어오는 변수(`results`)에 선언.
- 역할:
  * 여러 Worker 노드가 동시에 결과를 작성하여 돌려줄 때, 기본 동작인 
    "덮어쓰기(Overwrite)"를 방지함.
  * 기존 리스트 데이터 뒤에 새로운 결과물을 차곡차곡 이어 붙이는 `+` 연산 
    (List Concatenation) 규칙을 LangGraph 엔진에 부여함.
- 중요 포인트:
  * Reducer 선언(`Annotated`)은 오직 데이터가 집계되는 메인 State 필드에만 적용.
  * Worker 내부 처리 로직이나 Worker의 단일 `return` 문에는 사용하지 않음.


4. Orchestrator 노드 로직의 작동 원리
--------------------------------------------------------------------------------
Orchestrator 노드는 메인 State를 전달받아 전체 실행 계획(목차)을 수립하는 
첫 번째 핵심 노드이다.

- Step 1: 메인 State 참조
  * `state["request"]`를 통해 공유 메모장에서 사용자의 원본 주문 내용을 읽어옴.
- Step 2: Structured LLM 호출
  * `llm.with_structured_output(CurriculumPlan)`을 적용하여 작성된 LLM을 호출.
  * 사용자 요청 문장을 입력 프롬프트로 전달하여 구조화된 목차 데이터 획득.
- Step 3: 메인 State 업데이트
  * LLM이 반환한 `CurriculumPlan` 객체에서 목차 리스트(`response.items`)를 추출.
  * `return {"items": response.items}` 형태로 딕셔너리를 반환함.
  * LangGraph 엔진이 이 반환값을 받아 `CourseState`의 `items` 키에 자동 반영함.


5. 핵심 시사점 및 차후 진행 단계
--------------------------------------------------------------------------------
- State를 잘 설계하는 것은 복잡한 Multi-Agent 파이프라인의 안정성을 결정짓는 
  가장 중요한 기반 작업임.
- Orchestrator 노드가 세운 계획(`items`)을 바탕으로, 다음 단계에서는 `Send` API를 
  활용해 각 목차 아이템을 Worker 노드들에게 동적으로 라우팅(분배)하는 로직으로 이어짐.
================================================================================








# LangGraph 병렬 학습 자료 실습 코드

## 학습 목표

LangGraph를 이용해 학습 자료를 자동으로 만드는 코드를 한 줄씩 이해한다.
특히 다음 개념을 중심으로 정리한다.

- Pydantic `BaseModel`
- `Field`
- LangGraph State
- `TypedDict`
- LLM 구조화 출력과 Graph State의 연결
- 필드 이름과 데이터 전달 관계

---

# 1. 환경변수와 LLM 설정

## `load_dotenv(override=True)`

```python
load_dotenv(override=True)
```

`.env` 파일에 저장해 둔 환경변수 값을 프로그램에서 사용할 수 있도록 불러오는 함수이다.

예를 들어 `.env`에 다음과 같이 있을 수 있다.

```text
GOOGLE_API_KEY=...
```

`load_dotenv()`가 `.env`를 읽어서 환경변수로 사용할 수 있게 만든다.

### `override=True`의 의미

운영체제나 실행 환경에 같은 이름의 환경변수가 이미 존재할 수도 있다.

- 기본값: 이미 존재하는 환경변수를 우선
- `override=True`: `.env`의 값을 우선해서 덮어씀

즉,

> `.env`에 적어놓은 값을 기존 환경변수보다 우선해서 사용하겠다.

라는 의미이다.

---

## `MODEL_NAME = "gemini-3.6-flash"`

```python
MODEL_NAME = "gemini-3.6-flash"
```

사용할 모델 이름을 문자열로 변수에 저장한다.

`MODEL_NAME`은 개발자가 정한 변수 이름이다.

모델 이름 자체는 비밀키가 아니므로 코드에 직접 적어도 된다.
반면 API 키 같은 비밀 정보는 일반적으로 `.env` 등에 저장한다.

---

## `llm = ChatGoogleGenerativeAI(model=MODEL_NAME)`

```python
llm = ChatGoogleGenerativeAI(model=MODEL_NAME)
```

Google의 Gemini 모델을 사용하기 위한 LLM 객체를 설정한다.

중요한 점:

> 이 줄 자체가 LLM에게 질문을 보내는 것은 아니다.

실제 모델 호출은 뒤에서 사용하는 `invoke()`가 담당한다.

```python
llm.invoke(...)
```

정리하면:

- `ChatGoogleGenerativeAI(...)` → 사용할 LLM을 설정/준비
- `invoke(...)` → 실제로 LLM을 호출

---

# 2. `CurriculumItem` - 학습 목차 하나의 구조

```python
class CurriculumItem(BaseModel):
```

`class`는 데이터를 일정한 구조로 묶어서 사용할 수 있도록 새로운 구조를 정의하는 문법이다.

`CurriculumItem`은 개발자가 붙인 이름이다.

의미상:

> 학습 목차 하나를 표현하는 구조

이다.

`BaseModel`은 Pydantic에서 제공하는 기본 모델 클래스이다.

Pydantic은 LLM이 아니다.
LLM은 내용을 생성하고, Pydantic은 생성된 구조화 데이터를 정의하고 검증/관리하는 역할을 한다.

---

## `BaseModel`은 왜 사용하는가?

LLM에게 단순히 다음과 같이 프롬프트를 줄 수도 있다.

```text
title: 변수와 자료형
description: 파이썬의 기본 자료형을 학습
```

하지만 이렇게 하면 결과가 기본적으로 문자열이다.

프로그램에서 사용하려면 문자열을 다시 파싱해야 할 수 있다.

Pydantic `BaseModel`을 이용하면 LLM의 출력 구조를 프로그램에서 사용할 수 있는 형태로 정의할 수 있다.

예:

```python
class CurriculumItem(BaseModel):
    title: str
    description: str
```

이렇게 하면:

- `title`이라는 필드
- `description`이라는 필드

를 가진 구조화된 데이터를 기대할 수 있다.

---

# 3. `Field`

```python
title: str = Field(description="학습 목차 제목")
```

이 한 줄에는 여러 요소가 들어 있다.

### `title`

`title`은 필드 이름이다.

### `: str`

`str`은 타입 힌트이다.

즉:

> title이라는 필드에는 문자열이 들어간다.

라는 뜻이다.

### `= Field(...)`

`Field()`는 해당 필드에 대한 설정/메타데이터를 추가한다.

여기서:

```python
description="학습 목차 제목"
```

의 `description`은 `Field`가 제공하는 설정 이름이다.

따라서 왼쪽의 `title`, 오른쪽 `Field()` 안의 `description`은 서로 다른 역할이다.

---

## `Field`에서 사용할 수 있는 설정 예

### `description`

필드의 설명을 붙인다.

```python
Field(description="학습 목차 제목")
```

### `default`

기본값을 지정할 수 있다.

```python
Field(default="제목 없음")
```

### `examples`

예시를 제공할 수 있다.

여러 예시를 넣을 수도 있다.

```python
Field(
    examples=["변수와 자료형", "조건문", "반복문"]
)
```

`examples`는 여러 값을 허용한다는 뜻이 아니라, 참고용 예시를 여러 개 제공할 수 있다는 뜻이다.

### `alias`

외부에서 사용하는 이름을 별도로 지정할 수 있다.

예를 들어 내부적으로는 `title`을 사용하지만 외부 데이터에서는 다른 이름을 사용할 수 있다.

이 실습에서는 `alias`가 필요하지 않다.

---

# 4. `CurriculumPlan` - 전체 목차의 구조

```python
class CurriculumPlan(BaseModel):
```

`CurriculumItem`이 목차 하나를 나타낸다면,

`CurriculumPlan`은 전체 목차 계획을 나타낸다.

즉:

```text
CurriculumItem
→ 목차 하나

CurriculumPlan
→ 전체 목차 계획
```

---

## `items: list[CurriculumItem]`

```python
items: list[CurriculumItem] = Field(
    description="생성된 학습 목차 리스트 (3~5개)"
)
```

`items`는 필드 이름이다.

`list[CurriculumItem]`은 타입이다.

의미는:

> `CurriculumItem` 객체들을 담는 리스트

이다.

개념적으로는:

```text
CurriculumPlan
└── items (list)
    ├── CurriculumItem
    ├── CurriculumItem
    └── CurriculumItem
```

각 `CurriculumItem` 안에는:

```text
title
description
```

이 들어간다.

따라서 전체 구조는 대략:

```text
CurriculumPlan
└── items
    ├── { title, description }
    ├── { title, description }
    └── { title, description }
```

처럼 이해할 수 있다.

---

# 5. Pydantic 객체와 dict의 차이

Pydantic 모델은 겉으로 보면 dict와 비슷하게 느껴질 수 있지만 실제로는 같은 것이 아니다.

dict는 단순한 키-값 데이터이다.

```python
{
    "title": "파이썬 기초"
}
```

반면 Pydantic 모델은 정의된 필드와 타입을 가진 모델 객체이다.

그래서 다음과 같이 속성에 접근할 수 있다.

```python
item.title
item.description
```

여기서 `item`은 파이썬의 특별한 내장 함수가 아니다.

`.`은 객체의 속성이나 메서드에 접근하는 문법이다.

---

# 6. `:`는 상황에 따라 의미가 다르다

`:`가 항상 타입 힌트라는 것은 아니다.

### 타입 힌트

```python
title: str
```

여기서는 `title`의 타입을 나타낸다.

### dict

```python
{"title": "파이썬 기초"}
```

여기서는 `:`가 키와 값을 구분한다.

### 조건문

```python
if condition:
```

여기서는 실행할 코드 블록의 시작을 나타낸다.

### 슬라이싱

```python
text[1:3]
```

여기서는 슬라이스 범위를 나타낸다.

따라서 `:`의 의미는 문맥을 보고 판단해야 한다.

---

# 7. `TypedDict`와 LangGraph State

```python
class CourseState(TypedDict):
```

여기서는 `BaseModel`이 아니라 `TypedDict`를 사용한다.

`CourseState` 역시 개발자가 정한 이름이다.

`TypedDict`는:

> dict 형태의 데이터에 어떤 키가 있고, 각 키의 값이 어떤 타입인지 알려주는 타입 구조

라고 이해하면 된다.

중요한 점은 `CourseState` 자체가 실제 데이터 저장공간이라는 뜻은 아니라는 것이다.

실제로 움직이는 데이터는 일반적인 dict 형태라고 생각하면 된다.

예:

```python
state = {
    "request": "파이썬을 배우고 싶어",
    "items": [...]
}
```

`CourseState`는 이런 dict가 어떤 모양이어야 하는지를 설명한다.

---

# 8. 왜 `BaseModel`이 아니라 `TypedDict`를 사용하는가?

둘 다 데이터의 구조를 정의하기 때문에 처음 보면 비슷하다.

하지만 목적이 다르다.

## `BaseModel`

주 목적:

> 데이터 자체를 하나의 모델로 정의하고 검증/구조화하기

주로 이 실습에서는 LLM의 구조화된 출력에 사용한다.

```python
class CurriculumItem(BaseModel):
    title: str
    description: str
```

## `TypedDict`

주 목적:

> dict 형태의 데이터가 어떤 키와 타입을 가져야 하는지 설명하기

LangGraph에서는 노드 사이에 state가 dict 형태로 전달되므로 State 구조를 표현하는 데 자연스럽다.

```python
class CourseState(TypedDict):
    request: str
    items: list[CurriculumItem]
```

차이를 간단히 정리하면:

```text
BaseModel
→ 데이터 자체를 모델링

TypedDict
→ dict의 구조를 타입으로 설명
```

또한 `BaseModel`은 Pydantic을 통한 런타임 검증 등의 기능이 있지만,
`TypedDict`는 기본적으로 타입 체크를 위한 구조이며 일반 dict처럼 사용된다.

---

# 9. BaseModel과 TypedDict의 필드 이름 관계

중요한 질문:

> LLM이 BaseModel로 구조화된 결과를 만들었으면
> LangGraph의 TypedDict 필드 이름도 반드시 같아야 하는가?

정답:

> 반드시 같은 것은 아니지만, 데이터를 연결하려면 실제로 사용하는 키를 맞추거나 중간에서 변환해야 한다.

현재 코드에서는 이름을 같게 만들어 연결하기 편하게 했다.

LLM 결과:

```python
class CurriculumPlan(BaseModel):
    items: list[CurriculumItem]
```

LLM 결과에서:

```python
plan.items
```

로 `items`를 꺼낼 수 있다.

그리고 Graph State에:

```python
return {
    "items": plan.items
}
```

로 넣는다.

여기서:

```text
plan.items
   ↓
"items"
   ↓
Graph State의 items
```

로 연결된다.

즉, 자동으로 이름을 보고 연결되는 마법이 아니다.

개발자가 `return`에서 직접 연결하는 것이다.

---

## 이름이 달라도 가능하다

예를 들어:

```python
class CurriculumPlan(BaseModel):
    curriculum_items: list[CurriculumItem]

class CourseState(TypedDict):
    items: list[CurriculumItem]
```

처럼 이름이 달라도 된다.

그 경우:

```python
return {
    "items": plan.curriculum_items
}
```

처럼 개발자가 변환해주면 된다.

따라서:

> BaseModel 필드 이름 = TypedDict 필드 이름

이 반드시 지켜야 하는 규칙은 아니다.

다만 같은 데이터를 자연스럽게 전달하려면 이름을 동일하게 사용하는 경우가 많다.

---

# 10. 현재 배우는 `request: str`

```python
request: str
```

여기서:

- `request` → State에서 사용하는 키 이름
- `:` → 타입을 지정한다는 표시
- `str` → 문자열 타입

즉:

> State의 `request`라는 키에는 문자열이 들어온다.

라는 뜻이다.

예를 들어 실제 데이터는:

```python
{
    "request": "파이썬을 처음 배우고 싶어"
}
```

처럼 될 수 있다.

그리고 뒤에서는:

```python
state["request"]
```

처럼 사용한다.

여기서도 `request`라는 이름은 파이썬이 정해준 이름이 아니다.
개발자가 이 데이터의 역할을 보고 정한 키 이름이다.

---

# 지금까지의 전체 연결

현재까지 배운 내용을 하나로 연결하면:

```text
사용자 요청
   ↓
LLM
   ↓
CurriculumPlan (BaseModel)
   └── items
        ↓
create_curriculum()
   ↓
return {"items": plan.items}
   ↓
LangGraph State
   ↓
CourseState (TypedDict)
   ├── request: str
   ├── items: list[CurriculumItem]
   ├── results: ...
   └── final_material: str
```

핵심은 두 가지이다.

1. `BaseModel`은 LLM이 만들어낼 구조화된 결과를 정의한다.
2. `TypedDict`는 LangGraph에서 노드들이 공유하는 State dict의 구조를 설명한다.

그리고 둘 사이의 실제 데이터 연결은 노드 함수의 `return`과 `state[...]` 등을 통해 이루어진다.

---

# 질문하면서 생긴 중요한 포인트

## Q. `BaseModel`이 있으면 필드마다 LLM을 따로 호출하는가?

아니다.

예를 들어:

```python
class CurriculumItem(BaseModel):
    title: str
    description: str
```

이 구조가 있다고 해서 `title` 호출 한 번, `description` 호출 한 번이 되는 것이 아니다.

한 번의:

```python
planner.invoke(...)
```

호출에서 `title`과 `description`을 포함한 구조화된 결과를 받을 수 있다.

---

## Q. `Field(description=...)`의 description은 프롬프트인가?

일반적인 프롬프트 문자열을 직접 작성한 것은 아니다.

`Field()`에 메타데이터를 붙이고, LangChain의 구조화 출력 과정에서 이 정보가 스키마/구조화 출력 계약을 만드는 데 활용될 수 있다.

따라서 느슨하게 "LLM에게 이 필드가 뭔지 알려주는 설명"이라고 이해할 수 있지만,
정확하게는 Pydantic 필드의 메타데이터이다.

---

## Q. `TypedDict`가 `BaseModel`보다 무조건 좋은가?

아니다.

둘은 목적이 다르다.

이 코드에서는:

- LLM 출력 → `BaseModel`
- LangGraph State → `TypedDict`

라는 역할 분담을 하고 있다.

---

# 현재까지 한 줄 요약

```text
BaseModel
→ LLM 출력의 구조를 정의

TypedDict
→ LangGraph State로 사용할 dict의 구조를 정의

Field
→ BaseModel 필드에 설명/설정 추가

request: str
→ State의 request 키에는 문자열이 들어온다고 선언
```
