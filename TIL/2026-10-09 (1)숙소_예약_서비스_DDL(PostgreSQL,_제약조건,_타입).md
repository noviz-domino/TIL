---
tags: [database, postgresql, ddl, constraints, practice]
til: v2 2026-10-09
---

# 2026-10-09_숙소_예약_서비스_DDL(PostgreSQL,_제약조건,_타입)
> 작성일: 2026-10-09

## 🔗 관련 글

- [숙소 예약 서비스 ERD 설계](2026-10-08%20(1)숙소_예약_서비스_ERD_설계(기능_정의,_ERD_점검,_설계_결정).md) — 이 DDL의 바탕이 된 ERD와 설계 결정.
- [SQL 기초와 DDL (10-02)](2026-10-02%20(4)SQL_기초와_DDL(명령어_분류,_트랜잭션,_제약조건).md) — `PRIMARY KEY`, `UNIQUE`, `NOT NULL`, `CHECK`, `DEFAULT` 제약조건 표의 출처.
- [데이터베이스 설계](2026-10-07%20(5)데이터베이스_설계(키,_무결성,_정규화,_ERD_작성).md) — 키, 무결성, 정규화.
- 이어지는 글: [화면별 쿼리](2026-10-09%20(2)숙소_예약_서비스_화면별_쿼리(JOIN_추적,_GROUP_BY).md)

## ❓ 아직 모르겠는 것

- 아래 DDL은 **AI(Claude)가 만든 초안**이고 **직접 실행해서 결과를 확인하지는 못했다.** 실행 중 오류가 나는지 확인해야 한다.
- 예약금액의 의미(1박 요금 스냅샷 vs 총액)와 예약번호(별도 `UNIQUE` 컬럼)는 **가정**으로 정했다. 기능 정의와 맞는지 최종 확인이 필요하다.
- `INT`의 크기를 정하지 않는 것이 실무에서도 일반적인지, 큰 값(예: 금액)은 `BIGINT`가 필요한지는 이 글에서 확인하지 못했다.

---

## 주제 1: DDL 초안 (PostgreSQL)

ERD를 바탕으로 만든 테이블 7개이다. **부모 테이블부터** 만들어야 한다. (FK가 가리키는 테이블이 먼저 있어야 한다.)

```sql
CREATE TABLE users (
    user_id   INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_name VARCHAR(30)  NOT NULL,
    phone     VARCHAR(20),
    email     VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE host (
    host_id      INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id      INT NOT NULL UNIQUE REFERENCES users(user_id),
    introduction TEXT
);

CREATE TABLE accommodation (
    accommodation_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    host_id          INT NOT NULL REFERENCES host(host_id),
    name             VARCHAR(100) NOT NULL,
    region           VARCHAR(30)  NOT NULL,
    address          VARCHAR(200) NOT NULL,
    description      TEXT,
    phone            VARCHAR(20),
    amenities        TEXT,
    registered_at    DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE room (
    room_id          INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    accommodation_id INT NOT NULL REFERENCES accommodation(accommodation_id),
    room_name        VARCHAR(50) NOT NULL,
    max_guests       INT NOT NULL CHECK (max_guests > 0),
    price_per_night  INT NOT NULL CHECK (price_per_night >= 0),
    size             INT,
    room_count       INT,
    description      TEXT
);

CREATE TABLE booking (
    booking_id      INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    booking_no      VARCHAR(30) NOT NULL UNIQUE,
    user_id         INT NOT NULL REFERENCES users(user_id),
    room_id         INT NOT NULL REFERENCES room(room_id),
    check_in        DATE NOT NULL,
    check_out       DATE NOT NULL,
    guests          INT NOT NULL CHECK (guests > 0),
    price_per_night INT NOT NULL CHECK (price_per_night >= 0),
    status          VARCHAR(10) NOT NULL DEFAULT '확정'
                    CHECK (status IN ('확정', '이용 완료', '취소')),
    booked_at       DATE NOT NULL DEFAULT CURRENT_DATE,
    CHECK (check_out > check_in)
);

CREATE TABLE review (
    review_id  INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    booking_id INT NOT NULL UNIQUE REFERENCES booking(booking_id),
    rating     INT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    content    TEXT NOT NULL,
    created_at DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE wishlist (
    user_id          INT NOT NULL REFERENCES users(user_id),
    accommodation_id INT NOT NULL REFERENCES accommodation(accommodation_id),
    PRIMARY KEY (user_id, accommodation_id)
);
```

