---
tags: [fastapi, http-status, httpexception, response-model, dto]
til: v2 2026-10-25
---

# 2026-10-25 (2)상태 코드와 응답 모델(HTTPException, 201·204·404·422, response_model, 응답 DTO)
> 작성일: 2026-10-25

## 🔗 관련 글

- [Path·Query·Body 파라미터(경로 매개변수, 쿼리 파라미터, Query, 검색, 라우팅 순서)](2026-10-25%20(1)Path_Query_Body_파라미터(경로_매개변수,_쿼리_파라미터,_Query,_검색,_라우팅_순서).md) — 같은 레슨(쿼리 파라미터와 응답)의 앞부분이다.
- [Pydantic 요청 DTO(BaseModel, Request Body, 422 검증, PostCreate와 Post)](2026-10-24%20(2)Pydantic_요청_DTO(BaseModel,_Request_Body,_422_검증,_PostCreate와_Post).md) — 요청 DTO를 만들었다. 이 글은 반대쪽인 **응답 DTO**를 만든다. 또 그 글에서 없는 게시글에 대해 `{"message": ...}`를 200으로 돌려주던 것을 이 글에서 404로 고친다.
- [서버·클라이언트 설명](2026-07-30_서버_클라이언트_설명.md) — HTTP 요청·응답과 API의 기본을 다뤘다.

## ❓ 아직 모르겠는 것

이 글은 강의 레슨(쿼리 파라미터와 응답)의 뒷부분을 읽고 정리했고, 대화 중 사용자가 직접 한 질문은 없다. 아래는 사용자의 발언이 아니라, 함께 학습한 ChatGPT 정리본이 별도 항목으로 짚어 둔 헷갈리기 쉬운 지점이다. 복습 때 다시 확인한다.

- 오류 메시지를 `return`하면 왜 404가 되지 않는가? `raise HTTPException`과 무엇이 다른가? (주제 2, 아래 ➕ 참고)
- `response_model`로 `content`가 빠지면 저장된 게시글에서도 `content`가 지워지는가? (주제 4, 아래 ➕ 참고)
- 422와 500은 어떻게 구분하는가? (아래 ➕ 참고)

## 주제 1: HTTP 상태 코드

HTTP 상태 코드는 **요청의 처리 결과를 나타내는 숫자**이다. 응답 본문과 함께 전달되며, Swagger UI의 응답 영역에서 확인할 수 있다.

| 상태 코드 | 의미 | 게시판에서의 용도 |
|---|---|---|
| 200 | 요청 처리 성공 | 조회·수정 성공 |
| 201 | 자원 생성 성공 | 게시글 생성 성공 |
| 204 | 처리 성공, 응답 본문 없음 | 게시글 삭제 성공 |
| 404 | 자원을 찾을 수 없음 | 요청한 게시글이 없음 |
| 422 | 요청 데이터 검증 실패 | 필수 필드 누락, 허용되지 않는 타입 |

- 상태 코드를 지정하지 않은 정상 응답은 기본적으로 **200**이다.
- 요청 데이터가 `PostCreate`의 규칙에 맞지 않으면 FastAPI가 **자동으로 422**를 반환한다.

> 📝 **기사 시험 포인트** (정보보안기사)
> HTTP 응답 상태 코드는 첫 자리 숫자로 종류를 구분한다. 1xx는 정보, 2xx는 성공, 3xx는 리다이렉션, 4xx는 클라이언트 오류, 5xx는 서버 오류이다. 이 글의 200·201·204는 2xx, 404·422는 4xx에 속한다. 웹 보안에서는 응답 코드와 오류 메시지가 서버 내부 정보를 과하게 드러내지 않는지도 점검 대상이 된다.

## 주제 2: 예외 처리 — HTTPException

`{"message": "데이터를 찾을 수 없습니다."}`를 반환하는 것만으로는 **오류 상태 코드가 설정되지 않는다.** 게시글이 없다는 사실을 404로 전달하려면 `HTTPException`을 사용한다.

`mysite3/post_api.py`에서는 예외 처리에 사용할 `HTTPException`과 상태 코드 상수를 제공하는 `status`를 import한다.

```python
from fastapi import APIRouter, HTTPException, status
```

`get_post`, `update_post`, `delete_post`는 게시글을 찾지 못하면 다음 구문으로 예외를 발생시킨다.

```python
raise HTTPException(status_code=404, detail="게시글을 찾을 수 없습니다.")
```

`raise`를 실행하면 해당 요청의 처리가 **중단**된다. FastAPI가 예외를 처리하여 404와 다음 JSON 응답을 보낸다. **별도의 `try-except`는 필요하지 않다.**

```json
{"detail": "게시글을 찾을 수 없습니다."}
```

