---
tags: [sqlalchemy, postgresql, orm, transaction, pydantic]
til: v2 2026-10-27
---

# 2026-10-27_SQLAlchemy와_PostgreSQL_게시판_실습(Engine,_Session,_CRUD,_Pydantic)
> 작성일: 2026-10-27

## 🔗 관련 글

- [레이어드 게시판 구현](2026-10-26%20(2)레이어드_게시판_구현(mysite4,_schemas,_Repository,_Service,_Router,_요청_흐름).md) — 메모리 저장소를 DB 저장소로 바꾸기 전에 배운 Router·Service·Repository의 역할이다.
- [Transaction](2026-10-07%20(3)Transaction(BEGIN,_COMMIT,_ROLLBACK,_오류_처리).md) — flush로 실행한 SQL과 commit으로 확정한 트랜잭션의 차이를 연결한다.
- [SQL 기초와 DDL](2026-10-02%20(4)SQL_기초와_DDL(명령어_분류,_트랜잭션,_제약조건).md) — 테이블 구조를 만드는 DDL과 행 데이터를 다루는 DML을 구분한다.
- [Pydantic 요청 DTO](2026-10-24%20(2)Pydantic_요청_DTO(BaseModel,_Request_Body,_422_검증,_PostCreate와_Post).md) — 요청·응답 스키마와 SQLAlchemy 모델의 역할을 비교한다.

## ❓ 아직 모르겠는 것

아래는 실제 대화에서 질문하거나 여러 번 확인한 지점이다. 대부분 대화에서 풀렸지만 복습 때 다시 설명한다. 파일 저장 여부와 실행 여부는 개념 이해와 구분한다.

- Engine은 단순히 아이디·비밀번호를 기억하는 객체인가? DB 연결을 계속 하나만 유지하는가? (주제 2)
- `Path(__file__).with_name()`의 with는 무엇이며 실제 파일 이름도 바꾸는가? (주제 2)
- `URL.create()`와 `create_engine()`은 각각 무엇을 만들며, 접속 항목들은 필수인가? (주제 2)
- `if __name__ == "__main__"`, `with`, `execute()`, `scalar_one()`은 각각 언제 무엇을 실행하는가? (주제 3)
- 왜 Post가 DeclarativeBase를 바로 상속하지 않고 중간에 Base를 만드는가? (주제 4)
- `Mapped`와 `mapped_column`은 무엇이며 ORM은 AI처럼 SQL을 추론하는가? (주제 4)
- `Base.metadata`는 폴더인가? 여기에 실제 게시글이 저장되는가? (주제 5)
- Session 없이 테이블을 이미 만들었는데 왜 Session을 추가하는가? (주제 6)
- Session은 SQL 파일·스택·힙인가? 트랜잭션 자체로 보아도 되는가? (주제 6)
- Session이 DB 전체의 DML이나 다른 사람이 바꾼 테이블 구조를 감시하는가? (주제 6)
- 요청에서 SQL로 바로 처리하면 되는데 ORM 객체를 거치는 것이 불필요한 단계 아닌가? (주제 6)
- `add()`가 metadata에 객체를 넣는가? 추가한 순서대로 모두 처리되는가? (주제 7)
- flush에서 SQL이 실행되었다면 왜 commit이 필요하며, commit은 파이썬 객체를 확정하는가? (주제 7)
- flush와 commit 전후에 게시글 번호가 달라지는가? UPDATE가 INSERT인가? (주제 7, 주제 9)
- `refresh()`는 SELECT를 다시 하는가? 왜 같은 제목을 다시 읽는가? (주제 7)
- `select(Post)`에서 Post와 실제 SQL의 posts는 무엇이 다른가? (주제 8)
- `statement`에는 문자열·조회문·조회 결과 중 무엇이 들어가는가? `scalars()`는 컬럼 하나만 반환하는가? (주제 8)
- `get()`의 인자는 정해져 있는가? 복합 기본키라면 무엇을 전달하는가? (주제 8)
- 클래스 속성에 설정 딕셔너리를 적는 것만으로 `from_attributes`가 동작하는가? False면 스키마가 무의미한가? (주제 10)
- 응답 필드와 속성 이름은 같아야 하며, `a.b.c.id` 같은 중첩 구조는 불가능한가? (주제 10)
- 삭제 실습은 안내되었지만 기록 시점에 `delete_post.py`가 비어 있었다. 저장과 실행을 완료했는가? (미해결, 주제 11)

## 주제 1: 이번 학습 범위와 전체 흐름

강의 원문은 [DB를 연동한 게시판 구현](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1448/)과 [SQLAlchemy](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1447/)이다. 강의의 DB 연결·모델·세션·CRUD 개념을 기준으로, 대화에서는 작은 파일을 직접 작성하며 원리를 확인했다. 게시판 API 전체를 완성한 것이 아니라 DB 접속, 테이블 생성, Session CRUD 학습과 API 스키마 작성까지 진행했다.

강의에서는 PostgreSQL에 게시글을 저장하므로 FastAPI 서버를 재시작해도 데이터가 남는다. 메모리 리스트에만 넣은 게시글은 서버 프로세스가 종료되면 사라지는 것과 다르다.

| 강의 API | 역할 | 성공 상태 코드 |
|---|---|---|
| POST `/posts-db` | 게시글 등록 | 201 |
| GET `/posts-db` | 게시글 목록 조회 | 200 |
| GET `/posts-db/{id}` | 게시글 상세 조회 | 200 |
| PUT `/posts-db/{id}` | 제목·내용 수정 | 200 |
| DELETE `/posts-db/{id}` | 게시글 삭제 | 204 |

