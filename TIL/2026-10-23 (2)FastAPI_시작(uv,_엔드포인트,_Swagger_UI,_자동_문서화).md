---
tags: [fastapi, http, endpoint, swagger, uv]
til: v2 2026-10-23
---

# 2026-10-23 (2)FastAPI 시작(uv, 엔드포인트, Swagger UI, 자동 문서화)
> 작성일: 2026-10-23

## 🔗 관련 글

- [서버·클라이언트 설명](2026-07-30_서버_클라이언트_설명.md) — 클라이언트가 요청하고 서버가 응답한다는 구조, HTTP, API의 기본 개념을 처음 다뤘다. 이 글은 그 서버 쪽을 직접 Python으로 만든다.
- [파이썬 기초문법 학습4](2026-08-14%20파이썬%20기초문법%20학습4.md) — 라이브러리와 프레임워크의 차이, 제어의 역전(IoC)을 다뤘다. FastAPI가 "프레임워크"이고, `@app.get`으로 등록한 함수를 프레임워크가 요청이 올 때 호출한다.
- [MCP(Host·Client·Server, Tool·Resource·Prompt, JSON-RPC, FastMCP, A2A)](2026-10-23%20(1)MCP(Host_Client_Server,_Tool_Resource_Prompt,_JSON-RPC,_FastMCP,_A2A).md) — 같은 날 앞 수업이다. `@mcp.tool()`처럼 데코레이터로 함수를 등록하는 방식이 `@app.get("/")`과 닮았다.

## ❓ 아직 모르겠는 것

이 글은 강의 레슨(FastAPI Intro)을 읽고 정리했고, 대화 중 사용자가 직접 한 질문은 없다. 아래는 사용자의 발언이 아니라, 함께 학습한 ChatGPT 정리본이 따로 짚어 둔 헷갈리기 쉬운 지점이다. 복습 때 다시 확인한다.

- API와 Swagger UI는 같은 것인가? (주제 6, 아래 ➕ 참고)
- 서버의 Python 함수 이름이 URL을 결정하는가? (주제 3, 아래 ➕ 참고)
- `async def`를 쓰면 코드가 저절로 빨라지는가? (아래 ➕ 참고)

## 주제 1: FastAPI란 무엇인가

FastAPI는 Python 기반의 애플리케이션 개발을 위한 프레임워크이며, Python 기반 프레임워크 중 최상위권의 속도를 가진다.

- 웹사이트나 웹 서비스를 만들 때 필요한 많은 기능을 미리 구현해 놓아, 개발자가 **비즈니스 로직에만 집중**할 수 있게 해 준다.
- 비동기 처리와 스트리밍 응답을 지원하며, **LLM이나 에이전트 기능을 HTTP API로 제공**하는 데 활용할 수 있다.

### 클라이언트와 서버의 요청·응답

클라이언트는 브라우저나 앱처럼 서버에 기능이나 데이터를 요청하는 쪽이다. 서버는 요청을 처리하고 결과를 응답으로 돌려준다. FastAPI는 **서버에서 요청을 처리하는 애플리케이션**을 만드는 데 사용한다.

```text
클라이언트(브라우저·앱)  ── HTTP 요청 ──▶  서버(FastAPI 애플리케이션)
클라이언트(브라우저·앱)  ◀── HTTP 응답: JSON 등 ──  서버(FastAPI 애플리케이션)
```

## 주제 2: 프로젝트 준비와 설치

터미널에서 프로젝트를 생성하고 해당 폴더로 이동한다.

```bash
uv init fastapi_basic
cd fastapi_basic
```

`fastapi[standard]`를 설치하면 FastAPI와 함께 개발 서버 실행에 필요한 도구가 설치된다.

```bash
uv add "fastapi[standard]"
```

이 과정에서 프로젝트의 가상환경인 `.venv`가 생성되고 패키지가 설치된다.

## 주제 3: 첫 API 작성

