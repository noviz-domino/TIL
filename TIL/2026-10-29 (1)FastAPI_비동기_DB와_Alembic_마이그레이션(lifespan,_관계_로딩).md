---
tags: [fastapi, sqlalchemy, alembic, asyncio, lazy-loading]
til: v2 2026-10-29
---

# 2026-10-29_FastAPI_비동기_DB와_Alembic_마이그레이션(lifespan,_관계_로딩)
> 작성일: 2026-10-29

## 🔗 관련 글

- [SQLAlchemy와 PostgreSQL 게시판 실습](2026-10-27_SQLAlchemy와_PostgreSQL_게시판_실습(Engine,_Session,_CRUD,_Pydantic).md) — Engine·Session·ORM·flush·commit·응답 모델을 배운 뒤 비동기 연결과 구조 변경 이력 관리로 이어진다.
- [파이썬 비동기 기초 — 코루틴과 asyncio](2026-08-27%20(2)%20파이썬%20비동기%20기초%20-%20코루틴과%20asyncio.md) — await로 대기하는 동안 다른 작업이 실행되는 원리를 DB 조회에 적용한다.
- [관심사 분리와 레이어드 아키텍처](2026-10-26%20(1)관심사_분리와_레이어드_아키텍처(SoC,_Router_Service_Repository,_계층별_역할).md) — 설정·DB 연결·도메인 모델의 위치와 Router·Service·Repository의 역할을 연결한다.

## ❓ 아직 모르겠는 것

다음은 실제 대화에서 질문하거나 여러 번 되짚은 부분이다. 대화에서 설명과 교정을 거쳤지만 복습 때 다시 확인한다. 마지막에는 비동기 지연 로딩 기능의 존재와 자동 조회를 막는 목적을 확인했다.

- “이번에 하는 게 뭐 하는 거야?”, “마이그레이션이 뭐지?” — 이미 ORM으로 테이블과 데이터를 다뤘는데 마이그레이션은 무엇을 추가하는가? (주제 1·4 참고)
- “커밋이구나?” — 마이그레이션, Git 커밋, DB COMMIT은 같은 작업인가? (주제 4 참고)
- “Alembic이 마이그레이션하는 라이브러리야?” — SQLAlchemy·psycopg와 역할이 어떻게 다른가? (주제 2·4 참고)
- “라이프스팬은 전혀 모르겠네.” — 서버 시작·실행·종료에서 yield의 앞뒤는 언제 실행되는가? (주제 6 참고)
- “그러면 레이지로딩이 안되잖아.” — 비동기에서 관계 로딩 자체가 불가능한가, 현재 프로젝트가 자동 조회를 금지한 것인가? (주제 7·8 참고)
- “게시글은 기다려주는데 유저 정보 답변이 바로 안 왔다고 오류가 나는 거야?” — MissingGreenlet은 DB 응답 지연이나 시간 초과 오류인가? (주제 8 참고)
- “그럼 여기도 await로 하면 되지.” — 일반 속성 접근에 await를 붙이는 것과 비동기 조회 API를 사용하는 것은 무엇이 다른가? (주제 8 참고)
- “첫 번째 작업만 비동기가 가능하고 두 번째 지연 로딩은 불가능하게 짜여 있어?” — 조회 순서가 문제인가, 요청하는 인터페이스가 문제인가? (주제 8 참고)
- “속성 읽기를 비동기로 설정하면 되지, 왜 아예 막아?” — AsyncAttrs가 있는데도 lazy="raise"를 선택하는 이유는 무엇인가? (주제 8·9 참고)
- “그 기능이 이미 있고, DB 조회를 제한하려고 막은 거야?” — 조회 횟수 제한과 의도하지 않은 자동 조회 방지는 어떻게 다른가? (주제 9 참고)

## 주제 1: 오늘의 학습 범위와 프로젝트 구조

강의 원문은 [비동기 앱 초기 설정과 Alembic](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1467/)과 [비동기 CRUD](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1465/)이다. 첫 강의의 설정·초기 마이그레이션·컬럼 추가·앱 생명주기까지 진행하고, 다음 강의는 **「예시: 게시글 생성 후 작성자 정보 반환」까지** 다뤘다. 이후의 전체 비동기 CRUD 호출 구현을 완료한 것으로 기록하지 않는다.

프로젝트는 HTTP와 서비스 데이터, AI 실행 흐름, 문서 검색의 책임을 나눈다.

```text
src/
├── fastapi_agent/
│   ├── main.py
│   ├── core/
│   └── domains/
│       ├── users/
│       ├── posts/
│       ├── comments/
│       └── chat/
├── agents/
│   ├── chatbot/
│   └── rag/
└── rag/
```

`fastapi_agent`는 HTTP 요청·응답, 인증, 업무 규칙과 서비스 데이터를 담당한다. `domains`는 회원·게시글·댓글·채팅처럼 서비스의 기능 영역인 도메인(domain)을 기준으로 나눈다. Router는 요청·응답, Service는 업무 규칙, Repository는 DB 접근, models.py는 SQLAlchemy 모델, schemas.py는 Pydantic 요청·응답 모델의 자리이다.

AI 기능이 필요한 Service는 `agents`를 호출하고, Agent가 검색 기능을 필요로 하면 `rag`를 사용한다. `agents/rag`는 검색과 답변 생성을 연결하는 실행 흐름이고, `src/rag`는 문서 로딩·분할·임베딩·벡터 검색의 구현이다. 이름이 비슷하지만 책임이 다르다.

| 작업 파일 | 강의에서의 역할 |
|---|---|
| pyproject.toml | 의존성, FastAPI 실행 진입점 |
| .env | DB 접속 정보와 JWT 설정 |
| core/config.py | 환경변수 로딩·타입 변환·필수 값 검증 |
| core/database.py | 비동기 엔진·세션·공통 Base |
| domains/posts/models.py | Post에 작성 시각 컬럼 추가 |
| alembic.ini, alembic/ | 마이그레이션 설정과 변경 파일 |
| main.py | FastAPI 앱 생성과 lifespan 종료 정리 |

