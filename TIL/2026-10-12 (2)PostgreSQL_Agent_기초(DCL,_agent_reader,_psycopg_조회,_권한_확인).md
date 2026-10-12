---
tags: [langgraph, postgresql, dcl, psycopg, agent, text-to-sql, security]
til: v2 2026-10-12
---

# 2026-10-12_PostgreSQL_Agent_기초(DCL,_agent_reader,_psycopg_조회,_권한_확인)
> 작성일: 2026-10-12

## 🔗 관련 글

- [LangGraph PostgreSQL 저장(PostgresSaver, PostgresStore)](2026-10-12%20(1)LangGraph_PostgreSQL_저장(PostgresSaver,_PostgresStore,_Checkpointer와_Store).md) — 같은 날 앞의 수업(31번). `psycopg.connect`, `.env`, 접속 주소(`DB_URI`), `quote`, `autocommit`, `dict_row`가 먼저 나온다.
- [SQL 기초와 DDL(명령어 분류, 트랜잭션, 제약조건)](2026-10-02%20(4)SQL_기초와_DDL(명령어_분류,_트랜잭션,_제약조건).md) — DDL/DML/DCL 분류, 트랜잭션.
- [Transaction](2026-10-07%20(3)Transaction(BEGIN,_COMMIT,_ROLLBACK,_오류_처리).md) — `COMMIT`, `ROLLBACK`. 이 글의 `force_rollback`과 연결된다.
- [데이터베이스 설계(키, 무결성, 정규화, ERD 작성)](2026-10-07%20(5)데이터베이스_설계(키,_무결성,_정규화,_ERD_작성).md) — 속성(컬럼), 튜플(행), 릴레이션(테이블) 용어.
- [숙소 예약 서비스 화면별 쿼리(JOIN 추적, GROUP BY)](2026-10-09%20(2)숙소_예약_서비스_화면별_쿼리(JOIN_추적,_GROUP_BY).md) — 이 글에서 쓰는 `SELECT`, `LIMIT`, JOIN의 기초.

## ❓ 아직 모르겠는 것

- **이 글은 수업 32번(`32-postgres-agent.ipynb`)의 `DB 권한 확인` 구간(반복문 안쪽의 `try / except / else`와 `with conn.transaction(force_rollback=True)`)까지 정리했다.** 이후 구간(읽기 전용 조회 함수 `fetch_rows`, `%s` 바인딩과 `@tool`, 고정 SQL 도구, Text-to-SQL, SQL 검사, 재시도 그래프, 실습)은 아직 진행하지 못했다. 이어서 이 파일에 보강한다.
- **다음에 이어서 볼 곳**: `DB 권한 확인`의 마지막 `except` / `else` 줄을 코드 그대로 다시 읽고, 그다음 구간인 `SQL 조회 함수`(`conn.read_only = True`, `fetch_rows`)부터 진행한다.
- **VS Code PostgreSQL 확장이 SQL을 실행할 때 자동으로 확정(autocommit)하는지**는 설정으로 확인하지 못했다. `CREATE ROLE`, `GRANT`를 `COMMIT` 없이 실행하고 바로 접속이 됐다는 사실로 자동 확정으로 동작한다고 **추정**했다.
- 아래 "주제 6"의 **"`commit()` 없이 연결을 닫으면 변경이 남지 않는다"** 와 `autocommit=False`에서 `transaction()`이 이미 시작된 트랜잭션 안에서 쓰일 때의 동작(중첩)은 일반 지식이고 직접 실행해 확인하지 못했다.
- **다른 컴퓨터에서 이어서 하려면** 필요한 것: ① 이 저장소와 `aim-ai-agent-practice`를 `git pull`, ② 라이브러리 설치(`pip install -r requirements.txt python-dotenv ipykernel`), ③ `aim_home/.env`(Gemini 키, `POSTGRES_PASSWORD`, `AGENT_POSTGRES_PASSWORD`), ④ 그 컴퓨터의 PostgreSQL에 **`world` DB가 있고 `agent_reader` 계정이 만들어져 있어야 한다**(계정과 DB는 DB 안에 있어서 `git pull`로 오지 않는다).
- 노트북 셀을 **직접 실행한 결과**는 확인하지 못했다. 이 글의 출력 예시 중 "직접 실행해 확인"이라고 표시하지 않은 것은 코드로 추정한 것이다.
- `ORDER BY` 없이 `LIMIT 3`을 하면 **어떤 3줄이 나오는지는 보장되지 않는다.** 노트북 예시의 도시 이름은 확인한 결과가 아니라 정렬해서 본 값이다.
- psycopg의 **서버 쪽 커서**(결과가 아주 클 때 한 줄씩 받는 방식)와 `for row in 커서:` 반복은 일반 지식으로만 알고 직접 확인하지 못했다.
- `GRANT CONNECT ON DATABASE`가 **꼭 필요한지**(기본적으로 모두에게 열려 있는 경우)는 확인하지 못했다.

---

## 주제 1: 이번 수업이 하는 일 (수업 32번)

노트북 제목은 **"PostgreSQL을 활용하는 Agent"** 이다. 자연어로 질문하면 LLM이 `world` DB를 조회해서 답하는 에이전트를 만든다. 두 가지 방식을 비교한다.

| | **방식 A: 고정 SQL 도구** | **방식 B: Text-to-SQL** |
|---|---|---|
| SQL은 누가 쓰나 | **개발자가 미리** 작성 | **LLM이** 질문마다 생성 |
| LLM이 결정하는 값 | 도구와 인자 | 컬럼, 조건, JOIN, 집계 |
| 새 조회 요구가 오면 | 도구를 추가·수정 | 스키마를 바탕으로 SQL 생성 |
| 비유 | 메뉴판에서 고르기 | 주문서를 직접 써서 주기 |

노트북 설명: Text-to-SQL은 자연어 질문을 SQL로 변환하는 방식이다. `스키마 조회 → SQL 생성 → 검사·실행 → 답변`의 흐름이고, 실행 오류가 나면 `실행 오류 → SQL 재생성 → 재실행`으로 재시도한다.

### 왜 위험한가, 안전장치 3겹

노트북: "LLM은 잘못된 SQL이나 의도하지 않은 변경 명령을 생성할 수도 있다. 따라서 실행 전에 SQL을 검사하고, DB 계정의 권한과 읽기 전용 설정으로 데이터 변경을 제한한다."

프롬프트로 "변경하지 마"라고 부탁하는 것만으로는 LLM이 어기거나 속아 넘어갈 수 있어서 **DB·코드 차원의 방어**가 필요하다.

| 겹 | 방법 | 어디서 막나 | 노트북 구간 |
|---|---|---|---|
| ① | **DB 계정 권한** (조회만 허용하는 전용 계정) | DB가 막음 | `DCL`, `Agent용 DB 계정` |
| ② | **읽기 전용 연결** (`conn.read_only = True`) | 연결(트랜잭션)이 막음 | `SQL 조회 함수` |
| ③ | **SQL 검사** (`sqlglot`로 `SELECT` 한 개인지 확인) | 실행 전 코드가 막음 | `변경 SQL 차단` |

