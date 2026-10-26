---
tags: [fastapi, validation, pydantic, httpexception, field]
til: v2 2026-10-25
---

# 2026-10-25 (3)Validation(빈 문자열 검증, 422, 검증 순서, Field)
> 작성일: 2026-10-25

## 🔗 관련 글

- [Pydantic 요청 DTO(BaseModel, Request Body, 422 검증, PostCreate와 Post)](2026-10-24%20(2)Pydantic_요청_DTO(BaseModel,_Request_Body,_422_검증,_PostCreate와_Post).md) — Pydantic이 필수 여부와 타입을 검증한다는 것을 배웠고, 그 글에서 "`str`은 빈 문자열을 막지 못한다"고 남겨 둔 문제를 이 글에서 푼다.
- [상태 코드와 응답 모델(HTTPException, 201·204·404·422, response_model, 응답 DTO)](2026-10-25%20(2)상태_코드와_응답_모델(HTTPException,_201_204_404_422,_response_model,_응답_DTO).md) — 바로 앞 글이다. `HTTPException`과 422·404의 의미를 거기서 배웠고, 이 글에서 그것으로 검증 실패를 돌려준다.
- [Transaction(BEGIN, COMMIT, ROLLBACK, 오류 처리)](2026-10-07%20(3)Transaction(BEGIN,_COMMIT,_ROLLBACK,_오류_처리).md) — 잘못된 변경을 되돌리는 DB의 방법이다. 이 글은 변경 **전에** 검사해서 아예 시작하지 않는 방법이다.

## ❓ 아직 모르겠는 것

이 글은 강의 레슨(간단한 Validation)을 읽고 정리했고, 대화 중 사용자가 직접 한 질문은 없다. 아래는 사용자의 발언이 아니라, 함께 학습한 ChatGPT 정리본이 별도 항목으로 짚어 둔 헷갈리기 쉬운 지점이다. 복습 때 다시 확인한다.

- 검증을 왜 `post_id += 1`보다 먼저 해야 하는가? (주제 3)
- `Field(min_length=1)`이면 공백만 있는 `" "`도 막히는가? (아래 ➕ 참고)
- 없는 `id`에 빈 제목이 들어오면 404와 422 중 무엇이 먼저 나오는가? (아래 ➕ 참고)

## 주제 1: Validation이란

**Validation(유효성 검증)** 은 입력받은 데이터가 미리 정한 조건에 맞는지 검사하는 과정이다. 데이터를 **저장하거나 수정하기 전에** 검사하여 잘못된 데이터가 저장되는 것을 막는다.

게시글을 작성할 때는 다음과 같은 조건을 확인할 수 있다.

| 검증 기준 | 확인할 내용 |
|---|---|
| 필수 여부 | 제목과 내용이 요청에 포함되어 있는가? |
| 타입 | 제목과 내용이 문자열인가? |
| 값의 조건 | 제목과 내용이 빈 문자열은 아닌가? |

Pydantic은 `PostCreate`에 선언한 **필수 필드와 타입**을 검증한다. 하지만 현재 `title: str`, `content: str`에는 **빈 문자열도 허용**된다.

```json
{"title": "", "content": "내용"}
```

위 요청에는 `title`이 있고 값도 문자열이므로 기존 검증을 통과한다. 그러나 제목이 있는 게시글만 허용하려면 **빈 문자열인지 확인하는 조건을 추가**해야 한다. 타입이 올바르더라도 값이 필요한 조건을 만족하는지는 별도로 확인해야 한다.

이 실습에서는 제목이나 내용이 빈 문자열이면 생성·수정을 중단하고, `HTTPException`으로 **422 응답과 오류 메시지**를 보낸다. 검증을 통과한 데이터만 저장하거나 수정한다.

## 주제 2: 게시글 생성 검증

`mysite3/post_api.py`의 생성·수정 함수에서 빈 문자열을 검사한다. 검증에 실패하면 예외가 발생하여 저장·수정 코드가 실행되지 않는다.

