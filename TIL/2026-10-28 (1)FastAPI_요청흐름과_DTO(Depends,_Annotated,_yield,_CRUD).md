---
tags: [fastapi, dependency-injection, pydantic, dto, crud]
til: v2 2026-10-28
---

# 2026-10-28_FastAPI_요청흐름과_DTO(Depends,_Annotated,_yield,_CRUD)
> 작성일: 2026-10-28

## 🔗 관련 글

- [SQLAlchemy·DB 연결 선행 실습](2026-10-27_SQLAlchemy%EC%99%80_PostgreSQL_%EA%B2%8C%EC%8B%9C%ED%8C%90_%EC%8B%A4%EC%8A%B5%28Engine%2C_Session%2C_CRUD%2C_Pydantic%29.md) — Engine·Session·기본 ORM의 선행 기록이다.
- [레이어드 게시판](2026-10-26%20%282%29%EB%A0%88%EC%9D%B4%EC%96%B4%EB%93%9C_%EA%B2%8C%EC%8B%9C%ED%8C%90_%EA%B5%AC%ED%98%84%28mysite4%2C_schemas%2C_Repository%2C_Service%2C_Router%2C_%EC%9A%94%EC%B2%AD_%ED%9D%90%EB%A6%84%29.md) — 강의의 Router·Service·Repository 구조와 비교한다.

## ❓ 아직 모르겠는 것

대화에서 풀린 질문도 복습 때 다시 확인한다. 아래 항목은 실제 사용자의 질문·확인 발언에서 추렸다.

- 앱 객체는 하나만 만드는가? 요청이 올 때마다 새로 만드는가? (주제 1)
- GET만 등록했는데 POST로 요청하면 그 함수가 대신 실행되는가? (주제 1)
- `data`의 타입 힌트가 검증하라는 표시인가? FastAPI에 Pydantic 객체를 넣는 것인가? (주제 2)
- DTO는 검증 형식·JSON·검증된 객체 중 무엇인가? ORM 모델도 DTO인가? (주제 2)
- 타입 힌트는 생략 가능하다는데 왜 Depends를 반드시 남겨야 하는가? 기본값을 넣어주는 것인가? (주제 3)
- `Annotated`는 자바 애너테이션이나 파이썬 데코레이터인가? (주제 3)
- yield는 무엇을 기다리고 누가 다시 진행시키는가? (주제 4)
- db는 Session인데 왜 get의 첫 인자로 모델을 다시 지정하는가? 기본키 외 조건도 넣을 수 있는가? (주제 5)
- HTTPException의 raise와 정상 return은 무엇이 다른가? (주제 6)

## 주제 1: 앱·경로·HTTP 메서드·Swagger UI

기준 자료는 [DB를 연동한 게시판 구현](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1448/)이며, 아래 단일 main.py 코드는 대화에서 강의의 계층 구조를 단순화해 작성한 실습이다. 강의는 Router→Service→Repository로 나누지만 실습은 먼저 함수 안에서 요청과 DB 작업을 연결했다.

FastAPI 객체는 요청 경로와 처리 함수를 등록하는 앱 객체이다. 하나의 앱에 조회·등록·수정·삭제를 모두 등록할 수 있다. 앱 생성 자체가 HTTP 서버 시작은 아니다. 서버가 시작될 때 앱과 처리 규칙이 준비되고, 이후 각 요청에 해당하는 함수를 호출한다. 로컬 주소가 인터넷에 자동 공개되는 것은 아니다.

| 요청 방식과 경로 | 역할 | 정상 상태 코드 |
|---|---|---|
| GET /posts | 게시글 목록 | 200 |
| POST /posts | 새 게시글 등록 | 201 |
| GET /posts/{post_id} | 상세 조회 | 200 |
| PUT /posts/{post_id} | 제목·내용 수정 | 200 |
| DELETE /posts/{post_id} | 삭제, 응답 본문 없음 | 204 |

같은 경로여도 HTTP 메서드가 다르면 서로 다른 함수를 등록할 수 있다. GET만 등록된 경로에 POST를 보내면 일반적으로 405 Method Not Allowed이고, 존재하지 않는 경로는 404이다. 브라우저 주소창 접속은 보통 GET이며 PUT·DELETE 요청을 보낸다는 뜻이 아니다.

