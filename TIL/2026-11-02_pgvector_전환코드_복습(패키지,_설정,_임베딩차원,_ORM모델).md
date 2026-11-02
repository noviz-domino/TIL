---
tags: [pgvector, embeddings, sqlalchemy, config, fastapi]
til: v2 2026-11-02
---

# 2026-11-02_pgvector_전환코드_복습(패키지,_설정,_임베딩차원,_ORM모델)
> 작성일: 2026-11-02

## 🔗 관련 글

- [FastAPI Agent 도커 전환 준비](2026-10-30%20(2)FastAPI_Agent_도커전환_준비(실행구조,_TOML,_환경설정,_Alembic,_pgvector).md) — 변경 코드를 저장한 뒤 이번에는 파일별 원본·수정·이유를 나눠 복습한다.
- [Docker와 Compose 실습](2026-10-30%20(1)Docker와_Compose_실습(WSL2,_볼륨,_pgvector,_FastAPI,_바인드마운트).md) — DB 이미지·볼륨·연결 위치를 복습한다.
- [비동기 DB와 Alembic](2026-10-29%20(1)FastAPI_비동기_DB와_Alembic_마이그레이션(lifespan,_관계_로딩).md) — ORM 모델 정의와 실제 테이블 생성의 차이를 연결한다.

## ❓ 아직 모르겠는 것

대화에서 풀린 질문도 복습 대상으로 남긴다. 이번에는 사용자가 전체 설명을 한 번에 받기보다 파일 하나씩 원본·변경·이유를 보여 달라고 요청했다.

- “config.py 어디 있나?” — rag/config.py와 fastapi_agent/core/config.py를 구분하는가? (주제 1)
- “1번부터 다시, 원본을 어떤 코드로 바꿨는지 보여줘.” — 패키지 추가가 저장소 전환 자체와 다른 이유는 무엇인가? (주제 2)
- “Gemini 임베딩 모델 2는 차원 고정 아닌가? 지정 가능?” — 기본 차원과 지정 가능한 차원을 구분하는가? (주제 3·4)
- “차원은 높을수록 좋은 것 아닌가?” — 표현력·검색 품질·자원·청킹의 관계는 무엇인가? (주제 4)
- 임베딩 객체를 만들 때 문서를 보내는가? — 실제 API 요청은 어떤 메서드를 실행할 때 일어나는가? (주제 5, 앞선 질문에 필요한 선행 설명)
- ORM 모델의 테이블·컬럼 정의만 저장하면 DB에 반영되는가? — 생성은 Alembic, 데이터 등록은 PGVectorStore라는 역할을 다시 확인한다. (주제 6)
- 실제 pgvector PDF 재등록과 검색은 성공했는가? — 아직 이 대화에서 결과를 확인하지 않았으므로 다음 실습에서 검증한다. (미해결)

## 주제 1: config.py가 두 개인 이유와 현재 보는 파일

> ➕ **더 알아두기 — 실제 코드 탐색**
> 사용자가 찾던 것은 src/rag/config.py였다. 이 파일은 PDF 위치·검색 테이블·임베딩 모델·벡터 크기·청킹·공용 키 읽기를 다룬다. src/fastapi_agent/core/config.py는 앱 DB·JWT·로그 설정을 읽고 검증한다. 둘은 같은 이름이지만 폴더와 역할이 다르다.

```text
fastapi/
└─ fastapi_agent/
   └─ src/
      ├─ rag/
      │  ├─ config.py      PDF·벡터·모델·키 읽기
      │  ├─ embeddings.py  임베딩 객체 준비
      │  └─ models.py      벡터 테이블 ORM 정의
      └─ fastapi_agent/
         └─ core/
            └─ config.py   DB·JWT·앱 설정
```

> ➕ **더 알아두기**
> 설명은 설정→생성→실제 실행을 구분한다. config.py의 상수 이름만 적는다고 파일·테이블이 자동 생성되지 않는다. 다른 코드가 값을 읽어서 사용해야 한다. 실제 키·비밀번호·개인 절대 경로는 공개 학습 기록에 넣지 않는다.

## 주제 2: 변경 설명 1번 — 패키지를 교체가 아니라 추가

이번 복습은 앞서 저장한 코드와 pyproject.toml을 기준으로 한다. 설명만 다시 했으며 패키지 설치·코드 변경을 재실행하지 않았다.

### 변경 전

기존 의존성 중 관련 부분은 다음과 같다. 다른 항목은 생략한 발췌다.

```toml
[project]
dependencies = [
    "chromadb>=1.5,<2",
    "langchain-chroma>=1.1,<2",
    # 다른 패키지 생략
]
```