> ➕ **더 알아두기**
> 실제 작업 폴더는 `C:/Users/USER/Desktop/fastapi_agent`이다. 처음에는 기본 Python 패키지만 있었으며, 사용자가 알려준 `C:/Users/USER/Desktop/aim_home/260811/aim-ai-agent/fastapi/fastapi_agent`에서 설정과 User·Post·Comment·ChatRoom·ChatMessage 모델 정의를 확인했다. 이번 범위에 필요한 파일만 가져왔으며, 원본의 인증·전체 CRUD·AI·RAG 기능을 전부 복사한 것은 아니다. 실제 저장된 파일과 강의가 제시한 전체 구조를 구분한다.

## 주제 2: 실행 환경·환경변수·Pydantic Settings

### 라이브러리의 역할을 구분한다

강의는 프로젝트 폴더에서 `uv sync`로 pyproject.toml의 의존성을 설치한다. PostgreSQL 서버에는 별도로 프로젝트 DB를 만든다.

```powershell
uv sync
```

```sql
CREATE DATABASE fastapi_agent;
```

강의가 제시한 전체 의존성과 용도는 다음과 같다. 이 목록 전체를 이번 실습에서 설치·사용한 목록으로 보아서는 안 된다.

| 패키지 | 용도 |
|---|---|
| fastapi[standard] | API와 개발 서버 |
| pydantic-settings | 환경변수 읽기와 검증 |
| python-dotenv | RAG 모듈에서 .env 로딩 |
| sqlalchemy[asyncio] | 비동기 ORM·세션 |
| psycopg[binary] | PostgreSQL과 통신하는 드라이버 |
| alembic | DB 구조 변경과 이력 관리 |
| pwdlib[argon2] | 비밀번호 해싱·검증 |
| pyjwt | JWT 생성·검증 |
| langchain | LLM과 도구를 연결하는 Agent 구성 |
| langchain-core | 메시지·문서·도구·Runnable의 공통 구성 요소 |
| langchain-google-genai | Gemini와 Google 임베딩 연동 |
| langgraph | 상태·노드·분기로 실행 흐름 구성 |
| langgraph-checkpoint-postgres | PostgreSQL에 Agent 상태 저장·복원 |
| langchain-community | PyPDFLoader 등 외부 연동 구성 요소 |
| pypdf | PDF 읽기·텍스트 추출 |
| langchain-text-splitters | 문서 분할 |
| chromadb | 벡터·문서 저장과 유사도 검색 |
| langchain-chroma | LangChain과 Chroma 연동 |

> ➕ **더 알아두기**
> 현재 프로젝트가 직접 의존성으로 추가한 것은 fastapi[standard], pydantic-settings, sqlalchemy[asyncio], psycopg[binary], alembic이다. AI·RAG·인증을 실행한 것은 아니다. SQLAlchemy는 ORM·SQL 구성과 실행을 관리하고, psycopg는 PostgreSQL과 실제 통신하며, Alembic은 구조 변경을 관리한다. 라이브러리 이름이 비슷한 역할을 뜻하지 않는다.

### .env는 접속 설정을 코드에서 분리한다

강의의 .env 형식은 다음과 같다. 아래는 예시이며 실제 비밀번호·JWT 비밀 키를 학습 기록에 넣지 않는다.

```dotenv
DATABASE_URL=postgresql+psycopg://postgres:your_password@localhost:5432/fastapi_agent
JWT_SECRET_KEY=생성한_64자리_비밀_키
JWT_EXPIRE_MINUTES=3000
DB_ECHO=false
```

URL은 드라이버·계정·비밀번호·호스트·포트·DB 이름을 표현한다. `postgresql+psycopg` URL을 비동기 엔진 생성 함수에 전달하면 psycopg의 비동기 연결을 사용한다. psycopg는 동기·비동기 연결을 모두 지원한다.

강의는 `openssl rand -hex 32`로 JWT 비밀 키를 생성한다. 단순히 JWT 설정값을 준비한 것과 실제 로그인 기능을 구현한 것은 다르다.

> ➕ **더 알아두기**
> 이번 작업에서는 Python의 secrets로 무작위 키를 생성했다. `.env`는 Git에서 제외하고, 공유 가능한 자리표시자만 `.env.example`에 남겼다. DB 비밀번호를 사용자에게 전달받았더라도 공개 TIL에 다시 적지 않는다.

### 설정 클래스는 읽기뿐 아니라 검증도 한다

`os.getenv()`만 쓰면 .env 로딩, 문자열의 타입 변환, 필수 값 검사를 각각 처리해야 한다. Pydantic Settings는 선언한 타입과 Field 조건에 맞춰 설정을 읽고 검증한다. 실제 저장된 core/config.py는 다음과 같다.

```python
from functools import lru_cache

from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    database_url: str = Field(min_length=1)
    jwt_secret_key: SecretStr = Field(min_length=32)
    jwt_expire_minutes: int = Field(default=30, gt=0)
    db_echo: bool = False

    model_config = SettingsConfigDict(
        env_file=".env", env_file_encoding="utf-8", extra="ignore"
    )


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

database_url은 빈 문자열을 허용하지 않고, jwt_secret_key는 최소 32자이다. SecretStr은 일반적인 출력 표현에서 비밀값을 가리는 타입이며, 비밀값을 암호화해 저장하는 기능은 아니다. jwt_expire_minutes는 양수이고 기본값이 30이며, .env의 3000이 있으면 그 값이 적용된다. db_echo는 SQL 출력 여부이다.

`SettingsConfigDict`는 .env의 경로·인코딩과 추가 항목 처리 규칙을 지정한다. `get_settings()`의 `@lru_cache`는 설정 객체를 재사용하게 한다. 설정 항목이 많아지면 DB·인증·AI처럼 역할별로 클래스를 나눌 수도 있다.

## 주제 3: FastAPI와 비동기 DB 연결

### 비동기는 I/O 대기 시간을 활용한다

게시글 API는 DB 답을 기다리고, AI 채팅은 LLM API 답을 기다린다. 이런 입출력(Input/Output, I/O) 대기 중에도 다른 사용자의 요청을 처리하려고 비동기를 사용한다. 비동기로 바꾼다고 DB 쿼리 하나나 LLM 답변 생성 자체가 빨라지는 것은 아니다.

```text
요청 A: LLM에 요청 → 응답 대기 ───────────→ 답변 반환
요청 B:                    DB 조회 → 응답 반환
```

FastAPI의 일반 def 경로 함수도 스레드 풀에서 실행하므로 여러 요청을 처리할 수 있다. 다만 동기 I/O를 기다리는 동안 해당 스레드는 점유된다. 비동기 I/O는 대기 중 요청마다 스레드를 계속 점유하지 않아도 된다.

Router에 async def만 붙이는 것으로 끝나지 않는다. DB 엔진·세션과 외부 API 클라이언트도 비동기 API를 사용하고, 이를 호출하는 Service·Repository에서 await로 연결해야 한다. async def 안에서 동기 DB·HTTP 호출을 직접 실행하면 이벤트 루프를 막을 수 있다. FastAPI가 일반 Service·Repository 함수를 자동으로 스레드 풀에 보내는 것은 아니다.

| 동기 API | 비동기 API |
|---|---|
| create_engine() | create_async_engine() |
| sessionmaker() | async_sessionmaker() |
| Session | AsyncSession |
| def get_db() | async def get_db() |
| with SessionLocal() | async with SessionLocal() |

실제 저장된 core/database.py는 다음과 같다.

```python
from collections.abc import AsyncIterator

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase

from fastapi_agent.core.config import get_settings

settings = get_settings()
engine = create_async_engine(settings.database_url, echo=settings.db_echo)
SessionLocal = async_sessionmaker(bind=engine, expire_on_commit=False)


class Base(DeclarativeBase):
    pass


async def get_db() -> AsyncIterator[AsyncSession]:
    async with SessionLocal() as db:
        yield db
```

엔진은 연결 풀과 DB 접근의 공용 자원을 관리하고, SessionLocal은 요청에서 사용할 비동기 세션을 만드는 팩터리이다. 공통 Base를 상속하는 모델의 테이블 정보는 Base.metadata에 모인다. get_db는 세션을 열고 yield로 사용하게 한 뒤 정리한다.

### expire_on_commit=False의 의미

기본값 True에서는 commit 뒤 ORM 객체의 속성을 만료시킨다. 만료된 속성을 읽을 때 자동 재조회가 필요할 수 있는데, Pydantic이 응답을 만들며 속성을 읽는 과정에서는 이 비동기 조회를 await로 기다릴 수 없다. False로 두면 이미 읽은 값은 commit 후에도 유지해서 응답에 사용할 수 있다.

> ➕ **더 알아두기**
> False가 모든 관계를 미리 불러온다는 뜻은 아니다. 기존에 읽은 값의 만료를 막는 설정과, 아직 읽지 않은 post.user를 새로 로딩하는 작업은 별개이다. 또한 await가 있다는 이유만으로 단일 흐름의 모든 단계가 동시에 실행되는 것은 아니다. 이 작업이 I/O를 기다릴 때 이벤트 루프가 다른 실행 가능한 작업을 처리할 수 있다는 의미이다.

## 주제 4: ORM을 쓰는데 마이그레이션이 왜 필요한가

### 모델·변경 파일·실제 DB는 서로 다르다

SQLAlchemy 모델은 Python으로 표현한 테이블 구조이다. 모델을 수정했다고 PostgreSQL 안의 기존 테이블이 자동으로 바뀌지는 않는다. Alembic은 SQLAlchemy와 함께 사용하는 Python 마이그레이션 라이브러리이자 명령줄 도구이다. 마이그레이션(migration)은 테이블 생성·컬럼 추가처럼 DB 구조를 변경하고, 그 변경을 파일에 기록해 순서대로 적용하는 작업이다.

```text
Python 모델 정의
    ↓ 현재 DB와 비교
마이그레이션 파일 생성
    ↓ 적용 명령 실행
실제 DB 구조 변경
```

Alembic은 같은 변경 파일을 다른 개발 환경이나 서버 DB에도 적용할 수 있게 한다. 어떤 변경까지 적용했는지는 DB의 alembic_version 테이블에서 추적한다.

> ➕ **더 알아두기**
> 앞선 실습의 `Base.metadata.create_all(engine)`은 아직 없는 테이블을 만드는 데 사용할 수 있지만, 기존 테이블에 새 컬럼을 추가하는 구조 변경을 자동 반영하는 도구는 아니다. ORM을 없애고 Alembic으로 교체한 것이 아니라, ORM 모델과 실제 DB의 구조 변경을 관리하는 도구를 추가한 것이다.

### Git 커밋과 비슷한 점, 다른 점

> ♻️ 처음엔 “마이그레이션은 커밋이구나”로 이해했으나, 변경 이력을 기록한다는 비유만 비슷하고 Git 커밋·DB COMMIT·마이그레이션은 별개의 작업이라는 것이 맞다.

> ➕ **더 알아두기**
> Git 커밋은 저장소 파일의 변경 상태를 기록한다. 마이그레이션은 DB 구조를 바꿀 명령과 이전 변경과의 연결을 파일에 기록한다. DB COMMIT은 현재 트랜잭션의 변경을 확정한다. 마이그레이션 파일을 생성하는 것만으로 Git 커밋이 생기거나 DB 변경이 적용되지는 않는다. 생성된 마이그레이션 파일도 다른 소스 파일처럼 Git으로 관리할 수 있다.

> 📝 **기사 시험 포인트** (정보처리기사·정보보안기사)
> 데이터 정의어(Data Definition Language, DDL)는 CREATE·ALTER·DROP처럼 구조를 정의·변경하는 명령이다. 데이터 조작어(Data Manipulation Language, DML)는 INSERT·UPDATE·DELETE 등 행 데이터를 다루는 명령이다. COMMIT·ROLLBACK은 트랜잭션 확정·취소와 연결해 구분한다. Alembic 명령어 자체의 출제 여부나 빈도를 여기서 단정하지 않는다.

### 마이그레이션 대상과 초기화

| 모델 파일 | 클래스 | 테이블 |
|---|---|---|
| domains/users/models.py | User | users |
| domains/posts/models.py | Post | posts |
| domains/comments/models.py | Comment | comments |
| domains/chat/models.py | ChatRoom | chat_rooms |
| domains/chat/models.py | ChatMessage | chat_messages |

비동기용 Alembic 환경은 다음 명령으로 초기화한다.

```powershell
uv run alembic init -t async alembic
```

alembic.ini는 마이그레이션 경로 등의 설정, alembic/env.py는 접속·모델 메타데이터 연결과 실행 환경, alembic/versions는 변경 파일을 보관한다. 모델 모듈을 import해야 테이블 정의가 Base.metadata에 등록된다. Base만 import하면 실제 모델 목록이 빠질 수 있다.

실제 저장된 env.py는 다음과 같다.

```python
import asyncio
import selectors
import sys
from logging.config import fileConfig

