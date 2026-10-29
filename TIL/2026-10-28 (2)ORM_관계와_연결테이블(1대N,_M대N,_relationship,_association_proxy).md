---
tags: [sqlalchemy, one-to-many, many-to-many, relationship, association-proxy]
til: v2 2026-10-28
---

# 2026-10-28_ORM_관계와_연결테이블(1대N,_M대N,_relationship,_association_proxy)
> 작성일: 2026-10-28

## 🔗 관련 글

- [FastAPI 요청·DTO](2026-10-28%20%281%29FastAPI_%EC%9A%94%EC%B2%AD%ED%9D%90%EB%A6%84%EA%B3%BC_DTO%28Depends%2C_Annotated%2C_yield%2C_CRUD%29.md) — API 입력·출력과 의존성 구조를 관계 실습에 사용한다.
- [모델과 DB 연결 기초](2026-10-27_SQLAlchemy%EC%99%80_PostgreSQL_%EA%B2%8C%EC%8B%9C%ED%8C%90_%EC%8B%A4%EC%8A%B5%28Engine%2C_Session%2C_CRUD%2C_Pydantic%29.md) — Mapped·Base·테이블 생성의 기반이다.

## ❓ 아직 모르겠는 것

실제 대화에서 해결한 의문도 복습 대상으로 남긴다.

- ORM 모델·DTO·SQL·테이블 정의는 무엇이 다른가? (주제 1)
- 외래키로 번호를 저장했는데 왜 post라는 관계 속성이 또 필요한가? (주제 2)
- SQLAlchemy는 relationship 줄에 posts.id가 없어도 어느 외래키인지 어떻게 아는가? 여러 개면 어떻게 하는가? (주제 2)
- back_populates는 테이블명·컬럼명·상대 속성 중 무엇을 지정하는가? (주제 2)
- Comment(content=..., post=post)의 오른쪽 post가 Session인가? 왼쪽 post는 어디에 정의되는가? (주제 3)
- 게시글과 댓글의 부모 삭제는 왜 따로 정책을 정하는가? (주제 3)
- 연결 모델은 왜 id·post_id·tag_id를 모두 가지며 조합 중복을 막는가? (주제 4)
- __tablename__과 __table_args__는 변수인가? Mapped 타입 힌트 오른쪽 값은 실제 데이터인가? (주제 4)
- 태그 생성·연결도 기존 CRUD의 반복인가? 공유된 태그를 지우지 않으려면 무엇을 삭제하는가? (주제 5)
- association_proxy는 무슨 기능이며 왜 post.tags로 쓸 수 있는가? (주제 6)
- 강의의 상품 리뷰·기획전 실습도 실제 프로젝트에 구현한 것인가? (미구현 범위 확인, 주제 7)

## 주제 1: 모델·DTO·SQL을 먼저 구분한다

원문은 [1:N 관계](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1446/), [1:N 실습](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1449/), [M:N 관계](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1445/), [M:N 실습](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1450/)이다. 강의는 mysite_one_to_many·mysite_many_to_many의 계층 구조를 사용하고 실습은 단일 models.py·schemas.py·main.py에서 같은 개념을 연결했다. 이름과 경로가 강의와 다르므로 동일한 원본 코드라고 주장하지 않는다.

| 역할 | 실습의 예 |
|---|---|
| DB 스키마·매핑을 정의한 ORM 모델 클래스 | Post, Comment, Tag, PostTag |
| 요청·응답 전달용 DTO 클래스 | CommentCreate, CommentResponse, TagCreate, TagResponse |
| 실제 값이 담긴 객체 | data, post, comment, tag, link 등이 가리키는 인스턴스 |
| DB 작업 명령 | INSERT·SELECT·UPDATE·DELETE, select로 구성한 조회문 |

`id: Mapped[int] = mapped_column(primary_key=True)`에서 id는 클래스 속성 이름, 콜론 뒤는 타입 힌트, 우측은 실제 번호가 아니라 컬럼 정의 객체이다. SQLAlchemy가 정의를 읽어 매핑을 구성한 뒤 실제 인스턴스에서는 id 값으로 접근한다. 모델 정의만으로 SQL이 즉시 실행되지 않는다.

