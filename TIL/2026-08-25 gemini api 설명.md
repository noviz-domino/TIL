

# TIL 2026-08-25 (2) — 환경변수 통합, Gemini API

  

`.env` 파일 통합 작업, Gemini API 노트북 전체(REST/SDK, 대화 맥락, 오류 처리), 그리고 실습 중 만난 에러 두 가지.

  

---

  

## 1. `.env` 파일 하나로 통합하기

  

### `load_dotenv()`는 위로 올라가며 찾는다

  

`load_dotenv()`는 **지금 실행 위치에서 시작해서 부모 폴더로 계속 올라가며 `.env`를 찾는다.** 다만 **가장 가까운 것을 찾으면 거기서 멈추고, 더 위로는 안 올라간다.**

  

그래서 `과제/.env`, `aim-ai-agent-practice/python/api/.env`처럼 **각 폴더에 이미 `.env`가 있으면**, 최상위(`260811/.env`)에 새 키를 넣어도 하위 폴더에서는 못 찾음 (가까운 것에서 이미 멈춰버림).

  

### 해결: 기존 두 개를 지우고 최상위 하나로 합침

  

```

260811/

├── .env              ← DUST_API_KEY, GEMINI_API_KEY 전부 여기

├── .gitignore         ← 안전장치 (260811 은 git 저장소가 아니라 지금은 효과 없지만, 나중에 대비)

├── 과제/                (.env 없음, 위로 올라가서 260811/.env 를 찾음)

└── aim-ai-agent-practice/python/api/  (.env 없음, 마찬가지)

```

  

실제로 두 폴더 각각에서 실행 테스트해서 `DUST_API_KEY`, `GEMINI_API_KEY` 둘 다 정상적으로 로드되는 것 확인함.

  

### 이 구조가 오히려 더 안전한 이유

  

`aim-ai-agent-practice`는 자체 git 저장소. `.env`가 그 저장소 **바깥**(`260811`)에 있으면, 그 저장소 안에서 `git add .`를 잘못 눌러도 **애초에 저장소 범위 밖이라 커밋 대상이 될 수 없음.** `.gitignore`로 막는 것보다 한 단계 더 근본적으로 안전.

  

---

  

## 2. Gemini API — REST부터 SDK까지

  

### 큰 그림

  

파이썬 코드에서 AI(Gemini)에게 질문을 보내고 답을 받는 법. 인터넷으로 요청 보내고 응답 받는 원리 자체는 지난번 자판기/미세먼지 API와 같음 — **상대가 AI라는 것만 다름.**

  

### REST — 직접 만들어보기 (원리 확인용)

  

```python

url = "https://generativelanguage.googleapis.com/v1beta/interactions"

headers = {

    "Content-Type": "application/json",

    "x-goog-api-key": api_key,

}

body = {

    "model": model,

    "input": "오늘 몇일이니? 모르면 모른다고 해",

    "store": False,

}

  

response = requests.post(url, headers=headers, json=body, timeout=30)

response.raise_for_status()

  

response_data = response.json()

answer = response_data["steps"][-1]["content"][0]["text"]

```

  

- `requests.get`은 "받아오기", `requests.post`는 "보내기". 질문을 보내야 하니 `post`.

- 응답 구조: `{"steps": [{"type": "user_input", ...}, {"type": "model_output", "content": [{"type": "text", "text": "..."}]}]}`

- `["steps"][-1]` → 마지막 step(항상 AI의 최종 답변) → `["content"][0]["text"]` → 실제 글자.

  

### SDK — 같은 일을 편하게

  

```python

client = genai.Client(api_key=api_key)

  

response = client.interactions.create(

    model=model,

    input="...",

    store=False,

)

print(response.output_text)

```

  

| REST 방식 | SDK 방식 |

|---|---|

| `url`, `headers`, `body` 직접 작성 | `client` 객체 안에 이미 들어있음 |

| `requests.post(...)` | `client.interactions.create(...)` |