### 가정한 것들

| 항목 | 가정 | 이유 |
|---|---|---|
| 테이블 이름 `users` | `user`가 아니다 | `user`는 PostgreSQL **예약어**라서 테이블 이름으로 쓰기 곤란하다 |
| 호스트의 `user_id` | `UNIQUE` | 한 유저가 호스트 행을 둘 가질 수 없다 (유저 1 : 0..1 호스트) |
| 예약의 `price_per_night` | 예약 당시 1박 요금(**스냅샷**) | 나중에 객실 요금이 바뀌어도 과거 예약 금액이 안 변한다. 총액 = 요금 × 박 수(체크아웃 − 체크인)로 계산 |
| `booking_no` | 별도 컬럼 + `UNIQUE` | 사람이 보는 예약 번호(`BK-20250607-001`)와 내부 `booking_id`를 분리 |
| 후기 `booking_id` | `UNIQUE` | 예약 1건당 후기 최대 1개 (1:0..1) |
| 찜 | `(user_id, accommodation_id)` 복합 PK | 같은 숙소를 같은 유저가 두 번 찜하지 못하게 한다 |
| 편의시설 | `TEXT` 한 컬럼 | 의도적 단순화 (정석은 M:N 테이블) |
| 예약 상태 | `CHECK (status IN (...))` + `DEFAULT '확정'` | 허용되는 값만 넣고, 입력하지 않으면 확정 |

> ➕ **더 알아두기 — `GENERATED ALWAYS AS IDENTITY`**
> PostgreSQL에서 번호를 **자동으로 1씩 늘려 주는** 컬럼을 만드는 표준적인 방법이다. `INSERT`할 때 이 컬럼에는 값을 적지 않는다. (`ALWAYS`는 사용자가 값을 직접 넣는 것을 막는다는 뜻이다.) 수업에서 쓴 `SERIAL`과 비슷한 역할이다.

---

## 주제 2: 테이블별 설명

### users

- `user_name VARCHAR(30) NOT NULL` — 이름은 반드시 있어야 한다.
- `email ... UNIQUE` — 같은 이메일로 두 번 가입할 수 없다.
- `phone`은 `NOT NULL`이 아니어도 되지만, **직접 정한 결정**으로 `NOT NULL`을 붙이기로 했다. (아래 주제 5)

### host

- 호스트는 **유저에 붙는 역할**이다. `user_id`가 FK이면서 `UNIQUE`이다.
- FK만 있으면 1:N, **FK + `UNIQUE`이면 1:0..1**이다.

### accommodation

- `registered_at DATE NOT NULL DEFAULT CURRENT_DATE` — 등록할 때 날짜를 적지 않아도 **오늘 날짜**가 자동으로 들어간다. 화면 ①(최근 등록한 숙소)에 쓰인다.

### room

- 가격과 인원에 `CHECK`로 **0 이하 값을 막는다.**
- `price_per_night`는 **객실의 현재 요금**이다. 예약 테이블의 같은 이름 컬럼은 **예약 당시 요금**이다. 이름은 같지만 의미가 다르다.

### booking

- 박 수는 컬럼으로 저장하지 않고 `check_out - check_in`으로 **계산**한다. (PostgreSQL에서 `DATE`끼리 빼면 일수가 나온다.)
- 테이블 맨 끝의 `CHECK (check_out > check_in)`는 **여러 컬럼을 함께 검사**하는 제약이다. 컬럼 하나에 붙이지 않고 따로 적는다.

---

## 주제 3: 궁금했던 것 (질문과 답)

### `INT`는 크기를 정하지 않아도 되는가?

> ♻️ 처음엔 `INT(10)`처럼 **크기를 적어야** 한다고 알았으나, **PostgreSQL에서는 `INT`에 크기를 적지 않는다.** `INT(10)`은 **구문 오류**이다. (`INT(10)`은 **MySQL**에서 쓰던 표기이며 MySQL에서도 10은 저장 범위가 아니라 표시 너비일 뿐이다.)