> ♻️ 처음엔 쿼리를 DTO로 보거나 Comment를 DTO로 보았으나, 입력 DTO의 값을 읽어 별도의 ORM 객체를 만들고 Session으로 저장한다. ORM 클래스는 테이블·컬럼·관계 정의이며 SQL 문자열이 아니다.

## 주제 2: 1:N 관계와 객체 접근 설정

게시글 하나에 여러 댓글이 달리되 댓글 하나는 게시글 하나에 속한다. DB의 comments.post_id가 posts.id를 참조한다. 게시글 번호 3을 가진 댓글 여러 행이 존재할 수 있어 1:N이다.

ForeignKey는 DB 컬럼과 참조 무결성을 정의한다. relationship은 연결된 객체로 접근하는 속성을 만든다. DB에 post라는 컬럼이 추가되는 것은 아니다. post_id만으로도 번호를 읽어 직접 SELECT를 작성할 수 있으며 관계 속성은 필수가 아니다. 모델에서 관계 설정을 선언했다는 것과 실제 객체를 조회했다는 것은 다르다.

| 접근 | 의미 |
|---|---|
| comment.post_id | 소속 게시글 번호 |
| comment.post | 연결된 Post 객체 |
| post.comments | 연결된 Comment 객체 목록 |
| comment.post.title | 댓글의 게시글 제목 |

SQLAlchemy는 relationship의 대상 모델과 테이블 사이의 외래키 매핑을 살펴 연결 조건을 결정한다. post라는 이름을 추측해서 찾는 것이 아니다. 같은 대상 테이블로 향하는 외래키 경로가 여러 개이면 foreign_keys로 어느 컬럼을 사용할지 명시해야 한다. 모호한 상태에서는 임의 선택하지 않는다. 외래키 없이도 primaryjoin 등의 명시 설정으로 관계를 정의하는 별도 방식은 있지만 현재 실습은 자동 추론 가능한 외래키를 사용한다.

back_populates는 상대 모델의 관계 속성 이름이다. Post.comments의 상대는 Comment.post이고 Comment.post의 상대는 Post.comments이다. 한쪽 객체 관계를 설정하면 반대쪽 목록·참조도 동기화되며 이것이 commit을 대신하지는 않는다.

> ➕ **더 알아두기**
> 이미 읽은 댓글에서 게시글 제목도 필요한 경우 comment.post가 편리하다. 댓글 쪽 관계를 사용하지 않는 단방향 구성도 가능하지만 back_populates 양쪽을 일관되게 바꿔야 한다. 문자열 타입 이름은 아래에 선언될 클래스를 나중에 찾도록 한다. 파일이 나뉘면 TYPE_CHECKING과 문자열 참조로 타입 검사용 import와 런타임 import를 구분할 수 있고, 런타임 모델 등록은 별도로 필요하다.

## 주제 3: 댓글 API와 삭제 범위

| 강의의 게시글·댓글 API | 기능 | 정상 코드 |
|---|---|---|
| POST /posts-1n | 게시글 생성 | 201 |
| GET /posts-1n | 목록 | 200 |
| GET /posts-1n/{post_id} | 댓글 포함 상세 | 200 |
| PUT /posts-1n/{post_id} | 게시글 수정 | 200 |
| DELETE /posts-1n/{post_id} | 게시글과 댓글 삭제 | 204 |
| POST /posts-1n/{post_id}/comments | 댓글 등록 | 201 |
| PUT /posts-1n/{post_id}/comments/{comment_id} | 해당 글의 댓글 수정 | 200 |
| DELETE /posts-1n/{post_id}/comments/{comment_id} | 해당 글의 댓글 삭제 | 204 |