이 API 목록은 강의의 구현 목표이며 현재 실습에서 모두 연결한 API 목록이 아니다. 요청이 오면 Router가 입력과 의존성을 받고, Service가 처리 규칙·존재 여부·트랜잭션을 관리하고, Repository가 실제 저장·조회 코드를 수행한다. 같은 요청의 Session은 Router → Service → Repository로 전달한다. 강의에서는 변경 작업을 Service의 `with db.begin():`으로 묶어 정상 종료하면 commit하고 예외가 나면 rollback한다.

> ➕ **더 알아두기**
> 이번 실습은 이 계층들을 한꺼번에 만들지 않고, DB 연결과 객체 관리를 분리해서 확인했다. 1:N 댓글과 M:N 태그는 이후 학습할 범위이다. 1:N은 댓글에 게시글 외래키를 두고, M:N은 게시글·태그·연결 테이블로 표현한다는 개요만 확인했으며 아직 해당 모델을 구현하지 않았다.

## 주제 2: 프로젝트 준비·환경변수·Engine

### 프로젝트와 DB 서버는 별개이다

PostgreSQL은 데이터를 저장하는 별도 프로그램이다. VS Code의 PostgreSQL 확장(database-client.com)은 그 서버에 접속하는 클라이언트이다. 확장으로 연결했다고 파이썬 프로그램에 접속 정보가 자동 전달되는 것은 아니다. 확장으로 `board_practice` 데이터베이스를 만들었다.

> ➕ **더 알아두기**
> 기존 프로젝트 안에 새 프로젝트를 만들었다가 위치 혼동을 줄이기 위해 바탕화면의 별도 `board-practice` 폴더에서 다시 시작했다. 중첩 자체가 반드시 오류라는 뜻은 아니다. 이후 터미널·파일 경로는 새 프로젝트를 기준으로 한다.

초기화와 라이브러리 추가에 사용한 명령은 다음과 같다.

```bash
uv init --app
uv add "fastapi[standard]" sqlalchemy "psycopg[binary]" python-dotenv
```

`uv init --app`은 README만 만드는 명령이 아니라 앱 프로젝트의 기본 설정을 준비한다. 생성되는 구체적인 파일 구조는 초기화 방식에 따라 확인해야 하며 `src`가 반드시 없거나 있어야 하는 것은 아니다. `pyproject.toml`에는 프로젝트와 의존성을 기록하고, `.venv`에는 실제 라이브러리를 설치한다. `uv.lock`은 재현 가능한 환경을 위해 해결된 버전·의존 관계를 기록한다.

| 라이브러리 | 역할 |
|---|---|
| FastAPI | HTTP 요청과 응답 처리 |
| SQLAlchemy | DB 연결·SQL 구성/실행·ORM 객체 관리 |
| psycopg | PostgreSQL과 실제 통신하는 DB 드라이버 |
| python-dotenv | `.env`에서 환경변수 설정 읽기 |

`.env`의 DB_HOST·DB_PORT·DB_NAME·DB_USER·DB_PASSWORD는 우리가 정한 환경변수 이름이다. 실제 계정과 비밀번호는 TIL에 기록하지 않는다. `.gitignore`에는 `.env`, `.venv/`, `__pycache__/`를 제외하도록 설정했다. 이미 Git에 추적된 파일은 ignore 규칙만으로 추적 해제되지 않는다는 점도 구분한다.

### 경로와 환경변수

```python
load_dotenv(Path(__file__).with_name(".env"))
```

`__file__`은 현재 모듈 파일 경로이고 `Path()`는 경로 객체를 만든다. `with_name(".env")`는 같은 부모 폴더를 유지하며 마지막 이름만 `.env`로 바꾼 새 경로 객체를 반환한다. 실제 파일을 생성하거나 이름을 변경하지 않는다. 메서드 이름의 with는 with 문법과 무관하다.

`load_dotenv()`가 설정을 불러온 뒤 `os.environ["DB_USER"]`처럼 환경변수 이름으로 값을 꺼낸다. 대괄호 접근은 해당 키가 없으면 오류가 나므로 필요한 항목을 설정해야 한다. 기본적으로 이미 존재하는 환경변수는 `load_dotenv()`가 덮어쓰지 않는다.

```python
database_url = URL.create(
    "postgresql+psycopg",
    username=os.environ["DB_USER"],
    password=os.environ["DB_PASSWORD"],
    host=os.environ["DB_HOST"],
    port=int(os.environ["DB_PORT"]),
    database=os.environ["DB_NAME"],
)
```

`URL.create()`는 접속 정보를 담은 URL 객체를 만든다. `postgresql+psycopg`는 DB 종류와 드라이버를 구분하며 `+`는 덧셈이 아니다. `username` 등은 함수가 정한 매개변수 이름이고 `DB_USER` 등은 우리가 정한 설정 이름이다. URL 생성에 필수인 `drivername` 외 항목은 생략 가능하지만, 생략한 정보는 실제 접속 환경에서 해결되어야 한다. 함수 호출이 가능하다는 것과 DB 인증·접속 성공은 다르다.

### Engine과 연결 풀

```python
engine = create_engine(database_url)
```