> ➕ **더 알아두기**
> `@app.get(...)` 같은 데코레이터는 함수를 경로 처리자로 등록한다. 타입 옆에 정보를 붙이는 Annotated와는 문법·역할이 다르다. `/docs`의 Swagger UI는 FastAPI가 만든 OpenAPI 명세를 시각화한 API 시험 화면이다. Try it out은 입력, Execute는 실제 요청 전송이다. 별도로 만든 게시판 프런트엔드가 아니며 POST를 누를 때마다 실제 저장이 일어날 수 있다.

## 주제 2: 요청 DTO·ORM 모델·응답 DTO

DTO(Data Transfer Object)는 전달할 데이터를 담는 객체의 역할이다. PostCreate는 DTO 클래스, 함수의 data가 가리키는 실제 PostCreate 인스턴스는 값이 담긴 DTO 객체이다. JSON은 통신에서 데이터를 표현하는 형식이고 DTO 자체와 같지 않다. DTO가 반드시 검증 기능을 가져야 하는 것은 아니며, 이번에는 Pydantic으로 구현했기 때문에 검증까지 수행한다.

| 이름 | 역할 |
|---|---|
| PostCreate | 제목·내용 입력 구조와 검증을 정의한 요청 DTO 클래스 |
| data | 검증을 거친 제목·내용이 담긴 PostCreate 객체 |
| Post | DB의 posts 테이블과 컬럼을 매핑한 SQLAlchemy ORM 모델 클래스 |
| post | 조회하거나 새로 생성한 Post 객체 |
| PostResponse | 번호·제목·내용의 응답 구조 |
| PostDetailResponse | 댓글·태그 목록까지 포함하는 상세 응답 구조 |

사용자 JSON을 FastAPI가 읽고 Pydantic으로 검증한다. 통과하면 data를 전달하고, 서버 코드는 그 값을 읽어 별도의 Post 객체를 만든다. Session으로 저장한 후 FastAPI가 응답 모델을 기준으로 JSON을 만든다. DTO가 저절로 ORM 객체로 바뀌는 것이 아니다.

> ♻️ 처음엔 쿼리가 DTO이거나 Post도 DTO인 것으로 혼동했으나, 쿼리는 DB 작업 명령, DTO는 전달 객체의 역할, ORM 모델은 테이블·객체 매핑 정의이다. PostCreate와 테이블 이름이 같은 것도 아니다.

> ➕ **더 알아두기**
> DTO와 ORM 모델이 같은 필드를 가져도 목적은 다르다. 회원가입 입력은 비밀번호 원문·확인값을 받지만 DB 모델은 해시를 저장하고 응답 DTO에서는 원문과 해시를 모두 제외한다. 일반 클래스·dataclass도 DTO로 사용할 수 있다. schemas.py·models.py는 파일을 나누는 관례이며 파일명 자체가 기능을 만들지 않는다.

PostCreate의 조건은 제목 1~50자, 내용 1자 이상이다. 검증이 실패하면 기본 422이며 함수 본문은 실행하지 않는다. 응답 모델은 반환값의 구조도 검증한다. `list[PostResponse]`는 게시글 객체 여러 개를 목록으로 응답한다는 뜻이지 하나로 합치는 뜻은 아니다.

```python
class PostCreate(BaseModel):
    title: str = Field(min_length=1, max_length=50)
    content: str = Field(min_length=1)


class PostResponse(BaseModel):
    id: int
    title: str
    content: str

    model_config = ConfigDict(from_attributes=True)
```

`model_config`는 Pydantic이 읽는 클래스 속성이다. ConfigDict는 설정 딕셔너리를 구성하고 from_attributes=True는 객체의 속성에서도 값을 읽도록 허용한다. False여도 딕셔너리 키를 통한 입력은 검증할 수 있다. 필드 정의는 무엇이 필요한지, 이 설정은 어디에서 읽을 수 있는지를 정한다. 기본적으로 필드와 속성 이름을 맞추되 별칭·중첩 모델 등을 명시할 수 있다. id만 적었다고 임의의 a.b.c.id 경로를 자동 탐색하지 않는다.

## 주제 3: Annotated와 Depends는 누가 실행하는가

```python
def get_posts(db: Annotated[Session, Depends(get_db)]):
    ...
```

함수 매개변수는 db 하나이다. 콜론 뒤 Session은 타입, Depends(get_db)는 값 준비 방법, Annotated는 타입에 추가 메타데이터를 붙이는 typing의 표현이다. 일반 Python은 타입 힌트만으로 객체 생성·검증을 자동 수행하지 않는다. FastAPI가 선언을 읽고 의존성 주입을 수행한다. Pydantic은 본문 검증에 사용하지만 Depends의 실행 관리는 FastAPI 기능이다.