| 패키지 | 역할 |
|---|---|
| chromadb | Chroma 저장소 |
| langchain-chroma | LangChain에서 Chroma를 사용하도록 연결 |

### 변경 후

기존 항목을 유지하고 다음 항목을 추가했다.

```toml
[project]
dependencies = [
    "chromadb>=1.5,<2",
    "langchain-chroma>=1.1,<2",
    # 다른 패키지 생략
    "langchain-postgres>=0.0.19",
    "asyncpg>=0.32.0",
    "pgvector>=0.3.6",
]
```

```text
RAG 코드
  ↓ langchain-postgres: 문서·벡터 저장과 검색 인터페이스
  ↓ asyncpg: 비동기 PostgreSQL 통신 드라이버
PostgreSQL + vector 확장
```

Python의 pgvector는 ORM에 Vector(768) 같은 자료형을 표현하는 데 사용한다. PostgreSQL의 vector 확장과 이름은 비슷하지만 설치 위치가 다르다.

| 대상 | 설치 위치 | 하는 일 |
|---|---|---|
| vector 확장 | DB 서버 | 벡터 저장·비교 |
| Python pgvector 패키지 | .venv | Python·SQLAlchemy 벡터 자료형 |

사용자가 uv add langchain-postgres를 실행했고 조교가 uv add asyncpg pgvector를 실행해 직접 의존성을 명시했다. uv add가 pyproject.toml·uv.lock·실제 환경을 함께 반영했다. Chroma 패키지·데이터를 아직 제거하지 않았지만 변경한 활성 코드에서 사용하지 않는다. 두 저장소를 config로 선택하는 기능을 구현한 것은 아니다.

> ➕ **더 알아두기**
> 패키지가 설치되어 있다고 자동 실행되지 않는다. PostgreSQL 연결 코드를 작성해야 저장소가 전환된다. 기존 자료 보존과 활성 저장소 선택은 별도 문제다.

## 주제 3: 변경 설명 2번 — Chroma 위치에서 테이블·차원 설정으로

### 변경 전

```python
PERSIST_DIRECTORY = "./chroma_db"
COLLECTION_NAME = "spri_ai_brief"
```

PERSIST_DIRECTORY는 데이터 폴더 위치, COLLECTION_NAME은 그 안의 Chroma 컬렉션 이름이었다.

```text
chroma_db 폴더
└─ spri_ai_brief 컬렉션
   └─ 원문·임베딩·메타데이터
```

### 변경 후

```python
TABLE_NAME = "spri_ai_brief_vectors"
VECTOR_SIZE = 768
```

TABLE_NAME은 PostgreSQL에서 사용할 테이블 이름, VECTOR_SIZE는 문서 조각 하나를 나타낼 숫자의 개수다. 문서 수가 768개라는 뜻이 아니다.

```text
PostgreSQL fastapi_agent DB
└─ spri_ai_brief_vectors 테이블
   └─ 문서 조각별 원문·768차원 벡터·출처
```

> ➕ **더 알아두기**
> TABLE_NAME과 VECTOR_SIZE는 우리가 정한 Python 변수 이름이다. 이름만 쓰면 테이블을 생성하는 예약어가 아니다. 저장소 코드·ORM 모델·임베딩 코드가 import해서 사용한다. PDF 위치·모델 이름·700자 청킹·100자 겹침·공용 키 읽기 방식은 유지했다.

## 주제 4: Gemini Embedding 2의 기본 차원·선택·품질

사용자는 원래 차원이 고정된 것으로 알았다. 공식 문서와 설치된 라이브러리 코드를 확인해 기본값과 지정 가능 범위를 구분했다.

| 구분 | Gemini Embedding 2 |
|---|---|
| 기본 출력 | 3,072차원 |
| 지원 범위 | 128~3,072 |
| 권장 선택값 | 768·1,536·3,072 |