한 겹이 뚫려도 다음 겹이 막도록 쌓는다.

> 📝 **기사 시험 포인트** (정보보안기사, 확실도: 높음)
> 보안 장치를 한 곳에만 두지 않고 **여러 겹으로** 쌓아서 하나가 뚫려도 피해를 막는 원칙을 **다층 방어(심층 방어, Defense in Depth)** 라고 한다. 또 업무에 필요한 **최소한의 권한만** 주는 원칙을 **최소 권한의 원칙(Principle of Least Privilege)** 이라고 한다. (시험에서 이 사례가 어떻게 나오는지는 확인하지 못했다.)

사용하는 DB는 `world`의 테이블 3개이다.

| 테이블 | 내용 |
|---|---|
| `country` | 국가 (코드, 이름, 대륙, 인구…) |
| `city` | 도시 (이름, 국가 코드, 인구…) |
| `countrylanguage` | 국가별 언어 |

노트북이 직접 덧붙이는 연결 관계(이 DB에는 FK가 선언돼 있지 않다)는 다음과 같다.

```text
city.countrycode → country.code
countrylanguage.countrycode → country.code
country.capital → city.id
```

> ➕ **더 알아두기 — 이 컴퓨터에서 확인한 행 수 (직접 실행)**
> `country` 239행, `city` 4,079행, `countrylanguage` 984행.

---

## 주제 2: DCL — 데이터 접근 권한 제어 (`DCL`, `Agent용 DB 계정`)

노트북: DCL(Data Control Language, 데이터 제어어)은 DB에서 **누가 어떤 작업을 할 수 있는지** 권한을 관리하는 SQL이다. 데이터 조회(`SELECT`), 추가(`INSERT`), 수정(`UPDATE`), 삭제(`DELETE`) 등의 권한을 부여하거나 회수할 수 있다.

| 분류 | 이름 | 하는 일 | 예 |
|---|---|---|---|
| DDL | 데이터 정의어 | 테이블 구조를 만듦 | `CREATE`, `ALTER`, `DROP` |
| DML | 데이터 조작어 | 데이터를 다룸 | `SELECT`, `INSERT`, `UPDATE`, `DELETE` |
| **DCL** | **데이터 제어어** | **권한 관리** | **`GRANT`, `REVOKE`** |

| 명령 | 역할 | 예시 |
|---|---|---|
| `GRANT` | 권한 부여 | `GRANT SELECT ON country TO agent_reader;` |
| `REVOKE` | 부여된 권한 회수 | `REVOKE SELECT ON country FROM agent_reader;` |

### Agent용 DB 계정 만들기

권한 설정은 **관리자(`postgres`)가** 수행하고, Agent는 이 계정으로 DB에 연결한다. SQL 파일을 만들어 **`world`에 연결한 뒤** 실행한다. (노트북 셀이 아니라 `.sql` 파일에서 실행해야 한다.)

```sql
CREATE ROLE agent_reader LOGIN PASSWORD 'YOUR_AGENT_PASSWORD';
GRANT CONNECT ON DATABASE world TO agent_reader;
GRANT USAGE ON SCHEMA public TO agent_reader;
GRANT SELECT ON country, city, countrylanguage
TO agent_reader;
```

| 줄 | 뜻 | 비유 |
|---|---|---|
| `CREATE ROLE agent_reader LOGIN PASSWORD '...'` | **로그인할 수 있는 계정**을 만든다 (PostgreSQL에서 `ROLE`은 계정을 뜻한다) | 출입증 발급 |
| `GRANT CONNECT ON DATABASE world` | `world` DB에 **접속**할 권한 | 건물 정문 통과 |
| `GRANT USAGE ON SCHEMA public` | 스키마 안의 테이블·뷰 등 **객체에 접근**할 권한 | 층에 들어갈 권한 |
| `GRANT SELECT ON country, city, countrylanguage` | 세 테이블을 **조회**할 권한 | 방을 열람할 권한 |

`INSERT`, `UPDATE`, `DELETE`는 **부여하지 않았다.** 노트북: "데이터 변경 권한은 부여하지 않는다."

> ♻️ 처음엔 **"비밀번호를 만들라는 건 회원가입하라는 것인가"** 라고 생각했으나, 인터넷 서비스 가입이 아니라 **내 컴퓨터의 PostgreSQL 안에 접속용 계정 하나를 내가 직접 만드는 것**이다. `YOUR_AGENT_PASSWORD`는 노트북 예시의 **자리표시자**(내가 정한 비밀번호로 바꿔 쓰라는 표시)이다. 관리자 계정(마스터키) 대신 **조회만 되는 임시 출입증**을 AI에게 주는 것이다.

> ♻️ 자리표시자를 **바꾸지 않고 그대로 실행**해서, 계정이 `YOUR_AGENT_PASSWORD`라는 글자를 비밀번호로 해서 만들어졌다. 같은 `CREATE ROLE`을 다시 실행하면 "이미 존재한다"는 오류가 난다. **계정을 다시 만들지 않고 비밀번호만 바꾸는 명령**이 있다.
>
> | 명령 | 뜻 |
> |---|---|
> | `CREATE ROLE` | 계정을 **새로 만든다** (이미 있으면 오류) |
> | `ALTER ROLE agent_reader PASSWORD '새 비밀번호';` | **이미 있는 계정을 수정한다** (비밀번호 변경) |
>
> `GRANT`는 이미 있는 권한을 또 줘도 오류가 나지 않는 것으로 알고 있어서 다시 실행해도 안전하다. 바꾼 비밀번호는 `.env`의 `AGENT_POSTGRES_PASSWORD`에도 **똑같이** 넣어야 한다.

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음~중간)
> - SQL 분류: **DDL**(`CREATE`, `ALTER`, `DROP`, `TRUNCATE`), **DML**(`SELECT`, `INSERT`, `UPDATE`, `DELETE`), **DCL**(`GRANT`, `REVOKE`). `COMMIT`·`ROLLBACK`은 교재에 따라 DCL에 포함하기도 하고 **TCL**(트랜잭션 제어어)로 따로 분류하기도 한다.
> - `GRANT ... WITH GRANT OPTION`: 권한을 받은 사용자가 **다른 사용자에게 그 권한을 또 줄 수 있다.** `REVOKE ... CASCADE`: 회수할 때 그 사용자가 다시 준 권한까지 **연쇄로 회수**한다. (DBMS마다 세부 문법은 다르다.)

> 📝 **기사 시험 포인트** (정보보안기사, 확실도: 중간)
> 설치 시 기본값이나 **예시 비밀번호를 바꾸지 않고 그대로 쓰는 것**은 대표적인 보안 취약점이다. 계정을 만들면 기본·예시 비밀번호를 바꾸는 것이 기본 수칙으로 알고 있다. (시험 표현은 확인하지 못했다.)