`create_post`는 **검증을 통과한 경우에만** ID를 증가시키고 게시글을 저장한다.

```python
# mysite3/post_api.py

@router.post(
    "",
    response_model=PostDetailResponse,
    status_code=status.HTTP_201_CREATED,
)
def create_post(post_data: PostCreate):
    global post_id

    if post_data.title == "":
        raise HTTPException(
            status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
            detail="title을 입력하세요",
        )

    if post_data.content == "":
        raise HTTPException(
            status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
            detail="content를 입력하세요",
        )

    post_id += 1
    new_post = Post(post_id, post_data.title, post_data.content)
    posts.append(new_post)

    return new_post
```

## 주제 3: 게시글 수정 검증

`update_post`도 제목과 내용을 먼저 검사한다. 검증을 통과한 경우에만 기존 게시글의 값을 변경한다.

```python
# mysite3/post_api.py

@router.put("/{id}", response_model=PostDetailResponse)
def update_post(id: int, updated_post: PostCreate):
    if updated_post.title == "":
        raise HTTPException(
            status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
            detail="title을 입력하세요",
        )

    if updated_post.content == "":
        raise HTTPException(
            status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
            detail="content를 입력하세요",
        )

    for post in posts:
        if post.id == id:
            post.title = updated_post.title
            post.content = updated_post.content
            return post

    raise HTTPException(status_code=404, detail="게시글을 찾을 수 없습니다.")
```

> ➕ **더 알아두기**
> 이 레슨 페이지는 같은 내용이 두 번 나온다. 앞쪽은 `mysite3/post_api.py`에 검증을 적용한 설명이고, 뒤쪽은 "앞 실습에서 `mysite2/post_api2.py`를 `mysite2/post_api.py`로 이름을 바꾼 코드를 이어서 사용한다"는 설명이다. 폴더 이름(`mysite3`, `mysite2`)만 다르고 검증 코드는 같다. 이 글은 앞쪽(`mysite3`)을 기준으로 적었다.

## 주제 4: 응답 확인

빈 문자열 검증이 포함된 예제를 실행한 뒤, Swagger UI의 `mysite3` 그룹에서 `/posts-response` 경로로 요청을 보낸다.

| 요청 | 결과 |
|---|---|
| 제목이나 내용을 빈 문자열로 생성 | 422 |
| 제목이나 내용을 빈 문자열로 수정 | 422 |
| 정상 입력으로 생성 | 201, `id`·`title`·`content` 반환 |
| 정상 입력으로 수정 | 200, `id`·`title`·`content` 반환 |

간단한 입력 검증은 Pydantic의 `Field`로도 설정할 수 있다. 자세한 방법은 "데이터 검증 및 필드 제어"에서 다룬다고 레슨에 안내되어 있다.

## 헷갈리기 쉬운 부분

