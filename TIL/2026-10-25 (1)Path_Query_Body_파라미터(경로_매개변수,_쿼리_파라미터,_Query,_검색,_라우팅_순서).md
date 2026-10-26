---
tags: [fastapi, path-parameter, query-parameter, request-body, routing]
til: v2 2026-10-25
---

# 2026-10-25 (1)Path·Query·Body 파라미터(경로 매개변수, 쿼리 파라미터, Query, 검색, 라우팅 순서)
> 작성일: 2026-10-25

## 🔗 관련 글

- [Pydantic 요청 DTO(BaseModel, Request Body, 422 검증, PostCreate와 Post)](2026-10-24%20(2)Pydantic_요청_DTO(BaseModel,_Request_Body,_422_검증,_PostCreate와_Post).md) — 전 수업이다. 요청 본문(Body)을 Pydantic 모델로 받는 방법을 배웠고, 이 글은 거기에 경로(Path)와 쿼리(Query)를 더해 세 가지 입력 방식을 비교한다.
- [간단한 게시판 CRUD(RESTful 경로, APIRouter, Post 클래스, 메모리 리스트)](2026-10-23%20(3)간단한_게시판_CRUD(RESTful_경로,_APIRouter,_Post_클래스,_메모리_리스트).md) — 경로 매개변수 `/posts/{id}`를 처음 쓴 글이다.
- [SELECT 기본(DISTINCT, WHERE, ORDER BY, LIMIT OFFSET)](2026-10-06%20(1)SELECT_기본(DISTINCT,_WHERE,_ORDER_BY,_LIMIT_OFFSET).md) — 조건으로 데이터를 걸러 조회하는 SQL이다. 이 글의 검색(`?keyword=`)이 같은 일을 리스트에서 한다.
- [상태 코드와 응답 모델(HTTPException, 201·204·404·422, response_model, 응답 DTO)](2026-10-25%20(2)상태_코드와_응답_모델(HTTPException,_201_204_404_422,_response_model,_응답_DTO).md) — 같은 레슨(쿼리 파라미터와 응답)의 뒷부분이다.

## ❓ 아직 모르겠는 것

이 글은 강의 레슨(쿼리 파라미터와 응답)의 앞부분을 읽고 정리했고, 대화 중 사용자가 직접 한 질문은 없다. 아래는 사용자의 발언이 아니라, 함께 학습한 ChatGPT 정리본이 별도 항목으로 짚어 둔 헷갈리기 쉬운 지점이다. 복습 때 다시 확인한다.

- Path와 Query는 매개변수를 적는 순서로 정해지는가? (주제 2, 아래 ➕ 참고)
- `/{id}`보다 `/search`를 먼저 등록해야 하는 이유는 무엇인가? (주제 3)
- `keyword`를 생략했을 때와 빈 문자열일 때는 어떻게 다른가? (주제 3, 아래 ➕ 참고)

## 주제 1: 이번 실습의 구조

`fastapi_basic/mysite3`의 파일 구성은 다음과 같다.

| 파일 | 역할 |
|---|---|
| `mysite3/post.py` | 게시글 객체와 요청·응답 모델을 정의한다. |
| `mysite3/post_api.py` | 검색·상태 코드·예외 처리·응답 모델이 적용된 CRUD를 처리한다. |

`mysite3/post_api.py`의 라우터는 `/posts-response` 경로를 사용한다.

```python
router = APIRouter(prefix="/posts-response", tags=["mysite3"])
```

`main.py`에는 라우터가 다음과 같이 등록되어 있다.

```python
from mysite3.post_api import router as response_router

app.include_router(response_router)
```

`main.py`가 있는 프로젝트 폴더에서 서버를 실행하고, Swagger UI(`http://127.0.0.1:8000/docs`)의 `mysite3` 그룹에서 요청을 실행한다.

```bash
uv run fastapi dev main.py
```

## 주제 2: 경로 매개변수와 쿼리 파라미터

### 경로 매개변수