### 직접 확인한 결과 (이 컴퓨터)

비밀번호는 출력하지 않고 `agent_reader`로 읽기 전용 확인을 했다. 변경 시도는 `WHERE FALSE`와 롤백이라 데이터는 바뀌지 않았다.

| 항목 | 결과 |
|---|---|
| `agent_reader`로 `world` 접속 | ✅ |
| `SELECT country` / `city` / `countrylanguage` | ✅ 239 / 4,079 / 984행 |
| `INSERT` / `UPDATE` / `DELETE` 시도 | ✅ **모두 권한이 없어 차단됨** |

`WHERE FALSE`(바꿀 행이 0개)인데도 막혔다. **DB는 명령의 종류를 보고 실행 전에 권한을 검사**하기 때문이다.

---

## 주제 3: DB 접속 설정 — `.env`와 접속 주소 (`DB 접속 설정`, `LLM 설정`)

### `.env` 설정

```dotenv
AGENT_POSTGRES_USER=agent_reader
AGENT_POSTGRES_PASSWORD="YOUR_AGENT_PASSWORD"
AGENT_POSTGRES_DB=world
```

31번의 관리자용 `POSTGRES_*`와 구분하려고 앞에 `AGENT_`가 붙은 **에이전트용 계정 정보**이다. 코드에 기본값이 있어서 **비밀번호 한 줄만** 있어도 된다.

### 코드 (한 줄씩)

```python
load_dotenv(override=True)
AGENT_POSTGRES_USER = os.getenv("AGENT_POSTGRES_USER", "agent_reader")
AGENT_POSTGRES_PASSWORD = os.getenv("AGENT_POSTGRES_PASSWORD")
DB_NAME = os.getenv("AGENT_POSTGRES_DB", "world")
if not AGENT_POSTGRES_PASSWORD:
    raise ValueError(".env에 AGENT_POSTGRES_PASSWORD를 설정하세요.")

user = quote(AGENT_POSTGRES_USER, safe="")
password = quote(AGENT_POSTGRES_PASSWORD, safe="")
db = quote(DB_NAME, safe="")
DB_URI = f"postgresql://{user}:{password}@localhost:5432/{db}"
```

| 코드 | 뜻 |
|---|---|
| `load_dotenv(override=True)` | `.env`를 읽어 환경 변수로 등록. `override=True`는 컴퓨터에 **이미 같은 이름의 값이 있어도 `.env` 값이 이기게** 한다 (`False`면 원래 있던 값이 이긴다) |
| `os.getenv("이름", "기본값")` | 환경 변수를 꺼냄. **첫 번째가 찾을 이름, 두 번째가 못 찾았을 때 대신 쓸 값**이다 |
| `os.getenv("이름")` | 못 찾으면 `None` (비밀번호는 기본값을 코드에 적지 않는다) |
| `if not 변수: raise ValueError(...)` | 비밀번호가 비었으면(`None`, `""`) **일부러 오류를 내고 멈춘다** |
| `quote(..., safe="")` | 특수문자를 `%40` 같은 안전한 코드로 바꿈 |
| `f"...{변수}..."` | f-string. `{ }` 자리에 변수 값을 끼워 넣어 **접속 주소를 조립** |

> ♻️ 처음엔 기본값 `"agent_reader"`가 `.env`에 안 넣었으니 **의미가 없다**고 생각했으나, 이 이름은 `.env`가 아니라 **DB 안에 `CREATE ROLE`로 이미 만든 계정 이름**이다. `.env`에 이름이 없으면 코드가 기본값 `agent_reader`로 로그인을 시도하고, DB에 그 계정이 있으니 성공한다. 기본값이 DB의 실제 계정 이름과 **같아야** 한다. 마찬가지로 `DB_NAME`의 기본값 `"world"`도 DB에 이미 있는 데이터베이스 이름이다.

> ♻️ 처음엔 괄호 안의 `"agent_reader"`가 **찾는 대상**인 줄 알았으나, `os.getenv`는 **괄호 안에 두 가지를 함께 받는 함수**이고 두 번째는 못 찾았을 때의 **대안**이다. ("아메리카노 주세요, 없으면 라떼" 처럼 두 가지를 함께 말하는 것과 같다.)

변수에 담기는 것은 **문자열(`str`)** 이다. (리스트가 아니다. 리스트는 `[ ]`, 튜플은 `( )`.)

| 변수 | 지금 담긴 값 | 출처 |
|---|---|---|
| `AGENT_POSTGRES_USER` | `"agent_reader"` | 기본값 (DB에 만든 계정 이름) |
| `AGENT_POSTGRES_PASSWORD` | 내가 정한 비밀번호 | `.env` |
| `DB_NAME` | `"world"` | 기본값 (DB에 있는 DB 이름) |

**`raise`는 팝업이 아니라 셀 아래에 빨간 오류 글자**를 내고 그 셀의 나머지 실행을 멈춘다. 맨 아랫줄에 `ValueError: .env에 AGENT_POSTGRES_PASSWORD를 설정하세요.`처럼 우리가 적은 문장이 나온다.

> ➕ **더 알아두기 — 왜 `try`로 감싸지 않는가 (빠른 실패)**
> 비밀번호가 없으면 계속 진행해 봐야 소용없다. `try`로 오류를 삼키면 뒤에서 **더 알아내기 어려운 오류**가 난다. 그래서 **복구할 수 없는 문제는 일부러 일찍, 분명하게 멈추고**(빠른 실패, fail fast), `try / except`는 **오류를 처리하고 계속 갈 수 있을 때**(예: 권한 확인에서 차단되는 것이 정상일 때) 쓴다. 노트북에서는 **그 셀만** 멈추고 커널은 살아 있다. `.py` 파일이면 프로그램 전체가 종료된다.

> 📝 **기사 시험 포인트** (정보보안기사, 확실도: 중간)
> 비밀번호·API 키 같은 **비밀 정보를 소스 코드에 하드코딩하지 않고** 환경 변수나 별도 설정 파일로 분리하는 것은 기본적인 보안 수칙으로 알고 있다.

### 접속 주소(URI)가 주소 모양인 이유

웹 주소(URL)의 형식을 빌려 쓴 것이다.

```text
https://www.naver.com:443/news
postgresql://agent_reader:비밀번호@localhost:5432/world
 종류         사용자      비번      호스트   포트 DB
```

| 부분 | 웹 주소에서 대응 |
|---|---|
| `postgresql://` | `https://` (연결 방식) |
| `localhost:5432` | `www.naver.com:443` (호스트 + 포트) |
| `/world` | `/news` (그 안의 어디) |

정보 6개를 **한 줄**로 묶으면 변수 하나(`DB_URI`)로 전달할 수 있어서 편하다. **이 셀은 주소만 만들고 DB에 접속하지는 않는다.** 접속은 `psycopg.connect(DB_URI)`를 할 때 일어난다. 접속 주소는 `agent_reader`로 로그인하기 위한 **정보가 적힌 쪽지**이고 계정 자체는 DB 안에 이미 있다.

