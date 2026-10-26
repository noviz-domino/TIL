---
tags: [fastapi, restful, crud, apirouter, path-parameter]
til: v2 2026-10-23
---

# 2026-10-23 (3)간단한 게시판 CRUD(RESTful 경로, APIRouter, Post 클래스, 메모리 리스트)
> 작성일: 2026-10-23

## 🔗 관련 글

- [FastAPI 시작(uv, 엔드포인트, Swagger UI, 자동 문서화)](2026-10-23%20(2)FastAPI_시작(uv,_엔드포인트,_Swagger_UI,_자동_문서화).md) — 바로 앞 수업이다. 엔드포인트와 `@app.get`을 한 파일에서 만들었고, 이 글은 그것을 기능별 파일로 나누고 CRUD 5개를 만든다.
- [DML(INSERT, SELECT, WHERE, UPDATE, DELETE)](2026-10-02%20(5)DML(INSERT,_SELECT,_WHERE,_UPDATE,_DELETE).md) — DB에서 데이터를 넣고 읽고 고치고 지우는 SQL을 배웠다. 이 글의 Create·Read·Update·Delete가 같은 네 가지 작업이다. 이번에는 DB 대신 메모리 리스트에 한다.
- [파이썬 기초문법 학습3](2026-08-13%20파이썬%20기초문법%20학습3.md) — 가변(mutable)과 불변(immutable) 객체를 다뤘다. 이 글 주제 5의 `global`이 필요한 경우와 필요 없는 경우가 이 개념과 이어진다.
- [서버·클라이언트 설명](2026-07-30_서버_클라이언트_설명.md) — HTTP와 API의 기본 개념이다.

## ❓ 아직 모르겠는 것

이 글은 강의 레슨(간단한 게시판 구현)을 읽고 정리했고, 대화 중 사용자가 직접 한 질문은 없다. 아래는 사용자의 발언이 아니라, 함께 학습한 ChatGPT 정리본이 별도 항목으로 짚어 둔 헷갈리기 쉬운 지점이다. 복습 때 다시 확인한다.

- 왜 `post_id += 1`에는 `global`이 필요하고 `posts.append()`에는 필요 없는가? (주제 5, 아래 ➕ 참고)
- 게시글의 `id`와 리스트의 인덱스는 어떻게 다른가? `posts.pop(id)`로 쓰면 왜 안 되는가? (주제 9, 아래 ➕ 참고)
- `for post in posts`에서 `post.title`을 바꾸면 왜 리스트 안의 원본이 바뀌는가? (주제 8, 아래 ➕ 참고)

## 주제 1: RESTful API의 경로와 HTTP 메서드

게시글은 `id`, 제목, 내용으로 구성되며, API를 통해 생성·조회·수정·삭제(CRUD)할 수 있다.

RESTful API에서는 **자원을 URL로 표현하고, 자원에 수행할 행위를 HTTP 메서드로 구분**한다. 게시글 목록은 `/posts`, 특정 게시글은 `/posts/{id}`로 표현한다.

| 기능 | HTTP 메서드 | 경로 | 데코레이터 |
|---|---|---|---|
| 게시글 생성 | POST | `/posts` | `@router.post("")` |
| 전체 게시글 조회 | GET | `/posts` | `@router.get("")` |
| 단일 게시글 조회 | GET | `/posts/{id}` | `@router.get("/{id}")` |
| 게시글 수정 | PUT | `/posts/{id}` | `@router.put("/{id}")` |
| 게시글 삭제 | DELETE | `/posts/{id}` | `@router.delete("/{id}")` |

**PUT은 자원의 전체 수정에, PATCH는 일부 수정에 사용한다.** 이 예제에서는 제목과 내용을 모두 변경하므로 PUT을 사용한다.

## 주제 2: 게시판의 파일 구조와 Post 클래스

`fastapi_basic` 프로젝트의 게시판 관련 파일은 다음과 같다.

| 파일 | 역할 |
|---|---|
| `main.py` | FastAPI 앱을 생성하고 게시판 라우터를 연결한다. |
| `mysite/post.py` | 게시글의 데이터 구조를 정의한다. |
| `mysite/post_api.py` | 게시글을 저장하고 CRUD 요청을 처리한다. |

