---
tags: [fastapi, pydantic, dto, request-body, validation]
til: v2 2026-10-24
---

# 2026-10-24 (2)Pydantic 요청 DTO(BaseModel, Request Body, 422 검증, PostCreate와 Post)
> 작성일: 2026-10-24

## 🔗 관련 글

- [간단한 게시판 CRUD(RESTful 경로, APIRouter, Post 클래스, 메모리 리스트)](2026-10-23%20(3)간단한_게시판_CRUD(RESTful_경로,_APIRouter,_Post_클래스,_메모리_리스트).md) — 전 수업이다. 그때는 생성·수정의 제목과 내용을 `"제목"`, `"내용"`으로 고정했고, 이 글에서 클라이언트가 보낸 값을 받는다.
- [대화 메모리와 Structured Output](2026-09-01%20(3)%20대화%20메모리와%20Structured%20Output.md) — LLM 응답을 Pydantic 모델로 구조화하는 방법을 다뤘다. 같은 `BaseModel`로 이번에는 요청 데이터를 검증한다.

## ❓ 아직 모르겠는 것

이 글은 강의 레슨(상호작용이 가능한 게시판)을 읽고 정리했다. 아래는 사용자의 실습 코드에서 확인한 지점이다. 복습 때 다시 확인한다.

- 실습 코드(`aim-ai-agent/fastapi/fastapi_basic/mysite2/post.py`)에 `PostUpdate` 모델과 주석 "나는 제목 말고 content만 받아서 내용만 수정을 하게 하고 싶어."가 있고, 수정 함수에서 `post.title = ...` 줄은 주석 처리되어 있다. 제목은 두고 내용만 바꾸는 요청은 PUT과 PATCH 중 무엇이 맞는가? (주제 7, 아래 ➕ 참고)
- 같은 코드에서 `PostCreate`와 `PostUpdate`의 `content`가 `str`이 아니라 `int`로 적혀 있다. 강의 모델은 `content: str`이다. 숫자 입력이 통과하는지 보려던 실험이면 의도한 것이고, 아니라면 `str`로 되돌려야 한다. **어느 쪽인지 확인이 필요하다.**
- `str` 타입만 선언하면 빈 문자열 `""`도 통과하는가? (주제 3, 이 문제는 [Validation](2026-10-25%20(3)Validation(빈_문자열_검증,_422,_검증_순서,_Field).md)에서 다룬다.)

## 주제 1: Pydantic이란

**Pydantic**은 파이썬의 타입 힌트로 데이터의 구조를 정의하고 입력값을 검증하는 라이브러리이다. FastAPI에서는 클라이언트가 보내는 **요청 데이터의 형식을 정의**하는 데 사용한다.

- `BaseModel`을 상속하고 필드 이름과 타입을 선언한다.
- 기본값이 없는 필드는 **필수 입력값**이다. 기본값이 있으면 생략할 수 있다.
- 입력값은 모델의 규칙에 따라 타입 변환과 검증을 거친다. 필수 필드가 빠지거나 허용되지 않는 값이 들어오면 검증에 실패한다.

## 주제 2: HTTP Request Body

요청 본문(Request Body)은 클라이언트가 서버에 보낼 데이터를 담는 영역이다. 게시글을 생성하거나 수정할 때 제목과 내용을 JSON으로 담아 보낸다. FastAPI는 이 데이터를 **Pydantic 모델로 받아 검증**한다.

## 주제 3: 요청 데이터 모델 — PostCreate와 Post

`mysite2/post.py`는 데이터 모델을, `mysite2/post_api.py`는 게시판 API를 담는다. `PostCreate`는 클라이언트가 보낼 제목과 내용의 형식을 정의한다. 두 필드는 기본값이 없는 `str` 타입이므로 모두 문자열로 전달해야 한다.

```python
# mysite2/post.py
from pydantic import BaseModel

class PostCreate(BaseModel):
    title: str
    content: str

class Post:
    def __init__(self, id, title, content):
        self.id = id
        self.title = title
        self.content = content
```