실습은 /posts 경로를 사용한다. 댓글 내용은 1~200자로 검증하고 게시글 번호는 URL에서 받는다. Comment(content=data.content, post=post)는 요청 DTO의 내용과 이미 조회한 Post 객체를 사용해 새 ORM 객체를 만드는 표현이다. 왼쪽 post=는 관계 속성 이름, 오른쪽 post는 실제 객체이다. Session 객체를 넣는 것이 아니다. 객체를 연결하면 flush에서 post_id가 관계에 맞게 반영된다.

```python
class CommentCreate(BaseModel):
    content: str = Field(min_length=1, max_length=200)


class CommentResponse(BaseModel):
    id: int
    content: str
    post_id: int

    model_config = ConfigDict(from_attributes=True)


class PostDetailResponse(PostResponse):
    comments: list[CommentResponse] = Field(default_factory=list)
    tags: list[TagResponse] = Field(default_factory=list)
```

```python
@app.post(
    "/posts/{post_id}/comments",
    response_model=CommentResponse,
    status_code=201,
)
def create_comment(
    post_id: int,
    data: CommentCreate,
    db: Annotated[Session, Depends(get_db)],
):
    post = db.get(Post, post_id)

    if post is None:
        raise HTTPException(
            status_code=404,
            detail="게시글을 찾을 수 없습니다.",
        )

    comment = Comment(content=data.content, post=post)

    db.add(comment)
    db.commit()

    return comment


@app.put(
    "/posts/{post_id}/comments/{comment_id}",
    response_model=CommentResponse,
)
def update_comment(
    post_id: int,
    comment_id: int,
    data: CommentCreate,
    db: Annotated[Session, Depends(get_db)],
):
    comment = db.get(Comment, comment_id)

    if comment is None or comment.post_id != post_id:
        raise HTTPException(
            status_code=404,
            detail="해당 게시글의 댓글을 찾을 수 없습니다.",
        )

    comment.content = data.content

    db.commit()

    return comment
```

댓글이 없거나 URL의 게시글 소속이 아니면 404로 중단한다. 존재하는 댓글 번호만 확인하면 다른 게시글의 댓글을 잘못 수정·삭제할 수 있다. 이 소속 검사는 로그인 사용자에 대한 인가 검사와 별개이다.

상세 응답은 PostDetailResponse의 comments를 읽는다. from_attributes=True에 따라 ORM 객체 속성을 읽는 과정에서 미조회 댓글이 지연 로딩될 수 있다. default_factory=list는 입력값이 없을 때 사용할 독립 빈 목록을 만든다. 정상적으로 조회한 관계 컬렉션이 비어 있으면 댓글 응답도 빈 목록이다.

실습의 Post.comments에는 cascade="all, delete-orphan"를 설정했다. all에는 부모 삭제를 자식에게 전달하는 delete 동작 등이 포함되고, delete-orphan은 부모 관계에서 제거한 자식도 삭제 대상으로 만든다. 게시글 삭제 시 댓글을 지우지만 댓글 하나 삭제 시 게시글을 지우지 않는다. SQLAlchemy의 객체 단위 동작이며 DB의 ON DELETE CASCADE와 동일한 설정은 아니다.

> ➕ **더 알아두기**
> 현재 파일에는 댓글 생성·수정 함수가 있으며 댓글 삭제는 대화에서 안내된 예제이다. 이 문서를 작성하는 동안 저장된 코드와 설명한 코드를 혼동해 삭제 API가 완성됐다고 기록하지 않는다. 원문은 cascade="all" 예시이고 delete-orphan 추가는 실습에서 적용한 선택이다.

## 주제 4: M:N을 표현하는 연결 모델과 제약

게시글 하나가 여러 태그를 사용하고 태그 하나도 여러 게시글에서 공유된다. tags에 post_id 하나만 두면 여러 게시글 공유를 제대로 표현할 수 없으므로 post_tags 연결 테이블을 둔다.

| 연결 id | post_id | tag_id | 뜻 |
|---|---|---|---|
| 1 | 1 | 10 | 1번 글의 10번 태그 |
| 2 | 1 | 20 | 1번 글의 20번 태그 |
| 3 | 2 | 10 | 2번 글도 10번 태그 공유 |

