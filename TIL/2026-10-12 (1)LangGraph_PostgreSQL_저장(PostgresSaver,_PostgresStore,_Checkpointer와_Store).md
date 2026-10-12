---
tags: [langgraph, postgresql, checkpointer, store, memory, psycopg, langchain]
til: v2 2026-10-12
---

# 2026-10-12_LangGraph_PostgreSQL_저장(PostgresSaver,_PostgresStore,_Checkpointer와_Store)
> 작성일: 2026-10-12

## 🔗 관련 글

- [Memory와 State 관리(Checkpointer, Store, 대화 요약)](2026-09-09_(2)_Memory와_State_관리(Checkpointer,_Store,_대화_요약).md) — `MemorySaver`, `InMemoryStore`, `thread_id`, namespace/key/value. 이 글의 개념은 그 글과 같고 **저장 장소만 메모리 → PostgreSQL로 바뀐다.**
- [Time Travel(Checkpoint, Replay, Fork)](2026-09-16_Time_Travel(Checkpoint,_Replay,_Fork).md) — 체크포인트 이력, 되돌아가기, 분기.
- [Human-in-the-Loop](2026-09-15_Human-in-the-Loop(interrupt,_Command,_Checkpointer).md) — 중간에 멈췄다가 재개할 때 Checkpointer가 쓰이는 이유.
- [LangGraph 기초(State, Node, Edge)](2026-09-08_LangGraph_기초(State_Node_Edge와_조건부분기_반복_Reducer).md) — State, 노드, 엣지.
- [DDL(PostgreSQL, 제약조건, 타입)](2026-10-09%20(1)숙소_예약_서비스_DDL(PostgreSQL,_제약조건,_타입).md) — `CREATE DATABASE`, `DEFAULT`, 복합 키, PostgreSQL에 `USE`가 없다는 것.
- [Transaction](2026-10-07%20(3)Transaction(BEGIN,_COMMIT,_ROLLBACK,_오류_처리).md) — `COMMIT`. 이 글의 `autocommit`과 연결된다.
- [Self JOIN](2026-10-07%20(1)Self_JOIN(같은_테이블_조인,_별칭,_댓글_대댓글).md) — 같은 테이블 안에서 부모를 가리키는 구조(체크포인트의 `parent_checkpoint_id`와 같은 모양).

## ❓ 아직 모르겠는 것

- 노트북 31번의 **셀 15~35(챗봇 실행)는 직접 실행한 결과를 확인하지 못했다.** 이 글의 "예상 결과"는 코드를 읽고 추정한 것이다. 이 컴퓨터에서 확인한 것은 **패키지 설치, 접속 성공, 테이블 4개 생성**까지이다.
- Store의 `prefix` 컬럼에 namespace `("users", "user-001")`가 **어떤 글자로 변환되어 저장되는지** 확인하지 못했다. (`SELECT * FROM store;`로 확인해야 한다.)
- 대화가 길어질 때 `checkpoint_blobs`의 `blob` 크기가 **실제로 커지는지**(아래 ➕의 용량 설명) 쿼리로 확인하지 못했다.
- `prepare_threshold=0`이 정확히 무슨 효과인지는 **노트북 주석("첫 실행부터 준비된 쿼리 사용")** 이상으로 확인하지 못했다.
- `row_factory=dict_row`가 **LangGraph에서 필수인지**는 소스에서 확인하지 못했다. 노트북대로 따라 쓴다.
- Store용 연결(`store_conn`)을 Checkpointer용 `conn`과 **따로 만든 이유**는 노트북에 적혀 있지 않다.
- 오래된 체크포인트를 **정리·삭제하는 공식 방법**은 확인하지 못했다.
- 이 파일의 ➕ 블록 중 "UUID", "SQL 방언", "보안" 부분은 라이브러리 소스를 읽고 **추론**한 것이라 실제 동작은 실험해 봐야 한다.

---

## 주제 1: 이번 수업이 하는 일 (수업 31번)

노트북 제목은 **"PostgreSQL로 대화와 프로필 저장하기"** 이다. 앞에서 `MemorySaver`, `InMemoryStore`로 **메모리에** 저장하던 대화와 프로필을 **PostgreSQL에** 저장한다. 프로그램을 껐다 켜도 기억이 남는다.

| | 이전 (메모리) | 이번 (PostgreSQL) |
|---|---|---|
| 대화 저장(Checkpointer) | `MemorySaver()` | `PostgresSaver(conn)` + `setup()` |
| 프로필 저장(Store) | `InMemoryStore()` | `PostgresStore(store_conn)` + `setup()` |
| 커널을 껐다 켜면 | 사라짐 | **남음** |

바뀌는 것은 **저장소 객체와 DB 연결 코드**뿐이다. `thread_id`, `namespace`, 챗봇을 만드는 방법은 그대로이다.

수업 순서(노트북 셀 번호)는 다음과 같다.

| 셀 | 내용 |
|---|---|
| 1~4 | 패키지 설치, 실습용 DB 만들기, `.env` 설정, 임포트와 접속 주소 만들기 |
| 5~6 | PostgreSQL 연결 확인 |
| 7~9 | Checkpointer 준비와 테이블 4개 |
| 10~19 | 챗봇 만들기, `thread_id`, 대화 저장, 이어 가기, 저장된 메시지 확인 |
| 20~21 | 커널 재시작 후 대화 복원 |
| 22~28 | Store 준비, 프로필 저장·조회 |
| 29~35 | Store를 쓰는 챗봇 그래프, Checkpointer와 Store를 함께 사용, 새 스레드와 비교 |
| 36~37 | DB 연결 닫기 |

> ➕ **더 알아두기 — 개념 정리 (대화 중에 정리한 비유이며 수업 내용이 아니다)**
>
> | 용어 | 이해한 비유 |
> |---|---|
> | **LLM** | 질문할 때마다 처음 만난 사람처럼 대하는 **일회용 함수.** 이전 대화를 같이 넣어 줘야 기억하는 것처럼 보인다 |
> | **챗봇** | LLM에 **대화 기록 관리**를 붙인 프로그램 |
> | **에이전트** | LLM이 **도구를 고르고**, 프로그램이 그 함수를 대신 실행하는 구조. 이번 노트북의 `create_agent(tools=[])`는 도구가 없어서 사실상 챗봇이다 |
> | **State** | 그래프가 **한 번 실행되는 동안** 들고 다니는 데이터 서류철. 챗봇에서는 메시지 목록(`MessagesState`) |
> | **Checkpointer** | `invoke`가 끝날 때 State를 저장하고, 다음 `invoke`에서 복원해 주는 장치 |
>
> LLM도 일회용이고 State도 `invoke` 한 번이 끝나면 사라지므로, 둘 사이를 이어 주는 **저장 장치(Checkpointer)** 가 있어야 챗봇이 기억한다.

---

## 주제 2: 준비 — 패키지, DB, `.env`, 접속 주소 (셀 1~4)

### 패키지 설치

```bash
pip install langgraph-checkpoint-postgres "psycopg[binary]"
```

