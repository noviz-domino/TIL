---
tags: [fastapi, layered-architecture, repository, service, crud]
til: v2 2026-10-26
---

# 2026-10-26 (2)레이어드 게시판 구현(mysite4, schemas, Repository, Service, Router, 요청 흐름)
> 작성일: 2026-10-26

## 🔗 관련 글

- [관심사 분리와 레이어드 아키텍처(SoC, Router·Service·Repository, 계층별 역할)](2026-10-26%20(1)관심사_분리와_레이어드_아키텍처(SoC,_Router_Service_Repository,_계층별_역할).md) — 같은 레슨의 개념 부분이다. 이 글은 그 개념을 `mysite4` 코드로 구현한 부분이다.
- [Validation(빈 문자열 검증, 422, 검증 순서, Field)](2026-10-25%20(3)Validation(빈_문자열_검증,_422,_검증_순서,_Field).md) — `mysite3`의 `create_post` 한 함수 안에 있던 검증을 이 글에서 Service로 옮긴다.
- [간단한 게시판 CRUD(RESTful 경로, APIRouter, Post 클래스, 메모리 리스트)](2026-10-23%20(3)간단한_게시판_CRUD(RESTful_경로,_APIRouter,_Post_클래스,_메모리_리스트).md) — 처음 만든 리스트 기반 CRUD이다. 같은 CRUD가 이 글에서는 세 계층으로 나뉜다.
- [DML(INSERT, SELECT, WHERE, UPDATE, DELETE)](2026-10-02%20(5)DML(INSERT,_SELECT,_WHERE,_UPDATE,_DELETE).md) — Repository의 `save`·`find_*`·`update`·`delete`가 나중에 DB에서는 이 SQL이 된다.

## ❓ 아직 모르겠는 것

이 글은 강의 레슨(관심사 분리 및 레이어드 아키텍쳐)의 코드 부분을 읽고 정리했고, 대화 중 사용자가 직접 한 질문은 없다. 아래는 사용자의 발언이 아니라, 함께 학습한 ChatGPT 정리본이 별도 항목으로 짚어 둔 헷갈리기 쉬운 지점이다. 복습 때 다시 확인한다.

- `update_post`·`delete_post`에서 `self.get_post(id)`의 반환값을 쓰지 않는데 왜 호출하는가? (주제 5, 아래 ➕ 참고)
- `post_repository = PostRepository()`는 요청마다 새로 만들어지는가? (주제 2, 아래 ➕ 참고)
- Service가 `Post`를 만들 때 `id`가 `None`인 이유는 무엇인가? (주제 3)

## 주제 1: 파일 구조

`fastapi_basic/mysite4`의 하위 폴더는 각 계층의 역할에 따라 구분되어 있다.

```text
mysite4/
├── repositories/
│   └── post_repository.py
├── routers/
│   └── post_router.py
├── schemas/
│   └── post.py
└── services/
    └── post_service.py
```

| 파일 | 내용 |
|---|---|
| `schemas/post.py` | 게시글 객체와 요청·응답 모델 |
| `routers/post_router.py` | HTTP 요청을 처리하는 라우터 |
| `services/post_service.py` | 검증과 작업 흐름을 담당하는 `PostService` 클래스와 객체 |
| `repositories/post_repository.py` | 저장소의 CRUD를 담당하는 클래스와 객체 |
| `main.py` | 게시판 라우터를 앱에 연결하는 코드 |

- `Service` 메서드는 `PostService` 클래스에, `Repository` 메서드는 `PostRepository` 클래스에 정의되어 있다. 각 클래스 정의 아래에서 `post_service`와 `post_repository` 객체를 생성한다. Router 함수는 `post_router.py`의 모듈 수준에 정의되어 있다.
- Repository 객체는 `self.posts`와 `self.post_id`를 보관하며 **여러 요청에서 사용**된다. 서버를 다시 시작하면 메모리에 저장한 게시글이 초기화된다.

```python
# routers/post_router.py
from fastapi import APIRouter

router = APIRouter(prefix="/posts-layered", tags=["mysite4"])
```

```python
# main.py
from mysite4.routers.post_router import router as layered_router

app.include_router(layered_router)
```

```python
# services/post_service.py
post_service = PostService()
```

```python
# repositories/post_repository.py
class PostRepository:
    def __init__(self):
        self.posts = []
        self.post_id = 0

post_repository = PostRepository()
```

## 주제 2: schemas — 요청·응답 모델과 Post 객체