> ♻️ 처음엔 `localhost`가 **"내 IP"** 라고 생각했으나 **"내 컴퓨터 자신"** 을 뜻하는 이름이다. (숫자 주소는 `127.0.0.1`.) **포트**는 한 컴퓨터 안에서 어느 프로그램에 연결할지 정하는 번호이다. (웹 서버 443, PostgreSQL 5432)

### `quote`와 `safe` (실험으로 확인)

주소에서 `:`, `@`, `/`는 **구분 기호**라서 비밀번호에 이런 글자가 있으면 주소가 엉뚱하게 해석된다. `quote`가 안전한 코드로 바꿔 준다. 접속 주소를 읽는 쪽이 `%40` 같은 표기를 **알아서 원래 글자로 되돌린다.**

| 원래 글자 | `quote` 후 |
|---|---|
| `@` | `%40` |
| `/` | `%2F` |
| `:` | `%3A` |
| `#` | `%23` |
| `%` | **`%25`** |

원래 글자에 `%`가 있어도 `%25`로 바뀌어서 **서로 섞이지 않고, 한 번 바꾸면 항상 정확히 원래대로** 돌아온다(실험: `100%40abc` → `100%2540abc` → 되돌리면 `100%40abc`). 다만 **두 번 `quote`하면 망가진다**(`p@ss` → `p%40ss` → `p%2540ss`).

`safe="..."`는 **"여기 적은 글자는 바꾸지 마라"** 이다. 원본 `a/b@c`로 실험했다.

| 코드 | 결과 | 설명 |
|---|---|---|
| `quote("a/b@c")` | `a/b%40c` | 기본으로 `/`만 예외 |
| `quote("a/b@c", safe="")` | `a%2Fb%40c` | 예외 없음, 전부 바뀜 (**노트북이 쓰는 방식**) |
| `quote("a/b@c", safe="@")` | `a%2Fb@c` | `@`만 예외 |
| `quote("a/b@c", safe="/@")` | `a/b@c` | `/`와 `@` 둘 다 예외 |

- `safe`는 **예외로 둘 글자를 이어 붙인 하나의 문자열**이다. 콤마로 구분하지 않으며, 콤마를 쓰면 **콤마도 예외 글자**가 된다. 리스트(`["/", "@"]`)로 쓰는 것이 아니다.
- `safe`는 **글자 하나하나**만 본다. **`@/`처럼 붙은 글자 조합만 지키는 것은 불가능**하다(`safe="@/"`는 떨어져 있는 `@`, `/`까지 모두 안 바꾼다). 필요하면 `split`/`join`으로 직접 처리해야 한다. 이 노트북은 모두 바꾸는 `safe=""`로 충분하다.

### `LLM 설정`

```python
MODEL_NAME = "gemini-3.6-flash"
llm = ChatGoogleGenerativeAI(model=MODEL_NAME)
```

모델 이름을 변수에 담고 Gemini를 쓸 **창구를 준비**한다. **질문은 전송되지 않는다**(질문은 `llm.invoke(...)`로 보낸다). API 키(`GEMINI_API_KEY`)는 위 셀의 `load_dotenv`가 `.env`를 읽을 때 같이 불러와져서 자동으로 쓰인다. 그래서 `DB 접속 설정` 셀을 먼저 실행해야 한다.

---

## 주제 4: SQL 실행과 결과 가져오기 (`SQL 실행과 결과 가져오기`)

> ♻️ 이 구간의 코드는 **설명 글(마크다운) 안의 예시**라서 실행 버튼이 없다. 직접 돌려 보려면 **새 코드 셀**에 붙여 넣는다. (노트북 안내에 쓰인 "셀 번호"는 파일 안 순서(0부터)라 화면에 보이지 않는다. **제목과 코드 첫 줄**로 찾는다.)

```python
with psycopg.connect(DB_URI) as conn:
    result = conn.execute("SELECT current_user")
    print(result.fetchone())

    result = conn.execute("SELECT id, name FROM city LIMIT 3")
    print(result.fetchall())
```

| 메서드 | 역할 | 전화 비유 |
|---|---|---|
| `psycopg.connect(DB_URI)` | DB에 **접속** (연결 생성, `DB_URI`가 필요) | 전화 걸기 |
| `conn.execute(SQL)` | SQL을 **보내서 실행**하고 결과를 읽을 **커서를 반환** | 질문하기 |
| `fetchone()` | 결과에서 **다음 한 행**. 없으면 `None` | 대답 한 줄 듣기 |
| `fetchall()` | **남은 모든 행**을 리스트로. 없으면 `[]` | 대답 전부 듣기 |

기본적으로 **한 행은 튜플**로 반환된다. `fetchone()`은 튜플 하나, `fetchall()`은 튜플들의 리스트이다.

### `with ... as conn`과 `conn`

- `with`는 블록이 끝나면 **연결을 자동으로 닫는다.** 아래 들여쓴 줄(공백 4칸)이 블록 안이다.
- `as conn`은 연결 객체를 `conn`이라는 이름에 담는다. 변수에 직접 담는 `conn = psycopg.connect(...)`와 하는 일이 같고 **닫는 방법**만 다르다.

| | `with ... as conn` | `conn = psycopg.connect(...)` |
|---|---|---|
| 닫기 | **블록이 끝나면 자동** | **`conn.close()`를 직접** |
| 쓰기 좋은 때 | 잠깐 쓰고 끝낼 때 | 오래 열어 두고 계속 쓸 때 |

> ♻️ 처음엔 `conn`이 **`psycopg.connect(DB_URI)`의 축약(별칭)** 이라서 쓸 때마다 연결이 새로 생기는 줄 알았으나, **변수는 "오른쪽을 한 번 실행한 결과를 담은 그릇"** 이다. `conn = psycopg.connect(...)`는 `connect`를 **한 번만** 실행하고 그 연결을 담아 둔 것이라 `conn`을 몇 번 써도 **같은 연결**이다. SQL의 `AS` 별칭은 이름만 붙이지만 파이썬의 `=`는 실행한 결과를 저장한다.
>
> **실험으로 확인했다** (DB가 연결마다 붙이는 고유 번호 `pg_backend_pid()` 비교):
>
> | 방법 | 첫 번째 | 두 번째 | 같은 연결? |
> |---|---|---|---|
> | `connect`를 각각 | 12596 | 9116 | ❌ 다른 연결 |
> | 한 번 `connect` 후 `conn`으로 두 번 | 22996 | 22996 | ✅ 같은 연결 |
>
> `connect`는 부를 때마다 **새 연결**을 만든다.

### `result`는 데이터가 아니라 창구(커서)

`result = conn.execute(...)`의 `result`에는 **결과를 읽을 창구(커서)** 가 담긴다. 데이터는 `fetchone()`/`fetchall()`로 꺼내야 한다. (`conn`과 같은 느낌의 변수이고 담긴 대상만 다르다.)