| 패키지 | 역할 |
|---|---|
| `langgraph-checkpoint-postgres` | LangGraph가 PostgreSQL에 저장하게 해 주는 도구. `PostgresSaver`, `PostgresStore`가 들어 있다 |
| `psycopg` | 파이썬이 PostgreSQL과 통신하는 **드라이버.** `[binary]`는 설치가 쉬운 버전 |

### 실습용 DB 만들기

실행 중인 PostgreSQL에서 SQL로 직접 만든다. **이 셀은 노트북이 아니라 SQL로 실행해야 한다.**

```sql
CREATE DATABASE langgraph_prac;
```

### `.env`에 접속 정보 저장

```dotenv
POSTGRES_USER=postgres
POSTGRES_PASSWORD="YOUR_PASSWORD"
POSTGRES_DB=langgraph_prac
```

비밀번호는 원래 값으로 입력한다. 코드에서 사용자 이름과 비밀번호를 **URI용으로 인코딩**한 뒤 `postgresql://사용자:비밀번호@호스트:포트/DB이름` 형식으로 조합한다.

### 코드 (셀 4)

```python
load_dotenv(override=True)

POSTGRES_USER = os.getenv("POSTGRES_USER", "postgres")
POSTGRES_PASSWORD = os.getenv("POSTGRES_PASSWORD")
DB_NAME = os.getenv("POSTGRES_DB", "langgraph_prac")
if not POSTGRES_PASSWORD:
    raise ValueError(".env에 POSTGRES_PASSWORD를 설정하세요.")

user = quote(POSTGRES_USER, safe="")
password = quote(POSTGRES_PASSWORD, safe="")
db = quote(DB_NAME, safe="")

DB_URI = f"postgresql://{user}:{password}@localhost:5432/{db}"

MODEL_NAME = "gemini-3.6-flash"
llm = ChatGoogleGenerativeAI(model=MODEL_NAME)
```

| 코드 | 뜻 |
|---|---|
| `load_dotenv(override=True)` | `.env`를 읽어 환경 변수로 등록. `override=True`는 이미 있는 같은 이름의 값도 `.env` 값으로 덮어쓴다 |
| `os.getenv("이름", "기본값")` | 환경 변수를 꺼냄. 없으면 기본값 |
| `raise ValueError(...)` | **비밀번호가 없으면 일부러 오류로 멈춘다.** (`POSTGRES_USER`, `POSTGRES_DB`는 없어도 기본값으로 동작) |
| `quote(..., safe="")` | `@`, `/`, `:`, `#` 같은 특수문자를 `%40` 같은 안전한 코드로 바꿈. 비밀번호의 특수문자가 주소의 구분 기호로 오해되지 않게 한다 |
| `@localhost:5432` | **`localhost`는 이 코드를 실행하는 내 컴퓨터 자신**, `5432`는 PostgreSQL의 기본 **포트** |

> ♻️ 처음엔 `localhost`가 **"내 IP"** 라고 생각했으나, **"내 컴퓨터 자신"을 뜻하는 이름**이다. (숫자 주소는 `127.0.0.1`.) 공유기가 준 실제 IP와는 다르다. 포트는 한 컴퓨터 안에서 **어느 프로그램에 연결할지** 정하는 번호이다.

비밀번호를 코드에 직접 쓰는 것(**하드코딩**)은 깃에 올렸을 때 공개되므로 위험해서 `.env`에 분리한다. `.env`는 `.gitignore`에 등록되어 깃에 올라가지 않는다.

> 📝 **기사 시험 포인트** (정보보안기사, 확실도: 중간)
> 비밀번호·API 키 같은 **비밀 정보를 소스 코드에 하드코딩하지 않고** 환경 변수나 별도 설정 파일로 분리하는 것은 기본적인 보안 수칙으로 알고 있다. (시험에서 어떻게 묻는지는 확인하지 못했다.)

---

## 주제 3: DB 연결 확인과 Checkpointer 준비 (셀 5~9)

### 연결 확인 (셀 6)

```python
with psycopg.connect(DB_URI) as sql_conn:
    with sql_conn.cursor() as cursor:
        cursor.execute("SELECT current_database(), current_user")
        database, user = cursor.fetchone()
        print("데이터베이스:", database)
        print("접속 사용자:", user)
```

| 코드 | 뜻 (전화 비유) |
|---|---|
| `psycopg.connect(DB_URI)` | DB에 **접속**한다 (전화 걸기) |
| `sql_conn.cursor()` | SQL을 보내고 결과를 받는 **창구** (말하는 입) |
| `cursor.execute("SELECT ...")` | SQL을 **DB에 보냄** (질문하기) |
| `cursor.fetchone()` | 결과에서 **한 줄**을 가져옴 (대답 듣기) |
| `with ... as ...:` | 블록이 끝나면 **연결을 자동으로 닫는다** |

오늘 SQL 수업의 `SELECT current_database()`를 파이썬에서 보낸 것이다.

> ➕ **더 알아두기** — 셀 6의 `user`는 셀 4에서 만든 `user` 변수와 **이름이 같아서 덮어쓴다.** 이미 `DB_URI`를 만든 뒤라 문제는 없지만 같은 이름을 다시 쓰면 헷갈릴 수 있다.

### Checkpointer 준비 (셀 8)

```python
conn = psycopg.connect(
    DB_URI,
    autocommit=True,       # 변경 사항 즉시 반영
    prepare_threshold=0,   # 첫 실행부터 준비된 쿼리 사용
    row_factory=dict_row,  # 조회 결과를 딕셔너리로 반환
)
checkpointer = PostgresSaver(conn)
checkpointer.setup()

print("Checkpointer 테이블 준비 완료")
```

`conn`은 Python에서 PostgreSQL에 접속해 데이터를 읽고 쓸 때 사용하는 연결 객체이다. 이 연결로 `PostgresSaver`를 생성하고 이후 셀에서 재사용한다. `checkpointer.setup()`은 체크포인트 저장에 필요한 **테이블을 준비**한다.

| 옵션 | 의미 |
|---|---|
| `autocommit=True` | SQL을 실행하는 즉시 변경이 **확정(COMMIT)** 된다 |
| `prepare_threshold=0` | 노트북 주석: 첫 실행부터 준비된 쿼리를 사용 |
| `row_factory=dict_row` | 조회 결과를 `(1, "철수")` 튜플이 아니라 `{"id": 1, "name": "철수"}` **딕셔너리**로 반환 |

> ♻️ 처음엔 `autocommit=True`가 **"체크포인트를 자동으로 SQL로 저장해 준다"** 는 뜻인 줄 알았으나, **두 가지 "자동"을 섞은 것이다.**
>
> | | 누가 | 무엇을 |
> |---|---|---|
> | Checkpointer의 자동 저장 | LangGraph | 노드가 끝날 때마다 저장 요청을 DB에 보냄 |
> | `autocommit=True` | psycopg(DB 연결) | DB에 보낸 SQL의 변경을 **COMMIT 없이 바로 확정** |