Engine은 접속 정보, DB별 SQL 처리, 연결 풀을 바탕으로 DB 작업에 필요한 연결을 제공한다. 매 작업마다 개발자가 계정과 비밀번호를 다시 적지 않아도 된다. 연결 풀은 연결을 보관·재사용해 반복 접속 비용을 줄인다. Engine 하나가 연결 하나를 영원히 붙잡는다는 뜻은 아니다.

> ♻️ 처음엔 Engine을 만드는 순간 접속까지 성공하는 것으로 알았으나, 일반적으로 실제 연결은 첫 작업이 필요할 때 만들어지며 잘못된 계정 정보도 그때 오류가 날 수 있다.

## 주제 3: DB 연결 확인 코드와 수명

연결 확인 코드에서 사용한 문법은 다음과 같다.

```python
if __name__ == "__main__":
    with engine.connect() as connection:
        database_name = connection.execute(
            text("SELECT current_database()")
        ).scalar_one()
        print("연결된 DB:", database_name)
```

파일을 직접 실행하면 `__name__`이 `"__main__"`이므로 연결 확인을 실행한다. 다른 파일이 database 모듈을 import할 때는 이 블록을 실행하지 않는다. if 위의 설정 읽기와 Engine 생성은 import할 때도 실행된다.

`engine.connect()`는 실제 연결을 확보하고 `as connection`은 그 객체에 사용할 변수 이름을 붙인다. with 블록을 벗어나면 오류 여부와 무관하게 연결을 정리하며 일반적으로 풀에 반환한다. 이 with 자체가 자동 commit을 뜻하지는 않는다.

`text()`는 직접 적은 SQL 문자열을 SQLAlchemy가 실행할 SQL 구문 객체로 감싼다. `execute()`가 SQL을 실행하고 결과 객체를 반환한다. `scalar_one()`은 결과가 정확히 한 행인지 검사하고 첫 번째 컬럼 값을 반환한다. 여기서는 `board_practice`가 출력되어 연결·실행·결과 수신을 확인했다.

> ➕ **더 알아두기**
> FastAPI 서버는 종료할 때까지 요청을 기다리는 장기 실행 프로그램이다. Engine·Base.metadata 같은 객체는 프로세스에 남지만 요청용 Session은 요청마다 준비하고 닫는 방식으로 사용한다. 현재 연결 확인·테이블 생성 스크립트는 실행 후 종료된다. 파이썬 객체가 사라져도 PostgreSQL에 생성·확정한 테이블과 데이터는 남는다. 개발 모드의 재시작에서는 객체가 새로 구성된다.

## 주제 4: ORM·Post 모델·Base 상속

ORM(Object-Relational Mapping, 객체-관계 매핑)은 파이썬 클래스·객체와 관계형 DB 테이블·행을 연결하는 방식이다. 매핑은 서로 대응시킨다는 뜻이다.

| 파이썬 | PostgreSQL |
|---|---|
| Post 클래스 | posts 테이블 |
| Post 객체 하나 | 게시글 한 행 |
| post.title 속성 | 해당 행의 title 컬럼 값 |

```python
from sqlalchemy import String, Text
from sqlalchemy.orm import Mapped, mapped_column

from database import Base


class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(50), nullable=False)
    content: Mapped[str] = mapped_column(Text, nullable=False)
```

`Mapped[str]`은 ORM으로 관리하는 속성의 파이썬 타입을 표현한다. `mapped_column()`은 대응할 DB 컬럼을 정의한다. ORM 자체가 아니라 ORM 모델을 정의할 때 사용하는 함수이다. `String(50)`은 최대 50자의 문자열, `Text`는 긴 텍스트, `nullable=False`는 NULL 금지이다. 빈 문자열 금지와 NULL 금지는 서로 다르다. id는 정수 기본키이며 이 구성에서는 PostgreSQL이 새 번호를 자동 생성한다.

`Post`와 `posts`는 자동으로 단수·복수 변환된 이름이 아니다. 개발자가 `__tablename__ = "posts"`라고 연결했다. 모델은 단순한 조회 조건이 아니라 저장 구조·컬럼·제약·객체 매핑의 정의이다.

> ♻️ 처음엔 SQLAlchemy가 AI처럼 JSON의 의미를 보고 SQL을 추론한다고 생각했으나, 모델에 정해둔 테이블·컬럼 정보와 코드가 지정한 객체 상태에 따라 SQL을 구성하는 라이브러리이다. 제목에 명령 문장이 있어도 데이터로 취급한다. 실행 시 입력값은 SQL 명령과 구분해 바인딩한다.

### DeclarativeBase → Base → Post

```python
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass
```

이 선언형 방식에서 DeclarativeBase를 직접 상속하는 클래스는 프로젝트의 기반을 구성하고, 그 기반을 상속하는 Post가 테이블 모델이 된다. 여러 모델이 같은 Base와 metadata·registry를 공유할 수 있다. Base를 database.py에 둬야만 하는 것은 아니며 파일 위치는 구성상의 선택이다. pass는 추가 본문이 없다는 뜻이며 부모의 기능을 없애지 않는다.