> ➕ **더 알아두기**
> (ChatGPT 정리본과 함께 학습한 내용이며 수업에 나온 설명은 아니다.)
>
> **1. 검증 위치를 데이터 변경보다 앞에 두는 이유**
>
> `post_id += 1`을 검증보다 먼저 하면 빈 제목으로 요청이 실패해도 ID만 늘어난다. 검증을 먼저 하면 **실패한 요청이 데이터에 아무 영향도 주지 않는다.** 다만 DB의 자동 번호(시퀀스)는 실패한 트랜잭션에서도 번호가 건너뛸 수 있어서, ID가 끊기지 않고 이어지는 것이 보장되지는 않는다.
>
> **2. 두 필드가 모두 비어 있으면?**
>
> `raise`에서 실행이 멈추므로 첫 번째 `title` 오류 하나만 전달된다. 여러 오류를 한꺼번에 보여 주려면 Pydantic 같은 검증 도구가 더 적합하다.
>
> **3. `Field`로 같은 일을 하기**
>
> ```python
> from pydantic import BaseModel, Field
>
> class PostCreate(BaseModel):
>     title: str = Field(min_length=1)
>     content: str = Field(min_length=1)
> ```
>
> 이렇게 하면 함수에 들어오기 **전에** FastAPI가 검증한다. 그런데 공백 한 칸 `" "`은 길이가 1이므로 `min_length=1`을 통과한다. 공백만 있는 입력도 막으려면 `if not data.title.strip():` 같은 추가 검사가 필요하다. `strip()`은 앞뒤 공백을 제거한 **새 문자열**을 돌려줄 뿐이어서, 검사만 하고 원래 값을 저장하면 공백이 그대로 저장된다. **검증(validation)** 과 **값 정리(normalization)** 는 다른 일이다.
>
> **4. 검증 순서에 따라 응답이 달라진다**
>
> 수정 코드의 순서가 "제목 검증 → 내용 검증 → 게시글 존재 확인 → 수정"이면, 존재하지 않는 `id`에 빈 제목이 들어왔을 때 **404보다 422가 먼저** 나온다. 어떤 것을 먼저 알려 줄지는 서비스 규칙에 따라 정하되, 의도해서 정해야 한다.
>
> **5. Pydantic 검증과 직접 검증의 구분**
>
> 필수 여부·타입·길이 같은 **입력 형식**은 Pydantic이 주로 확인하고, 이 수업처럼 `if`로 직접 확인하는 것은 **값의 조건**이다. 다음 수업(레이어드 아키텍처)에서 이것을 "Pydantic이 하는 검증"과 "Service가 하는 검증"으로 나누는 기준을 배운다.

> 📝 **기사 시험 포인트** (정보보안기사)
> 웹 애플리케이션에서 입력값 검증은 기본 보안 대책이다. 입력값 검증이 부족하면 SQL 삽입, 크로스 사이트 스크립팅(XSS) 같은 취약점의 원인이 된다. 또 브라우저 쪽 검증은 우회될 수 있으므로 **서버 쪽에서도 반드시 검증**해야 한다. 이 글의 검증은 서버(FastAPI) 쪽 검증에 해당한다. (이 글의 빈 문자열 검사 자체가 그런 공격을 막는다는 뜻은 아니다.)

## 정리

- Validation은 입력이 조건에 맞는지 **저장·수정 전에** 검사하는 과정이다. 필수 여부, 타입, 값의 조건을 확인한다.
- Pydantic은 필수 필드와 타입을 검증한다. `str` 선언만으로는 빈 문자열을 막지 못하므로 값의 조건은 따로 검사한다.
- 이 실습은 `title == ""`, `content == ""`이면 `HTTPException`으로 **422**와 메시지(`title을 입력하세요`, `content를 입력하세요`)를 보내고 처리를 중단한다.
- 생성은 검증을 통과한 뒤에만 `post_id`를 증가시켜 저장한다. 수정도 검증을 통과해야 값을 바꾼다.
- 검증에 실패한 요청은 데이터를 바꾸지 않아야 한다.
- 간단한 검증은 Pydantic `Field`로도 설정할 수 있다.

## ✅ 확인 질문

1. Validation은 무엇이고, 게시글 작성에서 확인할 수 있는 세 가지 기준은 무엇인가요?
2. `{"title": "", "content": "내용"}`이 Pydantic 검증을 통과하는 이유는 무엇이고, 그래서 어떤 추가 조건이 필요한가요?
3. 빈 문자열 검증에 실패하면 어떤 상태 코드와 메시지가 나가나요? 왜 `return`이 아니라 `raise`를 쓰나요?
4. `create_post`에서 검증을 `post_id += 1`보다 먼저 두는 이유를 실패한 요청을 예로 설명해 보세요.
5. 수정 요청에서 존재하지 않는 `id`와 빈 제목이 함께 들어오면 어떤 코드가 나오나요? 순서를 바꾸면 결과가 어떻게 달라지나요?
6. `Field(min_length=1)`을 쓰면 `" "`(공백 한 칸)도 막히나요? 막으려면 무엇이 더 필요하고, 검증과 값 정리는 어떻게 다른가요?