`autocommit=False`(기본)에서는 `conn.commit()`을 직접 불러야 저장이 확정된다. 파이썬의 `conn.commit()`과 SQL의 `COMMIT;`은 **같은 커밋**이다. (10월 7일 Transaction 글 참고)

> ➕ **더 알아두기 — `autocommit=True`가 필수인 이유 (라이브러리 소스에서 확인)**
> `setup()`이 실행하는 SQL 목록에 `CREATE INDEX CONCURRENTLY`가 들어 있다. PostgreSQL에서 이 명령은 **트랜잭션 안에서 실행할 수 없는 명령**이라, `autocommit=True`가 아니면 오류가 날 수 있다. (PostgreSQL의 일반 규칙이며 이 노트북에서 직접 실행해 확인하지는 못했다.)

`checkpointer.setup()`은 **빈 테이블(틀)** 만 만든다. 대화 내용은 챗봇에 질문할 때 채워진다. (`CREATE TABLE`과 `INSERT`의 차이와 같다.) 테이블 정의에 `CREATE TABLE IF NOT EXISTS`가 쓰여서 **여러 번 실행해도 안전**하다.

### Checkpointer 테이블 4개 (셀 9)

| 테이블 | 역할 |
|---|---|
| `checkpoints` | 체크포인트의 상태 정보와 메타데이터 저장 |
| `checkpoint_blobs` | 메시지 목록 등 State 값의 **직렬화된 데이터** 저장 |
| `checkpoint_writes` | 노드 실행 중 발생한 중간 결과 저장. 복구할 때 활용 |
| `checkpoint_migrations` | Checkpointer 테이블에 적용한 스키마 변경 이력 관리 |

이 테이블들은 `PostgresSaver`가 자동으로 관리한다. (**이 컴퓨터에서 4개가 생긴 것을 확인했다.**)

> ➕ **더 알아두기 — 테이블 정의 (라이브러리 소스에서 확인)**
>
> ```sql
> CREATE TABLE IF NOT EXISTS checkpoints (
>     thread_id TEXT NOT NULL,
>     checkpoint_ns TEXT NOT NULL DEFAULT '',
>     checkpoint_id TEXT NOT NULL,
>     parent_checkpoint_id TEXT,
>     type TEXT,
>     checkpoint JSONB NOT NULL,
>     metadata JSONB NOT NULL DEFAULT '{}',
>     PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id)
> );
> ```
>
> `checkpoint_blobs`는 `(thread_id, checkpoint_ns, channel, version)`이 복합 PK이고 값은 `blob BYTEA`(바이너리) 컬럼에 들어간다. `checkpoint_writes`는 `(thread_id, checkpoint_ns, checkpoint_id, task_id, idx)`가 복합 PK이다. 10월 9일에 배운 **복합 PK**가 그대로 쓰인다.

---

## 주제 4: Checkpointer 실습 — 대화 저장과 복원 (셀 10~21)

### 챗봇 만들기 (셀 11)

```python
graph = create_agent(
    model=llm,
    tools=[],
    system_prompt=(
        "한국어로 간결하게 답하세요. "
        "사용자 정보는 대화 기록에 근거해 답하고, 없으면 모른다고 말하세요."
    ),
    checkpointer=checkpointer,
)
```

`create_agent`에 `PostgresSaver`를 연결한다. **`checkpointer=checkpointer` 한 줄이 대화를 DB에 저장되게 만든다.** `system_prompt`에 "없으면 모른다고 말하라"를 넣어 기억이 없을 때 지어내지 않게 한다.

### 대화 식별자 (셀 13)

```python
config = {"configurable": {"thread_id": "postgres-chat-demo2"}}
```

같은 `thread_id`로 대화를 이어 간다. 커널 재시작 후에도 이 값을 그대로 사용한다. `thread_id`는 **대화방 번호**(서랍 번호)이다.

### 대화 저장 → 이어 가기 → 확인 (셀 15, 17, 19)

```python
# 셀 15: 대화를 PostgreSQL에 저장하기
result = graph.invoke(
    {"messages": [("human", "내 이름은 민수야. PostgreSQL을 공부하고 있어.")]},
    config=config,
)
print(result["messages"][-1].text)
```

```python
# 셀 17: 같은 스레드에서 대화 이어 가기 (이전 메시지를 다시 보내지 않는다)
result = graph.invoke(
    {"messages": [("human", "내 이름과 공부 중인 내용을 알려줘.")]},
    config=config,
)
print(result["messages"][-1].text)
```

```python
# 셀 19: 저장된 메시지 확인하기
snapshot = graph.get_state(config)
for message in snapshot.values["messages"]:
    message.pretty_print()
```

| 코드 | 뜻 |
|---|---|
| `{"messages": [("human", "...")]}` | State에 넣을 값. `("human", ...)`은 사람이 한 말 |
| `result["messages"][-1].text` | 결과 메시지 목록의 **마지막(= AI의 답)** |
| `graph.get_state(config)` | 이 `thread_id`의 **최신 체크포인트** 조회 |
| `snapshot.values["messages"]` | 그 State 안의 메시지 목록 |

**셀 15는 한 번만 실행한다.** 같은 `thread_id`로 다시 실행하면 같은 질문이 대화에 또 추가된다.

### 커널 재시작 후 대화 복원 (셀 20~21)

커널 재시작 후 **환경 설정 → Checkpointer 준비 → 챗봇 만들기 → 대화 식별자 준비** 셀을 실행하고, 같은 `thread_id`로 아래 셀을 실행한다. 대화 저장 셀은 다시 실행하지 않는다.

```python
snapshot = graph.get_state(config)
if not snapshot.values.get("messages"):
    raise ValueError("저장된 대화가 없습니다. 대화 저장 실습과 thread_id를 확인하세요.")

for message in snapshot.values["messages"]:
    message.pretty_print()

result = graph.invoke(
    {"messages": [("human", "내가 공부 중인 주제로 짧은 복습 문제를 하나 내줘.")]},
    config=config,
)
print(result["messages"][-1].text)
```

커널을 재시작해도 파이썬 변수는 사라지지만 **기억은 PostgreSQL에 있으므로** 같은 `thread_id`로 복원된다. `ValueError` 검사는 저장된 대화가 없을 때 일부러 멈춰서 "복원 성공"으로 착각하지 않게 한다.

> ➕ **더 알아두기 — State는 `invoke` 한 번 동안만 산다**
> Checkpointer가 없으면 `invoke`를 새로 부를 때마다 **빈 State에서 시작**한다. State 자체에는 저장 기능이 없다.
>
> ```text
> [Checkpointer 없음]  invoke #1: 빈 State → 질문1+답1 → 끝 (State 소멸)
>                      invoke #2: 빈 State → 질문2+답2     (질문1을 모름)
> [Checkpointer 있음]  invoke #2 시작: 저장된 State를 불러옴 → 질문1,답1 + 질문2 → 답2 → 다시 저장
> ```
>
> **Checkpointer = `invoke`가 끝날 때 State를 저장하고, 다음 `invoke`가 시작될 때 복원해 주는 장치**이다.

---