### 생성·삭제의 상태 코드 지정

`create_post`의 데코레이터에는 `status_code=status.HTTP_201_CREATED`가 지정되어 있어 생성된 게시글과 함께 201을 반환한다.

`delete_post`는 삭제에 성공하면 204를 반환하고, **응답 본문은 보내지 않는다.**

```python
# mysite3/post_api.py

@router.delete("/{id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_post(id: int):
    for index, post in enumerate(posts):
        if post.id == id:
            posts.pop(index)
            return
    raise HTTPException(status_code=404, detail="게시글을 찾을 수 없습니다.")
```

## 주제 3: 응답 DTO

응답 모델은 서버가 클라이언트에게 보낼 **데이터의 구조를 정의하는 응답 DTO**이다. `PostCreate`가 요청 데이터의 형식을 정의하듯, 응답 모델은 반환할 데이터의 필드와 타입을 정의한다.

`mysite3/post.py`에는 두 응답 모델이 정의되어 있다. **목록과 검색 결과에는 `id`, `title`만** 보내고, **상세 조회·생성·수정 결과에는 `content`까지** 포함한다.

```python
# mysite3/post.py

class PostListResponse(BaseModel):
    id: int
    title: str

class PostDetailResponse(BaseModel):
    id: int
    title: str
    content: str
```

## 주제 4: response_model 지정

`mysite3/post_api.py`의 다음 import 구문은 응답 모델을 가져온다.

```python
from mysite3.post import PostDetailResponse, PostListResponse
```

각 함수의 `response_model`은 다음과 같다.

| 함수 | 데코레이터 |
|---|---|
| `get_posts` | `@router.get("", response_model=list[PostListResponse])` |
| `search_posts` | `@router.get("/search", response_model=list[PostListResponse])` |
| `get_post` | `@router.get("/{id}", response_model=PostDetailResponse)` |
| `create_post` | `@router.post("", response_model=PostDetailResponse, status_code=status.HTTP_201_CREATED)` |
| `update_post` | `@router.put("/{id}", response_model=PostDetailResponse)` |

FastAPI는 반환 데이터를 응답 모델에 맞게 **검증하고, 모델에 정의된 필드만 응답에 포함**한다. `Post` 객체에 `content`가 있어도 `PostListResponse`를 지정한 목록·검색 응답에서는 제외된다.

응답 모델은 **정상 응답의 형식**을 정의한다. 게시글이 없으면 `HTTPException`으로 오류 응답을 보낸다. 메시지 딕셔너리를 정상 응답으로 반환하면 응답 모델의 필수 필드가 없어 **응답 검증 오류**가 발생한다.

## 주제 5: 응답 확인

서버가 다시 시작되면 메모리의 게시글이 초기화된다. Swagger UI에서 게시글을 생성한 뒤, 반환된 `id`로 아래 요청을 실행한다.

| 요청 | 상태 코드 | 확인할 내용 |
|---|---|---|
| `POST /posts-response` | 201 | 생성된 게시글의 `id`, `title`, `content` |
| `GET /posts-response` | 200 | 각 게시글의 `id`, `title`만 반환 |
| `GET /posts-response/search?keyword=FastAPI` | 200 | 제목에 FastAPI가 포함된 게시글의 `id`, `title` |
| `GET /posts-response/{id}` | 200 | 해당 게시글의 `id`, `title`, `content` |
| `PUT /posts-response/{id}` | 200 | 변경된 게시글의 `id`, `title`, `content` |
| `DELETE /posts-response/{id}` | 204 | 응답 본문이 없음 |
| 삭제한 `id`로 조회·수정·삭제 | 404 | `detail`에 게시글을 찾을 수 없다는 메시지 |
| `content` 없이 게시글 생성 | 422 | `detail`에 필수 필드 누락에 대한 오류 |

## 헷갈리기 쉬운 부분