| `["steps"][-1]["content"][0]["text"]` | `.output_text` |

  

`create()`라는 이름이지만 로컬에서 뭘 만드는 게 아니라 **실제로 네트워크 요청이 나감** (이름과 동작이 다를 수 있다는 점 주의).

  

### `input`에 넣는 두 형태

  

```python

input="안녕하세요"                                    # 한 번의 간단한 질문

input=[{"type": "user_input", "content": [{"type": "text", "text": "안녕하세요"}]}]  # 대화 기록 등

```

  

바깥 `type`(user_input 등)은 "이 덩어리의 역할", 안쪽 `type`(text 등)은 "내용의 종류". 딕셔너리+리스트 조합일 뿐 새 문법 아님.

  

### `system_instruction` = 시스템 프롬프트

  

다른 서비스에서 "system prompt"/"system message"라고 부르는 것과 동일한 개념. Gemini SDK에서만 이름이 `system_instruction`.

  

```python

result = client.interactions.create(

    model=model, input=question,

    system_instruction="Python 입문 강사입니다. 쉬운 말로 40자 이내에서 답하세요.",

)

```

  

`input`(이번 질문)과 `system_instruction`(계속 유지되는 역할)은 역할이 다름 — 질문은 그대로 두고 역할만 바꾸면 답변 스타일이 달라짐.

  

---

  

## 3. AI는 스스로 기억하지 못한다 — 대화 이어가기 두 가지 방법

  

매 `create()` 호출은 서로 독립적인 사건. 이전 대화를 안 넣으면 AI는 정말로 모름.

  

### 방법 1: `history`를 직접 관리해서 매번 통째로 다시 보내기

  

```python

history = [{"type": "user_input", "content": [{"type": "text", "text": "내 이름은 민수야."}]}]

  

turn1 = client.interactions.create(model=model, input=history, store=False)

  

history.extend(step.model_dump(exclude_none=True) for step in turn1.steps)  # 응답도 기록에 추가

history.append({"type": "user_input", "content": [{"type": "text", "text": "내 이름이 뭐야?"}]})

  

turn2 = client.interactions.create(model=model, input=history, store=False)

```

  

`.model_dump(exclude_none=True)`는 SDK 객체를 다시 쓸 수 있는 딕셔너리로 바꿔주는 변환. **AI가 진짜 기억하는 게 아니라, 매번 전체 대화를 새로 읽고 답하는 것.**

  

### 방법 2: `store=True` — 서버가 대신 기억

  

```python

turn1 = client.interactions.create(model=model, input="내 이름은 민수야.", store=True)

  

turn2 = client.interactions.create(

    model=model, input="내 이름이 뭐야?",

    previous_interaction_id=turn1.id,   # 대화 전체 대신 번호표 하나만 전달

    store=True,

)

```

  

`history` 방식 = "내가 수첩에 다 적어서 매번 통째로 보여줌", `store=True` = "서버 서랍에 넣어두고 번호만 말함".

  

### 대화 분기

  

```python

turn1 = client.interactions.create(model=model, input="내 이름은 민수야.", store=True)

turn2 = client.interactions.create(model=model, input="...피자야...", previous_interaction_id=turn1.id, store=True)

turn3 = client.interactions.create(model=model, input="...", previous_interaction_id=turn1.id, store=True)  # 다시 turn1 에서

```

  

`previous_interaction_id`는 항상 최신 것을 가리켜야 하는 게 아니라, **어느 시점이든 다시 지정해서 그 지점부터 새 갈래로 뻗어나갈 수 있음.** turn2가 사라지는 게 아니라 별개 갈래로 남아있음.

  

---

  

## 4. `generation_config` — 답변을 "어떻게" 만들지

  

```python

generation_config={

    "max_output_tokens": 1000,   # 답변 최대 길이 (정확한 글자수 아님, 너무 작으면 중간에 끊김)

    "thinking_level": "high",    # low/medium/high — 답하기 전 얼마나 깊게 생각할지

}

```

  