## 주제 5: Store 실습 — 프로필 저장 (셀 22~28)

`PostgresStore`에 사용자 프로필을 저장한다. namespace와 key로 데이터를 구분한다. `PostgresStore`는 앞서 설치한 `langgraph-checkpoint-postgres` 패키지에 포함되어 있다. 기존 `DB_URI`로 연결하고 `store.setup()`으로 Store용 테이블을 준비한다. **Checkpointer와 같은 DB를 사용하지만 저장 테이블은 별도**이다.

### Store 준비 (셀 23)

```python
store_conn = psycopg.connect(
    DB_URI,
    autocommit=True,
    prepare_threshold=0,
    row_factory=dict_row,
)
store = PostgresStore(store_conn)
store.setup()

print("Store 테이블 준비 완료")
```

### Store 테이블 (셀 24)

| 테이블 | 역할 |
|---|---|
| `store` | namespace와 key별 JSON 데이터 저장 |
| `store_migrations` | Store 테이블에 적용한 스키마 변경 이력 관리 |

`store` 테이블의 `prefix`는 namespace를 문자열로 저장한 컬럼이고, `value`는 **JSONB** 타입이다. SQL은 저장 내용을 확인하는 용도로 사용하고, 프로필 저장과 수정은 Store 메서드로 수행한다.

> ➕ **더 알아두기 — `store` 테이블 정의 (라이브러리 소스에서 확인)**
>
> ```sql
> CREATE TABLE IF NOT EXISTS store (
>     prefix text NOT NULL,         -- 'prefix' represents the doc's 'namespace'
>     key text NOT NULL,
>     value jsonb NOT NULL,
>     created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     PRIMARY KEY (prefix, key)
> );
> ```
>
> **`PRIMARY KEY (prefix, key)`** 가 복합 기본키이다. 그래서 **같은 namespace와 key로 `put`하면 새 행이 아니라 기존 행이 갱신**된다. 내부 SQL도 `ON CONFLICT (prefix, key) DO UPDATE`이다.

### 프로필 저장과 조회 (셀 26, 28)

```python
user_id = "user-001"
user_namespace = ("users", user_id)

store.put(
    namespace=user_namespace,
    key="profile",
    value={"name": "철수", "interest": "Python"},
)
print("프로필 저장 완료")
```

```python
profile = store.get(user_namespace, "profile")
if profile is None:
    raise ValueError("저장된 프로필이 없습니다. 프로필 저장 셀을 먼저 실행하세요.")
print("namespace:", profile.namespace)
print("key:", profile.key)
print("value:", profile.value)
```

같은 namespace와 key로 다시 저장하면 기존 프로필이 **갱신**된다. 저장된 프로필은 연결을 닫아도 PostgreSQL에 남는다.

| 용어 | 뜻 | 비유 |
|---|---|---|
| **namespace** | 데이터를 묶는 **경로** | 폴더 (`("users", "user-001")`) |
| **key** | namespace 안에서 구분하는 이름 | 파일 이름 (`"profile"`) |
| **value** | 실제로 저장할 값 (딕셔너리) | 파일 내용 |

```text
("users", "user-001") / profile  → 철수의 프로필
("users", "user-002") / profile  → 영희의 프로필    (key가 같아도 namespace가 달라서 안 섞인다)
```

| `put` 두 번 | 결과 |
|---|---|
| namespace 같음 + key 같음 | **덮어씀** |
| namespace 같음 + key 다름 | 둘 다 남음 |
| namespace 다름 + key 같음 | 둘 다 남음 |

> ♻️ 처음엔 **Store가 여러 사람이 공동 작업할 때만 쓰는 것**이라고 생각했으나, **한 사람이어도 대화방(thread)이 여러 개면 필요하다.** Store가 필요한 이유는 "사람 수"가 아니라 **"대화방을 넘어서 정보를 유지"** 하기 위해서이다. 이번 노트북도 사용자는 `user-001` **한 명**이고, 새 대화방에서도 프로필을 아는지를 확인한다. namespace에 `user_id`를 넣는 것은 **나중에 사용자가 늘어도 데이터가 섞이지 않게 하는 관례**이다.

---

## 주제 6: Checkpointer와 Store를 함께 사용 (셀 29~35)

### Store를 사용하는 챗봇 그래프 (셀 30)

실행 설정의 `user_id`로 프로필을 조회하고 답변에 반영한다.

```python
def memory_chatbot(
    state: MessagesState,
    config: RunnableConfig,
    store: BaseStore,
):
    user_id = config["configurable"]["user_id"]
    profile = store.get(("users", user_id), "profile")

    system = (
        "한국어로 간결하게 답하세요. "
        "사용자 정보는 제공된 프로필과 대화에 근거해 답하고, 없으면 모른다고 말하세요."
    )
    if profile:
        system += (
            f"\n이름: {profile.value['name']}"
            f"\n관심 분야: {profile.value['interest']}"
        )
    else:
        system += "\n저장된 사용자 프로필이 없습니다."

    response = llm.invoke([SystemMessage(content=system), *state["messages"]])
    return {"messages": [response]}


profile_builder = StateGraph(MessagesState)
profile_builder.add_node("chatbot", memory_chatbot)
profile_builder.add_edge(START, "chatbot")
profile_builder.add_edge("chatbot", END)

profile_graph = profile_builder.compile(
    checkpointer=checkpointer,
    store=store,
)
```

| 노드 매개변수 | 정체 |
|---|---|
| `state: MessagesState` | 현재 스레드의 메시지가 들어 있는 State (Checkpointer가 복원한 것) |
| `config: RunnableConfig` | 호출 시 전달한 실행 설정. `user_id`를 여기서 읽는다 |
| `store: BaseStore` | `compile(store=store)`로 연결한 Store. LangGraph가 노드에 전달한다 |

`*state["messages"]`의 `*`는 목록을 풀어서 LLM에 같이 넘기는 파이썬 문법이다. **LLM이 일회용 함수라서, State에 들어 있는 이전 대화를 노드가 대신 챙겨서 매번 넣어 준다.**

### 호출과 비교 (셀 32, 33, 35)

```python
profile_config = {
    "configurable": {
        "thread_id": "postgres-profile-chat",
        "user_id": "user-001",
    }
}
```

`thread_id`는 **Checkpointer**가, `user_id`는 노드가 **Store** 조회에 쓴다. 이름과 관심 분야는 Store에서 읽고, 아래 대화의 취미는 Checkpointer의 State에 저장한다.

```python
result = profile_graph.invoke(
    {"messages": [("human", "나는 등산을 좋아해. 내 이름과 관심 분야도 알려줘.")]},
    config=profile_config,
)
print(result["messages"][-1].text)
print("저장된 프로필:", store.get(user_namespace, "profile").value)
```

셀 35는 **같은 `user_id`와 새 `thread_id`(`uuid4().hex`로 매번 새로 생성)** 로 기존 대화와 새 대화에 같은 질문("내 이름, 관심 분야, 취미를 알려줘. 모르는 정보는 모른다고 해줘.")을 한다.