| 타입 | 설명 |
|---|---|
| `INT` | 정수. 크기는 고정(약 ±21억). 크기를 적지 않는다 |
| `VARCHAR(n)` | **최대 n글자**의 문자열. 이 `(n)`은 필수는 아니지만 보통 적는다 |

### `TEXT`는 string과 다른 것인가? 크기를 안 쓰는가?

> ♻️ **string은 SQL의 타입 이름이 아니라 "문자열"이라는 개념**이다. SQL에서 문자열을 담는 타입이 `VARCHAR(n)`, `CHAR(n)`, `TEXT`이다.

| 타입 | 길이 | 쓰임 |
|---|---|---|
| `VARCHAR(n)` | 최대 n글자 | 이름, 이메일처럼 길이 한도가 예상되는 값 |
| `TEXT` | 제한 없음(크기를 적지 않는다) | 소개글, 후기 내용처럼 긴 글 |

### `CHECK`는 조건인가?

그렇다. **그 컬럼(또는 행)에 들어갈 수 있는 값의 조건을 정하는 제약**이다. 조건을 만족하지 않는 값을 넣으면 **오류**가 나며 **입력이 거부**된다.

> ♻️ 처음엔 **`CHECK`가 `WHERE`와 같은 것**처럼 느껴졌으나 쓰임이 다르다. `WHERE`는 **조회·수정·삭제 때 대상을 고르는** 조건이고, `CHECK`는 **테이블을 만들 때 정해 두는 데이터 규칙**이다.

### INT의 범위를 제한하고 싶다 (최대 20명)

```sql
max_guests INT NOT NULL CHECK (max_guests > 0 AND max_guests <= 20)
-- 또는
max_guests INT NOT NULL CHECK (max_guests BETWEEN 1 AND 20)
```

> ♻️ `CHECK (max_guests><20)`처럼 **한 식에 부등호 둘을 붙여 쓸 수 없다.** 두 조건은 `AND`로 잇거나 `BETWEEN`을 쓴다.

> ➕ **더 알아두기 — `CHECK`의 한계**
> `CHECK`는 **다른 테이블을 참조할 수 없다.** (예: "예약 인원은 객실의 `max_guests` 이하")는 `CHECK`로 만들 수 없고, 쿼리나 애플리케이션 코드 또는 트리거로 처리해야 한다.

### `booking_no`가 `UNIQUE`인 이유

예약 번호는 **사람이 화면에서 보고 예약을 찾는 번호**이다. 같은 번호가 두 예약에 있으면 안 되므로 `UNIQUE`를 건다. 내부 연결용 `booking_id`(PK)와 역할이 다르다.

### `status`의 기본값이 `확정`인 이유

예약이 만들어지는 순간 대부분 `확정` 상태로 시작하기 때문이다. 입력을 생략해도 `확정`이 들어가 **기본 상황을 매번 적는 수고**를 줄인다. 이후 `이용 완료`, `취소`로 바뀐다.

### SQL의 `IN`은 파이썬의 `in`과 같은가?

**비슷하다.** 둘 다 "목록 안에 있는가"를 묻는다.

| | SQL | 파이썬 |
|---|---|---|
| 문법 | `status IN ('확정', '취소')` | `status in ('확정', '취소')` |
| 뜻 | 값이 목록 중 **하나와 같은가** | 값이 목록 **안에 들어 있는가** |

SQL의 `IN`은 `status = '확정' OR status = '취소'`를 줄여 쓴 것과 같다. `NOT IN`도 가능하다.

### `DATE`는 무엇인가

**날짜(연-월-일)만** 담는 타입이다. 시·분·초가 필요하면 `TIMESTAMP`를 쓴다. 이 서비스에서 체크인·체크아웃·등록일·예약일은 날짜로 충분해서 `DATE`를 썼다.

| 타입 | 담는 것 | 예 |
|---|---|---|
| `DATE` | 날짜 | `2025-06-20` |
| `TIMESTAMP` | 날짜 + 시각 | `2025-06-20 14:30:00` |