> ➕ **더 알아두기**
> 내부 소스에서 직접 확인한 `__init_subclass__()`는 하위 클래스가 정의될 때 호출되는 훅이다. 직접 기반이면 `_setup_declarative_base()`를 호출하고, 그 다음 모델이면 선언형 매핑을 구성하는 분기를 탄다. 마지막 `super().__init_subclass__(**kw)`는 상속 순서의 다음 처리로 이어간다. 클래스의 긴 삼중 따옴표 부분은 설명문서이고 그 안의 예제는 실행 코드가 아니다. 들여쓰기가 끝난 다음 함수부터는 클래스 밖이다.
>
> 선언부의 Inspectable 관련 표현은 타입 검사 도구 관련 선언이고 `_DeclarativeTyping`도 내부 타입 표현을 돕는다. `metaclass=...`는 일반 부모 클래스 항목이 아니라 클래스 생성·속성 설정에 관여하는 메타클래스 설정이다. 상속한 기능이 자식 파일에 복사되어 보이는 것은 아니며 부모를 통해 사용할 수 있다. Ctrl+클릭 탐색 실패만으로 기능이 없다고 판단하지 않는다.

## 주제 5: Base.metadata와 테이블 생성

`Base.metadata`는 폴더·SQL 파일·게시글 저장소가 아니라 메모리에 있는 MetaData 객체이다. 모델에서 정의한 테이블·컬럼·제약을 나타내는 Table 객체들을 모은다. `Base.metadata.tables["posts"]`는 posts의 테이블 정의를 가리킨다. 실제 게시글 행 데이터는 PostgreSQL에 있다.

> ♻️ 처음엔 `add(post)`가 metadata에 게시글을 추가한다고 생각했으나, metadata에는 구조 정보가 등록되고 실제 Post 객체는 Session의 관리 대상이 된다.

```python
from database import Base, engine
import models

Base.metadata.create_all(bind=engine)

print("테이블 생성 완료")
```

위 코드는 대화에서 안내한 테이블 생성 예제이다. models를 import하면 Post 정의가 실행되어 테이블 정보가 등록된다. `create_all()`은 등록된 정의를 사용해 Engine이 연결하는 DB에 없는 테이블을 만든다. 이미 존재하는 테이블의 컬럼을 변경하는 마이그레이션 도구는 아니다.

models 이름을 뒤에서 직접 사용하지 않아도 등록이라는 부수 효과가 필요하다. Ruff의 F401 경고만 보고 import를 지우면 안 된다. 대화에서는 `# noqa: F401`로 의도적인 import임을 표시했다. Pylance의 미사용 표시는 별도 도구의 진단이므로 남을 수 있다.

> ➕ **더 알아두기**
> metadata와 모델 정보는 테이블 생성에만 쓰고 버리는 것이 아니다. 이후 SQL 구성과 매핑에도 사용된다. 테이블 생성 스크립트가 종료되면 그 실행의 메모리 객체가 사라지는 것과 실제 DB 구조가 유지되는 것을 구분한다. 강의는 FastAPI lifespan에서 시작 시 create_all, 종료 시 engine.dispose를 호출한다. 실습은 이 동작을 독립 스크립트로 먼저 확인했다.

실제 DB를 읽기 전용으로 확인했을 때 데이터베이스는 board_practice, 테이블은 posts였다. id는 INTEGER·기본키·자동 번호 생성, title은 VARCHAR(50)·NULL 금지, content는 TEXT·NULL 금지였고 당시 행 수는 0이었다. 이 수치는 확인 당시 상태이며 이후 등록·삭제 실습 뒤의 현재 행 수를 뜻하지 않는다.

## 주제 6: Session은 무엇이고 왜 사용하는가

Session은 SQL 문장·파일도 아니고 트랜잭션 자체도 아니다. ORM 객체의 새 등록·변경·삭제 상태를 관리하고 DB 작업·트랜잭션을 진행하는 객체이다. Session 하나에서 Post 여러 개와 Comment·Tag 같은 서로 다른 모델 객체들을 함께 관리할 수 있다.

```python
from sqlalchemy.orm import DeclarativeBase, sessionmaker
```

```python
SessionLocal = sessionmaker(bind=engine)
```

sessionmaker는 Session을 만드는 도구를 구성하고 bind는 사용할 Engine을 지정한다. SessionLocal은 우리가 붙인 이름이다. 이 줄만으로 실제 작업용 Session 하나를 만든 것은 아니다. `SessionLocal()`을 호출해야 실제 Session을 얻는다. 다른 파일이 `from database import SessionLocal`로 가져오려면 database.py에 정의하고 저장해야 한다. 테이블 생성에서는 필요 없었지만 Session 실습을 시작하며 추가한 코드이다.

### Session 없이도 SQL을 실행할 수 있다

Engine 연결에서 SQL을 직접 실행하거나 create_all로 테이블을 생성할 때는 Session이 필수가 아니다. SQLAlchemy에는 ORM 외에 연결·SQL 표현·실행 기능도 있다.

> ♻️ 처음 설명에서 Session이 있어야 SQL을 실행할 수 있다는 인상을 받았으나, 실제로는 Session 없이도 SQL을 실행한다. Session은 ORM 객체의 상태를 관리하는 편의를 제공한다.

직접 SQL 방식은 SELECT 결과를 받고 개발자가 UPDATE할 컬럼·값과 WHERE 조건을 명시해 실행한다. 파이썬 변수나 직접 만든 객체의 값을 바꿔도 자동으로 DB가 수정되지 않는다. Session으로 조회한 객체는 변경을 추적하므로 객체 속성을 수정하면 flush 때 필요한 SQL을 구성한다. 조회한 관리 객체를 수정 후 add로 다시 등록할 필요는 없다.