Post→PostTag와 Tag→PostTag는 각각 1:N이다. 연결 자체의 id는 게시글 번호·태그 번호와 별개이다. 각 post_id·tag_id의 반복은 허용하되 두 컬럼의 조합은 고유해야 같은 연결이 두 번 저장되지 않는다. Tag.name의 unique는 태그 자체의 이름 중복을 막고, 연결 조합의 UniqueConstraint는 동일한 연결 중복을 막는다.

> ♻️ 처음엔 UniqueConstraint가 중복 행을 자동 제거한다고 생각했으나, 중복 저장을 오류로 거부하는 제약이다. 기존 데이터를 지우거나 합치지 않는다.

__tablename__은 실제 테이블 이름을 담은 클래스 속성이고 __table_args__는 테이블 제약 등의 설정을 담은 속성이다. 이름 앞뒤 밑줄은 특별한 대입 문법이 아니며 SQLAlchemy가 정해진 이름을 읽는다. 튜플에 설정 하나만 넣어도 쉼표를 쓴다. 다른 설정으로 CheckConstraint·PrimaryKeyConstraint·ForeignKeyConstraint·Index 등이 있으며 필요할 때만 추가한다.

```python
from sqlalchemy import ForeignKey, String, Text, UniqueConstraint
from sqlalchemy.ext.associationproxy import AssociationProxy, association_proxy
from sqlalchemy.orm import Mapped, mapped_column, relationship

from database import Base


class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(50), nullable=False)
    content: Mapped[str] = mapped_column(Text, nullable=False)

    comments: Mapped[list["Comment"]] = relationship(
        "Comment",
        back_populates="post",
        cascade="all, delete-orphan",
    )

    post_tags: Mapped[list["PostTag"]] = relationship(
        "PostTag",
        back_populates="post",
        cascade="all, delete-orphan",
    )

    tags: AssociationProxy[list["Tag"]] = association_proxy(
        "post_tags", "tag", creator=lambda tag: PostTag(tag=tag)
    )


class Comment(Base):
    __tablename__ = "comments"

    id: Mapped[int] = mapped_column(primary_key=True)
    content: Mapped[str] = mapped_column(String(200), nullable=False)
    post_id: Mapped[int] = mapped_column(
        ForeignKey("posts.id"),
        nullable=False,
    )

    post: Mapped["Post"] = relationship(
        "Post",
        back_populates="comments",
    )


class Tag(Base):
    __tablename__ = "tags"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(
        String(50),
        unique=True,
        nullable=False,
    )

    post_tags: Mapped[list["PostTag"]] = relationship(
        "PostTag",
        back_populates="tag",
        cascade="all, delete-orphan",
    )


class PostTag(Base):
    __tablename__ = "post_tags"
    __table_args__ = (UniqueConstraint("post_id", "tag_id"),)

    id: Mapped[int] = mapped_column(primary_key=True)
    post_id: Mapped[int] = mapped_column(
        ForeignKey("posts.id"),
        nullable=False,
    )

    tag_id: Mapped[int] = mapped_column(
        ForeignKey("tags.id"),
        nullable=False,
    )

    post: Mapped["Post"] = relationship(
        "Post",
        back_populates="post_tags",
    )

    tag: Mapped["Tag"] = relationship(
        "Tag",
        back_populates="post_tags",
    )
```

위는 기록 시점 실습의 모델 코드이다. PostTag에는 post·tag 객체 관계가 있고 Post·Tag에는 연결 객체 목록인 post_tags가 있다. Post.post_tags↔PostTag.post, Tag.post_tags↔PostTag.tag가 각각 짝이다. Post.post_tags 목록의 원소가 Tag 자체는 아니라는 점을 구분한다.

> 📝 **기사 시험 포인트** (정보처리기사)
> 기본키(Primary Key)는 행을 식별하고 외래키(Foreign Key)는 다른 테이블의 키를 참조해 참조 무결성을 유지한다. M:N은 연결 테이블의 두 외래키를 통해 표현할 수 있다. 현재 예제의 기본키는 단일 id이고, post_id·tag_id 조합은 별도의 고유 제약이다. 두 컬럼을 함께 기본키로 정의하는 설계와 구분한다.