> 📝 **기사 시험 포인트** (정보처리기사)
> DDL 명령은 `CREATE`, `ALTER`, `DROP`(+ `TRUNCATE`)이며, 제약조건은 **개체 무결성(PK)**, **참조 무결성(FK)**, **도메인 무결성(`CHECK`, `NOT NULL`, `DEFAULT`)** 과 연결해 외워 둔다.

---

## 주제 4: 데이터베이스 만들기와 연결

테이블을 만들기 전에 **데이터베이스 자체**가 필요하다. (원통 모양 아이콘 = 데이터베이스)

```sql
CREATE DATABASE accommodation_db;
```

> ♻️ 처음엔 MySQL처럼 **`USE 데이터베이스명`** 으로 전환하면 된다고 생각했으나 **PostgreSQL에는 `USE`가 없다.** PostgreSQL은 **연결 하나가 데이터베이스 하나**에 붙어 있어서, 다른 데이터베이스를 쓰려면 **새로 연결(접속)** 해야 한다. (VS Code의 PostgreSQL 확장에서는 연결을 새로 만들 때 데이터베이스를 지정한다.)

현재 어느 데이터베이스에 연결돼 있는지 확인하는 쿼리:

```sql
SELECT current_database();
```

테이블을 엉뚱한 데이터베이스(예: 기본 `postgres`)에 만들지 않으려면 **연결한 데이터베이스를 먼저 확인**하는 습관이 좋다.

---

## 주제 5: 내가 낸 오류와 고친 것

직접 작성한 `users` DDL에서 나온 오류들이다.

| 오류 | 원인 | 고침 |
|---|---|---|
| `INT(10)` | PostgreSQL은 `INT`에 크기를 적지 않는다 → **구문 오류** | `INT` |
| `NOT NULL` 앞의 **쉼표** | 컬럼 정의 안에서 제약조건 사이에 쉼표를 넣었다. 쉼표는 **컬럼과 컬럼 사이**에만 쓴다 | 쉼표 제거 |
| `phone`에 `UNIQUE` | "연락처는 유니크하게?"라고 판단했지만 가정이 어긋났다 | 아래 설명 |

### `UNIQUE`는 하나만 걸 수 있는가?

> ♻️ 처음엔 **`UNIQUE`가 테이블에 하나만** 가능한 줄 알았으나, **여러 컬럼에 각각** 걸 수 있다. (예: `email`과 `phone` 둘 다 `UNIQUE`) 컬럼 여러 개를 **묶어서** 중복을 막는 **복합 `UNIQUE`** 도 있다. (예: `UNIQUE (student_id, course_id)`) 반면 **`PRIMARY KEY`는 테이블에 하나뿐**이다.

연락처는 가족이 같은 번호를 쓸 수도 있어서 `UNIQUE`가 꼭 필요하지는 않다. 결국 `phone`은 **`VARCHAR(20) NOT NULL`** 로 정했다. (연락처는 반드시 받는다는 결정)

> 📝 **기사 시험 포인트** (정보처리기사)
> **PK는 `UNIQUE` + `NOT NULL`을 합친 성격**이고 테이블에 하나이다. `UNIQUE`는 **NULL 허용 여부와 개수 제한이 PK와 다르다**(여러 개 가능). 후보키 중 하나가 기본키가 되고 나머지는 대체키이다.

---

## 주제 6: VS Code 편집 요령 (`INT(10)` 수정)

DDL 여러 곳의 `INT(10)`을 한꺼번에 `INT`로 고치고 싶을 때 쓴 방법이다.

### 정규식으로 한 번에 바꾸기

1. `Ctrl + H` (바꾸기 열기)
2. 정규식 켜기: 검색창 오른쪽 `.*` 아이콘 (`Alt + R`)
3. 찾기: `INT\(\d+\)` → 바꾸기: `INT`
4. 모두 바꾸기 (`Ctrl + Alt + Enter`)

### 단어를 선택해서 한꺼번에 바꾸기 (다중 커서)

정규식 없이도 같은 단어를 **한 번에 여럿 선택해서 같이 고칠 수 있다.**

| 단축키 | 동작 |
|---|---|
| `Ctrl + D` | 현재 단어와 **같은 다음 단어를 하나씩 추가 선택** |
| `Ctrl + Shift + L` | 문서 안의 **같은 단어를 전부 선택** |

