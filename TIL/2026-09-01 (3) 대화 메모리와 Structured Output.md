---
tags: [langchain, chat-memory, structured-output, pydantic, output-parser]
til: v2 2026-09-01
---

# (3) 대화 메모리(03)와 Structured Output(04)
> 작성일: 2026-09-01

## 🔗 관련 글

- [2026-09-01 LangChain 기초 - Runnable, LCEL 파이프라인, 실습 4종](2026-09-01%20LangChain%20기초%20-%20Runnable,%20LCEL%20파이프라인,%20실습%204종.md) — `with_fallbacks()`의 뒷부분, 오늘 `.with_retry()`와 짝지어 다룸
- [2026-08-25 (2) 환경변수 통합, Gemini API](2026-08-25%20(2)gemini%20api%20설명.md) — LLM stateless·`history` 직접 관리, Pydantic Structured Output의 뒷부분(오늘 LangChain의 `PydanticOutputParser`로 확장됨)

## Part 1. 대화 메모리 (Chat Memory)

### LLM은 원래 stateless (기억 못 함)

```python
response1 = llm.invoke([HumanMessage(content="내 이름은 철수야")])
response2 = llm.invoke([HumanMessage(content="내 이름이 뭐였지?")])  # 모름!
```

매번 `invoke()`는 완전히 새로운 요청. 이전 호출을 스스로 기억하는 기능이 아예 없음.
"대화가 이어지는 것처럼" 보이는 이유는 모델이 똑똑해서가 아니라, **매번 이전 대화 전체를 통째로 다시 보내주기 때문.**

### 수동으로 히스토리 전달

```python
messages = [SystemMessage("너는 친절한 상담사야."), HumanMessage("내 이름은 철수야")]
response1 = llm.invoke(messages)

messages.extend([response1, HumanMessage("내 이름이 뭐였지?")])  # 이전 응답까지 포함해서 누적
response2 = llm.invoke(messages)
```

### `InMemoryChatMessageHistory` — 세션별 자동 관리

```python
history_store: dict[str, InMemoryChatMessageHistory] = {}

def get_session_history(session_id):
    if session_id not in history_store:
        history_store[session_id] = InMemoryChatMessageHistory()
    return history_store[session_id]

def chat(session_id, user_input):
    history = get_session_history(session_id)
    user_message = HumanMessage(content=user_input)
    response = llm.invoke([SystemMessage("..."), *history.messages, user_message])
    history.add_messages([user_message, response])
    return response.content
```

`history_store`(딕셔너리)로 `session_id`별 대화를 독립적으로 관리 → `user-1`과 `user-2`의 기록이 안 섞임.

### ⚠️ 저장 위치는 RAM (컴퓨터 메모리)

**"In-Memory"** = RAM 안에 저장된다는 뜻. 디스크 파일이 아니라 **파이썬 프로세스가 실행되는 동안만 존재하는 임시 데이터.**
- 커널 재시작 → 사라짐
- 스크립트 종료 → 사라짐

실제 서비스라면 데이터베이스(PostgreSQL, Redis 등) 기반 저장소를 써야 서버 재시작에도 대화가 안 사라짐. LangGraph에서는 state + checkpointer로 이걸 관리.

### Context Window 관리 — 3가지 방법

| 방법 | 장점 | 단점 |
|---|---|---|
| 메시지 트리밍 | 간단, 추가 비용 없음 | 오래된 맥락 유실 |
| 대화 요약 | 핵심 맥락 압축 | 추가 API 호출, 요약 오류 가능 |
| 검색 기반 선택 | 관련 정보만 선택 | 검색/인덱싱 설계 필요 |

### `trim_messages()` — ⚠️ "압축"이 아니라 "삭제"

```python
trimmer = trim_messages(
    max_tokens=80,       # 예산(캐리어 무게 제한)이지, "중요도 기준"이 아님
    strategy="last",     # 최근 것 우선으로 남김
    token_counter=llm,   # 토큰 수를 셀 때 이 모델 기준 사용 (판단 X, 계산만)
    include_system=True, # system 메시지는 항상 유지
    start_on="human",    # 잘린 후 human 메시지부터 시작하게 정리
)
trimmed = trimmer.invoke(long_history)
```

**핵심 오해 정정**:
- `max_tokens`는 "이번에 보낼 수 있는 전체 예산"이지, "이 메시지가 크니까 중요하다"가 아님. 캐리어 무게 제한처럼, 최근 것부터 채우다가 넘치면 오래된 건 그냥 못 실음
- **메시지를 반토막 내지 않음** — 메시지 하나는 통째로 남거나 통째로 버려짐 (문장 중간을 자르지 않음)
- **압축이 아니라 삭제**: 문장을 다시 써서 짧게 만드는 게 아니라, 그냥 있거나 없거나. 진짜 "압축(내용 요약)"은 표의 다른 방법인 "대화 요약"에서 AI를 불러 별도로 하는 것
- 저장(`history.add_messages`)은 항상 원본 전체로 함. 트리밍은 "모델에게 뭘 보여줄지"만 결정, 저장소 자체를 지우지 않음
- 위험: 오래된 중요 정보("내 이름은 철수야")도 그냥 오래됐다는 이유로 잘려나갈 수 있음