`PostCreate`는 `title`과 `content`의 **타입과 필수 여부**를 검증한다. `str` 타입 선언만으로는 문자열의 길이를 제한하지 않으므로 **빈 문자열도 허용**한다.

### 요청 DTO와 게시글 객체

**DTO(Data Transfer Object)** 는 데이터를 주고받을 때 전달할 값을 묶어 담는 객체이다. Pydantic 모델인 `PostCreate`가 요청 DTO 역할을 한다.

| 클래스 | 포함하는 데이터 | 역할 |
|---|---|---|
| `PostCreate` | `title`, `content` | 클라이언트가 보낸 요청 데이터를 담고, Pydantic으로 필수 값과 타입을 검증한다. |
| `Post` | `id`, `title`, `content` | 서버에서 관리하는 게시글을 표현하며, `posts` 리스트에 저장된다. |

게시글을 생성할 때 클라이언트는 제목과 내용을 보낸다. 서버는 검증된 `PostCreate`의 값에 **서버가 생성한 `id`를 더해 `Post` 객체**를 만든다. 요청으로 받는 데이터와 서버에서 관리하는 데이터의 구성이 다르므로 두 클래스로 구분한다.

## 주제 4: 라우터와 저장소 설정

`mysite2/post_api.py`에는 게시글을 관리할 리스트, 식별자 카운터, 라우터가 있다. 라우터의 `prefix`는 공통 경로를 나타낸다. 이번에는 이전 실습(`/posts`)과 겹치지 않도록 `/posts-pydantic`을 쓴다.

```python
# mysite2/post_api.py
from fastapi import APIRouter
from mysite2.post import PostCreate, Post

# 경로 접두사 설정
router = APIRouter(prefix="/posts-pydantic", tags=["mysite2"])

# 게시글 저장소 역할을 수행하는 리스트
posts = []
# 고유 식별자 생성을 위한 카운터
post_id = 0
```

`main.py`의 `app.include_router(pydantic_router)`가 게시판 라우터를 앱에 연결한다.

```python
# main.py

from fastapi import FastAPI
from mysite2.post_api import router as pydantic_router

app = FastAPI()

app.include_router(pydantic_router)
```

## 주제 5: 게시글 생성과 요청 본문 처리

함수 매개변수를 `post_data: PostCreate`로 선언하면 FastAPI가 요청 본문의 JSON을 읽고 검증하여 `PostCreate` 객체로 전달한다. 함수 안에서는 `post_data.title`, `post_data.content`로 값을 꺼낸다.

```python
# mysite2/post_api.py

@router.post("")
def create_post(post_data: PostCreate):
    global post_id
    post_id += 1

    new_post = Post(post_id, post_data.title, post_data.content)
    posts.append(new_post)

    return new_post
```

요청 처리 흐름은 **JSON 요청 본문 → `PostCreate` 검증 → `create_post()` 실행 → JSON 응답**이다. 요청 데이터 검증에 실패하면 **함수가 실행되지 않고** 검증 오류 응답을 반환한다.

## 주제 6: 조회

저장소에 보관된 게시글 데이터를 조회한다. 단일 조회 시에는 경로 매개변수를 통해 특정 식별자를 전달받아 검색을 수행한다.

```python
# mysite2/post_api.py

@router.get("")
def get_posts():
    # 저장된 모든 게시글 반환
    return posts

@router.get("/{id}")
def get_post(id: int):
    # 식별자가 일치하는 데이터를 리스트에서 탐색
    for post in posts:
        if post.id == id:
            return post
    return {"message": "데이터를 찾을 수 없습니다."}
```

## 주제 7: 수정과 삭제

### 수정

`PUT /posts-pydantic/{id}`에서는 **경로로 게시글의 식별자**를 받고, **요청 본문으로 변경할 제목과 내용**을 받는다. 두 필드를 모두 받으므로 생성 요청과 같은 `PostCreate` 모델을 사용한다.