> ➕ **더 알아두기**
> (ChatGPT 정리본과 함께 학습한 내용이며 수업에 나온 설명은 아니다.)
>
> **1. `return`과 `raise`의 차이**
>
> ```python
> # 잘못된 예: 오류 메시지만 반환하면 일반 성공 응답(200)이 된다
> return {"message": "게시글이 없습니다."}
>
> # 올바른 오류 응답
> raise HTTPException(status_code=404, detail="게시글을 찾을 수 없습니다.")
> ```
>
> `return`은 함수의 정상 결과이고, `raise`는 정상 실행을 중단시켜 FastAPI의 예외 처리가 HTTP 오류 응답을 만들게 한다. 상태 코드와 JSON 본문은 서로 다른 개념이라서, 본문에 "없음"이라고 적었다고 상태 코드가 404가 되지는 않는다.
>
> **2. `detail`은 어디의 `detail`인가**
>
> `HTTPException(detail=...)`의 `detail`은 **예외 클래스의 정해진 인자 이름**이다. 오류 응답 본문이 `{"detail": "..."}` 모양이 되는 것도 이 때문이다. 반면 `return {"detail": ...}`처럼 일반 딕셔너리에 쓴 `detail`은 그냥 키 이름이어서 상태 코드를 바꾸지 않는다.
>
> **3. `response_model`은 숨기는 규칙이지 삭제가 아니다**
>
> 목록 응답에서 `content`가 빠져도 서버 메모리의 `Post` 객체에서 `content`가 지워진 것은 아니다. 출력할 때 필드를 걸러 내는 규칙일 뿐이다. 그래서 목록은 `PostListResponse`, 상세는 `PostDetailResponse`로 같은 데이터를 다르게 보여 줄 수 있다.
>
> **4. 422와 500은 다르다**
>
> 클라이언트가 보낸 **요청**의 필드·타입이 맞지 않으면 보통 **422**이다. 반대로 개발자가 `response_model`과 맞지 않는 값을 반환했다면 서버 쪽 약속 위반이어서 일반적으로 **500**이 된다. 위 주제 4의 "응답 검증 오류"가 이것이다.
>
> **5. 같은 게시글을 두 번 삭제하면**
>
> 첫 번째는 삭제되고 204, 두 번째는 이미 없으므로 404이다. 204 응답에는 본문을 넣지 않는다.
>
> **6. 코드 읽는 법**
>
> `status.HTTP_201_CREATED`는 숫자 `201`을 읽기 쉽게 쓴 상수이다. 같은 이름으로 `status.HTTP_404_NOT_FOUND`, `status.HTTP_422_UNPROCESSABLE_CONTENT` 등이 있다. 또 `delete_post`의 `return`은 값 없이 쓴 것이어서 `None`을 반환하지만, 상태 코드를 204로 지정했기 때문에 본문 없이 응답한다.

## 정리

- 상태 코드는 요청의 처리 결과를 숫자로 알려 준다. 200 성공, 201 생성, 204 성공이지만 본문 없음, 404 자원 없음, 422 요청 검증 실패이다. 지정하지 않은 정상 응답은 200이다.
- 오류 메시지를 `return`하면 200이다. 오류 상태 코드를 보내려면 `raise HTTPException(status_code=..., detail=...)`을 쓴다. `try-except`는 필요 없다.
- 데코레이터의 `status_code`로 생성은 201, 삭제는 204처럼 성공 상태 코드를 지정한다.
- 응답 DTO는 서버가 보낼 데이터의 필드와 타입을 정의한다. 목록은 `id`, `title`만, 상세는 `content`까지 보낸다.
- `response_model`을 지정하면 반환값을 검증하고 정의된 필드만 응답에 포함한다. 저장된 객체는 바뀌지 않는다.
- 오류 상황은 `HTTPException`으로 보내고, 정상 응답 모델에 맞지 않는 값(메시지 딕셔너리)을 정상 응답으로 반환하면 응답 검증 오류가 난다.

## ✅ 확인 질문

1. 200, 201, 204, 404, 422는 각각 어떤 의미이고 게시판의 어느 상황에서 쓰나요? 상태 코드를 지정하지 않으면 무엇이 되나요?
2. `return {"message": "게시글을 찾을 수 없습니다."}`와 `raise HTTPException(status_code=404, ...)`는 응답이 어떻게 다른가요?
3. `raise`를 실행하면 그 요청의 나머지 코드는 어떻게 되나요? 별도의 `try-except`가 필요 없는 이유는 무엇인가요?
4. `HTTPException`이 만드는 응답 본문은 어떤 모양이고, 거기의 `detail`은 일반 딕셔너리의 `detail` 키와 어떻게 다른가요?
5. 생성에는 201, 삭제에는 204를 지정하는 코드를 쓰고, 204 응답에 본문을 넣지 않는 이유를 설명해 보세요.
6. 응답 DTO란 무엇이고, `PostListResponse`와 `PostDetailResponse`는 어떻게 다르며 각각 어디에 쓰나요?
7. `Post` 객체에는 `content`가 있는데 목록 응답에는 왜 `content`가 나오지 않나요? 저장된 게시글에서 `content`가 지워진 것인가요?
8. 존재하지 않는 게시글에 대해 메시지 딕셔너리를 `response_model=PostDetailResponse`인 함수에서 정상 응답으로 반환하면 어떻게 되나요? 422와는 무엇이 다른가요?
9. 삭제한 `id`로 같은 `DELETE`를 한 번 더 보내면 어떤 상태 코드가 나오고 그 이유는 무엇인가요?