| 정보 | 기존 대화 | 새 대화 | 이유 |
|---|---|---|---|
| 이름·관심 분야 | 안다 | **안다** | **Store**는 `user_id` 기준 |
| 취미(등산) | 안다 | **모른다** | **Checkpointer**는 `thread_id` 기준 |

> 위 표는 **노트북의 설명과 코드로 예상한 결과**이며 실행해서 확인하지 못했다.

### 연결 닫기 (셀 36~37)

```python
conn.close()
store_conn.close()
```

모든 실습을 마친 뒤 연결을 닫는다. 커널을 종료하거나 재시작해도 DB 연결은 끊기지만, **PostgreSQL에 저장된 데이터는 남는다.**

### Checkpointer vs Store 비교

| | **Checkpointer** | **Store** |
|---|---|---|
| 저장하는 것 | 대화 State(메시지 기록 + 진행 정보) | 내가 정한 정보(프로필 등) |
| 구분 기준 | `thread_id` (대화방) | `namespace + key` (사람/주제) |
| 저장 방법 | **자동** (노드가 끝날 때마다) | **수동** (`put`, `get`) |
| 같은 키로 또 저장 | **새 행이 쌓임**(이력) | **덮어씀** |
| 대화방이 바뀌면 | 모름 | 같은 `user_id`면 앎 |
| 연결 코드 | `PostgresSaver(conn)` | `PostgresStore(store_conn)` |
| 테이블 | `checkpoints` 등 4개 | `store`, `store_migrations` |
| 비유 | 방별 대화 일지 | 단골 카드 |

> ♻️ 처음엔 **"체크포인트나 Store나 똑같고 저장 방식만 다르다"** 고 정리했으나, **저장하는 대상(대화 State vs 내가 정한 정보)과 열쇠(`thread_id` vs `namespace+key`)도 다르다.** 저장 방식만 다른 것이 아니다.

---

## 주제 7: 이 컴퓨터에서 겪은 문제와 해결

컴퓨터를 옮긴 뒤 노트북을 실행하자 여러 오류가 났다. 원인은 모두 **"코드는 `git pull`로 오지만 설치한 라이브러리와 `.env`는 오지 않는다"** 는 것이었다.

| 증상 | 원인 | 해결 |
|---|---|---|
| 첫 셀에서 라이브러리가 없다는 오류, 주피터 커널이 없다는 안내 | 새 컴퓨터에 `langchain`, `langgraph`, `langchain-google-genai`, `python-dotenv`가 **설치되지 않음** | `pip install -r requirements.txt python-dotenv ipykernel` 후 **커널 재시작**. 노트북 커널이 설치한 파이썬(3.12)인지 확인 |
| `.env`가 `260811` 폴더에 없음 | `.env`는 `.gitignore`에 있어 **깃에 올라가지 않음** | 키 파일은 상위 폴더 `aim_home/.env` 한 곳에서 관리하고 있었다. `load_dotenv()`는 **실행 위치에서 위 폴더로 올라가며** `.env`를 찾는다 |
| 접속은 되는데 비밀번호 인증 실패 | `.env`의 `POSTGRES_PASSWORD`가 **이 컴퓨터의 PostgreSQL 비밀번호와 달랐음** | 맞는 비밀번호로 수정 |
| `psql: command not found` | `psql` 명령이 PATH에 등록되지 않음 | 문제 아님. 파이썬(`psycopg`)으로 접속하면 된다 |
| 셀에 `SELECT ...`를 넣었더니 `SyntaxError` | **노트북 셀은 파이썬**이라 SQL을 해석하지 못함 | SQL은 `.sql` 파일에서 실행하거나, 파이썬에서 `cursor.execute("""SELECT ...""")`로 보낸다 |

> ➕ **더 알아두기 — 하위 폴더에 `.env`를 새로 만들면 안 되는 이유**
> `load_dotenv()`는 가장 가까운 `.env`에서 멈춘다. `langchain_` 안에 `.env`를 따로 만들면 위 폴더의 `aim_home/.env`가 **가려져서** 읽히지 않는다.

---

## 주제 8: 질문하며 이해를 깊게 한 것

### Checkpoint에는 State만 있는가?

> ♻️ 처음엔 **체크포인트 = State와 메타데이터**라고 생각했으나, 체크포인트는 **State 값 + 버전·순서 정보 + 메타데이터**를 한 덩어리로 묶은 것이다. **대화 내용(메시지)도 체크포인트에 들어 있다.**

| 구분 | 내용 |
|---|---|
| **State 값**(`channel_values`) | `messages` 등 앱이 쓰는 데이터 (**대화 내용 본체**) |
| 버전 정보 | 값이 몇 번 바뀌었는지, 각 노드가 어느 버전까지 봤는지 |
| 식별·순서 | 체크포인트 번호(`id`), 시각(`ts`), 부모 번호 |
| 메타데이터 | `source`(input/loop/update/fork), `step`(몇 번째 단계, 첫 입력은 -1), `run_id` |

> ➕ **더 알아두기 — 왜 State만 저장하지 않고 체크포인트로 구분하는가 (라이브러리 정의에서 확인한 필드의 용도를 풀어 쓴 것이다)**
> State는 **한 순간의 내용물**이고, 체크포인트는 **내용물 + 언제·어디까지·몇 번째로 찍었는지**이다. 게임의 세이브 파일과 같다.
> - 과거로 되돌아가기·분기·이력 조회를 하려면 **순서 정보**가 필요하다.
> - 중간에 멈춘 곳에서 이어 가려면 **"다음에 실행할 노드"** 를 알아야 한다. (`versions_seen`은 다음에 실행할 노드를 정하는 데 쓰인다.)
> - 오류 복구에는 `checkpoint_writes`의 **중간 결과**가 필요하다.

### 체크포인트의 순서와 연결 (라이브러리 소스에서 확인)

> ♻️ 처음엔 C언어의 **연결 리스트처럼 다음 것을 가리킨다**고 생각했으나, 체크포인트는 **이전(부모) 것을 가리킨다.**

```text
연결 리스트:   ① ──next──▶ ② ──next──▶ ③
체크포인트:    ① ◀──parent── ② ◀──parent── ③
```

- 각 행에 `parent_checkpoint_id` 컬럼이 **실제로 저장된다.** (`NULL`이면 맨 처음)
- 체크포인트는 만들어질 때 **이미 있는 것만** 알 수 있어서 앞이 아니라 **뒤(부모)** 를 가리킨다.
- **최신 것은 정렬해서 찾는다.** `get_tuple()`의 SQL이 `ORDER BY checkpoint_id DESC LIMIT 1`이다.

| 방식 | 용도 |
|---|---|
| **부모 번호 저장** (`parent_checkpoint_id`) | 한 단계씩 거슬러 올라가기, **분기(fork)** 표현 |
| **번호 정렬** (`ORDER BY ... DESC LIMIT 1`) | **가장 최근** 체크포인트 가져오기 |

분기가 일어나면 부모가 같은 체크포인트가 둘이 되므로 번호 정렬만으로는 구조를 알 수 없다.