| 변수 | 만드는 줄 | 담기는 것 |
|---|---|---|
| `conn` | `psycopg.connect(...)` | **연결**(DB와의 통로) |
| `result` | `conn.execute(...)` | **창구**(결과를 읽는 책갈피) |

`print(result)`를 하면 답이 아니라 **창구의 정보**가 나온다 (직접 실행한 결과).

```text
print(result)            → <psycopg.Cursor [TUPLES_OK] [INTRANS] (host=localhost user=agent_reader database=world) at 0x...>
print(result.fetchone()) → ('agent_reader',)
```

`('agent_reader',)`는 결과가 **한 행이고 컬럼이 하나**라서 **값이 하나뿐인 튜플**이 나온 것이다(끝의 `,`가 그 표시이다).

`conn.execute("SELECT ...").fetchall()`처럼 **한 줄로 이어 써도 된다.** `result`를 다시 안 쓰면 한 줄이 간결하고, 창구로 여러 번 읽거나 확인할 때는 나눠 쓴다. (노트북의 `fetch_rows`도 이어 쓴다.)

### `fetchone()`은 읽은 위치가 앞으로만 간다 (직접 실행해 확인)

| 순서 | 코드 | 결과 |
|---|---|---|
| 1 | `fetchone()` 첫 번째 | `('agent_reader',)` |
| 2 | `fetchone()` 두 번째 | **`None`** |
| 3 | `execute`를 다시 한 뒤 `fetchone()` | `('agent_reader',)` |
| a | (도시 3줄) `fetchone()` | `(1, 'Kabul')` |
| b | `fetchone()` | `(2, 'Qandahar')` |
| c | `fetchall()` | `[(3, 'Herat')]` (**남은 것만**) |
| d | `fetchone()` | `None` |

> ♻️ 처음엔 두 번째 `fetchone()`이 `None`인 것을 **"한 번 부르면 소모된다"** 고 이해했다. 맞는 방향이지만 **DB의 데이터가 사라지는 것이 아니라 창구(커서)의 "읽은 위치"가 앞으로 가 버리는 것**이다. 지나간 줄은 그 창구에서 다시 안 준다. **다시 읽으려면 `execute`를 다시 해서 새 창구**를 만든다.

> ♻️ 처음엔 스택처럼 동작한다고 생각했으나 **앞에서부터 하나씩 꺼내는 큐(선입선출)에 가깝다.** 스택(후입선출)이라면 마지막 줄부터 나온다.

리스트처럼 번호로 보고 싶으면 `rows = result.fetchall()`로 **변수에 저장**하면 `rows[0]`, `rows[2]`, `len(rows)`로 몇 번이든 볼 수 있다. 한 줄씩 읽는 방식(`fetchone`)은 **결과가 아주 클 때 한 줄씩 처리**하려는 설계이다.

### `cursor`는 왜 파이썬에서 만드는가

31번의 `conn.cursor()`는 **파이썬 쪽 도구**이다. DB는 SQL을 실행해 결과를 만들고, 그 결과를 받아 **어디까지 읽었는지 기억하는 책갈피**는 받는 쪽(파이썬)이 가진다. `conn.execute(...)`는 **내부에서 커서를 만들어** 그 커서를 돌려주는 줄임 표현이다. PostgreSQL에는 **SQL로 서버 안에 커서를 만드는 `DECLARE ... CURSOR`** 도 따로 있다(결과가 너무 클 때 쓰는 다른 종류이다. 직접 실행해 확인하지는 않았다).

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 중간)
> **커서(Cursor)** 는 SQL 실행 결과로 나온 **여러 행의 집합**을 **한 행씩 차례로 처리**하기 위한 **포인터**로 알고 있다. (정확한 출제 형태는 확인하지 못했다.)

### 컬럼 = 속성, 한 행 = 튜플

`SELECT id, name FROM city LIMIT 3`의 결과 `[(1, 'Kabul'), (2, 'Qandahar'), (3, 'Herat')]`에서 `(1, 'Kabul')` 한 묶음이 **행(튜플)** 이고, `1`은 `id` 속성, `'Kabul'`은 `name` 속성의 값이다.

> ➕ **더 알아두기 — 값의 순서는 타입이 아니라 `SELECT`에 쓴 순서**
> "숫자는 id, 문자는 name"은 이번 결과에서만 맞는 말이다. `SELECT name, id`로 쓰면 `('Kabul', 1)`이 나온다. 순서가 불편하면 `dict_row`를 쓰면 `{'id': 1, 'name': 'Kabul'}`처럼 이름표가 붙는다.

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
> 관계형 모델의 용어: **릴레이션**(테이블), **튜플**(행), **속성**(열, 컬럼), **도메인**(속성이 가질 수 있는 값의 범위), **차수(Degree)** = 속성의 수, **카디널리티(Cardinality)** = 튜플의 수. 같은 단어 "튜플"이 DB 용어(행)와 파이썬 용어(`( )`로 묶은 자료형)로 둘 다 있다.

### 여러 줄 SQL

한 줄 글자는 따옴표 하나(`"..."`)이고, **줄바꿈이 있는 글자는 따옴표 세 개(`"""..."""`)** 로 감싼다. SQL은 줄바꿈에 민감하지 않아서 한 줄로 쓴 것과 같게 실행된다.

```python
conn.execute("""
    SELECT id, name
    FROM city
    LIMIT 3
""")
```

---

## 주제 5: DB 권한 확인 (`DB 권한 확인`) — 진행 중

노트북: `city` 테이블에서 **조회는 성공하고**, INSERT·UPDATE·DELETE는 **DB 계정의 권한으로 차단되는지** 확인한다. `WHERE FALSE`로 변경할 행을 없애고, 각 실행 후에는 트랜잭션을 롤백한다. (이 셀은 진짜 코드 셀이다.)

### ① 조회 확인

```python
with psycopg.connect(DB_URI) as conn:
    result = conn.execute("SELECT id, name FROM city LIMIT 3")
    rows = result.fetchall()
    print(rows)
```

`SELECT` 권한이 정상인지 확인하는 부분이다. 3줄이 출력되면 정상이고 `InsufficientPrivilege` 오류가 나면 `GRANT SELECT`가 안 된 것이다.

### ② 변경 명령 3개를 글자로 모은 리스트

```python
queries = [
    "INSERT INTO city (name) SELECT 'TEST' WHERE FALSE",
    "UPDATE city SET population = 0 WHERE FALSE",
    "DELETE FROM city WHERE FALSE",
]
```

세 SQL 모두 끝에 **`WHERE FALSE`** (조건이 항상 거짓)가 붙어 있어서 **해당되는 행이 0개**이다. 그래서 권한이 **있어도** 실제로는 아무것도 추가·수정·삭제되지 않는다. 그런데도 권한이 없으면 **실행 전에 거부**되므로, **데이터를 건드리지 않고 권한만 확인**할 수 있다.

### ③ `for` 반복문