> ➕ **더 알아두기**
> ORM은 객체 생성·변경 추적이라는 중간 계층을 추가하므로 무조건 빠르거나 간단한 것은 아니다. 단순 작업이나 복잡한 조회는 직접 SQL이 더 명확할 수 있다. 반면 반복적인 객체 변환·수정·관계 저장 코드를 줄일 수 있다. 프로젝트에 맞게 선택하며 이번 학습은 ORM 방식을 익히기 위해 사용한다. FastAPI가 요청을 해석하고 애플리케이션 코드가 작업을 지정하는 것이지 Session이 REST 요청이나 JSON 의미를 자동 해석하지는 않는다.

### Session과 트랜잭션의 범위

트랜잭션은 DB 작업을 함께 확정하거나 취소하는 단위이다. Session은 그 트랜잭션을 사용하면서 객체 상태도 관리한다. 짧은 요청에서 Session 하나·트랜잭션 하나의 범위를 맞출 수 있으나, 같은 Session으로 commit 후 다음 트랜잭션을 시작할 수도 있다. Session을 만들자마자 반드시 실제 DB 트랜잭션이 시작되는 것은 아니다. 기본 자동 시작 방식에서 DB 접근이 필요해지면 연결·트랜잭션을 사용한다.

SELECT도 지금 방식에서는 트랜잭션 안에서 실행된다. 조회만 했다면 commit으로 확정할 데이터 변경이 없으며 Session을 닫을 때 열린 트랜잭션을 rollback해 정리할 수 있다. `with SessionLocal() as db:`는 닫기를 보장하는 문법이지 자동 commit 설정은 아니다.

> ♻️ 처음엔 Session을 트랜잭션과 동일한 것으로 보았으나, 특정 코드에서 수명이 비슷할 수 있을 뿐 객체 관리자와 DB 작업 단위는 구분한다. 또한 Session은 DB 전체를 감시하지 않으며 다른 연결의 데이터 변경이나 ALTER TABLE을 자동 통보받는 기능도 아니다.

DDL은 구조 생성·변경이고 DML은 행 삽입·수정·삭제 등을 다룬다. Session의 변경 추적은 관리 중인 파이썬 객체의 변화에 관한 기능이다. DDL이 의미 없어서 없는 것이 아니라 create_all 등 별도 도구가 담당한다. Session으로 직접 SQL을 실행할 수 있다는 사실과 테이블 구조 자동 추적은 다르다.

## 주제 7: add·flush·commit·rollback·refresh

### 등록 실습 코드

```python
from database import SessionLocal
from models import Post

with SessionLocal() as db:
    post = Post(title="첫 게시글", content="Session으로 저장합니다.")

    db.add(post)
    db.flush()

    print("flush 후 게시글 번호:", post.id)

    db.commit()
    db.refresh(post)

    print("DB에서 다시 읽은 제목:", post.title)
```

Post 객체를 생성하면 title·content가 객체 속성에 들어간다. add는 해당 객체를 Session에 새 저장 대상으로 등록한다. 이 새 객체에 대해 add 자체가 즉시 INSERT를 실행하는 것은 아니며 객체별 INSERT 문자열을 미리 보관하는 것도 아니다. flush 때 모델 매핑과 객체 상태·값을 바탕으로 SQL을 구성해 실행한다.

| 단계 | 객체·DB에서 일어나는 일 |
|---|---|
| 객체 생성 | 제목·내용이 파이썬 객체에 들어가고 자동 생성 id는 아직 없다 |
| add | Session에 새 저장 대상으로 등록 |
| flush | INSERT 실행, DB가 생성한 id를 객체에 반영 |
| commit | 필요한 flush를 수행한 뒤 DB 트랜잭션 확정 |
| refresh | SELECT로 DB 값을 다시 읽어 기존 객체 갱신 |
| close | 객체 관리 관계와 DB 연결 자원 정리 |

### SQL 실행과 확정은 다르다

flush는 변경을 DB로 흘려보내 실행한다는 의미이다. commit은 DB 트랜잭션 확정이다. Python 메서드이지만 드라이버를 통해 실제 DB commit을 요청하며 파이썬 객체만 확정하는 것이 아니다.

flush한 INSERT·UPDATE·DELETE는 현재 트랜잭션 안에서 실행되어 조회할 수 있지만 아직 rollback으로 취소 가능하다. 예를 들어 게시글 INSERT로 번호를 받은 뒤 댓글 저장이 실패하면 두 데이터 작업을 함께 취소할 수 있다. 다른 연결은 일반적으로 미확정 변경을 볼 수 없다. PostgreSQL의 시퀀스 번호 소비는 rollback으로 되돌아가지 않는다는 예외를 데이터 행 취소와 구분한다.

commit 후에는 이전 트랜잭션을 rollback으로 취소할 수 없다. 다시 UPDATE·DELETE하는 것은 새 트랜잭션이다. commit은 먼저 필요한 flush를 하므로 매번 flush를 따로 쓸 필요는 없다. 이번 등록 예제에서는 commit 전 id를 출력하려고 명시했다.

> ♻️ 처음엔 flush·commit 전후로 번호가 바뀌는 것으로 받아들였으나, 첫 번호가 1이라면 flush 후 1, commit 전 1, commit 후에도 1이다. 번호는 새 INSERT 때 생성되며 제목 UPDATE 때문에 바뀌지 않는다. 첫 삽입 시도·rollback·삭제 등의 이력이 있으면 반드시 1부터 출력되는 것은 아니다.

제목도 객체 생성 때 이미 정해져 있다. flush는 그 값을 DB에 실행하고 commit은 확정할 뿐 제목을 새 값으로 바꾸는 단계가 아니다. refresh 역시 DB에서 같은 제목을 읽으면 화면상 값이 바뀌지 않는다.

