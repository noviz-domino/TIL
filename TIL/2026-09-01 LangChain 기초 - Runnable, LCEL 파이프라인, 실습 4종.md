# 2026-09-01 LangChain 기초 — Runnable, LCEL 파이프라인, 실습 4종

## 1. LangChain이 뭔가

**한 줄**: 모델마다 제각각인 API 호출법을 **하나의 규격으로 통일**해주고, LLM 앱에 반복적으로 필요한 부품들을 미리 만들어둔 프레임워크.

- 모델은 다 다르고 API 호출법도 다르지만, LangChain이 규격화해서 어떤 모델이든 `invoke()` 하나로 다룰 수 있게 해줌
- 모델 호출뿐 아니라 프롬프트, 파서, 체인, 메모리, RAG 부품까지 전부 같은 규격으로 제공
- **단순한 한 번의 호출이면 직접 호출이 더 간단함.** 기능이 복잡해질수록(모델 교체, 프롬프트 재사용, 여러 단계 조립) 가치가 커짐

### 생태계 (패키지가 나뉜 이유)
```
langchain-core (핵심: Runnable, LCEL, 메시지 타입)
    ├── langchain (체인/메모리)
    ├── langchain-google-genai (Gemini 연동 → ChatGoogleGenerativeAI)
    ├── langchain-community (서드파티 통합)
    └── langgraph (Agent)
```
Google 쓰는 사람은 `langchain-google-genai`만, OpenAI 쓰는 사람은 `langchain-openai`만 설치하면 됨. 핵심 로직은 공통이고 **"제공자별 연결 부품"만 갈아끼우는 구조**.

## 2. Runnable — 부품들이 공유하는 "공통 자격증"

프롬프트, ChatModel, OutputParser, 체인까지 **전부 `Runnable`을 상속**받음. 그래서:
- 전부 `invoke()`로 실행 가능
- 전부 `|`로 연결 가능

### ⚠️ 중요: "규격 통일" = 형식이지 타입이 아니다

`invoke()`가 같다는 건 **"값 하나 받아서 값 하나 반환"이라는 패턴(모양)**이 같다는 뜻이지, **주고받는 데이터 타입이 같다는 게 아님.**

| 부품 | 입력 | 출력 |
|---|---|---|
| 프롬프트 | 변수 딕셔너리 `{role: "Python", ...}` | 메시지 묶음 `[SystemMessage, HumanMessage]` |
| 모델(llm) | 메시지 묶음 | AIMessage 객체 (텍스트+토큰정보+도구호출) |
| 파서 | AIMessage 객체 | 순수 텍스트 |

**앞 부품의 "출력" 칸 = 다음 부품의 "입력" 칸**이 딱 맞물려야 작동함.
→ 그래서 `prompt | llm | parser`는 사실상 **이 순서로만 성립**함. 순서 바꾸면 타입 안 맞아서 에러.

- `invoke()` 통일 = `|`를 쓸 수 있는 **자격**을 줌 (형식적 허용)
- 타입 일치 = 그 조합이 **실제로 돌아가는지**를 결정 (실질적 제약)

### 모델 두 개를 연달아 못 붙이는 이유
모델 출력(AIMessage)은 다음 모델 입력(메시지 묶음)이 될 수 없음. 실제 프롬프트 체이닝은:
```
프롬프트 | 모델 | 파서 | 변환기 | 프롬프트 | 모델 | 파서   (7단계)
```
모델 사이에 **파서(텍스트로) → 변환기(딕셔너리로) → 새 프롬프트(메시지로)** 가 끼어서 타입을 다시 맞춰줌.

## 3. ⭐ `|` 파이프 연산자의 정체 (제일 헷갈렸던 부분)

**`|`는 import 대상인 "이름"이 아니라 파이썬 문법 기호**라서, 경로도 import도 필요 없음.

파이썬은 `a | b`를 자동으로 이렇게 번역함:
```python
a | b   →   a.__or__(b)
```

즉 `prompt | llm | parser`는 내부적으로:
```python
prompt.__or__(llm).__or__(parser)
```

**찾던 "숨은 경로"는 `모듈.무언가`가 아니라 `|` 왼쪽의 객체 자신이었음.** `Runnable` 부모 클래스에 `__or__`가 "두 Runnable을 이어붙여 새 체인을 만들어라"로 정의돼 있고, 모든 부품이 그걸 상속받음.

비교:
```python
len(x)        # "len"이라는 이름 필요 → builtin
x + y         # 이름 없음, 문법 → x.__add__(y)
prompt | llm  # 이름 없음, 문법 → prompt.__or__(llm)
```