선택된 상태에서 그대로 입력하면 **모든 곳이 동시에 바뀐다.** 이 방법은 단어가 정확히 같을 때 쓴다. `INT(10)`처럼 괄호 안 숫자가 달라질 수 있으면 정규식이 편하다.

> ➕ **더 알아두기 — SQL 입력 도구**
> VS Code에서 `se`를 입력하고 `Tab`을 누르면 `SELECT` 구문 틀이 자동으로 나온다(IntelliSense/스니펫). Microsoft의 PostgreSQL 확장을 쓰면 쿼리를 바로 실행할 수 있다.

---

## 정리

- DDL은 ERD를 따라 **부모 테이블부터** 만든다: `users` → `host` → `accommodation` → `room` → `booking` → `review`, `wishlist`.
- PostgreSQL: **`INT`에 크기를 적지 않는다**(`INT(10)` 오류), `TEXT`는 크기 없이, `VARCHAR(n)`은 최대 길이. 자동 번호는 `GENERATED ALWAYS AS IDENTITY`. `user`는 예약어라서 `users`.
- 제약조건: `PRIMARY KEY`(테이블에 하나), `FOREIGN KEY ... REFERENCES`, `UNIQUE`(여러 개 가능, 복합 가능), `NOT NULL`, `CHECK`(조건, **다른 테이블 참조 불가**, 여러 조건은 `AND`/`BETWEEN`), `DEFAULT`.
- FK만 있으면 1:N, **FK + `UNIQUE`이면 1:0..1**(호스트–유저, 후기–예약).
- 예약은 **예약 당시 요금**(스냅샷)과 **상태**(`확정`/`이용 완료`/`취소`)를 저장하고, 박 수와 총액은 **계산**한다.
- PostgreSQL에는 **`USE`가 없다.** 다른 데이터베이스는 **새로 연결**한다. `SELECT current_database();`로 확인.
- SQL의 `IN`은 파이썬의 `in`과 비슷하다.

---

## ✅ 확인 질문

1. 테이블 7개를 어떤 순서로 만들어야 하는가? 순서가 틀리면 어떤 오류가 나는가?
2. `INT(10)`이 PostgreSQL에서 오류인 이유는? 이 표기는 어느 DB의 것인가?
3. `VARCHAR(n)`과 `TEXT`의 차이는? "string"이라는 SQL 타입이 있는가?
4. `CHECK`와 `WHERE`의 차이는? `CHECK`가 할 수 없는 일의 예를 하나 들라.
5. 인원 수를 1~20으로 제한하는 `CHECK`를 두 가지로 써라. `max_guests><20`이 틀린 이유는?
6. `users` 테이블 이름에 `s`가 붙은 이유는?
7. `booking_no`에 `UNIQUE`를 거는 이유는? `booking_id`와 역할은 어떻게 다른가?
8. `status`에 `DEFAULT '확정'`과 `CHECK (status IN (...))`를 함께 쓰는 이유는?
9. SQL의 `IN`과 파이썬의 `in`은 어떤 점이 같은가? `IN`을 `OR`로 풀어 써 보라.
10. `DATE`와 `TIMESTAMP`의 차이는? 이 서비스에서 `DATE`로 충분한 이유는?
11. `UNIQUE`는 한 테이블에 몇 개까지 걸 수 있는가? `PRIMARY KEY`와 어떻게 다른가? 복합 `UNIQUE`는 어떤 때 쓰는가?
12. 호스트의 `user_id`에 `UNIQUE`를 거는 이유는? FK만 있을 때와 무엇이 다른가?
13. 예약 테이블의 `price_per_night`와 객실 테이블의 `price_per_night`는 의미가 어떻게 다른가? 왜 예약에 요금을 또 저장하는가?
14. 박 수와 총 예약금액을 컬럼으로 저장하지 않고 계산하는 방법은?
15. PostgreSQL에서 다른 데이터베이스로 바꾸는 방법은? `USE`가 없는 이유와 현재 DB를 확인하는 쿼리는?
16. VS Code에서 같은 단어를 한꺼번에 바꾸는 방법 두 가지(정규식 / 다중 커서)를 말하라.
17. (기사 대비) 개체·참조·도메인 무결성을 각각 어떤 제약조건과 연결하는가? PK와 `UNIQUE`의 차이는?