Depends는 그 자리에서 get_db를 실행해 Session을 기본값으로 보관하는 호출이 아니다. 요청 때 FastAPI가 get_db의 값을 준비해 db 인자로 넘기도록 하는 설정이다. 따라서 함수를 일반 Python 방식으로 직접 호출하면 FastAPI의 주입이 자동 수행되지 않는다.

> ➕ **더 알아두기**
> `db: Session = Depends(get_db)`나 `db=Depends(get_db)`도 의존성 준비를 지정한다. 그러나 단순 `def get_posts(db)`는 FastAPI에 준비 방법을 알려주지 않는다. 함수 안에서 Session을 직접 준비하는 대안은 가능하다. `db=get_db`는 함수 자체, `db=get_db()`는 제너레이터이고, 전역 Session 하나를 공유하면 요청 간 트랜잭션과 상태가 섞일 수 있다. 요청마다 독립 Session을 준비한다.

db라는 이름은 관례일 뿐 Session을 DB 자체로 바꾸지 않는다. session으로 이름을 바꿔도 된다. 이름보다 전달되는 객체와 Depends 설정이 중요하다.

## 주제 4: yield로 Session 수명 관리

```python
def get_db():
    with SessionLocal() as db:
        yield db
```

get_db는 Session 그 자체가 아니라 Session을 준비하는 제너레이터 함수이다. 호출만 하면 제너레이터가 반환되고, FastAPI가 진행시킬 때 Session을 만들어 yield한다. yield는 실행 위치·지역변수를 보존하고 값을 넘겨 제어권을 호출자에게 돌려준다. 바쁜 대기나 자동 비동기 실행이 아니다. 일반 Python에서는 next()로 진행시키고 끝나면 StopIteration으로 종료한다. FastAPI는 정상 진행과 예외 시 정리 과정을 관리한다.

반환한 Session으로 요청 함수를 처리하고 의존성 정리 시 with를 끝내 닫는다. Session을 만드는 순간 반드시 실제 연결까지 확보하는 것은 아니다. `with SessionLocal()`은 종료·자원 정리를 위한 범위이며 자동 commit을 뜻하지 않는다.

## 주제 5: 조회문과 Session.get

```python
@app.get("/posts", response_model=list[PostResponse])
def get_posts(db: Annotated[Session, Depends(get_db)]):
    statement = select(Post).order_by(Post.id)
    return db.scalars(statement).all()
```

select(Post)는 Post의 매핑 컬럼을 조회할 Select 객체를 만들고 order_by는 정렬을 지정한다. statement에는 실행할 조회문 객체가 들어 있다. scalars가 Session으로 실행하고 각 결과 행에서 첫 번째 항목을 꺼낸다. select(Post)의 항목은 Post 엔티티이므로 id 하나가 아니라 Post 객체가 나온다. all은 리스트이며 결과가 없으면 빈 리스트이다. 다중 컬럼 조회에 scalars를 쓰면 첫 항목만 남으므로 모든 행 필드는 execute 등의 방식으로 받는다.

```python
@app.get("/posts/{post_id}", response_model=PostDetailResponse)
def get_post(
    post_id: int,
    db: Annotated[Session, Depends(get_db)],
):
    post = db.get(Post, post_id)

    if post is None:
        raise HTTPException(
            status_code=404,
            detail="게시글을 찾을 수 없습니다.",
        )

    return post
```

db.get(Post, post_id)의 첫 위치 인자는 모델 클래스, 둘째는 그 모델의 기본키 값이다. id라는 이름을 추측하지 않고 primary_key 매핑을 따른다. 같은 Session으로 db.get(Comment, 7)을 하면 댓글 테이블을 조회한다. 첫 작업에서 테이블이 고정되거나 Session 생성 때 모든 테이블을 읽는 것은 아니다. 반환값은 관리되는 ORM 객체 하나 또는 None이다. Session에 유효한 객체가 있으면 SELECT가 생략될 수 있다.

복합 기본키는 두 번째 인자에 튜플·딕셔너리로 묶는다. 세 번째·네 번째 위치에 조건을 추가하지 않는다. 그 외 설정은 populate_existing 같은 키워드 인자이다. 기본키 외 조건은 select.where로 표현한다.

> ♻️ 처음엔 get_db와 db.get을 같은 Session 생성 기능으로 보았으나, get_db는 의존성 제공 함수이고 db.get은 이미 전달받은 Session의 기본키 조회 메서드이다. 오른쪽 Post 클래스는 조회할 매핑을 알려주는 정보이며 Session 객체가 아니다.