```python
# schemas/post.py
from pydantic import BaseModel

class Post:
    def __init__(self, title, content):
        self.id = None
        self.title = title
        self.content = content

class PostCreate(BaseModel):
    title: str
    content: str

class PostDetailResponse(BaseModel):
    id: int
    title: str
    content: str

class PostListResponse(BaseModel):
    id: int
    title: str
```

- `PostCreate`는 **요청 DTO**이고, `PostDetailResponse`와 `PostListResponse`는 **응답 DTO**이다. `Post`는 저장소에서 관리할 게시글 객체이다.
- Service가 `Post`를 만들 때는 아직 ID가 없으므로 `id=None`으로 시작한다. Repository가 저장할 때 ID를 부여하므로, 기존 `Post(id, title, content)`와 달리 **생성자에는 제목과 내용만** 전달한다.

## 주제 3: Create — 게시글 생성

### Repository: ID를 부여하고 저장

Service가 만든 게시글에 ID를 부여하고 리스트에 저장한다.

```python
# repositories/post_repository.py
from mysite4.schemas.post import Post, PostCreate

class PostRepository:
    def __init__(self):
        self.posts = []
        self.post_id = 0

    def save(self, new_post: Post):
        self.post_id += 1
        new_post.id = self.post_id
        self.posts.append(new_post)
        return new_post
```

### Service: 검증 후 객체 생성과 저장 요청

제목과 내용을 검사한 뒤 게시글 객체를 만들고 저장을 요청한다.

```python
# services/post_service.py
from fastapi import HTTPException, status
from mysite4.repositories.post_repository import post_repository
from mysite4.schemas.post import Post, PostCreate

class PostService:
    def create_post(self, data: PostCreate):
        if data.title == "":
            raise HTTPException(
                status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
                detail="title을 입력하세요",
            )

        if data.content == "":
            raise HTTPException(
                status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
                detail="content를 입력하세요",
            )

        new_post = Post(title=data.title, content=data.content)

        return post_repository.save(new_post)
```

### Router: HTTP 요청 담당, Service 호출

```python
# routers/post_router.py
from fastapi import status
from mysite4.services.post_service import post_service
from mysite4.schemas.post import PostCreate, PostDetailResponse

@router.post("", response_model=PostDetailResponse, status_code=status.HTTP_201_CREATED)
def create_post(data: PostCreate):
    return post_service.create_post(data)
```

## 주제 4: Read — 전체 조회와 단일 조회

### 전체 게시글

```python
# repositories/post_repository.py — 게시글 조회에 대한 로직
def find_all(self):
    return self.posts
```

```python
# services/post_service.py — 추가 규칙 없이 Repository에서 받은 목록을 반환
def get_posts(self):
    return post_repository.find_all()
```

```python
# routers/post_router.py
from mysite4.schemas.post import PostCreate, PostDetailResponse, PostListResponse

@router.get("", response_model=list[PostListResponse])
def get_posts():
    return post_service.get_posts()
```

### 단일 게시글

```python
# repositories/post_repository.py
def find_by_id(self, id: int):
    for post in self.posts:
        if post.id == id:
            return post
    return None
```

```python
# services/post_service.py — Repository 호출, 없으면 404
def get_post(self, id: int):
    post = post_repository.find_by_id(id)
    if post is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="게시글을 찾을 수 없습니다.",
        )
    return post
```

```python
# routers/post_router.py
@router.get("/{id}", response_model=PostDetailResponse)
def get_post(id: int):
    return post_service.get_post(id)
```

## 주제 5: Update — 게시글 수정

```python
# repositories/post_repository.py — 게시글 수정에 대한 로직
def update(self, id: int, data: PostCreate):
    for post in self.posts:
        if post.id == id:
            post.title = data.title
            post.content = data.content
            return post
    return None
```

제목·내용을 검사하고 대상이 존재하는지 확인한 뒤 Repository에 수정을 요청한다.

```python
# services/post_service.py
def update_post(self, id: int, data: PostCreate):
    if data.title == "":
        raise HTTPException(
            status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
            detail="title을 입력하세요",
        )

    if data.content == "":
        raise HTTPException(
            status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
            detail="content를 입력하세요",
        )

    self.get_post(id)
    return post_repository.update(id, data)
```

```python
# routers/post_router.py
@router.put("/{id}", response_model=PostDetailResponse)
def update_post(id: int, data: PostCreate):
    return post_service.update_post(id, data)
```

## 주제 6: Delete — 게시글 삭제