> ➕ **더 알아두기 — 체크포인트 번호는 시간이 들어간 UUID (라이브러리 소스에서 확인)**
> 소스에서 `id=str(uuid6(clock_seq=step))`로 생성한다. **UUIDv6**은 시간 정보가 앞쪽에 들어 있어서 **정렬하면 시간순**이 되는 고유 번호이다. 숫자 순번(1, 2, 3)이 아니다. 별도로 **`step`은 숫자**(0, 1, 2…)이고 `ts`는 만든 **시각**이다.
> 1, 2, 3 순번 대신 UUID를 쓰는 이유는 "여러 서버가 동시에 저장해도 번호가 겹치지 않아서"라고 **일반적인 이유로 설명**했지만, 개발자가 이 이유로 골랐다고 확인한 것은 아니다.
>
> 부모 연결과 번호 정렬은 오늘 배운 **자기 참조(Self JOIN) 구조**와 **`ORDER BY ... DESC LIMIT`** 이 그대로 쓰인 것이다. 다만 테이블 정의에 `REFERENCES`가 없어서 **`parent_checkpoint_id`는 DB가 무결성을 강제하는 FK가 아니다.**

### `checkpointer=checkpointer`만 하면 왜 알아서 저장되는가?

> ♻️ 처음엔 **앞에서 형식을 지정해서** 저장되는 줄 알았으나, **저장하고 불러오는 시점과 방법은 LangGraph 안에 이미 내장**되어 있고 `checkpointer=`는 **"어디에 저장할지(저장소 객체)"** 만 넘긴다. 콘센트(저장소)에 꽂는 것과 같다.

| 코드 | 어디 문법 |
|---|---|
| `create_agent(...)`, `checkpointer=`, `thread_id`, `invoke` | **LangChain/LangGraph** |
| `PostgresSaver`, `PostgresStore` | LangGraph가 제공하는 PostgreSQL용 도구 |
| `psycopg.connect`, `cursor.execute` | **psycopg** (파이썬 ↔ PostgreSQL) |
| `SELECT`, `CREATE TABLE`, `->>` | **PostgreSQL(SQL)** |

```text
[내 코드]   create_agent(..., checkpointer=...)         ← LangGraph 문법
[LangGraph] PostgresSaver가 체크포인트를 SQL로 변환       ← 변환은 여기서
[psycopg]   INSERT INTO checkpoints ... 전달
[PostgreSQL] 받은 SQL대로 행을 넣음 (체크포인트인지 모름)
```

> ➕ **더 알아두기 — 실제로 보내는 SQL (라이브러리 소스에서 확인)**
>
> ```sql
> INSERT INTO checkpoints (thread_id, checkpoint_ns, checkpoint_id, parent_checkpoint_id, checkpoint, metadata)
> VALUES (%s, %s, %s, %s, %s, %s)
> ON CONFLICT (thread_id, checkpoint_ns, checkpoint_id)
> DO UPDATE SET checkpoint = EXCLUDED.checkpoint, metadata = EXCLUDED.metadata;
> ```
>
> - **글자·숫자 같은 단순한 값**은 `checkpoints`의 `checkpoint` 컬럼(JSONB) 안에, **메시지 목록 같은 복잡한 객체**는 `checkpoint_blobs`의 `blob` 컬럼(바이너리)에 따로 저장된다. (**직렬화**: 파이썬 객체를 저장할 수 있는 형태로 바꾸는 것)
> - 불러올 때(`get`)는 `checkpoints`와 `checkpoint_blobs`를 **JOIN**해서 State를 복원한다. 오늘 배운 JOIN이 라이브러리에서 쓰인다.
> - **PostgreSQL은 똑똑하지 않다.** 체크포인트라는 개념, 순서 번호, 직렬화는 모두 **파이썬 쪽(LangGraph)** 이 처리하고 DB는 행을 넣고 빼는 저장소일 뿐이다.

### SQL은 DB마다 문법이 다른데?

맞다. 그래서 LangGraph는 **DB마다 Saver를 따로** 만들었다. (노트북: "저장 위치에 따라 메모리, SQLite, PostgreSQL 등의 Saver를 선택할 수 있다.")

| 구분 | 같은가? |
|---|---|
| 겉모습(사용법: `put`, `get`, `list`) | 같다 |
| 속(만드는 SQL) | **DB마다 다르다** |

위 SQL의 `ON CONFLICT ... DO UPDATE`, `JSONB`, `BYTEA`, `->>`는 **PostgreSQL 전용 문법**이다. (MySQL은 `ON DUPLICATE KEY UPDATE` 등 다르게 쓴다고 알고 있으나 직접 확인하지는 않았다.) 직접 쓰는 DB가 PostgreSQL이면 변환할 필요가 없다. **지원하는 Saver가 없는 DB**를 쓸 때만 그 DB용 Saver를 직접 만들어야 한다. (보통은 지원되는 DB를 골라 쓴다.)

### State에 대화 전체가 들어 있으면 용량이 계속 커지지 않는가?

> ➕ **더 알아두기 — 용량 문제 (소스 구조에서 풀어 쓴 설명이며 실제 크기는 쿼리로 확인해야 한다)**
> 최신 체크포인트의 State에는 **지금까지의 대화가 전부** 들어 있다(전체 스냅샷). `put()`은 메시지 목록을 값이 바뀔 때마다 **새 버전의 행**으로 `checkpoint_blobs`에 **통째로** 저장하고, **과거 행도 지우지 않는다.** 그래서 대화가 길수록 점점 더 커진다.
>
> ```python
> cur.execute("""
>     SELECT version, length(blob) AS bytes
>     FROM checkpoint_blobs
>     WHERE thread_id = %s AND channel = 'messages'
>     ORDER BY version
> """, ("postgres-chat-demo2",))
> ```
>
> 용량보다 큰 문제는 **매번 전체 대화를 LLM에 같이 보낸다**는 점이다. 토큰 **비용·속도·최대 길이 한도**에 영향을 준다. 일반적인 대응은 **최근 것만 유지**, **요약**, **중요한 정보는 Store로**, **오래된 체크포인트 정리**이다. (이번 노트북에는 나오지 않는다. 라이브러리 소스에는 변화분만 저장하는 베타 기능으로 보이는 `DeltaChannel` 설명이 있으나 이번 수업과는 무관하다.)
>
> 용량 문제는 **저장 장소를 메모리로 바꿔도 똑같이 생긴다.** DB를 쓰는 이유는 용량이 아니라 **재시작해도 남고, 여러 서버가 공유하고, 트랜잭션으로 안전하고, SQL로 조회·백업할 수 있기 때문**이다.

### 최신 체크포인트 1개만 저장하면 안 되는가?