```python
# mysite2/post_api.py

@router.put("/{id}")
def update_post(id: int, updated_post: PostCreate):
    for post in posts:
        if post.id == id:
            # 전달받은 객체의 필드값으로 기존 데이터 갱신
            post.title = updated_post.title
            post.content = updated_post.content
            return post
    return {"message": "수정할 대상이 없습니다."}
```

### 삭제

`DELETE` 메서드와 경로 매개변수를 결합하여 특정 자원을 저장소에서 제거한다.

```python
# mysite2/post_api.py

@router.delete("/{id}")
def delete_post(id: int):
    for index, post in enumerate(posts):
        if post.id == id:
            # 해당 인덱스의 요소를 추출하여 제거
            return posts.pop(index)
    return {"message": "삭제할 대상이 없습니다."}
```

## 주제 8: 요청과 검증 결과 확인

`main.py`가 있는 프로젝트 폴더에서 서버를 실행하고 `http://127.0.0.1:8000/docs`에 접속한다.

```bash
uv run fastapi dev main.py
```

### 게시글 생성과 수정

`mysite2` 그룹의 `POST /posts-pydantic`을 펼쳐 Try it out을 누르고 요청 본문을 다음과 같이 입력한 뒤 Execute를 누른다.

```json
{
  "title": "첫 게시글",
  "content": "안녕하세요."
}
```

응답에서 서버가 부여한 `id`와 입력한 제목·내용을 확인한다. 이 `id`로 `GET /posts-pydantic/{id}`를 실행하면 저장된 게시글을 조회할 수 있다.

`PUT /posts-pydantic/{id}`에 같은 `id`를 입력하고 요청 본문을 바꿔 보낸다.

```json
{
  "title": "수정한 게시글",
  "content": "내용을 변경했습니다."
}
```

다시 조회하여 변경된 값을 확인한다.

### 요청 데이터 검증

`POST /posts-pydantic`에 다음 본문을 보내고 응답 내용을 비교한다.

| 요청 본문 | 예상 결과 |
|---|---|
| `{"title": "제목", "content": "내용"}` | 검증을 통과하고 게시글이 생성된다. |
| `{"title": "제목"}` | 필수 필드인 `content`가 없어 검증에 실패한다. |
| `{"title": ["제목"], "content": "내용"}` | `title`이 문자열이 아닌 리스트이므로 검증에 실패한다. |
| `{"title": "제목", "content": null}` | `content`는 `None`을 허용하지 않으므로 검증에 실패한다. |

검증에 실패한 응답의 `detail`에서 오류가 발생한 필드와 이유를 확인한다. `GET /posts-pydantic`으로 목록을 조회하여 **잘못된 요청으로는 게시글이 생성되지 않았는지**도 확인한다.

## 헷갈리기 쉬운 부분

> ➕ **더 알아두기**
> (ChatGPT 정리본과 함께 학습한 내용이며 수업에 나온 설명은 아니다.)
>
> **1. 요청 DTO에 `id`를 넣지 않는 이유**
>
> `id`는 서버가 정해야 하는 값이다. 클라이언트가 보낸 `id`를 그대로 믿으면 데이터 충돌이나 권한 검증 문제가 생길 수 있다. 그래서 `PostCreate`에는 클라이언트가 작성할 `title`, `content`만 둔다.
>
> **2. 모델에 없는 필드를 보내면?**
>
> Pydantic v2의 기본 설정에서는 모델에 선언되지 않은 추가 필드를 **무시**한다. 요청에 `"id": 500`을 더해도 `PostCreate`에 없으면 저장에 쓰이지 않는다. 예상치 못한 필드를 거부하려면 `model_config = ConfigDict(extra="forbid")`를 쓸 수 있다.
>
> **3. `BaseModel`과 일반 클래스 `Post`의 차이**
>
> 일반 클래스 `Post`는 `__init__`에 타입 힌트를 달아도 런타임 타입 검증을 하지 않는다. `BaseModel`을 상속한 클래스는 생성할 때 필드를 검증하고 변환한다. 다만 만든 뒤 `model.title = 123`처럼 속성을 다시 대입할 때는 기본적으로 검증하지 않는다. 필요하면 `validate_assignment=True`를 설정한다.
>
> **4. PUT과 PATCH — 내용만 고치고 싶을 때**
>
> PUT은 일반적으로 자원의 전체 상태를 교체하므로 이 수업처럼 `title`과 `content`를 모두 요구한다. 일부 필드만 고치려면 PATCH를 쓰고, 모든 필드를 선택 값으로 둔 모델을 쓴다.
>
> ```python
> class PostPatch(BaseModel):
>     title: str | None = None
>     content: str | None = None
>
> changes = patch_data.model_dump(exclude_unset=True)
> ```
>
> `exclude_unset=True`는 **아예 보내지 않은 필드**만 제외한다. 클라이언트가 명시적으로 `null`을 보냈다면 포함될 수 있으니 `null`을 허용할지는 따로 정해야 한다. 또 PUT은 같은 요청을 여러 번 보내도 결과가 같아야 한다는 **멱등성**을 의도한 메서드이다.
>
> **5. 이번 레슨 코드의 한계**
>
> 없는 게시글을 조회·수정·삭제하면 `{"message": ...}`를 **상태 코드 200**으로 돌려준다. 오류 상태 코드(404 등)를 쓰는 방법은 이후 수업에서 `HTTPException`으로 배운다. 또 `str`만 선언하면 빈 문자열이 통과하는 문제도 이후 Validation에서 다룬다.