### Short-term vs Long-term memory

| 구분 | Short-term | Long-term |
|---|---|---|
| 범위 | 현재 대화 세션 | 여러 세션 |
| 예시 | "아까 철수라고 했잖아" | "이 사용자는 백엔드 개발자다" |

이 노트북은 short-term만 다룸. long-term(세션 넘어 기억)은 이후 과정에서.

---

## Part 2. Structured Output과 Output Parser

### Output Parser 5종류

| Parser | 결과 타입 | 언제 |
|---|---|---|
| `StrOutputParser` | `str` | 텍스트만 |
| `CommaSeparatedListOutputParser` | `list[str]` | 간단한 목록 |
| `JsonOutputParser` | `dict`/`list` | 유연한 JSON |
| `PydanticOutputParser` | 검증된 객체 | JSON 파싱 + 스키마 검증 |
| `with_structured_output()` | Pydantic 객체 등 | 모델이 네이티브 지원할 때 |

**⚠️ Parser는 "강제"가 아니라 "유도"**: 프롬프트로 형식을 부탁하는 거지, 모델이 반드시 지킨다는 보장 없음. Parser는 사후에 변환·검증만 함.

### 검증 4단계 (Parser가 담당하는 건 앞 2개뿐)

```
JSON 문법 → schema → domain(업무 규칙) → policy(서비스 정책)
```
뒤 2단계(domain, policy)는 개발자가 직접 검증 코드를 짜야 함.

### `PromptTemplate.partial()` — 안 바뀌는 값 미리 채우기

```python
prompt = PromptTemplate.from_template("입력: {text}\n{format_instructions}")
prompt = prompt.partial(format_instructions=parser.get_format_instructions())
prompt.invoke({"text": "..."})  # text만 넘기면 됨
```
`functools.partial()`과 같은 발상. `format_instructions`는 호출마다 안 바뀌는 고정값이라 미리 박아둠. **이 시점에 실행/완성 안 됨** — 아직 안 채워진 변수만 받는 새 템플릿 객체를 반환할 뿐.

### `PydanticOutputParser` 흐름 4단계

```python
class SentimentResult(BaseModel):
    sentiment: Literal["긍정", "부정", "중립"] = Field(description="...")
    intensity: int = Field(ge=1, le=5, description="...")
    reason: str = Field(min_length=1, description="...")

pydantic_parser = PydanticOutputParser(pydantic_object=SentimentResult)
pydantic_prompt = PromptTemplate.from_template("...{format_instructions}").partial(
    format_instructions=pydantic_parser.get_format_instructions()
)
pydantic_chain = pydantic_prompt | llm | pydantic_parser
```

1. **클래스로 "원하는 틀" 정의** — `BaseModel` 상속받으면 자동으로 타입 검증 능력이 생김. `Literal`로 허용값 제한, `ge/le`로 숫자 범위, `min_length`로 문자열 길이 제한
2. **`PydanticOutputParser(pydantic_object=SentimentResult)`** — 그 틀을 생성자에 전달 → 객체 안에 저장됨 (그래서 나중에 `get_format_instructions()`를 인자 없이 호출해도 됨 — 이미 자기 안에 스키마를 기억하고 있어서)
3. **`get_format_instructions()`** — 저장해둔 스키마를 읽어서 "이 형식으로 답해라"는 문장을 자동 생성 (개발자가 직접 문장을 쓰는 게 아니라, 클래스 정의를 자동으로 "번역"만 함)
4. **`pydantic_chain`** — 모델 응답을 받아 그 스키마 객체로 변환 + 검증

### `with_structured_output()` vs `PydanticOutputParser`

| | `with_structured_output()` | `PydanticOutputParser` |
|---|---|---|
| 전달 방식 | 스키마를 모델 API에 직접 전달 | format instructions를 프롬프트에 텍스트로 전달 |
| 모델 요구사항 | 네이티브 구조화 출력 지원 필요 | 아무 텍스트 모델에서나 가능 |
| 제어 범위 | 간결하지만 provider 동작에 의존 | 전처리·예외처리·재시도를 직접 구성 가능 |

`PydanticOutputParser`가 필요한 경우: provider가 네이티브 지원 안 할 때, 여러 provider에서 같은 방식 유지해야 할 때, 이미 저장된 텍스트를 검증해야 할 때, `FakeListChatModel` 같은 테스트 모델로 재현 테스트할 때.

### `FakeListChatModel` — 테스트용 가짜 응답기 (실제로 확인함)