## 주제 5: 태그 생성·공유·연결 해제의 흐름

강의의 기능을 실습 경로로 정리하면 다음과 같다.

| 메서드·경로 | 입력 | 결과·상태 |
|---|---|---|
| POST /tags | name | 태그 생성, 201 |
| GET /tags | 없음 | 목록, 200 |
| POST /posts/{post_id}/tags | name | 기존 태그 재사용 또는 새 태그 생성 후 연결, 201 |
| POST /posts/with-tags | title·content·tags | 태그와 게시글을 함께 생성, 201 |
| GET /posts/{post_id} | 경로 번호 | 댓글·태그 포함 상세, 200 |
| DELETE /posts/{post_id}/tags/{tag_id} | 경로 번호들 | 연결만 해제, 204 |

태그 단독 생성의 같은 이름은 400, 같은 게시글에 같은 태그 재연결도 400, 없는 게시글·없는 연결은 404, 입력 조건 위반은 422이다. 실습은 동시 요청에서 DB 고유 제약이 충돌하면 트랜잭션을 취소하고 409로 재시도를 안내한다. 이 409 처리와 태그명 길이 검증은 실습에서 보강한 것이다.

태그와 함께 생성할 때 이름 목록의 중복을 먼저 제거하고 IN 조건으로 기존 태그를 한 번에 조회해 이름별로 재사용한다. 없는 이름만 새 객체를 만든다. 입력 tags가 없거나 비어 있으면 태그 없이 게시글을 만든다. 태그명은 현재 대소문자·공백 정규화 없이 입력값 기준으로 비교하므로 Python과 python은 다르게 취급된다.

```python
class TagCreate(BaseModel):
    name: str = Field(min_length=1, max_length=50)


class TagResponse(BaseModel):
    id: int
    name: str

    model_config = ConfigDict(from_attributes=True)


class PostCreateWithTags(PostCreate):
    tags: list[Annotated[str, Field(min_length=1, max_length=50)]] = Field(
        default_factory=list
    )
```

```python
@app.post("/posts/with-tags", response_model=PostDetailResponse, status_code=201)
def create_post_with_tags(
    data: PostCreateWithTags,
    db: Annotated[Session, Depends(get_db)],
):
    try:
        with db.begin():
            names = list(dict.fromkeys(data.tags))
            tags_by_name = {
                tag.name: tag
                for tag in db.scalars(select(Tag).where(Tag.name.in_(names))).all()
            }
            post = Post(title=data.title, content=data.content)
            for name in names:
                tag = tags_by_name.get(name)
                if tag is None:
                    tag = Tag(name=name)
                post.tags.append(tag)
            db.add(post)
    except IntegrityError:
        raise HTTPException(status_code=409, detail="태그가 동시에 변경되었습니다. 다시 시도하세요.") from None

    return post


@app.post("/tags", response_model=TagResponse, status_code=201)
def create_tag(data: TagCreate, db: Annotated[Session, Depends(get_db)]):
    try:
        with db.begin():
            if db.scalar(select(Tag).where(Tag.name == data.name)) is not None:
                raise HTTPException(status_code=400, detail="이미 존재하는 태그 이름입니다.")
            tag = Tag(name=data.name)
            db.add(tag)
    except IntegrityError:
        raise HTTPException(status_code=409, detail="태그가 동시에 변경되었습니다. 다시 시도하세요.") from None

    return tag


@app.get("/tags", response_model=list[TagResponse])
def get_tags(db: Annotated[Session, Depends(get_db)]):
    return db.scalars(select(Tag).order_by(Tag.id)).all()


@app.post("/posts/{post_id}/tags", response_model=PostDetailResponse, status_code=201)
def add_tag_to_post(
    post_id: int,
    data: TagCreate,
    db: Annotated[Session, Depends(get_db)],
):
    try:
        with db.begin():
            post = db.get(Post, post_id)
            if post is None:
                raise HTTPException(status_code=404, detail="게시글을 찾을 수 없습니다.")

            tag = db.scalar(select(Tag).where(Tag.name == data.name))
            if tag is None:
                tag = Tag(name=data.name)
            elif db.scalar(
                select(PostTag.id).where(
                    PostTag.post_id == post_id, PostTag.tag_id == tag.id
                )
            ) is not None:
                raise HTTPException(status_code=400, detail="이미 이 게시글에 연결된 태그입니다.")

            post.tags.append(tag)
    except IntegrityError:
        raise HTTPException(status_code=409, detail="태그가 동시에 변경되었습니다. 다시 시도하세요.") from None

    return post


@app.delete("/posts/{post_id}/tags/{tag_id}", status_code=204)
def remove_tag_from_post(
    post_id: int,
    tag_id: int,
    db: Annotated[Session, Depends(get_db)],
):
    with db.begin():
        post = db.get(Post, post_id)
        if post is None:
            raise HTTPException(status_code=404, detail="게시글을 찾을 수 없습니다.")

        link = db.scalar(
            select(PostTag).where(
                PostTag.post_id == post_id, PostTag.tag_id == tag_id
            )
        )
        if link is None:
            raise HTTPException(status_code=404, detail="해당 게시글에 연결된 태그를 찾을 수 없습니다.")

        db.delete(link)
```