```python
# repositories/post_repository.py — 게시글 삭제에 대한 로직
def delete(self, id: int):
    for index, post in enumerate(self.posts):
        if post.id == id:
            return self.posts.pop(index)
```

대상이 없으면 예외를 발생시키고, 존재하면 삭제를 요청한다.

```python
# services/post_service.py
def delete_post(self, id: int):
    self.get_post(id)
    post_repository.delete(id)
```

```python
# routers/post_router.py
@router.delete("/{id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_post(id: int):
    post_service.delete_post(id)
```

## 주제 7: 실행 및 응답 확인

`main.py`가 있는 `fastapi_basic` 폴더에서 서버를 실행한다.

```bash
uv run fastapi dev main.py
```

Swagger UI(`http://127.0.0.1:8000/docs`)의 `mysite4` 그룹에서 요청을 실행한다. 생성 응답에서 받은 `id`를 조회·수정·삭제에 사용한다.

| 요청 | 상태 코드 | 확인할 내용 |
|---|---|---|
| 정상 입력으로 생성 | 201 | `id`, `title`, `content` 반환 |
| 전체 조회 | 200 | 각 게시글의 `id`, `title`만 반환 |
| 단일 조회·정상 수정 | 200 | `id`, `title`, `content` 반환 |
| 빈 제목이나 내용으로 생성·수정 | 422 | 검증 메시지 반환, 데이터 변경 없음 |
| 없는 ID 조회·삭제 또는 정상 입력으로 수정 | 404 | "게시글을 찾을 수 없습니다." 반환 |
| 삭제 성공 | 204 | 응답 본문 없음 |

## 헷갈리기 쉬운 부분

> ➕ **더 알아두기**
> (ChatGPT 정리본과 함께 학습한 내용이며 수업에 나온 설명은 아니다.)
>
> **1. 메서드별 계층의 책임**
>
> | API | Router | Service | Repository | 성공 상태 |
> |---|---|---|---|---|
> | POST `/posts-layered` | `create_post` | 빈 값 검증 후 `Post` 생성 | `save`로 ID 부여·저장 | 201 |
> | GET `/posts-layered` | `get_posts` | 목록 요청 | `find_all` | 200 |
> | GET `/posts-layered/{id}` | `get_post` | 없으면 404 | `find_by_id` | 200 |
> | PUT `/posts-layered/{id}` | `update_post` | 입력 검증·존재 확인 | `update` | 200 |
> | DELETE `/posts-layered/{id}` | `delete_post` | 존재 확인 | `delete` | 204 |
>
> **2. POST 한 번이 처리되는 순서**
>
> ```text
> 1. FastAPI가 POST /posts-layered 라우트를 찾는다.
> 2. Pydantic이 JSON을 PostCreate 인스턴스로 검증한다.
> 3. Router의 create_post(data)가 실행된다.
> 4. Router가 post_service.create_post(data)를 호출한다.
> 5. Service가 빈 문자열을 검사하고 Post(title, content)를 생성한다. (이때 id는 None)
> 6. Service가 post_repository.save(new_post)를 호출한다.
> 7. Repository가 self.post_id를 1 증가시키고 new_post.id를 할당한다.
> 8. Repository가 self.posts에 객체를 저장하고 반환한다.
> 9. Service → Router 순서로 같은 객체가 반환된다.
> 10. FastAPI가 PostDetailResponse 기준으로 응답을 직렬화한다.
> 11. HTTP 201과 {"id":1,"title":"...","content":"..."}를 보낸다.
> ```
>
> 목록과 상세의 응답이 다른 것은 같은 `Post` 객체를 `PostListResponse`와 `PostDetailResponse`로 다르게 걸러 내기 때문이다.
>
> **3. `self.get_post(id)`의 반환값을 쓰지 않아도 의미가 있는 이유**
>
> `update_post`와 `delete_post`에서는 게시글이 **있는지 확인**하려고 호출한다. 못 찾으면 안에서 `HTTPException(404)`이 발생해 뒤 코드가 실행되지 않는다. 반환된 게시글을 쓰지 않아도 **예외를 발생시키는 효과**가 있다.
>
> **4. 검증과 존재 확인의 순서**
>
> 수정 코드는 제목·내용 검증을 먼저 하고 존재 확인을 한다. 그래서 없는 `id`에 빈 제목이 들어오면 404가 아니라 422가 먼저 나온다.
>
> **5. 객체는 요청마다 새로 만들어지지 않는다**
>
> `PostRepository`는 클래스(설계도), `PostRepository()`는 인스턴스 생성, `post_repository`는 그 인스턴스를 가리키는 모듈 변수이다. `self.posts`와 `self.post_id`는 그 인스턴스의 상태이다. 모듈을 불러올 때 한 번 만든 인스턴스를 같은 프로세스에서 계속 쓰므로 **요청마다 새 저장소가 생기지 않는다.** 하지만 서버를 재시작하면 데이터가 사라지고, 서버 프로세스를 여러 개 띄우면 프로세스마다 별도의 메모리를 갖는다. 동시에 요청이 몰리면 ID 발급이 꼬일 수 있어서 실서비스에서는 DB의 기본키와 트랜잭션을 쓴다.
>
> **6. 코드의 한계**
>
> - `find_all()`은 원본 리스트의 참조를 돌려주므로, 받은 쪽이 리스트를 직접 바꿀 수 있다. 학습용이라 단순하게 둔 것이다.
> - `update_post`와 `delete_post`는 먼저 존재를 확인한 뒤 변경하는데, 실제 서비스에서는 그 사이에 다른 요청이 끼어들 수 있어서 동시성 제어가 필요할 수 있다.
> - Router의 `delete_post`는 값을 반환하지 않아 `None`이 되지만, `status_code=204`를 지정했기 때문에 본문 없이 응답한다.
>
> **7. `__init__.py`**
>
> 폴더를 패키지로 표시하는 편의 파일이다. 최신 Python에서는 없어도 동작하는 경우가 있지만, 학습 프로젝트에서는 두는 편이 이해하기 쉽다.