from alembic import context
from sqlalchemy import pool
from sqlalchemy.ext.asyncio import async_engine_from_config

from fastapi_agent.core.config import get_settings
from fastapi_agent.core.database import Base
from fastapi_agent.domains.chat.models import ChatMessage, ChatRoom
from fastapi_agent.domains.comments.models import Comment
from fastapi_agent.domains.posts.models import Post
from fastapi_agent.domains.users.models import User

config = context.config
config.set_main_option("sqlalchemy.url", get_settings().database_url.replace("%", "%%"))

if config.config_file_name and config.attributes.get("configure_logging", True):
    fileConfig(config.config_file_name, disable_existing_loggers=False)

target_metadata = Base.metadata


def include_name(name, type_, parent_names):
    # Checkpoint tables are managed by AsyncPostgresSaver.
    checkpoint_tables = {
        "checkpoints", "checkpoint_blobs", "checkpoint_writes", "checkpoint_migrations"
    }
    return type_ != "table" or name not in checkpoint_tables


def run_migrations_offline():
    context.configure(
        url=config.get_main_option("sqlalchemy.url"),
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        include_name=include_name,
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection):
    context.configure(
        connection=connection, target_metadata=target_metadata, include_name=include_name
    )
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations():
    engine = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    try:
        async with engine.connect() as connection:
            await connection.run_sync(do_run_migrations)
    finally:
        await engine.dispose()


if context.is_offline_mode():
    run_migrations_offline()
elif sys.platform == "win32":
    asyncio.run(
        run_async_migrations(),
        loop_factory=lambda: asyncio.SelectorEventLoop(selectors.SelectSelector()),
    )
else:
    asyncio.run(run_async_migrations())
```

접속 URL은 .env에서 읽고, Alembic 설정에 넣을 때 `%`를 `%%`로 이스케이프한다. target_metadata는 비교할 모델 정의이다. include_name은 AsyncPostgresSaver가 별도로 관리하는 checkpoints·checkpoint_blobs·checkpoint_writes·checkpoint_migrations 테이블을 자동 비교 대상에서 제외한다. 채팅방·메시지 테이블은 여전히 Alembic으로 관리한다.

오프라인 실행은 접속 없이 SQL을 생성하고, 온라인 실행은 DB에 연결해 마이그레이션을 처리한다. 비동기 연결의 run_sync로 Alembic의 동기 마이그레이션 함수를 실행하고, finally에서 엔진을 정리한다. Windows에서는 psycopg 비동기 실행을 위해 Selector 이벤트 루프를 사용하는 분기가 있다.

## 주제 5: 초기 마이그레이션과 Post.created_at 추가

### 첫 번째 변경 파일을 만든다

초기 파일이 없는 새 프로젝트에서 다음 명령을 실행한다. 이미 초기 파일이 제공된 프로젝트라면 무조건 새로 만들기보다 해당 파일을 확인한다.

```powershell
uv run alembic revision --autogenerate -m "create initial tables"
```

autogenerate는 Base.metadata와 DB 구조의 차이를 찾아 변경 파일을 작성한다. 이 시점에는 테이블 생성 SQL을 아직 적용하지 않는다. 파일 이름 앞의 revision ID는 실행마다 달라진다.

```text
<revision_id>_create_initial_tables.py
```

실제 초기 파일은 `78e4ebcebcdb_create_initial_tables.py`이다. users·posts·comments·chat_rooms·chat_messages의 컬럼, 기본키, 외래키, 고유 제약조건, 인덱스 등이 들어 있다. 처음 생성했을 때 프로젝트 DB는 애플리케이션 테이블이 없는 상태였다.

```powershell
uv run alembic upgrade head
```

head는 마이그레이션 파일의 최신 버전이다. upgrade head는 현재 DB에 아직 적용되지 않은 upgrade()를 순서대로 실행한다.

### 게시글에 작성 시각을 추가한다

Post 클래스에 날짜·시간 타입과 DB 기본값을 추가한다. 기존 컬럼과 관계는 유지한다. 실제 저장된 Post 모델은 다음과 같다.

```python
from datetime import datetime
from typing import TYPE_CHECKING

from sqlalchemy import DateTime, ForeignKey, String, Text, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from fastapi_agent.core.database import Base

if TYPE_CHECKING:
    from fastapi_agent.domains.comments.models import Comment
    from fastapi_agent.domains.users.models import User


class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    title: Mapped[str] = mapped_column(String(50), nullable=False)
    content: Mapped[str] = mapped_column(Text, nullable=False)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), nullable=False)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )

    user: Mapped["User"] = relationship(back_populates="posts", lazy="raise")
    comments: Mapped[list["Comment"]] = relationship(
        back_populates="post",
        cascade="save-update, merge, delete, delete-orphan",
        passive_deletes=True,
        order_by="Comment.id",
        lazy="raise",
    )
```

| 표현 | 의미 |
|---|---|
| Mapped[datetime] | Python에서 날짜·시간 값으로 다루는 ORM 매핑 |
| DateTime(timezone=True) | 시간대를 고려하는 PostgreSQL 날짜·시간 타입 사용 |
| server_default=func.now() | INSERT에서 값을 생략하면 DB가 시각을 채움 |
| nullable=False | NULL 허용 안 함 |

PostgreSQL의 now()는 현재 트랜잭션 시작 시각을 반환한다. 매번 Python이 datetime을 계산해 넘기는 것과 DB가 기본값을 채우는 것은 구분한다.

### 기존 파일을 고치지 않고 다음 변경을 만든다

초기 마이그레이션은 남겨 두고 새 파일을 생성한다.

```powershell
uv run alembic revision --autogenerate -m "add post created_at"
```

실제 생성 파일은 `dc2f73773774_add_post_created_at.py`이며, down_revision은 초기 ID인 `78e4ebcebcdb`이다. 아래 두 함수가 정방향 변경과 되돌리기를 표현한다.

```python
def upgrade() -> None:
    """Upgrade schema."""
    # ### commands auto generated by Alembic - please adjust! ###
    op.add_column('posts', sa.Column('created_at', sa.DateTime(timezone=True), server_default=sa.text('now()'), nullable=False))