with db.begin은 정상 종료 시 flush·commit, 예외 시 rollback한다. 조회가 자동 시작한 트랜잭션과 begin이 충돌하지 않도록 이 코드의 관련 조회도 블록 안에 둔다. 삭제 시 Tag가 아니라 PostTag 연결 객체를 지우므로 태그와 다른 게시글의 연결은 유지된다. 게시글 삭제 cascade도 연결 행만 제거하고 공유 태그는 남긴다.

## 주제 6: association_proxy는 연결 객체 접근의 지름길

원래는 post.post_tags[0].tag로 태그 객체를 얻는다. association_proxy는 post_tags 목록의 각 연결 객체에서 tag 속성을 대신 꺼내 post.tags로 제공한다. 연결 테이블을 제거하거나 새 DB 컬럼을 추가하지 않는다.

실습의 첫 인자 post_tags는 현재 모델의 연결 목록 속성, 두 번째 tag는 연결 객체의 속성 이름이다. creator의 lambda는 새 태그를 추가할 때 PostTag(tag=tag)를 만드는 함수이다. 조회 시 creator를 호출하는 것은 아니다. post.tags.append(tag)는 새 연결 객체를 만들며 cascade가 저장 대상으로 포함시킨다.

> ➕ **더 알아두기**
> 사용자가 처음 본 기능이라고 질문한 부분이다. 어렵게 외우기 전에 ‘Post→PostTag→Tag의 중간 접근을 대신 해주는 통로’라고 이해한다. 별도 proxy 없이 연결 객체를 직접 만들어도 같은 DB 관계를 저장할 수 있다. 기존 ORM 관계를 활용하는 편의 기능이지 새로운 SQL 관계 종류는 아니다.

메모리 DB 테스트에서는 태그 이름 중복·길이 검증, 기존 태그 공유·중복 연결 거부, 없는 게시글에 연결 시 새 태그가 남지 않음, 연결 해제 후 다른 게시글의 태그 유지, 부모 삭제 후 공유 태그 보존, 일괄 생성의 중복 이름 제거를 확인했다. PostgreSQL의 실제 동시 요청 충돌이나 부하 성능까지 검증한 것은 아니다.

## 주제 7: 강의의 상품 리뷰·기획전 실습 요구사항

아래는 이번 요청으로 원문을 읽은 확장 과제이며 현재 게시판 코드에 구현한 API가 아니다.

### 상품 리뷰 1:N

| Review 필드 | 조건 |
|---|---|
| id | 자동 생성 정수 기본키 |
| content | 1자 이상 필수 |
| rating | 정수 1~5 필수 |
| product_id | products.id 외래키, 필수 |