URL 경로의 일부로 전달하는 값이다. `/posts-response/{id}`의 `id`는 조회·수정·삭제할 게시글을 **식별**한다. 함수 매개변수를 `id: int`로 선언하면 FastAPI가 경로의 값을 정수로 변환하고 검증한다.

### 쿼리 파라미터

URL의 `?` 뒤에 `key=value` 형태로 전달하는 값이다. **검색·필터링·정렬 조건**을 전달할 때 사용한다.

`/posts-response/search?keyword=FastAPI`에서 `keyword`는 매개변수 이름이고 `FastAPI`는 검색할 값이다.

### 경로·쿼리 파라미터의 자동 구분

FastAPI는 **경로 선언과 함수 매개변수의 이름·타입**을 보고 값을 받을 위치를 자동으로 구분한다.

- 경로의 `{변수명}`과 **이름이 같은** 매개변수는 경로에서 받는다. `/{id}`에 대응하는 `id: int`가 여기에 해당한다.
- 경로에 포함되지 않은 `int`, `str`, `bool` 같은 **단순 타입** 매개변수는 쿼리 파라미터로 받는다. `/search`에 없는 `keyword`가 여기에 해당한다.

경로·쿼리의 구분은 **매개변수의 선언 순서와 관계없다.** 타입 힌트는 전달된 값을 변환하고 검증하는 데 사용한다.

## 주제 3: 게시글 검색

`mysite3/post_api.py`의 검색 함수는 단일 게시글 조회 함수보다 **위에** 정의되어 있다. FastAPI는 **등록 순서대로** 경로를 확인하므로 `/search`가 `/{id}`보다 먼저 등록되어야 한다.

```python
# mysite3/post_api.py

@router.get("/search", response_model=list[PostListResponse])
def search_posts(keyword: str | None = None):
    if keyword:
        return [post for post in posts if keyword in post.title]
    return posts
```

- `keyword`는 경로에 포함되지 않은 단순 타입 매개변수이므로 **쿼리 파라미터**로 받는다.
- `str | None`은 문자열과 `None`을 허용하고, `= None`은 값을 생략했을 때 사용할 기본값이다.
- `keyword`를 생략하거나 빈 문자열로 보내면 **전체 게시글**을 반환한다.
- `GET /posts-response/search?keyword=FastAPI`로 요청하면 제목에 `FastAPI`가 포함된 게시글만 반환한다.

## 주제 4: Query()로 입력 위치 명시

`Query()`로 쿼리 파라미터임을 명시할 수도 있다. 다음은 같은 검색 조건을 `Query()`로 표현한 예제이다.

```python
from fastapi import Query

@router.get("/search", response_model=list[PostListResponse])
def search_posts(keyword: str | None = Query(default=None)):
    if keyword:
        return [post for post in posts if keyword in post.title]
    return posts
```

- `Query()`는 서버가 값을 받을 위치와 설정을 **선언하는 도구**이다. 클라이언트는 동일하게 `?keyword=FastAPI`로 값을 전달한다.
- `default=None`은 검색어를 생략할 수 있다는 뜻이며, 검증 조건이나 설명도 설정할 수 있다.
- 경로 매개변수는 `Path()`, 요청 본문은 `Body()`로 명시할 수 있다.

## 주제 5: 데이터 전달 방식 세 가지 비교

| 전달 방식 | 요청에서의 위치 | 함수 매개변수 | 역할 |
|---|---|---|---|
| Path | `/posts-response/1` | `id: int` | 특정 게시글 식별 |
| Query | `/posts-response/search?keyword=FastAPI` | `keyword: str \| None = None` | 검색 조건 전달 |
| Body | `{"title": "제목", "content": "내용"}` | `post_data: PostCreate` | 생성·수정할 데이터 전달 |

`PostCreate` 같은 **Pydantic 모델 매개변수는 별도의 입력 위치 지정 없이 선언하면 요청 본문으로 받는다.**

## 헷갈리기 쉬운 부분