def downgrade() -> None:
    """Downgrade schema."""
    # ### commands auto generated by Alembic - please adjust! ###
    op.drop_column('posts', 'created_at')
```

자동 생성 뒤에는 의도한 컬럼 추가만 있는지, 원치 않는 변경·삭제가 없는지 사람이 확인한다. 파일 생성과 DB 적용을 혼동하지 않는다. 적용 단계에서는 다시 `uv run alembic upgrade head`를 사용하고 DB의 posts에 created_at이 있는지 확인한다.

> ➕ **더 알아두기**
> 작업은 단계별로 진행했다. 처음에는 초기 파일만 생성했다. 다음 요청에서 초기 테이블을 적용하고 created_at 변경 파일만 생성했다. 이후 작성자 정보 반환 예제까지 진행하면서 최신 버전 적용 상태와 실제 컬럼 존재를 확인했다. 최종 확인된 DB 버전은 `dc2f73773774`이며, 현재는 created_at이 실제 DB에도 있다. 이전 답변의 “아직 미적용”은 당시 단계의 상태이다.

> ➕ **더 알아두기**
> downgrade가 있다는 이유만으로 모든 변경이 데이터 손실 없이 되돌아가는 것은 아니다. 이 변경의 downgrade는 created_at 컬럼을 삭제하므로 그 안의 값도 사라진다. 테스트의 오프라인 SQL 생성과 실제 DB에서 downgrade를 실행한 것은 다르다. 이번에는 실제 DB에서 컬럼을 되돌리는 작업을 하지 않았다.

## 주제 6: lifespan은 서버의 시작과 종료를 관리한다

### Alembic과 실행 시점이 다르다

lifespan은 FastAPI 앱이 요청을 받기 전에 공용 자원을 준비하고, 앱이 종료될 때 정리하는 실행 흐름이다. 요청마다 실행되는 Router나 get_db와 다르다. DB 구조 변경은 Alembic 명령으로 관리하고, 서버 시작 코드에서 자동으로 테이블을 만들거나 수정하지 않는다.

실제 저장된 main.py는 다음과 같다.

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI

from fastapi_agent.core.database import engine


@asynccontextmanager
async def lifespan(app: FastAPI):
    try:
        yield
    finally:
        await engine.dispose()


app = FastAPI(lifespan=lifespan)
```

| 시점 | 동작 |
|---|---|
| yield 전 | 시작 준비 작업. 현재 코드는 별도 준비 없이 yield로 이동 |
| yield에서 실행을 넘긴 동안 | 앱이 요청을 처리함 |
| 앱 종료 후 finally | await engine.dispose()로 연결 풀 정리 |

`@asynccontextmanager`는 yield를 기준으로 비동기 준비·정리를 한 쌍으로 다루게 한다. `FastAPI(lifespan=lifespan)`으로 등록해야 FastAPI가 앱 생명주기에 맞춰 사용한다. try/finally는 종료 정리가 실행되도록 배치한 것이다.

> ➕ **더 알아두기**
> 대화에서는 가게의 개점 준비 → 영업 → 마감 정리에 비유했다. yield 뒤의 정리는 첫 번째 HTTP 요청이 끝날 때 실행되는 것이 아니라 앱의 lifespan이 끝날 때 실행된다. engine.dispose()는 DB나 게시글을 지우는 명령이 아니다. 엔진이 관리하는 연결 풀의 자원을 정리한다. 프로세스를 강제로 종료하는 모든 경우까지 정리 실행을 보장한다는 의미로 넓혀 이해하지 않는다.

### 앱 실행 위치를 지정한다

```toml
[tool.fastapi]
entrypoint = "fastapi_agent.main:app"
```

fastapi_agent.main은 src/fastapi_agent/main.py 모듈이고 app은 해당 모듈의 FastAPI 객체이다. uv sync로 패키지를 설치하므로 모듈 경로에 src를 넣지 않는다.

```powershell
uv run fastapi dev
```

강의의 개발 서버 실행 명령이다. 기본 주소는 `http://127.0.0.1:8000`이고 코드가 바뀌면 재시작한다.

> ➕ **더 알아두기**
> 이번 세션에서는 앱 import와 설정을 확인했다. 개발 서버를 띄워 HTTP 요청으로 전체 게시판 기능을 시험한 것은 아니다. main.py에는 기본 FastAPI 객체와 lifespan이 있고, 게시글·인증 Router는 아직 연결하지 않았다.

## 주제 7: 게시글의 작성자 번호와 작성자 객체는 다르다

강의는 관계 속성에 `lazy="raise"`를 설정한다. 아직 로딩하지 않은 관계를 읽으면 추가 SQL을 자동 실행하는 대신 InvalidRequestError를 내고, 이미 로딩한 관계는 일반 속성처럼 읽게 한다.

```python
user: Mapped["User"] = relationship(back_populates="posts", lazy="raise")
```

게시글을 `user_id=current_user.id`로 만들면 작성자 번호를 저장한 것이지 post.user까지 로딩한 것은 아니다. 응답에 작성자 정보가 필요하면 먼저 가져와야 한다.

> ➕ **더 알아두기**
> 대화에서는 다음 표로 관계를 설명했다. 실제 모델에는 이름 대신 email 등의 컬럼이 있으며, 아래 이름은 개념 설명용이다.
>
> | 회원 id | 이름 |
> |---|---|
> | 1 | 민수 |
> | 2 | 지수 |
>
> | 게시글 id | 제목 | user_id |
> |---|---|---|
> | 10 | 안녕하세요 | 2 |
>
> 게시글 행에는 “작성자는 2번 회원이다”라는 외래키만 있다. `post.user_id`는 2라는 번호이고, `post.user`는 회원 객체이다. 번호를 알고 있다는 것과 회원 객체의 속성을 사용할 수 있도록 관계가 로딩됐다는 것은 별개이다. 예제에서는 회원 객체를 먼저 만들었더라도 lazy="raise"인 post.user 관계를 명시적으로 로딩하는 것을 확인한다. 모든 ORM 상황에서 외래키가 있으면 반드시 추가 SELECT가 한 번 발생한다고 일반화하지 않는다.