`Post` 클래스는 게시글의 식별자, 제목, 내용을 담는다.

```python
# mysite/post.py

class Post:

    def __init__(self, id, title, content):
        self.id = id
        self.title = title
        self.content = content
```

## 주제 3: 게시판 로직 구성과 Router 연결

### 라우터와 저장소 설정

- `posts` 리스트에 게시글 객체들을 저장한다.
- `post_id`는 **마지막으로 부여한 게시글의 식별 번호**를 저장한다.
- `APIRouter`는 `main.py`와 별도 파일에 정의된 게시판 엔드포인트를 묶는다.
- `prefix="/posts"`는 각 엔드포인트 경로 앞에 붙는 공통 경로이다.
- `tags=["mysite"]`는 Swagger UI에서 게시판 API를 `mysite` 그룹으로 묶는다.

```python
# mysite/post_api.py

from fastapi import APIRouter
from .post import Post

router = APIRouter(prefix="/posts", tags=["mysite"])

# 게시글 객체들을 저장하는 리스트와 초기 데이터
posts = [Post(1, "기본 제목", "기본 내용")]
post_id = 1
```

게시글은 **메모리에 저장**되므로 서버가 재시작되거나 코드 변경으로 다시 로드되면 초기 데이터로 돌아간다.

### Router 연결

`app.include_router(post_router)`는 게시판 라우터를 FastAPI 앱에 등록한다.

```python
# main.py

from fastapi import FastAPI
from mysite.post_api import router as post_router

app = FastAPI()

app.include_router(post_router)
```

### 요청 실행 방법

`main.py`가 있는 프로젝트 폴더에서 서버를 실행한다.

```bash
uv run fastapi dev main.py
```

브라우저에서 `http://127.0.0.1:8000/docs`에 접속한다. `mysite` 그룹의 해당 항목을 펼쳐 Try it out → Execute로 요청을 보낸다. 브라우저 주소창으로 URL에 접속하면 GET 요청을 보낸다. 생성·수정·삭제 요청은 Swagger UI에서 해당 HTTP 메서드의 항목을 선택하여 실행한다.

## 주제 4: Create — 게시글 작성 (POST)

`POST /posts`로 요청을 보내면 새로운 게시글 객체를 생성하고 리스트에 추가한다.

```python
# mysite/post_api.py

@router.post("")
def create_post():
    # 식별자 1 증가
    global post_id
    post_id += 1

    # 새로운 게시글 객체 생성
    post = Post(post_id, "제목", "내용")

    # 리스트에 추가
    posts.append(post)

    return post
```

이 단계에서는 요청 본문을 받지 않고 제목과 내용을 `"제목"`, `"내용"`으로 고정해 둔다. 요청으로 받는 방법은 다음 수업에서 배운다.

## 주제 5: Read — 전체 조회와 경로 매개변수

### 전체 게시글 조회

`GET /posts`로 요청을 보내면 현재 리스트에 저장된 모든 게시글 데이터를 반환한다.

```python
# mysite/post_api.py

@router.get("")
def get_posts():
    # 게시글 리스트 전체를 반환
    return posts
```

### 경로 매개변수(path parameter)

- 주소 경로에 변수를 포함하여 데이터를 전달하는 방식이다.
- 중괄호를 사용하여 변수명을 지정한다.
- `/posts/{id}`에서 `id`는 조회할 게시글의 식별자이다. `/posts/1`로 요청하면 함수의 `id` 매개변수에 `1`이 전달된다.
- 함수 매개변수를 `id: int`로 선언하면 FastAPI가 경로의 값을 **정수로 변환하고 검증**한다.

## 주제 6: Read — 단일 게시글 조회

`GET /posts/{id}`로 요청을 보내면 리스트를 순회하며 식별자가 일치하는 게시글을 반환한다.

```python
# mysite/post_api.py

@router.get("/{id}")
def get_post(id: int):
    # 리스트 내 객체들을 하나씩 검사
    for post in posts:
        # 객체의 식별자와 입력받은 식별자가 일치하는지 확인
        if post.id == id:
            return post
    return None
```

일치하는 게시글이 없으면 `None`을 반환한다. 이 시점의 코드에는 "없는 게시글" 처리가 따로 없다.