프로젝트 폴더의 `main.py`에 다음 코드를 작성한다.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"Hello": "World"}
```

| 코드 | 의미 |
|---|---|
| `app = FastAPI()` | 요청을 처리할 애플리케이션 객체를 생성한다. |
| `@app.get("/")` | 루트 경로 `/`로 들어오는 **HTTP GET 요청**을 아래 함수와 연결하는 데코레이터이다. |
| 함수(`read_root`) | 해당 요청을 받으면 실행되는 함수이다. |
| `return {"Hello": "World"}` | 딕셔너리를 반환하면 FastAPI가 **JSON 응답으로 변환**하여 클라이언트에 전달한다. |

> ➕ **더 알아두기**
> 레슨의 설명문은 함수 이름을 `get_root()`라고 쓰고 있으나, 같은 레슨의 예제 코드는 `def read_root():`이다. 설명문과 코드의 표기가 어긋난 것이며, 함수 이름은 URL과 무관하므로 어느 쪽으로 써도 동작은 같다. 이 글은 코드를 기준으로 `read_root`로 적었다.

## 주제 4: 개발 서버 실행과 응답 확인

`main.py`가 있는 프로젝트 폴더에서 개발 서버를 실행한다.

```bash
uv run fastapi dev main.py
```

- 코드 변경 사항을 저장하면 서버가 **자동으로 다시 로드**된다.
- 터미널에서 `Ctrl + C`를 누르면 서버가 종료된다.

서버가 실행 중인 상태에서 브라우저로 `http://127.0.0.1:8000/`에 접속하면 다음 JSON 응답을 확인할 수 있다.

```json
{"Hello": "World"}
```

## 주제 5: 엔드포인트와 응답

**엔드포인트**는 클라이언트가 특정 기능을 요청하는 API의 접점을 말하며, **HTTP 메서드와 URL 경로로 구분**한다. 위 코드의 엔드포인트는 `GET /`이다.

하나의 애플리케이션은 여러 엔드포인트를 가질 수 있다. 다음은 `/hello` 경로에서 문자열을 반환하는 예제이다.

```python
@app.get("/hello")
def hello():
    return "Hello World!"
```

`GET /hello` 요청에 대한 응답 본문은 다음과 같은 JSON 문자열이다.

```json
"Hello World!"
```

### 리스트 응답

`GET /hi`는 문자열 세 개를 담은 리스트를 반환한다.

```python
@app.get("/hi")
def hi():
    return ["hello", "World", "!"]
```

리스트는 다음과 같은 JSON 배열로 응답한다.

```json
["hello", "World", "!"]
```

| Python 반환값 | JSON 응답 |
|---|---|
| 딕셔너리 `{"Hello": "World"}` | JSON 객체 `{"Hello": "World"}` |
| 문자열 `"Hello World!"` | JSON 문자열 `"Hello World!"` |
| 리스트 `["hello", "World", "!"]` | JSON 배열 `["hello", "World", "!"]` |

## 주제 6: 자동 문서화 (Swagger UI와 ReDoc)

FastAPI는 등록된 엔드포인트를 바탕으로 **API 명세서를 자동으로 생성**한다. 서버가 실행 중일 때 다음 주소에서 문서를 확인할 수 있다.

| 주소 | 문서 도구 | 용도 |
|---|---|---|
| `http://127.0.0.1:8000/docs` | Swagger UI | API 명세 확인 및 요청 실행 |
| `http://127.0.0.1:8000/redoc` | ReDoc | API 명세 확인 |

Swagger UI에서 `GET /`를 펼치고 **Try it out → Execute**를 누른다. 응답 상태 코드 `200`과 응답 본문의 `{"Hello": "World"}`를 확인한다.

## 헷갈리기 쉬운 부분