### 필요할 때 가져오는 것이 지연 로딩이다

게시글을 가져올 때 작성자는 가져오지 않았다가, 나중에 post.user가 필요해졌을 때 조회하는 것이 지연 로딩(Lazy Loading)이다. 미리 읽어 두는 즉시 로딩(Eager Loading)과 조회 시점이 다르다.

강의의 대안은 두 가지이다. 게시글을 저장한 후 refresh로 관계를 명시적으로 가져오거나, joinedload를 사용한 조회에서 작성자를 함께 가져온다.

```python
await db.refresh(post, ["user"])
```

```python
loaded_post = await db.scalar(
    select(Post).options(joinedload(Post.user)).where(Post.id == post.id)
)
```

selectinload도 관계를 미리 로딩하는 전략이다. 관계를 불러오는 DB 작업은 await 조회 범위에서 끝내고, 응답 변환 시에는 이미 로딩한 값을 읽는다.

## 주제 8: 왜 일반 속성 접근에서 비동기 자동 조회가 문제인가

AsyncSession에서는 일반 속성을 읽는 도중 발생한 DB I/O를 그 자리에서 await할 수 없어 MissingGreenlet 오류가 날 수 있다. Pydantic 응답 변환도 ORM 속성을 동기적으로 읽으므로 필요한 관계를 사전에 준비해야 한다.

> ♻️ 처음엔 “비동기에서는 지연 로딩 자체가 안 된다”로 이해했으나, 일반 속성 접근의 암묵적 I/O와 명시적인 비동기 관계 로딩은 다르며 후자는 가능하다는 것이 맞다. 앞선 설명도 이 둘을 구분하지 않아 혼동을 키웠으므로 정정된 내용만 남긴다.

> ➕ **더 알아두기**
> 사용자는 “기존에도 DB에 요청하면 답을 줬는데 왜 안 되느냐”고 물었다. DB의 응답 능력이 달라진 것이 아니다. 동기 호출은 실행 중인 스레드에서 답을 기다릴 수 있다. 비동기 호출은 기다릴 수 있는 작업과 실행 제어를 넘기는 지점이 연결되어야 하며, 일반 속성 읽기는 자동으로 이 인터페이스로 바뀌지 않는다.
>
> 순서는 다음과 같다.
>
> 1. 게시글 조회를 명시적인 비동기 DB 호출로 실행하고 기다린다.
> 2. 게시글의 작성자 번호는 있지만 작성자 관계는 아직 로딩되지 않았다.
> 3. 응답을 만들다가 일반 속성인 post.user를 읽는다.
> 4. 관계가 없으니 일반 lazy loader가 추가 DB I/O를 시도한다.
> 5. 그 속성 접근은 비동기 드라이버의 대기를 처리할 SQLAlchemy 실행 문맥으로 연결되어 있지 않아 오류가 날 수 있다.
>
> async def 안에 들어 있다는 사실만으로 그 안의 모든 속성 접근이 await 가능한 작업으로 변하지 않는다. SQLAlchemy가 await 기반 API 안에서 마련하는 실행 문맥과 일반 속성 읽기 사이의 차이이다.

> ♻️ 처음엔 “작성자 답이 바로 안 와서 오류가 난다”로 이해했으나, 응답 지연·시간 초과가 아니라 암묵적 비동기 I/O를 수행할 실행 문맥이 없는 것이 문제라는 설명이 맞다. DB가 빨라도 같은 문제가 생길 수 있다.

> ♻️ 처음엔 “첫 번째 게시글 조회만 비동기가 가능하고 두 번째 작성자 조회는 불가능하다”로 이해했으나, 순서와 무관하게 두 조회 모두 적절한 비동기 API를 사용하면 기다릴 수 있다는 것이 맞다.

### “그럼 여기도 await를 붙이면 되지?”의 정확한 답

> ➕ **더 알아두기**
> 사용자의 해결 방향은 맞다. 두 번째 조회도 명시적으로 기다리면 된다. 다만 await는 일반 값을 임의로 비동기 작업으로 바꾸는 표시가 아니다. `await post.user`라고 쓰면 먼저 post.user를 평가해야 하므로 일반 속성 조회 단계의 문제를 해결하지 못한다. await가 받을 수 있는 작업을 반환하도록 설계된 API를 사용해야 한다. 현재 코드에서는 `await db.refresh(post, ["user"])`가 그 역할을 한다.

### 비동기 지연 로딩 기능도 존재한다