## 주제 7: Update — 게시글 수정 (PUT)

`PUT /posts/{id}`로 요청을 보내면 해당 게시글의 제목과 내용을 변경한다.

```python
# mysite/post_api.py

@router.put("/{id}")
def update_post(id: int):
    for post in posts:
        if post.id == id:
            # 객체의 속성 값 변경
            post.title = "수정된 제목"
            post.content = "수정된 내용"
            return post
    return None
```

## 주제 8: Delete — 게시글 삭제 (DELETE)

`DELETE /posts/{id}`로 요청을 보내면 해당 게시글을 리스트에서 제거하고 **삭제한 게시글을 반환**한다.

```python
# mysite/post_api.py

@router.delete("/{id}")
def delete_post(id: int):
    # 리스트를 순회하며 인덱스(i)와 내용(post)을 함께 추출
    for i, post in enumerate(posts):
        if post.id == id:
            # 해당 인덱스의 게시글을 제거하고 반환
            return posts.pop(i)

    return "삭제 실패"
```

- `enumerate(posts)`는 `(인덱스, 요소)` 쌍을 하나씩 꺼낸다.
- `posts.pop(i)`는 `i`번째 요소를 리스트에서 제거하고 그 요소를 반환한다.
- 일치하는 게시글이 없으면 문자열 `"삭제 실패"`를 반환한다.

## 주제 9: 실행 확인

서버를 재시작하여 초기 데이터로 돌아간 뒤 Swagger UI에서 아래 요청을 표의 순서대로 실행한다. 각 요청의 응답을 예상하고 실제 결과와 비교한다.

| 요청 | 확인할 내용 |
|---|---|
| `GET /posts` | 초기 게시글(id=1)이 조회된다. |
| `POST /posts` | 제목이 "제목", 내용이 "내용"인 게시글(id=2)이 생성된다. |
| `GET /posts/2` | 생성한 게시글이 조회된다. |
| `PUT /posts/2` | 제목과 내용이 "수정된 제목", "수정된 내용"으로 변경된다. |
| `GET /posts/2` | 변경된 내용이 조회된다. |
| `DELETE /posts/2` | 게시글이 삭제되고 삭제한 게시글이 응답으로 반환된다. |
| `GET /posts` | 삭제한 게시글이 목록에 나타나지 않는다. |

## 헷갈리기 쉬운 부분

> ➕ **더 알아두기**
> (ChatGPT 정리본과 함께 학습한 내용이며 수업에 나온 설명은 아니다.)
>
> **1. `global`이 `post_id`에는 필요하고 `posts`에는 필요 없는 이유**
>
> `post_id += 1`은 함수 안에서 모듈 변수 `post_id`에 **다시 대입**하는 연산이어서 `global post_id`가 필요하다. 반면 `posts.append(post)`는 `posts`라는 이름을 새 리스트에 다시 붙이지 않고 **기존 리스트 객체를 변경**할 뿐이어서 `global posts`가 필요 없다. 핵심은 값이 가변이냐뿐 아니라 **변수 이름을 다시 바인딩하느냐**이다. 함수 안에서 `posts = [new_post]`나 `posts += [new_post]`처럼 이름 자체에 대입하면 `global`이 필요하다. (정수와 문자열은 불변, 리스트와 일반 클래스 인스턴스는 가변이다.)
>
> **2. 게시글의 `id`와 리스트 인덱스는 다르다**
>
> `post.id`는 게시글의 고유 식별자이고 `index`는 현재 리스트에서의 위치이다. 2번 게시글을 지우면 뒤의 요소가 앞으로 당겨지지만 남은 게시글의 `id`는 변하지 않는다. 그래서 `id`가 1, 3만 남았을 때 `posts.pop(3)`처럼 `id`를 인덱스로 쓰면 엉뚱한 요소를 지우거나 오류가 난다. 삭제 코드가 `enumerate`로 인덱스를 따로 꺼내는 이유이다.
>
> **3. `for post in posts`에서 속성을 바꾸면 원본이 바뀌는 이유**
>
> 반복 변수 `post`는 리스트 안 게시글 객체를 **참조**한다. 그래서 `post.title = "변경"`은 저장된 객체의 상태를 바꾼다. 새 객체를 만들어 지역 변수 `post`에 대입하는 것과는 다르다.
>
> **4. 탐색 비용**
>
> 리스트에서 `id`를 찾는 순차 검색은 O(N)이고, 리스트 중간을 삭제할 때 뒤 요소를 당기는 것도 O(N)이다. `id`를 키로 쓰는 딕셔너리는 평균적으로 조회·삭제가 O(1)이지만 순서·동시성·영속성도 따져야 한다. DB에서는 인덱스 설계에 따라 조회 성능이 달라진다.
>
> **5. `@router.post("")`와 `@router.post("/")`는 다르다**
>
> `prefix="/posts"`에 `""`를 붙이면 `/posts`, `"/"`를 붙이면 `/posts/`가 된다. 슬래시 유무가 다른 주소로 요청하면 기본 설정에서는 FastAPI(Starlette)가 올바른 주소로 307 리다이렉트를 보낸다. 이 수업은 `""`를 쓴다.
>
> **6. 이 실습 코드의 한계**
>
> 게시글은 서버 메모리에만 있어서 서버를 다시 켜면 사라진다. 없는 게시글을 조회하거나 수정하면 `null`과 상태 코드 200이 나오고, 없는 게시글을 삭제하면 `"삭제 실패"` 문자열과 상태 코드 200이 나온다. 이 부분은 이후 수업에서 `404`와 `HTTPException`으로 고친다.