[Google 모델 문서](https://ai.google.dev/gemini-api/docs/models/gemini-embedding-2)와 [임베딩 안내](https://ai.google.dev/gemini-api/docs/embeddings)에서 확인했다. 이 표는 확인 당시 해당 모델 기준이며 모든 임베딩 모델의 공통 성질이 아니다.

```python
GoogleGenerativeAIEmbeddings(
    model="gemini-embedding-2",
    api_key=get_api_key(),
    output_dimensionality=768,
)
```

> ♻️ 처음엔 Gemini Embedding 2가 768차원으로 고정된 것으로 알았으나, 기본 출력과 지정 가능한 차원은 다르며 output_dimensionality로 선택할 수 있다는 것이 맞다.

> ➕ **더 알아두기 — 높을수록 무조건 좋은가?**
> 같은 모델에서 큰 벡터는 더 풍부한 표현에 도움이 될 수 있지만 모든 질문·문서에서 품질 향상을 보장하지 않는다. 저장 숫자와 거리 비교 연산량도 늘어난다. 3,072는 768의 네 배 숫자지만 정확도가 네 배이거나 검색 시간이 정확히 네 배라는 뜻은 아니다.
>
> PDF 추출 품질, 청크 분리, 문서에 정답 근거가 존재하는지, 질문·검색 설정도 중요하다. 청킹이 잘못되면 차원을 늘려도 해결되지 않을 수 있다. 같은 질문 집합으로 근거 검색 성공·속도·용량을 비교해야 우리 자료에 맞는 선택을 평가할 수 있다.
>
> 768은 수업 예제와 자원 절감을 고려한 실습 선택이다. 우리 PDF에서 최적이라고 비교 검증한 결과는 아니다. 모델 차원 변경과 요금 절감을 동일시하는 주장도 이번에 확인하지 않았다.

```text
저장할 문서 임베딩 768
         ↕ 같은 모델·호환 설정
검색할 질문 임베딩 768
         ↕
DB의 vector(768)
```

길이만 같으면 서로 다른 모델 벡터를 섞어도 된다는 뜻은 아니다. 기존 Chroma 벡터를 그대로 재사용하지 않고 원본 PDF를 현재 설정으로 다시 임베딩할 계획이다.

## 주제 5: 변경 설명 3번 — 설정값을 임베딩 객체에 전달

### 변경 전

```python
from langchain_google_genai import GoogleGenerativeAIEmbeddings

from rag.config import EMBEDDING_MODEL, get_api_key


def get_embeddings() -> GoogleGenerativeAIEmbeddings:
    return GoogleGenerativeAIEmbeddings(
        model=EMBEDDING_MODEL,
        api_key=get_api_key(),
    )
```

모델·키를 설정했지만 출력 차원을 명시하지 않았다. 설정이 없으면 서비스·라이브러리의 기본 동작을 사용한다.

### 변경 후

```python
from langchain_google_genai import GoogleGenerativeAIEmbeddings

from rag.config import EMBEDDING_MODEL, VECTOR_SIZE, get_api_key


def get_embeddings() -> GoogleGenerativeAIEmbeddings:
    return GoogleGenerativeAIEmbeddings(
        model=EMBEDDING_MODEL, api_key=get_api_key(), output_dimensionality=VECTOR_SIZE,
    )
```

원본 대비 VECTOR_SIZE import와 output_dimensionality 인자를 추가했다. config에 값을 적기만 하면 효과가 없고 실제 생성 코드가 받아야 한다.

```text
config.py의 VECTOR_SIZE=768
       ↓ import
embeddings.py의 output_dimensionality=VECTOR_SIZE
       ↓ 실제 임베딩 요청 시
서비스에 768차원 출력 요청
```

> ➕ **더 알아두기 — 객체 준비와 API 호출**
> embeddings=get_embeddings()는 요청에 사용할 객체를 준비한다. 문서를 바로 임베딩하는 것은 아니다. embed_query·embed_documents 또는 비동기 대응 메서드가 호출될 때 입력을 API로 보낸다. 실제 등록·검색에서는 저장소 라이브러리가 이 임베딩 서비스의 메서드를 호출한다. 문서·질문이 같은 준비 함수를 사용하므로 모델·차원을 맞추기 쉽다.

```python
# 직접 사용하는 원리를 보여 주는 예시, 이 대화에서 실행하지 않았다.
embeddings = get_embeddings()
vector = embeddings.embed_query("볼륨은 무엇인가?")
```

이 단계의 설명에서는 실제 외부 요청·PDF 재등록을 새로 실행하지 않았다. 파일은 앞선 전환 작업에서 수정된 것이다.

## 주제 6: 변경 설명 4번 — 새 ORM 모델과 역할

기존 Chroma에서는 저장 구조를 Chroma가 관리해 PostgreSQL 벡터 모델 파일이 없었다. 이번에는 src/rag/models.py를 추가했다.

```python
from pgvector.sqlalchemy import Vector
from sqlalchemy import JSON, Text
from sqlalchemy.orm import Mapped, mapped_column

from fastapi_agent.core.database import Base
from rag.config import TABLE_NAME, VECTOR_SIZE


class RagChunk(Base):
    __tablename__ = TABLE_NAME

    id: Mapped[str] = mapped_column(Text, primary_key=True)
    content: Mapped[str] = mapped_column(Text)
    embedding: Mapped[list[float]] = mapped_column(Vector(VECTOR_SIZE))
    langchain_metadata: Mapped[dict | None] = mapped_column(JSON, nullable=True)
```

| 표현 | 뜻 |
|---|---|
| class RagChunk(Base) | 공통 Base에 연결되는 ORM 모델 |
| __tablename__ | 실제 DB 테이블 이름 |
| Mapped[str] | Python에서 문자열로 다룰 매핑 속성 |
| mapped_column(Text) | DB 컬럼 자료형·조건 정의 |
| primary_key=True | 행을 식별하는 기본키 |
| Mapped[list[float]] | Python에서는 실수들의 리스트 |
| Vector(VECTOR_SIZE) | DB에서는 지정 길이 벡터 자료형 |
| JSON | 딕셔너리 형태 메타데이터 저장 |
| dict 또는 None / nullable=True | 값 없음 허용 |

클래스 RagChunk와 테이블 spri_ai_brief_vectors는 이름이 달라도 된다. id에는 기존 chunk_id 문자열을, content에는 원문을, embedding에는 숫자 768개를, langchain_metadata에는 출처·페이지 등을 담는다.

```json
{
  "source": "문서.pdf",
  "page_number": 3,
  "chunk_id": "12"
}
```

```text
문서 조각 한 개 = 테이블 한 행
├─ id
├─ content
├─ embedding
└─ langchain_metadata
```

> ➕ **더 알아두기 — 왜 모델도 필요한가?**
> 우리 코드가 사용할 구조를 명시하고 Alembic의 모델 메타데이터에도 등록하기 위해 추가했다. 하지만 실제 문서 쓰기는 Session.add(RagChunk(...))가 아니라 PGVectorStore의 등록 메서드로 수행한다. ORM 모델은 정의, migration은 DB 생성·변경, ingest는 실제 데이터 등록이라는 역할을 구분한다.

```text
models.py 저장         → 원하는 구조 정의
Alembic upgrade 적용   → 실제 테이블 생성
rag-ingest 실행        → 문서·벡터 저장
```

> 📝 **기사 시험 포인트** (정보처리기사)
> 스키마(Schema), 기본키(Primary Key), 자료형(Data Type), NULL 허용 조건과 실제 행 데이터를 구분한다. ORM 코드 작성과 DDL 적용은 같은 시점의 작업이 아니다.

## 정리

이번 복습은 패키지·설정·임베딩·ORM 네 부분을 원본과 비교했다. PostgreSQL 벡터 도구는 추가했지만 기존 Chroma 자료는 보존했다. 폴더·컬렉션을 테이블 설정으로 바꾸고 문서·질문·DB 차원을 맞췄다. Gemini Embedding 2는 출력 차원을 지정할 수 있으며 768 선택이 최적이라는 실험 결론은 아니다. 모델은 구조 정의이고 실제 생성·등록과 구분한다. 다음 자세한 설명은 Alembic 변경 파일부터 이어갈 수 있다.

## ✅ 확인 질문

1. rag/config.py와 core/config.py는 각각 무엇을 관리하며 위치가 왜 중요한가?
2. 설정 상수를 적는 것과 실제 DB·API 작업이 실행되는 것은 어떻게 다른가?
3. langchain-postgres·asyncpg·Python pgvector와 DB vector 확장의 역할을 구분하라.
4. Chroma 패키지를 남겨둔 것과 활성 코드가 Chroma를 사용하는 것은 어떻게 다른가?
5. PERSIST_DIRECTORY·COLLECTION_NAME을 TABLE_NAME·VECTOR_SIZE로 바꾼 이유는 무엇인가?
6. VECTOR_SIZE는 문서 수인가? 임베딩 요청·DB 컬럼에 각각 어떻게 연결되는가?
7. Gemini Embedding 2의 기본 차원과 지원·권장 차원은 어떻게 다른가?
8. 차원이 커져도 품질·검색 시간이 단순 비례하지 않는 이유와 비교 실험 방법을 설명하라.
9. get_embeddings()와 embed_query()는 실행 시 무엇이 다른가?
10. 문서·질문·DB가 같은 차원이어야 하며 모델 호환성도 필요한 이유는 무엇인가?
11. RagChunk 모델의 id·content·embedding·langchain_metadata를 설명하라.
12. ORM 모델·Alembic migration·PGVectorStore 등록이 각각 언제 무엇을 하는가?