```python
fake = FakeListChatModel(responses=["첫번째", "두번째"])
```
- 생성자에 넘긴 `responses` 리스트를 객체 안에 저장 + 내부 카운터도 같이 가짐
- `.invoke()` 호출될 때마다 리스트에서 **순서대로** 하나씩 꺼내 반환, 카운터 증가
- **실험으로 확인**: 리스트 길이(2개)를 넘어 3번째, 4번째 호출하면 **처음으로 되돌아가서 순환**(1번째, 2번째, 1번째, 2번째...) — 에러 안 나고 계속 돎
- 진짜 AI처럼 내용을 이해해서 답하는 게 아니라, 그냥 "미리 적어둔 카드를 순서대로 뒤집는" 것뿐. 역전파나 학습과 무관

> ♻️ 처음엔 `FakeListChatModel`을 "미분 가능한 프롬프트 학습용 대리 모델"이라고 이해했으나, 이는 완전히 다른 개념과 혼동한 오답이었다. `FakeListChatModel`은 미리 적어둔 응답 리스트를 순서대로 반환하기만 하는 테스트용 가짜 모델일 뿐, 학습이나 역전파와는 무관하다.

### `.with_retry()` vs `.with_fallbacks()` — 핵심 구분

둘 다 "원래 입력을 처음부터 다시 흘려보낸다"는 공통점이 있지만:

| | 무엇을 다시 실행하나 |
|---|---|
| `.with_retry()` | **같은 파이프라인 하나**를 반복 재실행 |
| `.with_fallbacks([...])` | **미리 만들어둔 다른 파이프라인**으로 교체해서 실행 |

```python
# retry: 파이프라인 1개
chain = (RunnableLambda(count) | prompt | fake_llm | parser).with_retry(
    retry_if_exception_type=(OutputParserException,), stop_after_attempt=3
)

# fallback: 파이프라인 2개 (미리 따로 만들어둠)
primary_chain = RunnableLambda(항상_실패하는_함수)
fallback_chain = prompt | fake_llm2 | parser
resilient_chain = primary_chain.with_fallbacks([fallback_chain])
```

- retry는 `01-langchain-basic.ipynb`에서 이미 배운 `.with_fallbacks()`(다른 모델로 전환, 예: `gemini-3.6-flash` 실패 → `gemini-2.5-flash-lite`)의 자매 개념. 실전에서 fallback은 진짜 다른 모델로 갈아타는 경우가 흔함
- retry_if_exception_type으로 재시도할 에러 종류 한정, stop_after_attempt로 무한 반복 방지

### 모든 실패를 재시도하면 안 된다

| 실패 상황 | 권장 |
|---|---|
| 일시적 형식 오류 | 횟수 제한 재시도 |
| 필수 입력 부족 | 사용자에게 추가 정보 요청 |
| 허용 범위 벗어남 | 오류 알리고 중단/재입력 |
| **정책 위반** | **재시도하지 않고 중단** (반복해도 똑같이 잘못될 가능성 높음, 비용만 낭비) |
| 일시적 provider 장애 | fallback 모델 |

### 선택 기준 요약

- 단순 텍스트 → `StrOutputParser`
- 유연한 JSON → `JsonOutputParser`
- 엄격한 검증 필요 → `PydanticOutputParser`
- 모델이 지원 + 스키마 명확 → `with_structured_output()`

## 핵심 한 줄 정리

- 메모리: LLM은 stateless, "기억"은 전부 우리가 매번 통째로 다시 보내주는 착시. RAM 저장이라 프로세스 꺼지면 사라짐. 트리밍은 압축이 아니라 삭제
- Structured Output: 클래스로 "원하는 틀"을 정의하면 파서가 그 틀을 저장해두고, 지시문 생성과 검증을 자동으로 대신해줌. 재시도(같은 체인 반복)와 fallback(다른 체인 교체)은 목적이 다름

---

## ✅ 확인 질문

1. "대화가 이어지는 것처럼" 보이는 게 왜 모델이 똑똑해서가 아닌가?
2. `InMemoryChatMessageHistory`를 `session_id`별 딕셔너리로 관리하는 이유는?
3. "In-Memory" 저장 방식의 한계는 무엇이며, 실제 서비스에서는 무엇으로 대체하는가?
4. `trim_messages()`가 "압축"이 아니라 "삭제"라는 말은 무슨 뜻인가? `max_tokens`는 무엇을 기준으로 자르는가?
5. Output Parser가 "강제"가 아니라 "유도"라는 것은 무슨 의미인가?
6. 검증 4단계 중 Parser가 담당하는 범위는 어디까지이며, 나머지는 누가 처리하는가?
7. `PromptTemplate.partial()`은 호출 시점에 무엇을 하는가? `functools.partial()`과 어떤 발상이 같은가?
8. `PydanticOutputParser`가 `get_format_instructions()`를 인자 없이 호출할 수 있는 이유는?
9. `with_structured_output()`과 `PydanticOutputParser`는 스키마를 모델에 전달하는 방식이 어떻게 다른가?
10. `FakeListChatModel`이 실제 AI와 근본적으로 다른 점은 무엇인가?
11. `.with_retry()`와 `.with_fallbacks()`는 각각 무엇을 다시 실행하는가?
12. 정책 위반으로 인한 실패를 재시도하면 안 되는 이유는?