```python
for query in queries:
```

**`queries` 리스트에서 항목을 하나씩 꺼내 `query`라는 이름으로 부르며** 아래 블록을 반복한다.

| 반복 | `query`에 담기는 것 |
|---|---|
| 1번째 | `"INSERT ..."` |
| 2번째 | `"UPDATE ..."` |
| 3번째 | `"DELETE ..."` |

`queries`(리스트 전체, 복수)와 `query`(한 번에 하나, 단수)를 구분한다. 반복문이 없으면 같은 코드를 세 번 복사해야 한다.

### ④ `try / except / else`

```python
    try:
        with psycopg.connect(DB_URI) as conn:
            with conn.transaction(force_rollback=True):
                conn.execute(query)
    except psycopg.errors.InsufficientPrivilege:
        print("권한이 없어 차단됨:", query)
    else:
        print("실행 허용됨 — 계정의 권한을 확인하세요.:", query)
```

```text
try 안의 코드 실행
   ├─ 오류 없음 → 끝까지 실행 → except 건너뜀 → else 실행
   └─ 오류 남   → 그 줄에서 중단 → 맞는 except로 이동
```

| 상황 | 실행되는 것 |
|---|---|
| `try` 안에서 **권한 오류**(`InsufficientPrivilege`)가 남 | `except` 줄 → "차단됨" 출력 |
| `try` 안에서 **오류가 없음** | `else` 줄 → "실행 허용됨, 권한을 확인하세요" 출력 |

이 코드의 목적이 **"차단되는 것이 정상"** 이라서 오류가 없으면 오히려 경고(`else`)를 낸다. `raise`는 오류를 **내는** 것이고 `try / except`는 오류를 **받아서 처리하는** 쪽이다.

### ⑤ `with conn.transaction(force_rollback=True):`

> ♻️ 처음엔 쿼리를 실행하는 명령이 `execute`인데 **왜 `transaction`이라는 명령이 나오는가** 의아했으나, **`transaction()`은 쿼리를 실행하는 명령이 아니다.** 쿼리 실행은 **여전히 `execute`** 가 하고, `transaction()`은 그 실행을 **하나의 묶음으로 감싸는 범위**(울타리)를 만든다.

| 코드 | 하는 일 | SQL 대응 |
|---|---|---|
| `conn.transaction(...)` | 트랜잭션 범위를 연다 | `BEGIN;` |
| **`conn.execute(query)`** | **쿼리를 실행** | `INSERT ...;` 등 |
| 블록이 끝날 때 (`force_rollback=True`) | **무조건 되돌린다** | `ROLLBACK;` |

권한이 없으면 `execute`에서 오류가 나서 어차피 실행되지 않지만, **권한이 실수로 있다면** `execute`는 성공하고 **울타리가 끝날 때 롤백**되어 데이터가 확정되지 않는다. (`WHERE FALSE`에 롤백까지 두 겹이다.)

**블록이 끝날 때 무슨 일이 일어나는가 (임시 테이블로 직접 실행해 확인)**

| 단계 | 블록 | 행 수 | 의미 |
|---|---|---|---|
| 처음 | | 0 | |
| ① | `with c.transaction():` 안에서 INSERT | **1** | **블록이 끝나자 자동 COMMIT됨** |
| ② | `with c.transaction(force_rollback=True):` 안에서 INSERT | **1** (그대로) | **ROLLBACK**되어 두 번째 INSERT가 사라짐 |
| ③ | 블록 안에서 **오류**가 남 | **1** (그대로) | 오류가 나면 **자동 ROLLBACK** |

| 상황 | 블록이 끝날 때 |
|---|---|
| 오류 없이 끝남 (일반) | **자동 `COMMIT`** |
| `force_rollback=True` | **무조건 `ROLLBACK`** |
| 블록 안에서 오류가 남 | **자동 `ROLLBACK`** |

> 이 실험은 `autocommit=True` 연결에서 했다. 노트북의 권한 확인 코드는 연결 직후 `transaction()`을 바로 열어서 같은 방식이다.

### 왜 3줄을 한 번에 보내지 않고 하나씩 보내는가 (직접 실행해 확인)

SQL 세 개를 `;`로 이어 **한 번의 `execute`로** 보내면 어떻게 되는지 실험했다.

| 방법 | 결과 |
|---|---|
| 한 번에 보냄 | 오류가 **하나만** 나온다 (`InsufficientPrivilege`, "city 테이블에 대한 접근 권한 없음"). 어느 명령 때문인지 오류 문장에 안 나온다 |
| 하나씩 따로 보냄 | **INSERT, UPDATE, DELETE 3개 모두 개별로 차단**됨을 확인 |

한 번에 보내면 **처음 실패한 명령에서 멈춰서** 나머지가 각각 막히는지는 알 수 없다. 하나씩 보내야 **각 명령의 권한을 따로 확인**할 수 있다. 또한 오류가 나면 그 트랜잭션은 중단된 상태가 되어 다음 명령에 영향을 주므로(10월 7일 Transaction 글), 노트북은 **반복마다 연결과 트랜잭션을 새로** 만든다.

---

## 주제 6: 트랜잭션과 COMMIT 개념 (질문으로 정리)

`테이블 생성`과 `검색` 정도만 알던 상태에서 `COMMIT`, `ROLLBACK`, `autocommit`이 계속 흐릿해서 개념부터 다시 정리했다.

### 왜 트랜잭션이 필요한가 — 은행 이체

철수가 영희에게 1만 원을 이체하면 **① 철수 계좌에서 빼기 ② 영희 계좌에 더하기** 두 단계이다. ①만 하고 정전이 나면 **돈이 허공으로 사라진다.** 그래서 두 단계를 **한 덩어리**로 묶고 **전부 성공하거나 전부 취소**해야 한다.

| 용어 | 뜻 | 쉬운 말 |
|---|---|---|
| **트랜잭션** | 여러 작업을 묶은 한 덩어리 | 한 묶음 |
| **`COMMIT`** | 묶음의 변경을 **확정**한다 | 저장 |
| **`ROLLBACK`** | 묶음의 변경을 **취소**한다 | 되돌리기 |

문서 편집에 비유하면 **고치는 중**(저장 안 함)이 트랜잭션 진행 중이고, **저장**이 `COMMIT`, **저장 안 하고 되돌리기**가 `ROLLBACK`이다. 확정하기 전의 변경은 **임시 상태**이고 다른 사람에게는 아직 안 보인다. ①만 하고 오류가 나면 묶음 전체를 **`ROLLBACK`** 한다.

`SELECT`는 데이터를 **바꾸지 않아서 확정할 것이 없다.** `INSERT`/`UPDATE`/`DELETE`처럼 데이터가 바뀔 때 확정이 의미가 있다.

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
> "전부 성공하거나 전부 취소"는 트랜잭션의 **원자성(Atomicity)** 이다. 트랜잭션의 특성은 **ACID**(원자성, 일관성, 격리성, 영속성)이다. `COMMIT`·`ROLLBACK`은 교재에 따라 DCL에 포함하기도 하고 **TCL**(트랜잭션 제어어)로 따로 분류하기도 한다.