`system_instruction`은 "무엇을" 답할지, `generation_config`는 "생성 엔진 설정값"을 조절 — 서로 다른 층위.

  

---

  

## 5. 오류 처리 — 이미 배운 것의 재사용

  

```python

try:

    result = client.interactions.create(model=model, input=prompt, store=False)

    return result.output_text

except errors.ClientError as error:      # 내 쪽 문제 (키/모델명/요청 오류)

    if error.code in [401, 403]:

        ...

    elif error.code == 429:              # 사용량 제한

        ...

except errors.ServerError as error:      # 구글 서버 쪽 문제

    ...

```

  

지난번 자판기의 `NotFoundException`/`OutOfStockException`과 같은 패턴 — 원인별로 다르게 대응하려면 종류를 나눠야 함.

  

### `.output_text`, `.code`, `.message`는 어디서 오나

  

**함수가 아니라 속성.** `google-genai` 라이브러리가 미리 만들어둔 클래스들에 정의되어 있음.

  

- `.output_text` → `create()`가 돌려주는 `Interaction` 객체의 속성 (우리가 만든 `Person.name` 같은 것과 같은 구조, 다만 클래스를 구글이 만듦)

- `.code`, `.message` → `errors.ClientError`/`errors.ServerError` 클래스의 속성. 우리가 `class MyCustomError(Exception): pass`로 아무것도 안 채웠던 것과 달리, 구글은 `__init__`에서 `self.code`, `self.message`를 직접 저장하도록 만들어놓은 것.

  

---

  

## 6. 실습 중 만난 에러 두 가지

  

### `NameError: name 'client' is not defined`

  

주피터 노트북은 셀들이 **하나의 메모리 공간을 공유**하지만, **실행한 셀만** 그 효과가 남는다. `client = genai.Client(...)`를 만드는 셀을 실행 안 하고 그걸 쓰는 셀만 실행하면 이 에러가 남. 위에서부터 순서대로 실행해야 함.

  

→ 이후 각 셀을 **독립적으로 실행 가능한 완전한 코드**로 요청 → 매 셀 맨 위에 `import`, `load_dotenv()`, `client` 생성까지 전부 다시 넣는 방식으로 정리.

  

### `NotFoundError: Model 'Gemini 3.5 Flash Lite' not found`

  

```python

model = os.getenv("GEMINI_MODEL", "Gemini 3.5 Flash Lite")   # 잘못된 기본값

```

  

사람이 읽는 표시 이름(`Gemini 3.5 Flash Lite`)과 API에 실제로 넘겨야 하는 이름(`gemini-3.5-flash-lite`, 소문자+하이픈)이 다름. `ClientError` 404로 분류되는 사례 — 요청 쪽(내 코드)의 잘못이라 직접 고쳐야 하는 경우.

  

```python

model = os.getenv("GEMINI_MODEL", "gemini-3.5-flash-lite")   # 수정

```

  

---

  

## 오늘의 핵심 정리

  

1. **`load_dotenv()`는 가까운 `.env`에서 멈춘다** — 여러 폴더에서 같은 키를 쓰려면 하위 폴더의 개별 `.env`를 없애고 상위 폴더 하나로 모아야 함.

2. **REST와 SDK는 같은 일을 하는 두 가지 방법.** SDK(`client.interactions.create`)는 REST의 `url`+`headers`+`body`+`requests.post`를 대신 처리.

3. **AI는 요청 간 기억을 안 한다.** `history`를 직접 매번 다시 보내거나(`store=False`), `previous_interaction_id`로 서버에 저장된 대화를 이어받는(`store=True`) 두 가지 방법.

4. **오류 종류(ClientError/ServerError)로 나눠서 처리하는 이유는 원인별로 대응이 다르기 때문** — 지난 자판기 예외 처리와 동일한 논리.

5. **모델 이름은 표시용 이름과 API용 ID가 다르다** — 에러 메시지가 정확한 이름을 알려주므로 그대로 따르면 됨.