### refresh가 필요한 이유와 한계

refresh는 SELECT를 다시 보내 기존 객체 속성을 DB 값으로 갱신한다. DB 기본값·트리거로 설정한 값이나 다시 읽어야 할 값을 가져오는 데 사용한다. 이번에는 저장 후 재조회 동작을 확인하는 학습용 단계이며 저장에 필수는 아니다. id는 flush 때 이미 받았다. 관계 로딩에 따라 정확히 SELECT 한 번만 발생한다고 일반화하지 않는다.

> ➕ **더 알아두기**
> 기본 Session 설정은 commit 후 객체 값을 만료시킨다. 그 객체의 속성에 다시 접근하면 SELECT가 자동 발생할 수 있다. 따라서 refresh를 없애도 뒤의 출력에서 조회가 일어날 수 있다. refresh는 해당 속성의 미반영 변경을 버릴 수 있으므로 로컬 수정과 DB 재조회가 필요한 시점을 구분해야 한다. 이 메서드는 무조건 commit 뒤에 붙이는 의식 같은 절차가 아니다.

### Session은 스택이나 힙인가

Session은 LIFO 스택이나 우선순위 힙으로 처리 순서를 정의하는 자료구조가 아니다. 단순한 새 객체들은 보통 추가 순서로 INSERT되지만 관계 의존성·작업 종류에 따라 순서가 달라질 수 있다. 게시글 번호가 필요한 댓글은 게시글 저장이 먼저 필요하다. 추가 순서를 모든 작업의 보장으로 사용하지 않는다.

flush는 가장 먼저 추가한 하나를 꺼내 버리는 것이 아니라 당시 미반영 변경들을 처리한다. flush 후 새 저장 대기 상태는 끝나도 객체는 관리 중이므로 이후 속성 변경을 다시 추적할 수 있다. commit도 Session 자체를 삭제하지 않는다. close 후에도 변수가 참조하면 파이썬 객체는 남지만 Session과 분리된다.

> 📝 **기사 시험 포인트** (정보처리기사)
> 트랜잭션(Transaction)의 원자성(Atomicity)은 묶인 작업이 전부 반영되거나 전부 취소되는 성질이고, 지속성(Durability)은 성공적으로 확정한 결과가 유지되는 성질이다. COMMIT·ROLLBACK의 구분과 연결해 설명한다. flush는 SQLAlchemy의 ORM 작업 개념이며 ACID의 별도 속성이 아니다. 특정 문제의 출제 빈도는 단정하지 않는다.

## 주제 8: select·statement·scalars·get

### read_posts.py 전체

```python
from sqlalchemy import select

from database import SessionLocal
from models import Post

with SessionLocal() as db:
    statement = select(Post).order_by(Post.id)
    posts = db.scalars(statement).all()

    for post in posts:
        print(post.id, post.title, post.content)
```

select는 SQLAlchemy의 조회문 구성 함수이다. Post는 클래스이며 클래스에 정의한 posts가 SQL의 테이블 이름이다. 모델 전체를 지정했으므로 모델에 매핑된 id·title·content 컬럼들을 조회한다. 실제 DB에만 있고 모델에 매핑하지 않은 컬럼까지 자동으로 전부 읽는 것은 아니다. `order_by(Post.id)`는 id 오름차순 정렬을 지정한다.

statement는 개발자가 정한 변수 이름으로 Select 객체를 담는다. SQL 문자열이나 조회된 게시글 데이터 자체가 아니다. 실행 때 DB에 맞는 SQL로 컴파일한다. `print(statement)`로 SQL 표현을 볼 수 있지만 그것이 곧 실제 실행 로그는 아니다.

`db.scalars(statement)`에서 조회를 실행한다. scalars는 각 결과 행의 첫 번째 항목을 꺼낸다. select(Post)에서는 그 항목이 id 하나가 아니라 Post 엔티티이므로 Post 객체를 받는다. all은 객체들을 전부 리스트로 만든다. 결과가 없으면 빈 리스트이다. for는 객체들을 하나씩 꺼내 출력한다.

> ➕ **더 알아두기**
> `select(Post.title)`이면 scalar 항목은 제목 문자열이다. `select(Post.title, Post.content)`처럼 여러 컬럼을 조회하면서 scalars를 쓰면 첫 번째 항목만 꺼낸다. 모든 컬럼을 받으려면 행 결과를 다루는 execute 등의 방식을 사용한다. posts는 결과 리스트 변수 이름이고 실제 테이블 이름과 문법적으로 연결되는 것은 아니다.

### 기본키 조회 get

```python
post = db.get(Post, 1)
```

첫 번째 인자는 ORM 모델 클래스(entity), 두 번째는 기본키 식별값(ident)이다. Post에서는 id를 기본키로 정의했으므로 1번 게시글을 찾는다. 찾으면 Post 객체, 없으면 None이다. Session에 유효한 객체가 이미 있으면 DB SELECT 없이 반환할 수 있다. db.get은 조회까지 수행하지만 select는 실행할 구문을 구성한다는 차이가 있다.

조건을 명시해 비슷한 결과를 얻는 코드는 다음과 같다.

```python
statement = select(Post).where(Post.id == 1)
post = db.scalars(statement).one_or_none()
```