## 주제 6: 생성·수정·삭제와 실제 코드 상태

```python
@app.post("/posts", response_model=PostResponse, status_code=201)
def create_post(
    data: PostCreate,
    db: Annotated[Session, Depends(get_db)],
):
    post = Post(title=data.title, content=data.content)

    db.add(post)
    db.commit()

    return post


@app.put("/posts/{post_id}", response_model=PostResponse)
def update_post(
    post_id: int,
    data: PostCreate,
    db: Annotated[Session, Depends(get_db)],
):
    post = db.get(Post, post_id)

    if post is None:
        raise HTTPException(
            status_code=404,
            detail="게시글을 찾을 수 없습니다.",
        )

    post.title = data.title
    post.content = data.content

    db.commit()

    return post


@app.delete("/posts/{post_id}", status_code=204)
def delete_post(
    post_id: int,
    db: Annotated[Session, Depends(get_db)],
):
    post = db.get(Post, post_id)

    if post is None:
        raise HTTPException(
            status_code=404,
            detail="게시글을 찾을 수 없습니다.",
        )

    db.delete(post)
    db.commit()
```

생성 함수는 입력 DTO의 값을 읽어 Post 객체를 만들고 add로 등록한 후 commit한다. 수정은 관리 중인 객체의 속성을 바꿔 commit하므로 다시 add하지 않는다. commit은 필요한 flush를 수행한 뒤 DB 트랜잭션을 확정한다. 삭제는 delete로 지정하고 commit의 flush에서 DELETE를 실행한다. 204는 본문 없는 응답이고 201은 생성 성공이다.

없는 게시글은 None 검사를 거쳐 HTTPException으로 404를 발생시킨다. raise는 정상 return과 달리 실행을 예외 경로로 중단한다. 댓글 수정·삭제에서는 객체 존재 여부뿐 아니라 URL의 게시글과 실제 소속도 비교한다. 왼쪽 조건이 참이면 or의 오른쪽은 평가하지 않아 None 속성 접근을 피한다.

현재 저장 코드에서 게시글 CRUD와 댓글 등록·수정은 확인했다. 댓글 삭제 API는 대화에서 안내했으나 확인한 main.py에는 아직 없었다. 이 기록 작성을 이유로 앱 코드를 수정하거나 API 전체를 실행하지 않았다. 태그 테스트의 검증 범위는 다음 관계 파트에서 별도 기록한다.

## 정리

요청 경로·메서드 선택 → 의존성 Session 준비 → 입력 DTO 검증 → ORM 작업 → 트랜잭션 확정 → 응답 DTO 검증·JSON 변환이라는 역할을 나눈다. DTO 클래스와 실제 인스턴스, JSON 형식, ORM 모델, SQL 조회문은 서로 다르다.

## ✅ 확인 질문

1. 같은 /posts 경로에 GET과 POST를 등록해도 구분되는 이유는 무엇인가?
2. Swagger UI의 Execute는 어떤 실제 작업을 수행하며 앱 생성과 서버 실행은 어떻게 다른가?
3. JSON·PostCreate 클래스·data 객체·Post 객체의 차이를 등록 요청의 흐름으로 설명할 수 있는가?
4. from_attributes=True는 무엇을 바꾸며 False일 때도 스키마 검증이 의미 있는 이유는 무엇인가?
5. 타입 힌트를 생략할 수 있어도 단순 db 매개변수만으로 Session이 주입되지 않는 이유는 무엇인가?
6. Annotated와 Depends 각각의 역할, 그리고 해당 정보를 실제로 처리하는 주체는 무엇인가?
7. yield는 무엇을 보존하고 제어권을 어디로 넘기며, FastAPI는 언제 정리 과정을 진행시키는가?
8. get_db와 db.get이 서로 다른 이유를 객체 생성·조회 관점에서 설명할 수 있는가?
9. select(Post)를 만든 시점과 db.scalars(statement).all()을 실행한 시점은 어떻게 다른가?
10. get에 모델과 기본키를 모두 넘기는 이유와 복합 기본키 전달 방식은 무엇인가?
11. add 없이 조회 객체의 제목을 바꾸고 commit해도 UPDATE가 실행되는 이유는 무엇인가?
12. 201·204·404·405·422가 각각 발생하는 상황을 이 게시판에서 설명할 수 있는가?