> ➕ **더 알아두기**
> SQLAlchemy의 AsyncAttrs는 `await obj.awaitable_attrs.관계속성` 형태로 필요할 때 관계를 기다려 읽는 기능을 제공한다. 사용자가 제안한 비동기 지연 로딩은 이미 제공되는 기능이다. 다만 **AsyncAttrs를 추가해도 lazy="raise" 설정을 자동으로 해제하지 않는다.** 그 방식을 쓰려면 관계 로딩 설정도 허용하는 방향으로 선택해야 한다. 현재 Base는 DeclarativeBase만 상속하고, Post.user는 lazy="raise"이므로 이 대안을 적용한 상태는 아니다. 오늘은 기능의 존재와 설계 선택을 설명했으며 실제 AsyncAttrs 방식의 코드를 저장·실행하지 않았다.
>
> 근거: [SQLAlchemy 공식 문서 — 비동기 세션에서 암묵적 I/O 방지](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html#preventing-implicit-io-when-using-asyncsession).

## 주제 9: 가능한 기능을 왜 lazy="raise"로 막는가

> ➕ **더 알아두기**
> 사용자는 “속성 읽기를 비동기로 설정할 수 있다면 왜 아예 막느냐”고 반문했다. 기술적으로 비동기 지연 로딩을 할 수 있다는 사실과, 프로젝트에서 어떤 조회 방식을 허용할지는 별개의 문제이다. 강의는 응답에 필요한 관계를 명시적으로 로딩하도록 lazy="raise"를 선택했다.
>
> 목적은 DB 조회 횟수의 상한을 설정하는 것이 아니라, 값만 읽는 줄 알았던 위치에서 예상하지 못한 추가 조회가 일어나는 것을 막는 것이다. 특히 응답 직렬화는 Python 객체를 JSON 등 전송할 표현으로 바꾸는 과정인데, Pydantic은 각 관계 속성을 자동으로 await해서 읽어주지 않는다. 따라서 응답 생성 전에 필요한 데이터가 준비됐는지 확인하는 규칙으로 사용한다.
>
> 게시글 목록 100개를 조회한 뒤 각 작성자를 하나씩 lazy loading하면, 조건에 따라 최대 100번의 추가 조회가 발생할 수 있다. 처음 목록 조회 1번에 관계 조회 N번이 뒤따르는 N+1 문제이다. 같은 회원의 재사용이나 이미 로딩된 객체 등에 따라 실제 조회 수는 달라지므로 항상 정확히 101번이라고 단정하지 않는다. joinedload·selectinload처럼 관계를 미리 읽는 전략으로 관리할 수 있다.

> ♻️ 처음엔 “DB 조회를 제한하려고 막았다”로 이해했으나, 숫자로 횟수를 제한하는 기능이 아니라 의도하지 않은 자동 조회를 금지하고 조회 시점을 관리하는 설정이라는 것이 맞다.

> ➕ **더 알아두기**
> 현재 설정에서 로딩되지 않은 post.user를 읽으면 SQLAlchemy의 InvalidRequestError가 먼저 발생한다. Pydantic으로 감싸서 응답 변환을 시도하면 그 원인이 ValidationError의 속성 읽기 오류로 나타날 수 있다. 일반 lazy loading을 허용한 상태에서 발생할 수 있는 MissingGreenlet과, lazy="raise"가 의도적으로 내는 오류를 같은 것으로 기록하지 않는다. 이미 읽어 둔 post.user를 사용하는 것은 lazy="raise"에서도 허용된다.

## 주제 10: 실제로 저장·실행한 예제와 확인 결과

> ➕ **더 알아두기**
> 강의의 해당 구간은 게시글 저장 후 작성자를 명시적으로 로딩하는 원리를 설명한다. 현재 프로젝트에는 전체 게시글 API 대신 `examples/create_post_with_user.py` 실행 예제를 추가했다. 회원·게시글을 임시로 생성하고 로딩 전후 응답 변환을 비교한다. 아래는 실제 저장·실행한 파일 그대로이며, UserResponse는 id·email만 포함해 비밀번호 해시를 응답에서 제외한다.

```python
"""게시글 작성 후 작성자를 로딩하고 응답으로 변환한다. 데모 데이터는 롤백한다."""

import asyncio
import selectors
import sys
from uuid import uuid4

from pydantic import BaseModel, ConfigDict, ValidationError
from sqlalchemy import inspect, select
from sqlalchemy.orm import joinedload

from fastapi_agent.core.database import SessionLocal, engine
from fastapi_agent.domains.comments.models import Comment
from fastapi_agent.domains.posts.models import Post
from fastapi_agent.domains.users.models import User


class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: str


class PostResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    title: str
    content: str
    user: UserResponse


async def main():
    try:
        async with SessionLocal() as db:
            try:
                user = User(
                    email=f"loading-demo-{uuid4().hex}@example.com",
                    hashed_password="demo-only-not-a-password-hash",
                )
                db.add(user)
                await db.flush()

                post = Post(
                    title="작성자가 있는 게시글",
                    content="작성자를 명시적으로 로딩한 뒤 응답을 만듭니다.",
                    user_id=user.id,
                )
                db.add(post)
                await db.flush()
                assert post.created_at is not None
                assert "user" in inspect(post).unloaded

                # user_id만 지정하면 post.user는 아직 로딩되지 않는다.
                try:
                    PostResponse.model_validate(post)
                except ValidationError as exc:
                    assert any(error["loc"] == ("user",) for error in exc.errors())
                    print('로딩 전: lazy="raise"가 작성자 자동 조회를 차단했습니다.')
                else:
                    raise AssertionError("로딩하지 않은 작성자의 응답 변환은 실패해야 합니다.")

                await db.refresh(post, ["user"])
                response = PostResponse.model_validate(post)
                assert response.user.id == user.id
                assert "hashed_password" not in response.user.model_dump()
                print("refresh 후 응답:")
                print(response.model_dump_json(indent=2))

                # 같은 관계를 조회할 때 함께 로딩하는 대안도 확인한다.
                db.expire(post, ["user"])
                loaded_post = await db.scalar(
                    select(Post).options(joinedload(Post.user)).where(Post.id == post.id)
                )
                assert loaded_post is not None
                assert "user" not in inspect(loaded_post).unloaded
                assert PostResponse.model_validate(loaded_post) == response
                print("joinedload로 조회한 응답도 동일합니다.")
            finally:
                await db.rollback()
                print("데모 데이터를 롤백했습니다.")
    finally:
        await engine.dispose()


if __name__ == "__main__":
    if sys.platform == "win32":
        asyncio.run(
            main(),
            loop_factory=lambda: asyncio.SelectorEventLoop(selectors.SelectSelector()),
        )
    else:
        asyncio.run(main())
```

> ➕ **더 알아두기**
> 실행 순서는 다음과 같다.
>
> 1. 임시 회원을 생성하고 flush하여 ID를 받는다.
> 2. user_id만 지정한 게시글을 생성하고 flush한다. created_at이 채워졌는지 확인한다.
> 3. user 관계가 아직 unloaded 상태인지 검사한다.
> 4. 로딩 전 Pydantic 응답 변환이 실패하는지 확인한다.
> 5. refresh로 user 관계를 불러온 뒤 작성자 정보를 포함한 응답으로 변환한다.
> 6. 비교 실험을 위해 관계를 expire한 뒤 joinedload 조회로 같은 응답이 나오는지 확인한다. expire는 두 번째 전략을 시험하기 위한 것이며 일반적인 비동기 처리에서 매번 해야 하는 단계는 아니다.
> 7. finally에서 rollback하여 임시 데이터가 남지 않게 한다.
> 8. 엔진을 dispose하여 연결 풀을 정리한다.
>
> 이 예제는 학습용이라 commit하지 않는다. 실제 게시글을 영구 저장하는 서비스에서는 성공한 트랜잭션을 확정하는 별도 처리가 필요하다. 또한 데모의 hashed_password는 실제 비밀번호를 해싱한 값이 아니며 인증 기능 예제가 아니다.

```powershell
uv run python examples/create_post_with_user.py
uv run python -m unittest discover -s tests -v
```

> ➕ **더 알아두기**
> 실제 검증 결과는 다음과 같다. 이는 오늘 대화에서 실행해 확인한 결과이며 TIL 작성 시 새로 전체 테스트를 반복한 결과는 아니다.
>
> | 확인 항목 | 확인한 결과 |
> |---|---|
> | 설정값 검증 | 짧은 JWT 키·0 이하 만료 시간 거부 테스트 통과 |
> | 초기 마이그레이션 | 메모리 SQLite에서 초기 생성·롤백과 예상 스키마 차이 검사 통과 |
> | created_at 변경 | PostgreSQL용 정방향·역방향 SQL 생성 검사 통과 |
> | 실제 PostgreSQL 예제 | 로딩 전 변환 실패, refresh 후 변환 성공, joinedload 응답 동일성 확인 |
> | 응답 정보 | 작성자 id·email 포함, 비밀번호 해시 제외 |
> | 실제 DB 구조 | dc2f73773774 버전과 posts.created_at 존재 확인 |
> | 임시 데이터 | 예제 회원 데이터가 롤백되어 남지 않은 것 확인 |
>
> 단위 테스트는 최종 3개가 통과했다. 실제 PostgreSQL에서 downgrade를 실행하거나, 전체 HTTP CRUD·로그인·Agent 기능을 시험한 것은 아니다. 롤백하더라도 PostgreSQL 시퀀스는 사용한 번호를 되돌리지 않아 자동 증가 ID가 건너뛸 수 있다. 데모 데이터 삭제 실패를 뜻하는 것은 아니다.

> 📝 **기사 시험 포인트** (정보처리기사·정보보안기사)
> 트랜잭션의 원자성(Atomicity)은 작업 묶음이 전부 반영되거나 취소되는 성질이다. flush로 SQL을 실행한 것과 COMMIT으로 확정한 것은 구분한다. 이 예제는 SQL을 실행한 뒤 ROLLBACK으로 임시 행 변경을 취소한다. 외래키(Foreign Key, FK)와 참조 무결성(Referential Integrity)은 게시글의 user_id가 존재하는 회원을 참조하는 구조와 연결한다.

## 정리

- ORM 모델과 실제 DB 구조는 별개이며 모델 수정만으로 DB 구조가 바뀌지 않는다.
- Alembic은 DB 구조 변경 파일을 생성하고 순서대로 적용·추적한다. revision과 upgrade는 역할이 다르다.
- 초기 변경 다음에 created_at 변경을 연결했고, 최종 DB는 두 변경이 적용된 상태이다.
- lifespan은 앱 시작 준비와 종료 정리를 담당하며 DB 구조 변경 명령이 아니다.
- 작성자 번호 user_id를 저장했다고 작성자 관계 user가 로딩된 것은 아니다.
- 비동기 관계 조회 자체는 가능하다. 문제는 일반 속성 읽기 도중의 암묵적 I/O이다.
- lazy="raise"는 자동 조회를 막는 프로젝트의 선택이며, 조회 횟수 제한 기능은 아니다.
- 현재 예제는 refresh 또는 joinedload로 작성자를 준비한 다음 응답으로 변환한다.
- AsyncAttrs는 대화 중 확인한 대안이며 현재 코드에 적용하지 않았다.

## ✅ 확인 질문

1. fastapi_agent·agents·rag는 각각 어떤 책임을 가지며, agents/rag와 src/rag는 어떻게 다른가?
2. Router·Service·Repository·models.py·schemas.py는 각각 어떤 코드를 두는 자리인가?
3. SQLAlchemy·psycopg·Alembic·pydantic-settings의 역할을 서로 구분해 설명할 수 있는가?
4. .env, SettingsConfigDict, Field, SecretStr, lru_cache는 설정 처리에서 각각 무엇을 하는가?
5. FastAPI에서 비동기를 쓰면 개별 DB 쿼리 자체가 반드시 빨라지는가? I/O 대기와 연결해 설명하라.
6. expire_on_commit=False가 해결하는 문제는 무엇이며, 이 설정만으로 post.user가 로딩되는가?
7. ORM으로 테이블을 만들 수 있는데도 Alembic이 필요한 이유는 무엇인가?
8. Git 커밋·DB COMMIT·마이그레이션 파일 생성·마이그레이션 적용은 어떻게 다른가?
9. 초기 마이그레이션 파일을 만든 직후와 upgrade를 실행한 뒤의 실제 DB 상태는 어떻게 다른가?
10. revision·down_revision·upgrade·downgrade·head는 각각 어떤 의미이며, 생성 파일은 왜 검토해야 하는가?
11. lifespan의 yield 전·실행을 넘긴 동안·종료 후에는 각각 어떤 일이 일어나는가?
12. engine.dispose()는 테이블을 삭제하는가? 요청마다 세션을 정리하는 get_db와 실행 시점이 어떻게 다른가?
13. post.user_id를 알고 있는데도 post.user가 아직 로딩되지 않을 수 있는 이유는 무엇인가?
14. refresh와 joinedload는 작성자 정보를 어떤 시점에 준비하며, 응답 변환 전에 필요한 이유는 무엇인가?
15. MissingGreenlet을 DB 응답이 늦어서 나는 오류라고 설명하면 왜 틀리는가?
16. 두 번째 조회도 비동기로 가능한데 일반 post.user 접근과 await 가능한 조회 API는 무엇이 다른가?
17. AsyncAttrs가 지원하는 기능은 무엇이며, 현재 lazy="raise" 설정이 자동으로 해제되는가?
18. lazy="raise"를 쓰는 목적을 응답 직렬화·예상하지 못한 조회·N+1 문제와 연결해 설명하라.
19. 오늘의 실행 예제는 어떤 순서로 로딩 전후를 비교했으며, 실제 인증·HTTP API까지 구현한 것인가?
20. 임시 데이터를 rollback했는데 자동 증가 ID가 건너뛸 수 있는 이유와, flush·commit의 차이는 무엇인가?