### 왜 하필 `|` 기호인가
대충 고른 게 아니라 **리눅스 터미널의 파이프(`ls | grep foo`)** 관례를 그대로 가져온 것. "앞 명령의 출력을 뒤 명령의 입력으로 넘긴다"는 의미가 수십 년째 통용돼서, 개발자에게 익숙한 기호를 재사용한 것. 그래서 이름도 "LCEL **파이프**라인".

## 4. 주요 부품들

### ChatModel과 메시지 역할
`("system", ...)`, `("human"/"user", ...)`, `("ai"/"assistant", ...)` — **고정된 약속어**임. 소문자로 써야 하고 마음대로 지어내면 안 됨.

`"ai"` 역할의 두 가지 용도:
1. 진짜 이전 대화 기록
2. **few-shot 예시** — AI가 실제로 답한 적 없지만 "이렇게 답해라"는 견본으로 우리가 지어내서 넣음

`("human", ...)`, `("ai", ...)`를 **여러 쌍 번갈아 넣어도 됨.** 전체가 **한 번의 요청**으로 통째로 전달되고, 모델은 이걸 "이미 오갔던 대화"로 인식해서 패턴을 따라감.

### PromptTemplate vs ChatPromptTemplate
- `PromptTemplate`: 역할 구분 없는 단일 문자열
- `ChatPromptTemplate`: system/human/ai 역할 구분

`from_messages([...])`는 **형식(틀)만 지정.** `{context}` 같은 자리는 아직 빈칸이고, 실제 값은 `invoke()` 호출 시점에 채워짐. (빈칸 있는 초대장 양식을 만드는 것 ≠ 초대장을 보내는 것)

### OutputParser
AIMessage에서 부가정보(토큰 사용량 등) 다 떼고 **순수 텍스트만** 추출. 다음 단계로 깔끔한 문자열을 넘기기 위해 필요.

### RunnableLambda — 일반 함수를 파이프라인에 끼우기
일반 파이썬 함수는 `invoke()`가 없어서 파이프라인에 못 낌. `RunnableLambda`로 감싸면 **함수 내용은 그대로, 부르는 방식만 `invoke()`로 통일**되어 `|`로 연결 가능해짐.

`Runnable`을 직접 상속해도 되지만 `invoke()` 내부 로직을 다 짜야 해서 번거로움. `RunnableLambda`는 그 작업이 이미 완료된 "함수 전용 어댑터".

### RunnablePassthrough — 입력 유지 + 새 값 추가
`RunnablePassthrough.assign(context=fn)` = 기존 입력은 그대로 두고 새 키를 추가.

**중요**: Passthrough 자체가 "알아서 똑똑하게 찾아주는" 게 아님. **AI가 아니라 우리가 짠 일반 파이썬 함수가** 실행돼서 값을 채움. Passthrough는 "유지 + 추가"라는 **틀**만 제공.

```
1. 사용자가 질문만 보냄
2. 일반 파이썬 함수가 배경지식을 만들어냄   ← AI 아님
3. Passthrough가 질문 + 배경지식을 합침
4. 그제서야 프롬프트 → 모델(AI)로 넘어감    ← AI는 여기서만 관여
```

## 5. 실행 방법 (invoke 외)

| 목적 | 동기 | 비동기 |
|---|---|---|
| 입력 하나 | `invoke()` | `ainvoke()` |
| 입력 여러 개 | `batch()` | `abatch()` |
| 스트리밍 | `stream()` | `astream()` |

### batch vs asyncio.gather
**기능은 같음** (동시 실행). 차이는 **소속**:
- `gather()` = 파이썬 표준 도구, 서로 다른 코루틴을 자유롭게 조합
- `batch()` = LangChain 전용, 같은 체인에 여러 입력 적용 시 동시성 제한(`max_concurrency`)·추적 등을 LangChain 방식으로 관리

`batch()`가 더 비동기적이어서 쓰는 게 아니라, **LangChain 생태계 안에서 통합 관리되니까** 쓰는 것.

### stream과 중단
`stream()` 자체가 토큰을 아껴주진 않음 (끝까지 받으면 총 토큰은 동일). **핵심은 "중간에 멈출 선택권"을 준다는 것.**

중단 방법은 전용 API가 아니라 **그냥 `for` + `break`**:
```python
for chunk in chain.stream(...):
    print(chunk, end="")
    if 조건:
        break   # 더 이상 다음 chunk를 안 받아옴
```

## 6. 에러 핸들링 / 캐싱

**에러 대응 3단계**: 자동 재시도(`max_retries`, exponential backoff로 1→2→4초) → fallback 모델(`with_fallbacks`) → `try/except` 최종 처리