## 정리

- Pydantic은 타입 힌트로 데이터 구조를 정의하고 검증하는 라이브러리이다. FastAPI에서는 `BaseModel`을 상속해 요청 본문의 형식을 정의한다.
- 기본값이 없는 필드는 필수이다. 필수 필드가 없거나 타입이 맞지 않으면 검증에 실패하고, 함수는 실행되지 않으며 검증 오류 응답(422)이 나간다.
- 요청 본문은 함수 매개변수를 `post_data: PostCreate`처럼 Pydantic 모델로 선언해서 받는다.
- `PostCreate`(요청 DTO)는 클라이언트가 보내는 `title`, `content`만 담고, `Post`는 서버가 `id`를 더해 관리하는 게시글 객체이다.
- 수정은 경로에서 `id`를, 본문에서 새 제목·내용을 받는다.
- `str` 타입 선언은 빈 문자열을 막지 못한다. 검증 실패한 요청으로는 게시글이 생성되지 않는다.

## ✅ 확인 질문

1. Pydantic은 어떤 라이브러리이고 FastAPI에서 어떤 용도로 쓰나요? `BaseModel`을 상속한 모델에서 기본값이 없는 필드는 어떻게 취급되나요?
2. HTTP 요청 본문(Request Body)은 무엇이고, 게시글 생성·수정에서 어떤 데이터를 담나요?
3. `create_post(post_data: PostCreate)`처럼 매개변수 타입을 Pydantic 모델로 선언하면 FastAPI가 하는 일을 순서대로 설명해 보세요.
4. 요청 DTO `PostCreate`와 게시글 객체 `Post`는 어떻게 다르고, 왜 클래스를 둘로 나누나요? `id`는 어느 쪽이 가지고 있나요?
5. 검증에 실패한 요청은 `create_post` 함수 안으로 들어오나요? 그때 응답은 무엇이고, 게시글은 생성되나요?
6. `{"title": "제목"}`, `{"title": ["제목"], "content": "내용"}`, `{"title": "제목", "content": null}` 세 요청이 모두 검증에 실패하는 이유를 각각 설명해 보세요.
7. `PUT /posts-pydantic/{id}`는 `id`와 변경할 제목·내용을 각각 어디로 받나요? 수정에도 `PostCreate`를 그대로 쓰는 이유는 무엇인가요?
8. 제목은 그대로 두고 내용만 수정하고 싶다면 PUT과 PATCH 중 무엇이 맞고, 요청 모델은 어떻게 바꿔야 하나요?
9. `title: str`만 선언했을 때 `""`가 통과하는 이유는 무엇인가요? 이 문제는 다음에 어떻게 해결하나요?