> ➕ **더 알아두기 — 이번 노트북 같은 단순 챗봇에서는 최신 1개로도 충분하다 (대화 중 정리한 판단이며 공식 설명이 아니다)**
> 메시지 목록 하나뿐이고 아무것도 지우지 않으면, 최신 1개에서 **앞부분을 잘라** 과거 시점을 재현할 수 있다. 그러나 다음 경우에는 과거 체크포인트가 필요하다.
> 1. 메시지를 **삭제·요약**해서 줄이는 경우 (노트북 13번에도 "삭제된 메시지는 현재 State에서 사라지지만 Checkpointer 히스토리에는 과거 스냅샷이 남아 있다"는 문장이 있다)
> 2. State에 메시지 말고 **덮어쓰는 칸**(요약문, 카운터 등)이 있는 경우
> 3. 노드가 여러 개라 **"그래프의 어디까지 갔는지"** 가 필요한 경우 (사람 승인 대기, 오류 복구)
>
> LangGraph는 단순 챗봇부터 복잡한 에이전트까지 대응하려고 기본값을 **"다 남기기"** 로 둔 것이다. 그리고 "몇 개를 저장할까"는 설계 문제이고 "어디에 저장할까"는 별개여서, 1개만 저장하더라도 수많은 대화방의 최신 상태를 안전하게 보관하려면 DB가 여전히 유용하다.

### 체크포인트에 가짜 대화를 끼워 넣으면 컨텍스트가 오염되는가?

> ➕ **더 알아두기 — 보안 관점 (소스 구조에서 추론한 것이며 직접 실험하지는 않았다)**
> - 이 구조에는 **변조를 막는 장치가 없다.** `parent_checkpoint_id`에 FK가 없고, 서명·해시 컬럼도 없으며, 불러올 때 **부모 연결을 검증하지 않는다.** 연결 끊긴 행도 이력 목록(`list`)에는 번호순으로 나온다.
> - **과거 자리에 끼워 넣기**는 컨텍스트를 거의 오염시키지 못한다. 이유는 부모 연결이 막아 줘서가 아니라 ① 최신을 **번호 정렬**로 고르고 ② 체크포인트 하나가 **전체 스냅샷**이라 과거 행을 합쳐서 쓰지 않기 때문이다.
> - 위험한 것은 **최신보다 큰 번호로 삽입**하거나 **최신 행의 내용을 수정**하는 것, 그리고 **과거 체크포인트를 불러오도록 유도**하는 것이다.
> - **진짜 방어는 DB 접근 통제**(최소 권한 계정, 방화벽, 강한 비밀번호, 감사 로그)이다. 부모 id 연결은 보안 장치가 아니라 **이력 관리용**이다.
>
> 이런 공격을 **컨텍스트(기억) 오염(poisoning)** 이라고 부른다.

> 📝 **기사 시험 포인트** (정보보안기사·정보처리기사, 확실도: 중간)
> - 보안의 3요소 **CIA**: **기밀성**(Confidentiality), **무결성**(Integrity), **가용성**(Availability). 데이터가 변조되지 않는 성질이 무결성이며, 해시·전자서명·접근통제로 지킨다고 알고 있다.
> - **SQL 삽입(SQL Injection)** 의 대응책으로 **매개변수화된 쿼리(Parameterized Query, Prepared Statement)** 가 있다. 값을 SQL 문자열에 이어 붙이지 않고 `%s` 자리표시자로 **따로 전달**해 입력이 명령이 아닌 **데이터로만** 처리되게 한다.
>
> ```python
> # ❌ 위험: 문자열로 직접 이어 붙임
> cur.execute(f"SELECT * FROM checkpoints WHERE thread_id = '{x}'")
> # ✅ 안전: 값을 따로 전달
> cur.execute("SELECT * FROM checkpoints WHERE thread_id = %s", (x,))
> ```
>
> (이 사례가 시험에서 어떻게 출제되는지는 확인하지 못했다.)

### 결국 LLM에는 글자를 붙여서 보내는 것 아닌가?

맞다. **LLM에게 가는 것은 결국 "글자(메시지 목록)"를 이어 붙인 것**뿐이다. 이번 노트북에서도 그렇다.

| 저장소 | LLM에 들어가는 모습 |
|---|---|
| Checkpointer | 복원한 `state["messages"]`를 `*state["messages"]`로 **풀어서 그대로 이어 붙임** |
| Store | 프로필을 꺼내 `system += f"\n이름: {profile.value['name']}"`처럼 **시스템 지시문 글자에 덧붙임** |

```text
LLM에 보내는 것 = [시스템 지시 글자(+ Store 내용)] + [대화 기록 글자(Checkpointer 내용)] + 새 질문
```

> ➕ **더 알아두기 — 단순한 원리를 왜 복잡하게 관리하는가 (대화 중 정리한 설명이다)**
> LLM 쪽은 단순하다. **"받은 글자를 읽고 답한다"** 가 전부이고 기억이 없다. 복잡해지는 것은 LLM이 아니라 **LLM에 보낼 글자를 준비하는 쪽**이다.
>
> | 관리해야 하는 일 | 이유 |
> |---|---|
> | **어디에 보관하나** | 껐다 켜도 남고 여러 서버가 같이 봐야 한다 |
> | **누구의 것인가** | 사용자·대화방이 수천 개여도 섞이면 안 된다 (`thread_id`, `namespace`) |
> | **무엇을 보낼까** | 전부 보내면 토큰 비용·속도·길이 한도가 문제다 (요약, 최근 것만, Store로 분리) |
> | **중간에 멈추면** | 오류·승인 대기 후 이어서 해야 한다 (체크포인트, 중간 결과) |
> | **안전한가** | 변조·접근 권한 문제 |
>
> 식당에 비유하면 **손님의 말은 단순하지만, 주문서를 보관하고 테이블별로 구분하고 주방에 전달하는 시스템**이 필요한 것과 같다. 이번 노트북처럼 한 명이 짧게 쓰면 과해 보이지만 **사용자가 많아지고 대화가 길어질 때를 위한 구조**이다.

### DB에서 직접 확인하는 쿼리

노트북 셀에서 실행하려면 파이썬으로 감싼다.

```python
with psycopg.connect(DB_URI) as c:
    with c.cursor() as cur:
        cur.execute("""
            SELECT checkpoint_id, parent_checkpoint_id,
                   metadata->>'source' AS source, metadata->>'step' AS step
            FROM checkpoints
            WHERE thread_id = %s
            ORDER BY checkpoint_id
        """, ("postgres-chat-demo2",))
        for row in cur.fetchall():
            print(row)
```

`metadata->>'source'`는 JSONB 컬럼에서 값을 글자로 꺼내는 PostgreSQL 문법이다. 각 행의 `parent_checkpoint_id`가 바로 위 행의 `checkpoint_id`와 같은지 확인한다. (오늘 DB에 연결된 테이블 이름 목록은 `information_schema.tables`로 조회할 수 있다.)

---

## 정리