## 정리

- RESTful API는 **자원은 URL**(`/posts`, `/posts/{id}`), **행위는 HTTP 메서드**(POST·GET·PUT·DELETE)로 표현한다. PUT은 전체 수정, PATCH는 일부 수정이다.
- 게시판 코드는 데이터 구조(`post.py`)와 API(`post_api.py`)로 나누고, `APIRouter(prefix="/posts", tags=["mysite"])`로 묶어 `main.py`에서 `include_router`로 등록한다.
- 경로 매개변수는 `{id}`로 선언하고, `id: int`로 쓰면 FastAPI가 정수로 변환·검증한다.
- Create는 `post_id`를 증가시키려면 `global`이 필요하다. 목록 `append`는 이름을 바꾸지 않아서 필요 없다.
- Read는 리스트를 순회하며 `post.id`와 비교한다. Update는 객체의 속성을 바꾼다. Delete는 `enumerate`로 인덱스를 얻어 `pop(i)`한다.
- 데이터가 메모리에 있어서 서버를 다시 로드하면 초기 데이터로 돌아간다.

## ✅ 확인 질문

1. RESTful API에서 자원과 행위는 각각 무엇으로 표현하나요? 게시글 CRUD 5개 기능의 HTTP 메서드와 경로를 표로 말해 보세요.
2. PUT과 PATCH는 어떻게 다른가요? 이 예제에서 PUT을 쓴 이유는 무엇인가요?
3. `APIRouter(prefix="/posts", tags=["mysite"])`에서 `prefix`와 `tags`는 각각 무엇을 하나요? `@router.get("")`로 등록한 함수는 어떤 경로가 되나요?
4. 라우터를 만들었는데 `app.include_router(post_router)`를 빼면 어떻게 되나요?
5. `post_id`는 무엇을 저장하나요? 서버를 재시작하면 `posts`와 `post_id`는 어떻게 되나요? 그 이유는 무엇인가요?
6. `create_post`에서 `post_id += 1`에는 `global`이 필요한데 `posts.append(post)`에는 필요 없는 이유를 설명해 보세요.
7. `/posts/{id}`에서 `{id}`와 함수 매개변수 `id: int`는 어떻게 연결되나요? `int`라고 적으면 무엇이 달라지나요?
8. `get_post`에서 일치하는 게시글이 없으면 어떤 값이 반환되나요? 이 방식의 문제점은 무엇일까요?
9. `delete_post`에서 `enumerate`를 쓰는 이유는 무엇인가요? 게시글 `id`와 리스트 인덱스가 다르다는 것을 예로 설명해 보세요.
10. 실행 확인 표의 순서(GET → POST → GET → PUT → GET → DELETE → GET)대로 요청했을 때 각 단계의 결과를 예상해 보세요.