전체를 정렬해 all로 받는 코드는 리스트이고, 이 코드는 하나 또는 None이므로 다르다. one_or_none은 결과가 여러 개면 오류이다. 기본키 조건에서는 DB 제약 때문에 여러 행이 나오지 않는다. `.first()`는 첫 결과 또는 None, `.one()`은 정확히 하나가 아니면 오류, `.all()`은 전체 리스트라는 강의의 반환 차이도 구분한다.

> ➕ **더 알아두기**
> 복합 기본키는 get의 두 번째 인자에 튜플이나 딕셔너리를 전달한다. 예를 들어 두 컬럼이 실제 기본키인 PostTag 모델이라면 `(1, 3)`은 매핑된 기본키 순서를 따르고, `{"post_id": 1, "tag_id": 3}`은 속성 이름을 명시한다. 아직 실제 복합키 모델을 구현한 것은 아니다.

## 주제 9: UPDATE·DELETE 실습

### update_post.py

```python
from database import SessionLocal
from models import Post

with SessionLocal() as db:
    post = db.get(Post, 1)

    if post is None:
        raise ValueError("해당 게시글이 없습니다.")

    print("수정 전:", post.title)

    post.title = "제목을 수정했습니다."

    db.commit()

    print("수정 완료")
```

1은 실제 수정할 실습 게시글 번호로 바꾼다. None 검사는 없는 객체의 title에 접근하는 것을 막고 오류로 중단한다. Session이 조회한 객체의 title 변경을 추적하므로 add를 다시 호출하지 않는다. commit이 flush를 수행하며 UPDATE를 실행하고 확정한다. 제목만 바꿨으므로 id는 그대로이다.

### delete_post.py: 대화에서 안내한 예제

```python
from database import SessionLocal
from models import Post

with SessionLocal() as db:
    post = db.get(Post, 1)

    if post is None:
        raise ValueError("해당 게시글이 없습니다.")

    db.delete(post)
    db.commit()

    print("삭제 완료")
```

delete는 객체를 삭제 대상으로 지정하고 commit의 flush 과정에서 DELETE가 실행된다. SQLAlchemy 객체를 이용하는 코드이지만 실제 DB 행을 삭제한다. 파일을 다시 실행해도 같은 id가 계속 존재하는 것은 아니므로 이미 삭제했다면 None 검사를 통과하지 못한다. 별도 read_posts 실행으로 결과를 확인한다. 삭제가 끝난 테이블에 행이 없으면 조회 반복문은 출력하지 않는다.

> ♻️ 처음엔 UPDATE와 INSERT를 같은 것으로 혼동했으나, INSERT는 새 행 생성, UPDATE는 기존 행 수정, DELETE는 행 삭제이다. 수정·삭제에도 flush는 필요하지만 commit이 수행하므로 명시 호출이 없는 것이다.

## 주제 10: Pydantic 스키마와 from_attributes

```python
from pydantic import BaseModel, ConfigDict, Field


class PostCreate(BaseModel):
    title: str = Field(min_length=1, max_length=50)
    content: str = Field(min_length=1)


class PostResponse(BaseModel):
    id: int
    title: str
    content: str

    model_config = ConfigDict(from_attributes=True)
```

PostCreate는 요청 제목·내용의 형식과 길이를 검증한다. 사용자가 id를 생성하는 것이 아니므로 요청 스키마에 넣지 않았다. PostResponse는 응답에 필요한 번호·제목·내용의 형식이다. SQLAlchemy 모델은 저장 구조·테이블 매핑, Pydantic 스키마는 주고받을 데이터의 검증·응답 형식을 담당한다.

### 클래스 변수 설정만으로 왜 동작하는가

`model_config`는 Pydantic이 읽는 정해진 클래스 속성 이름이다. ConfigDict는 설정용 딕셔너리를 구성한다. BaseModel 쪽 구현이 모델을 구성할 때 이 설정을 해석하기 때문에 동작하며 파이썬 자체가 임의 변수 이름을 알아보는 것은 아니다. 이 설정을 작성하는 순간 DB 객체를 조회하는 것도 아니다.

from_attributes=True이면 딕셔너리의 키뿐 아니라 입력 객체의 id·title·content 속성에서도 데이터를 읽어 모델로 검증할 수 있다. False여도 딕셔너리나 해당 모델 인스턴스 같은 지원 입력은 처리하므로 필드 정의가 무의미해지지 않는다. 일반 SQLAlchemy 객체의 속성을 직접 읽도록 허용하지 않는다는 차이이다. 객체를 딕셔너리로 명시 변환하는 대안도 있다.

### 필드 이름·중첩 구조

기본 설정에서는 id 필드에 대해 입력 객체의 id를 찾는다. 별칭 등을 설정하지 않았다면 이름을 맞춰야 한다. 원본에 응답에 필요 없는 다른 속성이 더 있어도 된다. from_attributes가 모든 깊은 경로를 자동 검색하지는 않는다.

> ♻️ 처음엔 점 하나로 접근하는 속성만 가능하고 `a.b.c.id` 같은 중첩 구조는 불가능하다고 생각했으나, 중첩 모델이나 경로 설정을 명시해 처리할 수 있다. id만 선언했다고 임의의 깊이에 있는 id를 자동 탐색하는 것은 아니다. 나중에 게시글 응답 안의 댓글 목록을 중첩 응답 모델로 구성할 수 있다.

## 주제 11: 실제 확인 상태와 다음 실습