### 트랜잭션을 안 만들면 `COMMIT`을 안 써도 되는가?

> ♻️ 처음엔 **"트랜잭션으로 안 만들면 커밋은 안 써도 된다"** 고 이해했다. 반은 맞다. **"트랜잭션이 없는 것"이 아니라 "문장 하나가 곧 하나의 트랜잭션"** 이다. 내가 묶지 않아도 DB가 문장마다 알아서 묶고 확정한다(자동 커밋).

| 환경 | 기본 동작 | `COMMIT`을 직접 써야 하나 |
|---|---|---|
| **SQL 파일**에서 문장을 그냥 실행 | 문장마다 자동 확정 | ❌ 필요 없음 |
| SQL에서 **`BEGIN;`으로 직접 열었음** | 끝날 때까지 임시 상태 | ✅ `COMMIT;` 또는 `ROLLBACK;`으로 끝내야 함 |
| **파이썬 psycopg 기본**(`autocommit=False`) | **묶음이 자동으로 시작**됨 | ✅ `commit()`을 해야 확정 |
| 파이썬 **`autocommit=True`** | 문장마다 자동 확정 | ❌ 필요 없음 |

- 31번에서 `autocommit=True`를 쓴 이유가 이것이다. "문장마다 바로 확정해 줘"라는 설정이다.
- 앞에서 `print(result)`로 본 **`[INTRANS]`** 는 "지금 트랜잭션 안에 있다"는 뜻이다. `SELECT`만 했는데도 파이썬(`autocommit=False`)은 묶음을 열어 두고 있었다.
- 묶음이 열려 있지 않은데 `COMMIT;`을 실행해도 오류로 멈추지 않고 경고 정도가 나오는 것으로 알고 있다.
- 파이썬의 `conn.commit()`과 SQL의 `COMMIT;`은 **같은 커밋**이다.

### "먼저 해 보고 결과를 보고 확정할지 결정"하려면

```sql
BEGIN;                                              -- ① 묶음 시작
UPDATE city SET population = 0 WHERE id = 1;        -- ② 변경 (아직 임시)
SELECT id, name, population FROM city WHERE id = 1; -- ③ 결과 확인 (내 연결에서는 바뀐 값이 보임)
COMMIT;                                             -- ④-a 마음에 들면 확정
-- ROLLBACK;                                        -- ④-b 아니면 취소 (둘 중 하나만)
```

```python
conn = psycopg.connect(DB_URI)       # autocommit 꺼짐 → 묶음이 자동 시작
conn.execute("UPDATE city SET population = 0 WHERE id = 1")     # 변경 (임시)
row = conn.execute("SELECT population FROM city WHERE id = 1").fetchone()   # 결과 확인
conn.commit()          # 마음에 들면 확정
# conn.rollback()      # 아니면 취소
conn.close()
```

> 위 코드는 일반적인 사용법이며 실제 데이터를 바꾸는 코드라서 직접 실행해 확인하지는 않았다.

| 방법 | 끝났을 때 |
|---|---|
| `with conn.transaction():` | 오류가 없으면 **자동 `COMMIT`** (직접 결정할 수 없음) |
| `conn.commit()` / `conn.rollback()`을 **직접** | **내가 결정** (확인한 뒤 확정·취소) |
| `force_rollback=True` (노트북) | **무조건 `ROLLBACK`** (확인만 하고 흔적을 남기지 않음) |

**"결과를 보고 결정"** 하려면 `with conn.transaction():`이 아니라 `commit()`/`rollback()`을 직접 쓴다. 지켜야 할 것: **같은 연결을 유지**해야 하고(묶음은 하나의 연결 안에서만 살아 있다), **너무 오래 열어 두지 않는다**(같은 행을 쓰려는 다른 사람이 기다릴 수 있다).

---

## 정리

- **`DB 권한 확인`의 `try / except / else`**: 오류가 나면 `except`, **오류가 없으면 `else`**. 이 코드는 **차단이 정상**이라서 오류가 없으면 경고(`else`)를 낸다. `raise`는 오류를 내고 `try/except`는 받아서 처리한다.
- **`transaction()`은 쿼리 실행이 아니라 묶음 범위**이다. 실행은 `execute`가 한다. `with conn.transaction():`은 블록이 끝나면 **자동 COMMIT**(오류면 ROLLBACK), `force_rollback=True`는 **무조건 ROLLBACK**이다(임시 테이블 실험으로 확인).
- **트랜잭션**: 여러 작업을 묶어 전부 성공하면 `COMMIT`(확정), 하나라도 실패하면 `ROLLBACK`(취소). 문장 하나가 곧 하나의 트랜잭션이라 SQL 파일에서는 `BEGIN` 없이 `COMMIT`이 필요 없고, **파이썬 기본(`autocommit=False`)** 에서는 `commit()`이 필요하다. 먼저 해 보고 결정하려면 `BEGIN → 변경 → SELECT 확인 → COMMIT/ROLLBACK`.
- **이번 수업 32번의 목표**: LLM이 `world` DB를 조회하는 에이전트. **고정 SQL 도구**(개발자가 SQL 작성, LLM은 도구·인자 선택)와 **Text-to-SQL**(LLM이 SQL 생성)을 비교한다. Text-to-SQL은 위험해서 **안전장치 3겹**(DB 계정 권한, 읽기 전용 연결, 실행 전 SQL 검사)을 둔다 — 다층 방어, 최소 권한의 원칙.
- **DCL**: `GRANT`(권한 부여), `REVOKE`(회수). `agent_reader`에게 `CONNECT`, `USAGE`, `SELECT`만 준다. 계정은 `.sql` 파일에서 **관리자로 `world`에 연결해** 만든다. 이미 만든 계정의 비밀번호는 `CREATE`가 아니라 `ALTER ROLE`로 바꾼다. 예시 비밀번호를 그대로 쓰지 않는다.
- **접속 주소 만들기**: `.env` → `load_dotenv` → `os.getenv("이름", "기본값")`(기본값은 DB에 만든 계정·DB 이름) → 비밀번호가 없으면 `raise ValueError`(빠른 실패) → `quote(safe="")`(특수문자 안전 변환, 한 번만) → f-string으로 `DB_URI` 조립. 이 단계는 **DB에 접속하지 않는다.**
- **`localhost`는 내 컴퓨터 자신, `5432`는 PostgreSQL 포트**이다. 접속 주소는 웹 주소와 같은 URI 형식이다.
- **psycopg 조회**: `connect`(전화 걸기) → `execute`(질문) → `fetch`(대답). `conn`은 연결을 담은 변수이고 `connect`는 부를 때마다 **새 연결**을 만든다. `result`는 **창구(커서)** 이며 `fetchone`은 **읽은 위치가 앞으로만** 간다(큐처럼). 다 읽으면 `None`/`[]`이고 `execute`를 다시 하면 처음부터 읽는다. 한 행은 **튜플**, 컬럼은 **속성**.
- **권한 확인**: `WHERE FALSE`로 **데이터를 건드리지 않고** 권한만 검사한다. 하나씩 보내야 각 명령의 권한을 따로 확인할 수 있다. 변경 3개 모두 차단됨을 확인했다.
- **셀 찾는 법**: 마크다운 셀 안의 코드는 실행 불가(새 코드 셀에 붙여 넣기). 셀 번호 대신 **제목과 코드 첫 줄**로 찾는다.