> ➕ **더 알아두기**
> (ChatGPT 정리본과 함께 학습한 내용이며 수업에 나온 설명은 아니다.)
>
> - **API와 Swagger UI는 다르다.** API는 서버가 공개한 HTTP 요청·응답 규약이고, Swagger UI는 그 명세를 바탕으로 API를 브라우저에서 조회·시험하게 해 주는 화면이다. Swagger UI가 없어도 `requests`, `curl`, Postman, 프론트엔드에서 같은 API를 호출할 수 있다.
> - **URL을 결정하는 것은 함수 이름이 아니다.** `@app.get("/hello")`처럼 데코레이터에 적은 **HTTP 메서드와 경로**가 엔드포인트를 정한다. 함수 이름을 `hello`에서 `abc`로 바꿔도 `GET /hello`는 그대로 동작한다.
> - **JSON과 Python 딕셔너리는 같은 객체가 아니다.** 모양이 비슷할 뿐이고, HTTP로 전송될 때는 JSON 텍스트가 된다. FastAPI가 딕셔너리·리스트를 JSON으로 바꿔 준다.
> - **`async def`를 쓴다고 내부 연산이 저절로 빨라지지는 않는다.** I/O 작업(네트워크, 파일, DB 등)을 비동기로 처리해야 이점이 나타난다. 이번 레슨의 예제는 모두 일반 `def`이다.
> - `uv init`은 프로젝트를 처음 만들 때 한 번만 한다. 이미 만든 프로젝트라면 `main.py`가 있는 폴더에서 `uv run fastapi dev main.py`만 실행하면 된다.

## 정리

- FastAPI는 Python 웹 프레임워크로, 서버 쪽에서 HTTP 요청을 받아 처리하는 애플리케이션을 만든다. LLM·에이전트 기능을 HTTP API로 제공하는 데도 쓴다.
- 준비 순서는 `uv init 프로젝트` → `cd` → `uv add "fastapi[standard]"` → `uv run fastapi dev main.py`이다. 저장하면 서버가 자동으로 다시 로드된다.
- `@app.get("/")` 같은 데코레이터가 HTTP 메서드와 경로를 함수에 연결한다. 이 둘이 **엔드포인트**를 구분한다.
- 함수가 딕셔너리·문자열·리스트를 반환하면 FastAPI가 JSON(객체·문자열·배열)으로 변환한다.
- 서버를 켜 두면 `/docs`(Swagger UI)와 `/redoc`(ReDoc)에서 자동 생성된 명세를 보고, Swagger UI에서 Try it out → Execute로 요청을 실행할 수 있다.

## ✅ 확인 질문

1. FastAPI는 무엇이고 클라이언트·서버 중 어느 쪽을 만드는 데 쓰나요? 이 수업에서 LLM·에이전트와 연결해 쓸 수 있다고 한 이유는 무엇인가요?
2. `uv init`, `uv add "fastapi[standard]"`, `uv run fastapi dev main.py`는 각각 무엇을 하나요? `.venv`는 언제 만들어지나요?
3. `@app.get("/")` 데코레이터는 무엇과 무엇을 연결하나요? `app = FastAPI()`는 무슨 일을 하나요?
4. 함수가 딕셔너리를 반환했을 때 클라이언트는 어떤 형태로 받나요? 문자열과 리스트를 반환할 때의 응답은 각각 어떻게 다른가요?
5. 엔드포인트는 무엇으로 구분하나요? 같은 경로 `/`라도 `GET /`과 `POST /`가 다른 엔드포인트인 이유를 설명해 보세요.
6. 개발 서버를 실행한 상태에서 코드를 고치고 저장하면 어떻게 되나요? 서버는 어떻게 종료하나요?
7. `/docs`와 `/redoc`은 각각 무슨 도구이고 어떤 용도인가요? Swagger UI에서 요청을 실행하는 순서를 설명해 보세요.
8. API와 Swagger UI는 어떻게 다른가요? Swagger UI 없이도 API를 호출할 수 있는 방법을 들어 보세요.