> ➕ **더 알아두기**
> (ChatGPT 정리본과 함께 학습한 내용이며 수업에 나온 설명은 아니다.)
>
> **1. Path·Query·Body가 정해지는 기준**
>
> 매개변수를 나열한 순서가 아니다. (1) 경로 템플릿 `{...}`에 같은 이름이 있는가, (2) 타입이 단순 타입인가 Pydantic 모델인가, (3) `Query()`·`Path()`·`Body()`로 명시했는가로 정해진다.
>
> **2. 라우팅 순서를 잘못 쓰면**
>
> `/{id}`를 먼저 등록하면 `GET /search` 요청이 동적 경로 `/{id}`에 먼저 걸린다. 이때 `id: int` 검증에서 `"search"`를 정수로 바꾸지 못해 **422**가 발생할 수 있고, FastAPI는 나중에 등록된 `/search`로 다시 찾아가지 않는다. 충돌할 수 있는 경로에서는 **정적 경로(`/search`)를 앞에** 둔다. 참고로 `/search/3`은 경로 조각이 두 개라서 `/search`나 `/{id}`와는 별개의 경로이다.
>
> **3. 검색 코드의 세부 동작**
>
> - `keyword in post.title`은 **대소문자를 구분하는 부분 문자열 포함 여부** 검사이다.
> - `if keyword:`에서 `None`과 빈 문자열 `""`은 둘 다 거짓이라서 전체 목록이 반환된다. 일치하는 게시글이 없는 검색어는 빈 리스트가 반환된다.
>
> **4. `Query()`로 조건 걸기**
>
> `Query()`에는 검증 조건도 줄 수 있다. 예를 들어 `keyword: str = Query(min_length=3)`은 기본값이 없으니 **필수**이고, 길이가 3 미만이면 422이다. `/keyword?keyword=AI`는 422, `?keyword=Python`은 통과한다. 경로 매개변수에는 `id: int = Path(gt=0)`처럼 "0보다 큰 정수" 조건을 줄 수 있다. 필수 `str`이라고 해서 빈 문자열이 자동으로 금지되지는 않는다.

## 정리

- **Path**: 경로의 일부(`/posts-response/1`). 특정 자원을 식별한다. 경로의 `{변수명}`과 매개변수 이름이 같으면 자동으로 경로에서 받는다.
- **Query**: `?` 뒤의 `key=value`. 검색·필터·정렬 조건이다. 경로에 없는 단순 타입 매개변수는 자동으로 쿼리로 받는다.
- **Body**: HTTP 요청 본문의 JSON. Pydantic 모델 매개변수로 선언하면 본문으로 받는다.
- 위치는 선언 순서가 아니라 이름·타입·명시적 선언으로 정해진다.
- `keyword: str | None = None`처럼 기본값 `None`을 두면 검색어를 생략할 수 있고, 생략·빈 문자열이면 전체 목록을 반환한다.
- 고정 경로 `/search`는 `/{id}`보다 **먼저** 등록한다.
- `Query()`·`Path()`·`Body()`는 입력 위치와 조건을 명시하는 도구이다.

## ✅ 확인 질문

1. 경로 매개변수와 쿼리 파라미터는 각각 URL의 어느 부분에 들어가고, 어떤 용도로 쓰나요?
2. FastAPI는 함수 매개변수가 Path인지 Query인지 어떻게 구분하나요? 매개변수를 적는 순서가 영향을 주나요?
3. `search_posts(keyword: str | None = None)`에서 `str | None`과 `= None`은 각각 무슨 뜻인가요? `keyword`를 생략했을 때와 빈 문자열일 때 결과는 어떻게 되나요?
4. `/search`를 `/{id}`보다 먼저 등록해야 하는 이유를 `GET /posts-response/search` 요청이 처리되는 과정으로 설명해 보세요.
5. `Query(default=None)`으로 바꿔 써도 클라이언트가 보내는 요청은 달라지지 않습니다. 그럼 `Query()`를 쓰면 무엇을 더 할 수 있나요?
6. Path, Query, Body 세 가지를 요청에서의 위치, 함수 매개변수, 역할로 비교해 보세요. Pydantic 모델 매개변수는 왜 별도 지정 없이도 Body로 받나요?