**캐싱**: 같은 질문 반복 시 API를 다시 안 부르고 저장된 답 반환 → **토큰(비용)과 시간 절약**. 개발 중 후처리 코드만 고치며 테스트할 때 유용.

## 7. Interactions API vs generateContent (조사한 내용)

2026년 6월 기준 **Interactions API가 정식 출시(GA)되어 신규 프로젝트에 권장**. generateContent는 레거시로 분류되나 계속 지원됨.

| | generateContent | Interactions API |
|---|---|---|
| 대화 기록 | 매 턴 **전체를 다시 통째로** 전송 | 서버가 기억, **새 메시지만** 전송 (`previous_interaction_id`) |
| 효율 | 대화 길어질수록 눈덩이 | 10턴 기준 입력 79% 감소, 전송량 평균 85% 작음 |

**부담이 클라이언트 → 서버로 옮겨간 구조.** TMDB 에이전트에서 `previous_interaction_id`를 쓴 게 정확히 이 장점을 활용한 것.

- LangChain의 `ChatGoogleGenerativeAI`는 아직 내부적으로 `generate_content`를 씀
- 그래서 실행 시 뜨는 `"Direct use of AFC in Models.generate_content is not recommended"` 경고는 **에러가 아님.** SDK 3.0부터 AFC(자동 도구 호출)가 generate_content에서 빠질 예정이라는 사전 안내일 뿐이고, LangChain 뒤에 숨어있고 도구도 안 쓰는 지금 상황엔 무관.

## 8. 실습 4종

### 공통 패턴
```python
prompt = ChatPromptTemplate.from_messages([...])
llm = ChatGoogleGenerativeAI(model=MODEL_NAME)
parser = StrOutputParser()
chain = prompt | llm | parser
result = chain.invoke({...})
```

### 1. 말투 변환기
`{tone}`, `{sentence}` 두 변수. `for tone in tones:`로 **같은 체인을 재사용**하며 tone만 교체.

### 2. 감정분석기 (few-shot)
`("system", 지시)` 다음, `("human", "{sentence}")` **이전에** few-shot 예시 쌍을 끼워넣음:
```python
("human", "괜찮은 것 같아요. 가격 대비 무난합니다."),
("ai", "- 감정: 중립\n- 강도: 약함\n- 근거: ..."),
```
예시 답변을 **원하는 출력 형식 그대로** 써주면 모델이 그 형식을 따라감.

> 변수(`{check}`)로 목록을 넘기든 문장에 직접 쓰든 **모델이 받는 텍스트는 동일**함. 다만 카테고리 목록을 명시하는 것 자체는 제약을 거는 효과가 있고, 재사용성 면에서 변수도 나쁘지 않음. 진짜 강제하려면 `with_structured_output()` (스키마 기반 구조화 출력)이 답.

### 3. 번역기
`"문장 -언어"` 형식으로 목표 언어를 지정. few-shot으로 "한→영", "영→일"만 보여줬는데 **예시에 없던 한→프랑스어도 같은 형식으로 정확히 출력됨.**
→ few-shot의 핵심 가치: **규칙을 다 알려주지 않아도 패턴만 보여주면 새 상황에 일반화함.**

### 4. QA 체인 (RunnablePassthrough)
```python
docs = {"FastAPI": "...", "Django": "..."}          # 키워드-문서 딕셔너리

def search_docs(input_dict):                         # 일반 파이썬 함수
    question = input_dict["question"]
    for keyword, doc in docs.items():
        if keyword in question:
            return doc
    return "관련 문서를 찾을 수 없습니다."

add_context = RunnablePassthrough.assign(context=search_docs)
chain = add_context | prompt | llm | parser
chain.invoke({"question": "FastAPI"})                # 사용자는 질문만 넘김
```

**`{context}`가 어디서 오는지 헷갈렸던 부분**: 프롬프트 코드만 보면 안 보이는 게 당연함. `add_context`가 **prompt보다 먼저 실행되어** 딕셔너리에 `context` 키를 추가해두고, prompt는 **이미 채워진 상태로 도착한 딕셔너리**에서 값을 꺼내는 것.

## 핵심 한 줄 정리

- **LangChain** = 제각각인 모델 API를 하나의 규격(`Runnable`)으로 통일한 프레임워크
- **`invoke()`** = 모든 부품의 공통 실행 방식 (형식 통일이지 타입 통일이 아님)
- **`|`** = import 필요 없는 문법 기호. `a.__or__(b)`로 번역되고, 유닉스 파이프 관례에서 따옴
- **파이프라인 순서** = 앞 출력 타입 == 뒤 입력 타입이어야만 성립
- **few-shot** = 패턴만 보여주면 새로운 입력에도 일반화됨