- **이번 수업**: 메모리(`MemorySaver`, `InMemoryStore`)에 저장하던 대화와 프로필을 **PostgreSQL**(`PostgresSaver`, `PostgresStore`)에 저장해 **껐다 켜도 기억이 남게** 한다. 개념은 그대로이고 **저장소 객체와 DB 연결 코드**만 바뀐다.
- **준비**: 패키지 설치 → `CREATE DATABASE` → `.env`(비밀번호는 코드에 쓰지 않는다) → `quote()`로 인코딩해 접속 주소 `postgresql://사용자:비밀번호@localhost:5432/DB` 조립.
- **연결**: `psycopg.connect` → `conn`. 옵션 `autocommit=True`(COMMIT 없이 즉시 확정, `CREATE INDEX CONCURRENTLY`가 있어서 필수), `dict_row`(딕셔너리로 반환).
- **`setup()`**: 저장용 **빈 테이블**을 만든다(`IF NOT EXISTS`라서 여러 번 실행해도 안전). Checkpointer 4개(`checkpoints`, `checkpoint_blobs`, `checkpoint_writes`, `checkpoint_migrations`), Store 2개(`store`, `store_migrations`).
- **Checkpointer**: `create_agent(..., checkpointer=...)`로 꽂으면 **노드가 끝날 때마다 자동 저장**, 같은 `thread_id`로 부르면 **최신 체크포인트를 복원**한다. State는 `invoke` 한 번 동안만 사는 서류철이고 Checkpointer가 이를 이어 준다.
- **체크포인트 = State 값 + 버전·순서 + 메타데이터.** 부모 번호(`parent_checkpoint_id`)를 저장하고, 최신은 `ORDER BY checkpoint_id DESC LIMIT 1`로 찾는다. 번호는 **시간이 들어간 UUID**이다.
- **Store**: `put`(직접 저장, 같은 `namespace+key`면 **덮어씀**) / `get`. `store` 테이블은 `PRIMARY KEY (prefix, key)`이다. **한 사람이어도 대화방이 여러 개면 필요하다.**
- **함께 사용**: `compile(checkpointer=..., store=...)`. `config`에 `thread_id`(Checkpointer용)와 `user_id`(Store용)를 같이 넣는다. 새 대화방에서도 **프로필(Store)은 알고 대화 내용(Checkpointer)은 모른다.**
- **LangGraph가 똑똑하고 DB는 저장소**: 체크포인트를 SQL로 변환하는 것은 LangGraph의 `PostgresSaver`이다. SQL은 DB마다 문법이 달라서 **DB별 Saver**가 있다.
- **용량·보안**: 대화가 길수록 `blob`과 LLM 토큰이 늘어난다(요약·정리·Store 활용). 체크포인트 구조에는 변조 방지 장치가 없어서 **DB 접근 통제**가 방어이다.
- **환경 이동**: 라이브러리와 `.env`는 `git pull`로 오지 않는다. `.env`는 `aim_home/.env` 한 곳에서 공유하고, SQL은 노트북 셀이 아니라 `.sql` 파일이나 `cursor.execute`로 실행한다.

---

## ✅ 확인 질문

1. LLM을 "일회용 함수"라고 부르는 이유는? 챗봇이 이전 대화를 기억하는 것처럼 보이게 하려면 무엇을 해야 하는가?
2. 챗봇과 에이전트의 차이는? `create_agent(tools=[])`가 사실상 챗봇인 이유는?
3. State는 언제까지 살아 있는가? Checkpointer가 없으면 두 번째 `invoke`에서 첫 질문을 모르는 이유는?
4. 이번 수업에서 `MemorySaver()`와 `PostgresSaver(conn)`의 차이는? 바뀌는 코드와 그대로인 코드를 구분하라.
5. 비밀번호를 코드에 직접 쓰지 않고 `.env`에 두는 이유는? `.env`가 `git pull`로 오지 않는 이유는?
6. `postgresql://사용자:비밀번호@localhost:5432/DB이름`에서 `localhost`와 `5432`는 각각 무엇인가?
7. `quote(..., safe="")`는 왜 쓰는가?
8. `.env`에 `POSTGRES_PASSWORD`가 없을 때와 있는데 **틀렸을 때** 각각 어떤 오류가 나는가? (`raise ValueError` vs 인증 실패)
9. `connect`, `cursor`, `execute`, `fetchone`은 각각 무슨 일을 하는가?
10. `autocommit=True`의 뜻은? Checkpointer의 "자동 저장"과 어떻게 다른가? 이 설정이 필수인 이유는?
11. `dict_row`를 쓰면 조회 결과가 어떤 모양으로 오는가?
12. `checkpointer.setup()`은 무엇을 하는가? 두 번 실행해도 안전한 이유는? 대화 내용은 언제 채워지는가?
13. Checkpointer 테이블 4개의 역할을 말하라. 메시지 목록 본체는 어느 테이블에 저장되는가?
14. `checkpointer=checkpointer`만 해도 저장되는 이유는? 이 줄은 어느 문법인가(PostgreSQL / LangGraph)? 체크포인트를 SQL로 변환하는 것은 누구인가?
15. `thread_id`를 다른 값으로 바꾸면 어떻게 되는가? 커널을 재시작해도 대화가 복원되는 이유는?
16. 체크포인트에는 어떤 것들이 들어 있는가? State와의 차이는? 왜 State만 저장하지 않는가?
17. 체크포인트가 **이전(부모)** 을 가리키는 이유는? 최신 체크포인트는 어떻게 찾는가? 부모 번호와 정렬은 각각 언제 쓰이는가?
18. `checkpoint_id`는 숫자 순번인가? 어떻게 만들어지며 왜 시간순 정렬이 되는가? `step`과는 어떻게 다른가?
19. SQL은 DB마다 문법이 다른데 LangGraph는 이를 어떻게 해결했는가? 직접 Saver를 만들어야 하는 경우는?
20. Store의 namespace, key, value를 폴더·파일에 비유해 설명하라. 같은 namespace·key로 `put`을 두 번 하면 DB에서는 어떻게 되는가? (PK 관점)
21. `store` 테이블에서 `prefix`는 무엇인가? `(prefix, key)`가 복합 PK인 이유는?
22. Store는 여러 사람이 공동 작업할 때만 쓰는가? 사용자가 한 명이어도 필요한 이유는?
23. 셀 35에서 새 대화방이 이름·관심 분야는 알지만 취미는 모르는 이유는? `thread_id`와 `user_id`는 각각 어느 저장소가 쓰는가?
24. Checkpointer와 Store를 저장 대상, 열쇠, 저장 방법, 같은 키로 또 저장했을 때로 비교하라.
25. 대화가 길어지면 용량이 커지는 이유는? DB에 저장하는 이유가 용량 때문이 아닌 이유는? 용량을 관리하는 일반적인 방법은?
26. 최신 체크포인트 1개만 저장해도 되는 경우와 과거 체크포인트가 필요한 경우를 구분하라.
27. 체크포인트 테이블에 가짜 대화를 끼워 넣으면 컨텍스트가 오염되는가? 어떤 경우가 위험하고 진짜 방어는 무엇인가?
28. 노트북 셀에 `SELECT ...`를 그대로 넣으면 `SyntaxError`가 나는 이유는? SQL을 실행하는 방법 두 가지는?
29. (기사 대비) SQL 삽입을 막는 방법과 `%s` 자리표시자의 원리는? 보안의 3요소 중 데이터 변조와 관련된 것은?