---

## ✅ 확인 질문

1. 고정 SQL 도구와 Text-to-SQL은 SQL을 누가 쓰는지, LLM이 무엇을 결정하는지에서 어떻게 다른가? 어느 쪽이 더 위험한가?
2. "변경하지 마"라고 프롬프트로 부탁하는 것만으로 부족한 이유는? 이 노트북의 안전장치 3겹은?
3. 다층 방어와 최소 권한의 원칙을 각각 설명하라. `agent_reader`에게 `SELECT`만 준 것은 어느 원칙의 예인가?
4. DDL, DML, DCL의 차이와 대표 명령은? `GRANT`와 `REVOKE`는? (기사 대비) `WITH GRANT OPTION`과 `REVOKE ... CASCADE`는?
5. `CONNECT`, `USAGE`, `SELECT`는 각각 무엇을 허용하는가? 이 계정은 어떤 순서로 열어 줘야 데이터를 볼 수 있는가?
6. `CREATE ROLE`과 `ALTER ROLE`의 차이는? 예시 비밀번호를 그대로 쓰면 왜 안 되는가? 이미 만든 계정의 비밀번호는 어떻게 바꾸는가?
7. 계정을 만드는 SQL은 어디에서(노트북 셀 / `.sql` 파일), 어떤 계정으로 어느 DB에 연결해서 실행하는가?
8. `os.getenv("이름", "기본값")`에서 첫 번째와 두 번째 인자의 역할은? `.env`에 사용자 이름을 안 적어도 로그인이 되는 이유는? (기본값 `"agent_reader"`는 어디에 있는 값인가)
9. `AGENT_POSTGRES_PASSWORD`가 `.env`에 없으면 변수에 무엇이 담기고 어떤 일이 일어나는가? 왜 비밀번호에는 기본값을 쓰지 않는가?
10. `raise ValueError(...)`를 하면 팝업이 뜨는가? 왜 `try`로 감싸지 않고 일부러 멈추는가(빠른 실패)? `try / except`는 어떤 때 쓰는가?
11. `load_dotenv(override=True)`와 `override=False`의 차이는?
12. `quote(..., safe="")`는 왜 쓰는가? `safe=""`, `safe="/@"`의 차이는? `@/` 같은 글자 조합만 지킬 수 있는가?
13. 원래 비밀번호에 `%40`이 글자 그대로 들어 있어도 `quote`와 되돌리기가 안전한 이유는? `quote`를 두 번 하면 어떻게 되는가?
14. 접속 주소 `postgresql://사용자:비밀번호@호스트:포트/DB이름`에서 `localhost`와 `5432`는 무엇인가? 이 주소를 만드는 셀이 DB에 접속하는가?
15. `connect`, `execute`, `fetchone`, `fetchall`은 각각 무슨 일을 하는가? `conn`과 `result`에는 각각 무엇이 담기는가?
16. `conn`이 `psycopg.connect(...)`의 축약이 아닌 이유는? `connect`를 두 번 부르면 연결은 몇 개인가(PID 실험)? `with ... as conn`과 `conn = ...`의 차이는?
17. `print(result)`를 하면 왜 `agent_reader`가 안 보이는가? 답을 보려면 무엇을 해야 하는가?
18. `fetchone()`을 같은 `result`에서 두 번 부르면 두 번째가 `None`인 이유는? DB 데이터는 사라지는가? 다시 읽으려면?
19. `fetchone`이 스택이 아니라 큐에 가까운 이유는? 리스트처럼 번호로 보려면 어떻게 하는가? 한 줄씩 읽는 방식은 언제 유리한가?
20. `cursor`는 왜 파이썬에서 만드는가? SQL에도 커서가 있는가?
21. `SELECT id, name`의 결과 `(1, 'Kabul')`에서 `1`과 `'Kabul'`은 각각 어느 속성의 값인가? 값의 순서는 무엇으로 정해지는가? (기사 대비) 릴레이션, 튜플, 속성, 차수, 카디널리티는?
22. 여러 줄 SQL은 어떻게 쓰는가? `"..."`와 `"""..."""`의 차이는?
23. `WHERE FALSE`를 쓰는 이유는? 권한이 있어도 `DELETE FROM city WHERE FALSE`는 행을 지우는가?
24. `for query in queries:`에서 `queries`와 `query`의 차이는? 두 번째 반복에서 `query`에는 무엇이 담기는가?
25. 변경 SQL 3개를 `;`로 이어 한 번에 보내지 않고 하나씩 보내는 이유는? (실험 결과로 설명하라)
26. 마크다운 셀 안의 예시 코드는 왜 실행되지 않는가? 직접 돌려 보려면 어떻게 하는가?
27. `try / except / else`에서 `else`는 언제 실행되는가? 이 권한 확인 코드에서 `except`와 `else`는 각각 어떤 의미인가(차단이 정상인 이유)?
28. `conn.transaction()`은 쿼리를 실행하는 명령인가? `execute`와 역할이 어떻게 다른가? SQL의 `BEGIN`/`ROLLBACK`과 어떻게 대응되는가?
29. `with conn.transaction():`은 블록이 끝날 때 자동으로 `COMMIT`되는가? `force_rollback=True`일 때와 블록 안에서 오류가 났을 때는 각각 어떻게 되는가? (실험 결과로 설명하라)
30. 트랜잭션이 필요한 이유를 은행 이체로 설명하라. `COMMIT`과 `ROLLBACK`은 각각 무엇인가? ①만 하고 오류가 나면 무엇을 해야 하는가?
31. SQL 파일에서 `BEGIN` 없이 `UPDATE`를 실행하면 `COMMIT`이 필요한가? `BEGIN`으로 직접 열었을 때는? "문장 하나가 곧 하나의 트랜잭션"이란 무슨 뜻인가?
32. 파이썬 psycopg의 기본(`autocommit=False`)에서는 왜 `commit()`이 필요한가? `autocommit=True`는 무엇을 바꾸는가? `[INTRANS]`는 무슨 뜻인가?
33. 변경을 먼저 해 보고 결과를 확인한 뒤 확정·취소를 결정하는 흐름을 SQL과 파이썬으로 써라. 이때 지켜야 할 것 두 가지는?
34. (기사 대비) 트랜잭션의 원자성은 무엇인가? ACID는? `COMMIT`·`ROLLBACK`은 DCL과 TCL 중 어디에 분류하는가?