| 메서드·경로 | 작업 | 정상 코드 |
|---|---|---|
| POST /products/{product_id}/reviews | content·rating으로 리뷰 생성, id·content·rating 응답 | 201 |
| PUT /products/{product_id}/reviews/{review_id} | 내용·평점 수정 후 같은 형식 응답 | 200 |
| DELETE /products/{product_id}/reviews/{review_id} | 리뷰만 삭제, 본문 없음 | 204 |
| GET /products/{product_id} | 상세의 reviews에 id·content·rating 포함 | 200 |
| DELETE /products/{product_id} | 상품과 그 리뷰 함께 삭제 | 204 |

상품·리뷰가 없거나 URL 상품과 리뷰 소속이 다르면 404, 내용·평점 검증 실패는 422, 리뷰가 없으면 빈 목록이다.

### 상품·기획전 M:N

Collection은 collections 테이블의 id 자동 생성 기본키와 name 1~50자를 가진다. ProductCollection은 product_collections 테이블의 id·product_id·collection_id로 구성한다. 두 외래키는 products.id·collections.id를 참조하고 필수이며 조합은 고유하다.

| 메서드·경로 | 작업 | 정상 코드 |
|---|---|---|
| POST /collections | name으로 생성, id·name 응답 | 201 |
| GET /collections | id·name 목록 | 200 |
| GET /collections/{collection_id} | id·name·products, 각 상품 id·name·final_price·category | 200 |
| POST /collections/{collection_id}/products | product_id로 기존 상품 연결, 연결 id·두 FK 응답 | 201 |
| DELETE /collections/{collection_id}/products/{product_id} | 연결만 제거, 상품과 다른 연결 유지 | 204 |

없는 상품·기획전·연결은 404, 동일 기획전의 동일 상품 중복은 400, 이름 길이 검증 실패는 422이다. 상품이 없으면 products는 빈 목록이다.

## 정리

외래키는 DB의 번호 관계, relationship은 Python 객체의 접근 관계, back_populates는 양쪽 속성의 짝, association_proxy는 연결 객체 접근을 줄이는 편의 기능이다. DTO는 입출력 형식이고 ORM 모델은 저장 매핑이다. 공유 데이터 자체와 연결 행의 삭제 범위를 구분한다.

## ✅ 확인 질문

1. 같은 content 값을 가져도 DTO와 ORM 객체가 다른 이유는 무엇인가?
2. Mapped와 mapped_column 각각의 역할과 실제 값이 들어오는 시점은 무엇인가?
3. 외래키만 있어도 가능한 작업과 relationship이 추가로 제공하는 편의는 무엇인가?
4. 외래키 경로가 여러 개일 때 SQLAlchemy가 임의로 선택하면 안 되는 이유와 지정 방법은 무엇인가?
5. Comment(content=data.content, post=post)의 왼쪽 post와 오른쪽 post는 무엇인가?
6. 댓글 소속 검사와 사용자 권한 검사, ORM cascade와 DB ON DELETE CASCADE는 어떻게 다른가?
7. PostTag의 id와 post_id·tag_id는 각각 무엇을 식별하는가?
8. 조합 고유 제약이 중복 제거 작업이 아니라는 것을 저장 시 동작으로 설명할 수 있는가?
9. 여러 게시글의 Python 태그를 재사용할 때 테이블 세 개에 어떤 행들이 필요한가?
10. 태그 연결 해제·게시글 삭제·태그 삭제의 범위를 각각 설명할 수 있는가?
11. association_proxy의 post_tags·tag·creator 인자는 각각 어떤 대상을 의미하는가?
12. post.tags.append(tag)는 왜 연결 객체를 생성하고, 별도 연결 저장을 줄일 수 있는가?
13. 상품 리뷰에서 다른 상품의 리뷰 수정을 막으려면 어떤 값을 비교해야 하는가?
14. 기획전과 상품의 연결을 해제할 때 상품까지 삭제하면 안 되는 이유는 무엇인가?