| 항목 | 기록 시점에 확인한 내용 |
|---|---|
| DB 연결·테이블 | 읽기 전용 조회로 board_practice와 posts 구조를 확인했다 |
| database.py | 최종 저장본에 sessionmaker import와 SessionLocal 정의가 추가된 것을 확인했다 |
| models.py | Post 모델과 세 컬럼 정의를 확인했다 |
| create_tables.py | models 등록과 create_all 코드가 있다 |
| practice_session.py | add·flush·commit·refresh 예제가 작성되어 있다 |
| read_posts.py | select·scalars·all·출력 예제가 작성되어 있다 |
| update_post.py | 최신 파일이 저장되어 있으며 수정 예제를 학습했다. 이 기록 작성 과정에서 재실행하지 않았다 |
| delete_post.py | 삭제 예제는 대화에서 설명했지만 확인 당시 파일이 비어 있어 실행 완료로 기록하지 않는다 |
| schemas.py | 사용자가 저장 후 PostCreate·PostResponse와 from_attributes 설정을 확인했다 |
| FastAPI 연결 | DB CRUD를 HTTP 엔드포인트·의존성·계층에 연결하는 코드는 아직 작성하지 않았다 |

> ➕ **더 알아두기**
> 에디터에서 입력한 코드와 디스크에 저장된 코드는 다를 수 있다. 처음 확인 때 schemas.py가 비어 있었고 SessionLocal 정의도 없었으나, 사용자 확인 후 저장된 파일에서 내용을 다시 읽었다. TIL은 작성한 코드·설명한 코드·실제 실행 확인을 구분한다. 실제 비밀번호·인증 정보는 기록하거나 커밋하지 않는다.

다음은 저장된 수정·삭제 스크립트를 확인하고, DB Session을 요청마다 제공하는 의존성 및 FastAPI Router를 연결하는 단계이다. 강의에서는 get_db가 Session을 만들고 yield로 전달한 뒤 finally에서 닫는다. Depends는 필요한 Session을 FastAPI가 준비해 함수로 전달하도록 지정한다. 아직 이 의존성을 실습으로 구현하지 않았으므로 완료 목록에 넣지 않는다.

## 정리

- Engine은 DB 연결을 제공·관리하고, Base.metadata는 테이블 구조 정의를 모은다.
- ORM은 객체와 테이블을 연결한다. Post는 클래스, posts는 실제 테이블 이름, post는 객체 변수이다.
- create_all은 구조를 만드는 DDL이고 Session이 SQL 실행의 필수 조건은 아니다.
- Session은 객체 상태와 DB 작업을 관리하며 트랜잭션 자체와 동일하지 않다.
- add는 새 객체 등록, flush는 SQL 실행, commit은 DB 트랜잭션 확정이다.
- refresh는 SELECT로 기존 객체를 갱신한다. 저장마다 반드시 필요하지 않다.
- select는 조회문을 구성하고 scalars·get 등 Session 실행 메서드로 결과를 받는다.
- Pydantic의 from_attributes는 객체 속성에서도 값을 읽는 설정이며 테이블 정의가 아니다.

## ✅ 확인 질문

1. 메모리 리스트 기반 게시판을 DB에 연결하면 서버 재시작 후 데이터 보존이 어떻게 달라지는가?
2. Router·Service·Repository를 거칠 때 같은 Session을 전달하는 이유는 무엇인가?
3. uv add가 pyproject.toml·.venv·uv.lock에 각각 하는 일은 무엇인가?
4. Engine 생성과 실제 DB 접속의 시점은 어떻게 다르며 연결 풀은 왜 사용하는가?
5. `Path(__file__).with_name(".env")`는 어떤 경로를 만들며 실제 파일을 변경하는가?
6. `if __name__ == "__main__"`과 연결용 with가 각각 제어하는 범위는 무엇인가?
7. ORM에서 클래스·객체·속성은 DB의 무엇에 대응하는가?
8. DeclarativeBase를 상속하는 Base와 그 Base를 상속하는 Post는 역할이 어떻게 다른가?
9. Base.metadata에는 어떤 정보가 있고 실제 게시글 값은 어디에 있는가?
10. 사용하지 않는 것처럼 보이는 import models를 테이블 생성 전에 남겨두는 이유는 무엇인가?
11. Session 없이 SQL을 실행할 수 있는데도 ORM에서 Session을 사용하는 이유는 무엇인가?
12. Session 하나가 여러 트랜잭션을 사용할 수 있다는 사실을 예로 설명할 수 있는가?
13. add·flush·commit 각각의 시점에 객체와 DB의 상태는 어떻게 다른가?
14. flush 후 id가 1인 경우 commit 전후에도 1인 이유와 refresh를 선택적으로 쓰는 이유는 무엇인가?
15. statement에는 무엇이 들어가며 select(Post)만으로 DB 조회가 실행되지 않는 이유는 무엇인가?
16. select(Post)와 select(Post.title)를 scalars로 실행했을 때 반환 항목은 어떻게 다른가?
17. get의 두 인자는 무엇이며 복합 기본키는 어떻게 지정하는가?
18. get(Post, 1)과 전체 조회 후 all로 받는 코드는 대상과 반환 형태가 어떻게 다른가?
19. 조회한 객체의 제목을 수정한 후 add 없이 commit해도 UPDATE가 실행되는 이유는 무엇인가?
20. delete와 commit의 사이에는 어떤 상태이며 이미 삭제한 id를 다시 조회하면 무엇을 받는가?
21. DB 모델과 요청·응답 스키마를 나누는 이유는 무엇인가?
22. model_config를 클래스 속성으로 적는 것만으로 from_attributes가 적용되는 이유와 False일 때 가능한 입력은 무엇인가?