## 정리

- `mysite4`는 `schemas`(데이터 구조), `repositories`(저장), `services`(규칙과 흐름), `routers`(HTTP) 폴더로 나뉘고, `main.py`에서 `include_router(layered_router)`로 연결한다.
- `Post`는 `id=None`으로 시작하고, Repository의 `save()`가 ID를 부여한다. 그래서 생성자에는 제목과 내용만 전달한다.
- Service는 빈 값 검증과 존재 확인(없으면 404)을 하고, Repository는 저장·조회·수정·삭제만 한다. Router는 `response_model`과 성공 상태 코드를 지정하고 Service를 호출한다.
- `get_post`는 `find_by_id()`가 `None`이면 404를 던진다. `update_post`와 `delete_post`는 `self.get_post(id)`로 존재를 먼저 확인한다.
- `post_service`와 `post_repository` 객체는 서버가 켜져 있는 동안 여러 요청에서 계속 쓰이고, 서버를 재시작하면 메모리의 데이터가 초기화된다.
- 응답: 생성 201, 목록은 `id`·`title`만, 단건·수정 200, 빈 값 422, 없는 ID 404, 삭제 204.

## ✅ 확인 질문

1. `mysite4`의 폴더 네 개(`schemas`, `repositories`, `services`, `routers`)에는 각각 무엇이 들어 있나요? `post_service`와 `post_repository` 객체는 어디에서 만들어지나요?
2. `Post`의 생성자가 `Post(title, content)`로 바뀌고 `self.id = None`으로 시작하는 이유는 무엇인가요? ID는 어느 계층에서 부여하나요?
3. `PostCreate`, `PostDetailResponse`, `PostListResponse`, `Post`는 각각 어떤 역할인가요?
4. 생성 요청에서 Router, Service, Repository가 각각 하는 일을 순서대로 설명해 보세요.
5. 전체 조회에서 Service가 Repository의 결과를 그대로 돌려주는 이유는 무엇인가요? 목록 응답에 `content`가 없는 것은 어느 계층에서 정해지나요?
6. 단일 조회에서 "게시글이 없다"는 사실은 어느 계층이 판단하고, 어떻게 404가 되나요?
7. `update_post`에서 `self.get_post(id)`를 호출하고 반환값을 쓰지 않는 이유는 무엇인가요? 없는 `id`에 빈 제목을 보내면 404와 422 중 무엇이 나오나요?
8. `delete_post`가 성공하면 왜 본문 없이 204가 나오나요? 같은 `id`로 한 번 더 삭제하면 어떻게 되나요?
9. `post_repository`는 요청마다 새로 만들어지나요? 서버를 재시작하거나 서버 프로세스를 여러 개 띄우면 데이터는 어떻게 되나요?
10. 리스트 저장소를 DB로 바꿀 때 어느 계층을 주로 고치고, 무엇을 유지해야 다른 계층의 변경을 줄일 수 있나요?
