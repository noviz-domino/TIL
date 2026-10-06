---
tags: [sql, dml, insert, select, update, delete]
til: v2 2026-10-02
---

# 2026-10-02_DML(INSERT,_SELECT,_WHERE,_UPDATE,_DELETE)
> 작성일: 2026-10-02

## 🔗 관련 글

- [SQL 기초와 DDL](2026-10-02%20(4)SQL_기초와_DDL(명령어_분류,_트랜잭션,_제약조건).md) — DML을 DDL·DCL·TCL과 나누는 분류표, 이 글에서 쓰는 `student` 테이블을 만든 `CREATE TABLE`, 위험한 `UPDATE`/`DELETE`를 되돌리는 트랜잭션(`BEGIN`/`ROLLBACK`)이 거기에 있다.

## ❓ 아직 모르겠는 것

- VS Code에서 `demodb`로 **연결을 바꾸는 방법**을 안내받았으나, 사용하는 **확장(Extension) 이름**을 알려주지 않아 정확한 메뉴 경로는 확인되지 않았다.
- 연습 문제를 풀지 않았다.
  - `UPDATE student SET grade = 3;`(`WHERE` 없음)을 실행하면 어떻게 되는가?
  - `DELETE FROM student;`와 `TRUNCATE TABLE student;`는 결과가 비슷해 보이는데 어떤 차이가 있는가?
  - `UPDATE student SET major = '수학' WHERE grade = 3;`을 한국어로 읽고, `SET` 안의 `=`와 `WHERE` 안의 `=`의 뜻을 각각 말하라.
  - 한국(`KOR`) 도시 중 이름이 `S`로 시작하는 도시의 이름만 뽑는 쿼리를 써 보라.
  - 독립 연도(`indepyear`)가 NULL인 나라를 찾는 조건은 어떻게 쓰는가?
  - `WHERE`에서 `SELECT`로 만든 별칭을 쓰는 것과 `ORDER BY`에서 쓰는 것 중 오류가 날 가능성이 큰 쪽은 어느 것이며 이유는?
- 실습에서 쓴 쿼리 7개는 정상 문법이었다. 다만 5번 쿼리의 **세미콜론 누락**과 `indepyear`의 **NULL 행이 결과에서 빠지는 점**을 확인해야 한다. (주제 8 참고)

---

## 주제 1: DML이란

- **DML(Data Manipulation Language, 데이터 조작어)** 은 데이터를 **추가, 조회, 수정, 삭제**하는 기본적인 데이터 처리를 담당한다.

| 명령 | 하는 일 | 기본 모양 |
|---|---|---|
| `INSERT` | 새 데이터 **추가** | `INSERT INTO 테이블명 (컬럼1, 컬럼2) VALUES (값1, 값2);` |
| `SELECT` | 데이터 **조회** | `SELECT 컬럼 FROM 테이블명;` |
| `WHERE` | 조건으로 **필터링** | `SELECT * FROM 테이블명 WHERE 조건;` |
| `UPDATE` | 기존 데이터 **수정** | `UPDATE 테이블명 SET 컬럼 = 새값 WHERE 조건;` |
| `DELETE` | 데이터 **삭제** | `DELETE FROM 테이블명 WHERE 조건;` |

---

## 주제 2: INSERT — 데이터 추가

- 새로운 데이터를 테이블에 **추가**할 때 사용한다.
- **모든 컬럼의 값을 순서대로 다 적으면 컬럼을 생략**할 수 있다.

```sql
INSERT INTO 테이블명 (컬럼1, 컬럼2) VALUES (값1, 값2);
```

```sql
INSERT INTO student (id, name, grade, major)
VALUES ('2024001', '김철수', 1, '컴퓨터공학');
```

컬럼을 생략하고 **여러 행을 한 번에** 넣는 예이다.

```sql
INSERT INTO student VALUES
('2024002', '이영희', 2, '경영학'),
('2024003', '박민수', 3, '물리학');
```

- 문자열(`'2024001'`, `'김철수'`)은 작은따옴표로 감싸고, 숫자(`1`)는 따옴표 없이 쓴다.

> ➕ **더 알아두기 — 컬럼을 생략하는 방식의 주의점**
> 컬럼을 생략하는 방식은 **컬럼 순서에 의존**한다. 나중에 `ALTER TABLE`로 컬럼을 추가하거나 순서가 바뀌면 값이 엉뚱한 칸에 들어가거나 오류가 날 수 있어서, 실무에서는 **컬럼 이름을 명시하는 방식**을 더 권장하는 편으로 알고 있다. (수업에는 나오지 않은 일반 지식이다.)

---

## 주제 3: SELECT — 데이터 조회

데이터를 **조회**할 때 사용한다.

### 모든 필드 조회

```sql
SELECT * FROM 테이블명;
```

```sql
SELECT * FROM student;
```

### 특정 필드 조회

```sql
SELECT 테이블명.컬럼명1, 테이블명.컬럼명2 FROM 테이블명;
```

- 컬럼명 앞에 **`테이블명.`** 을 붙여 어느 테이블의 컬럼인지 나타낼 수 있다. **여러 테이블에서 같은 이름의 컬럼을 사용할 때 이를 구분**하며, 모호하지 않으면 생략할 수 있다.

```sql
SELECT 컬럼명1, 컬럼명2 FROM 테이블명;
```

```sql
SELECT id, name FROM student;
```

---

## 주제 4: WHERE — 조건으로 필터링

- `WHERE` 절은 데이터를 **필터링**할 때 사용하는 조건절이다.

```sql
SELECT * FROM 테이블명 WHERE 조건;
```

```sql
SELECT * FROM student WHERE grade = 2;
```

> ➕ **더 알아두기 — 문장 읽는 법**
> SQL은 "키워드 / 내가 지은 이름 / 연산자 / 값"으로 나눠 읽는다. `SELECT * FROM student WHERE grade = 2;`는 **"student 테이블에서, grade가 2인 행의, 모든 컬럼을 조회한다"** 로 읽는다. 앞에서 정리한 대로 `SELECT`, `FROM`, `WHERE`는 키워드이고, `student`, `grade`는 내가 지은 이름, `=`는 연산자, `2`는 값이다.

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
> **SELECT 문의 구조**는 `SELECT … FROM … WHERE … GROUP BY … HAVING … ORDER BY`처럼 절이 이어지는 형태로 시험에 나오는 것으로 알고 있다. 이 수업에서는 `SELECT`, `FROM`, `WHERE`까지만 나왔으므로, 나머지 절은 수업에서 나오면 정리한다.

---

## 주제 5: UPDATE — 데이터 수정

- **기존 데이터를 수정**할 때 사용한다.

```sql
UPDATE 테이블명 SET 컬럼 = 새값 WHERE 조건;
```

```sql
UPDATE student
SET grade = 2, major = '경제학'
WHERE id = '2024001';
```

> ➕ **더 알아두기 — `SET`은 무엇인가**
> (수업에서 나온 설명이 아니라 대화 중 정리한 설명이다. 레슨 본문은 문법 모양과 예제이다.)
>
> `SET`은 **`UPDATE` 문장 안에서 "무엇을 어떤 값으로 바꿀지" 적는 자리**이다. 영어 뜻 그대로 **"~로 설정하다"** 이다.
>
> | 부분 | 읽는 법 |
> |---|---|
> | `UPDATE student` | student 테이블을 수정한다 |
> | `SET grade = 2, major = '경제학'` | grade를 2로, major를 '경제학'으로 바꾼다 |
> | `WHERE id = '2024001'` | id가 '2024001'인 행만 |
>
> 한 줄로 읽으면 **"student 테이블에서 id가 2024001인 학생의 학년을 2로, 전공을 경제학으로 바꾼다"** 이다.
>
> - **바꿀 컬럼이 여러 개면 쉼표로 잇는다.** `SET grade = 2, major = '경제학'`. 하나만 바꿀 때는 `SET grade = 2`만 쓴다.
> - **`SET`은 "무엇을" 바꿀지, `WHERE`는 "어느 행을" 바꿀지**이다. 그래서 `WHERE`를 빼면 모든 행에 `SET`이 적용된다.
> - **같은 `=`라도 자리에 따라 뜻이 다르다.**
>
> | 위치 | `=`의 뜻 | 예 |
> |---|---|---|
> | `SET` 안 | **대입** ("이 값으로 바꿔라") | `SET grade = 2` |
> | `WHERE` 안 | **비교** ("같은 것만 골라라") | `WHERE id = '2024001'` |
>
> `SET grade = 2`는 "grade가 2냐?"가 아니라 "grade를 2로 만들어라"이다.
> - `SET` 오른쪽에는 새 값뿐 아니라 **기존 값을 이용한 계산식**도 쓸 수 있다. 트랜잭션 예시의 `SET balance = balance - 50000`은 "balance를 (지금 balance에서 50000을 뺀 값)으로 설정한다"는 뜻이다.
> - `UPDATE`는 `SET`과 짝으로 다닌다. (`INSERT`는 `VALUES`와 짝이다.)

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
> `UPDATE 테이블 SET 컬럼 = 값 WHERE 조건;` 형태는 빈칸 채우기나 결과 예측 문제로 나오는 기본 문법으로 알고 있다. `UPDATE`의 짝이 `SET`이라는 것, `WHERE`가 없으면 전체 행이 바뀐다는 것을 같이 기억한다.

---

## 주제 6: DELETE — 데이터 삭제

- **데이터를 삭제**할 때 사용한다.

```sql
DELETE FROM 테이블명 WHERE 조건;
```

```sql
DELETE FROM student
WHERE id = '2024002';
```

### ⚠️ WHERE를 생략하면

> **`UPDATE`와 `DELETE`에서 `WHERE`을 생략하면 모든 행이 수정 또는 삭제된다.** 특정 행만 처리하려면 반드시 조건을 지정해야 한다.

> ➕ **더 알아두기 — 위험한 명령의 안전장치**
> `DELETE FROM student;`처럼 `WHERE` 없이 실행하면 학생 전체가 삭제된다. 이럴 때 앞에서 배운 **트랜잭션**이 안전장치가 된다. `BEGIN;` 후 명령을 실행하고, 결과가 이상하면 `ROLLBACK;`으로 되돌리고, 맞으면 `COMMIT;`으로 확정한다. (상세는 [SQL 기초와 DDL](2026-10-02%20(4)SQL_기초와_DDL(명령어_분류,_트랜잭션,_제약조건).md)의 트랜잭션 부분)

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
> **DELETE / TRUNCATE / DROP의 차이**는 시험에서 자주 비교되는 것으로 알고 있다.
>
> | 명령 | 분류 | `WHERE` 사용 | 하는 일 |
> |---|---|---|---|
> | `DELETE` | DML | **가능** (조건 행만 삭제) | 행 삭제, 테이블 구조 유지 |
> | `TRUNCATE` | **DDL** | 불가 | **모든 행** 삭제, 구조 유지 |
> | `DROP` | DDL | 불가 | **테이블 구조까지** 삭제 |
>
> `DELETE`는 DML이고, 이름이 "데이터를 지우는 것"처럼 보이는 `TRUNCATE`는 DDL이다. DML의 4가지 명령어는 `INSERT`, `UPDATE`, `DELETE`, `SELECT`이다. (정리: 시험에서는 SELECT를 DML에 포함하는 분류가 일반적이다.)

> 📝 **기사 시험 포인트** (정보보안기사, 확실도: 높음)
> **SQL 인젝션**은 `WHERE` 조건에 들어갈 입력값을 조작해서 의도하지 않은 SQL이 실행되게 하는 공격이다. 로그인 쿼리의 조건 부분에 `' OR '1'='1` 같은 문자열을 끼워 넣어 **조건을 항상 참**으로 만드는 방식이 대표적이라고 알고 있다. 수업의 **"문자열은 작은따옴표로 감싼다"** 규칙과 이어지는데, 입력값이 따옴표를 닫아 버리면 뒤에 SQL을 끼워 넣을 수 있기 때문이다. 대응으로는 **입력값 검증**과 **매개변수화된 쿼리(Prepared Statement)** 가 거론된다.

---

## 주제 7: 실습 환경 — PostgreSQL에는 `USE`가 없다 (수업 외 · 대화 중 보충)

실습 중 "VS Code에서 `demodb`를 만들었는데 연결이 `postgres` DB에 붙어 있어서 `USE`로 `demodb`를 쓰고 싶다"는 질문이 있었다.

> ➕ **더 알아두기 — `USE`가 안 되는 이유와 해결**
> (수업에 없는 일반 지식이다.)
>
> - `USE demodb;`는 **MySQL이나 SQL Server**에서 쓰는 문법이며, **PostgreSQL에는 `USE` 문이 없다.** 수업의 "세부 문법과 동작은 DBMS마다 다를 수 있다"의 실제 사례이다.
> - PostgreSQL은 **접속할 때 어느 DB에 들어갈지 정하고, 그 연결 안에서는 DB를 바꾸지 못하는** 방식이다. `CREATE DATABASE demodb`로 DB를 만들어도 **연결은 여전히 `postgres`에 붙어 있다.** "방을 지었지만 아직 그 방에 들어가지 않은 것"과 같다.
> - 해결은 **demodb로 다시 연결**하는 것이다.
>   - **VS Code 연결 설정**의 Database 항목을 `demodb`로 바꾸거나, demodb용 **새 연결을 추가**한다. (확장 프로그램마다 메뉴가 다르다.)
>   - **psql**(PostgreSQL 전용 터미널 도구)에서는 `\c demodb`로 전환할 수 있다. 이것은 SQL 문장이 아니라 psql의 **메타 명령어**라서 VS Code의 SQL 편집창에서는 실행되지 않을 가능성이 크다.
>
> ```bash
> psql -U postgres -d demodb
> ```
>
> 한 연결이 한 DB라고 기억하면 된다. 같은 서버 안의 다른 DB를 조회하는 것은 기본적으로 안 되는 것으로 알고 있다.

---

## 주제 8: 실습 기록과 실습에서 나온 질문 (수업 외 · 대화 중 보충)

이 주제는 수업 내용이 아니라, `city`·`country` 테이블로 실습 문제를 풀면서 나온 쿼리와 질문을 정리한 것이다. 쿼리에 쓴 컬럼(`code`, `continent`, `region`, `indepyear`, `countrycode`, `population` 등)으로 보아 두 테이블이 있는 구조로 **추정**했다. 실제 DB 구조와 다를 수 있다.

### 실습 쿼리

```sql
-- 1. 인구가 800만을 넘는 도시
SELECT name, population
FROM city
WHERE population > 8000000;

-- 2. 한국(KOR)의 도시 이름과 국가코드
SELECT ct.name, countrycode
FROM city ct, country cr
WHERE (countrycode = cr.code) AND cr.code = 'KOR';

-- 3. 유럽 대륙에 속한 나라의 이름과 지역
SELECT name, region
FROM country cr
WHERE continent = 'Europe';

-- 4. 이름이 San으로 시작하는 도시
SELECT name
FROM city
WHERE name LIKE 'San%';

-- 5. 1901년 이전에 독립한 나라 (원래 쿼리 끝에 세미콜론이 없었다)
SELECT name, indepyear
FROM country
WHERE indepyear < 1901;

-- 6. 한국의 인구 100만 초과 200만 미만 도시
SELECT name
FROM city
WHERE (population > 1000000 AND population < 2000000) AND countrycode = 'KOR';

-- 7. 한국·일본·중국의 인구 500만 초과 도시
SELECT city.name, countrycode, city.population
FROM city
WHERE (city.population > 5000000) AND (countrycode IN ('KOR', 'JPN', 'CHN'));
```

> 위 코드는 실습에서 쓴 쿼리를 정리한 것이다. 5번에만 세미콜론을 보충했고 나머지는 키워드 대소문자와 공백만 맞췄다.

### 8-1. 문자열은 작은따옴표 — `"Europe"`이 오류가 나는 이유

실습 중 `WHERE continent = "Europe"`처럼 **큰따옴표**로 쓰면 되지 않았다. [SQL 기초와 DDL](2026-10-02%20(4)SQL_기초와_DDL(명령어_분류,_트랜잭션,_제약조건).md)의 수업 규칙 그대로이다.

- 수업: **문자열과 날짜 값은 작은따옴표(`'`)** 로 감싸고, 큰따옴표(`"`)는 이름(예약어를 이름으로 쓸 때)을 감쌀 때 쓴다.

> ➕ **더 알아두기 — 값은 작은따옴표, 이름은 큰따옴표**
>
> | 따옴표 | 뜻 | 예 |
> |---|---|---|
> | 작은따옴표 `' '` | **값**(문자열) | `'Europe'` → "Europe"라는 글자 |
> | 큰따옴표 `" "` | **이름**(컬럼명, 테이블명) | `"Europe"` → "Europe"이라는 **이름의 컬럼** |
>
> `"Europe"`을 쓰면 PostgreSQL이 `Europe`이라는 이름의 컬럼을 찾다가 오류를 낸다. (`column "Europe" does not exist` 같은 형태로 알고 있다. 실제 메시지는 직접 확인한다.) 이름은 **식별자(identifier)**, 값은 **리터럴(literal)** 이라는 정식 용어도 있으며, 검색할 때는 `string literal`, `identifier`, `PostgreSQL single quotes vs double quotes`처럼 정식 용어로 바꾸면 잘 나온다.
> 값 `'Europe'`은 **대소문자를 구분**하므로 `'europe'`로 쓰면 행이 안 나올 수 있다.

> ➕ **더 알아두기 — 표준 SQL인가, PostgreSQL만의 규칙인가**
>
> | 규칙 | 표준인가? |
> |---|---|
> | 값 = 작은따옴표 | ✅ 표준 |
> | 이름 = 큰따옴표 | ✅ 표준 |
> | 큰따옴표로 감싼 이름은 대소문자 구분 | ✅ 표준 |
> | 큰따옴표 없는 이름을 **소문자로** 처리 | ❌ PostgreSQL의 특징 (표준은 대문자로 처리, Oracle이 그렇다) |
>
> 다른 DBMS는 다르다. 아래는 일반 지식이며 버전과 설정에 따라 달라질 수 있다.
>
> | DBMS | 문자열 값 | 이름 감싸기 |
> |---|---|---|
> | PostgreSQL | `'값'` | `"이름"` |
> | Oracle | `'값'` | `"이름"` |
> | MySQL | `'값'` (`"값"`도 기본 설정에서 허용) | 백틱 `` `이름` `` |
> | SQL Server | `'값'` | `[이름]` 또는 `"이름"` |
>
> 가장 안전한 습관은 **값은 항상 작은따옴표**로 쓰는 것이다. 이번 실습에서 만난 두 오류(`USE`, `"Europe"`)는 모두 **다른 DBMS의 습관** 때문이었다.

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 중간)
> 시험의 SQL 문제는 보통 **표준 SQL 기준**으로 나오는 것으로 알고 있다. 따라서 **"값은 작은따옴표"** 로 외워 두면 안전하다. DBMS별 차이(백틱, 대괄호)가 출제되는지는 확인하지 못했다.

### 8-2. 쿼리를 검토하며 나온 포인트

> ➕ **더 알아두기 — 실습 쿼리에서 짚은 점**
>
> - **세미콜론 누락(5번)**: `;`가 없으면 파일 전체를 한 번에 실행할 때 다음 문장과 이어져 문법 오류가 날 수 있다. 모든 문장 끝에 `;`를 붙인다.
> - **NULL은 결과에서 빠진다(5번)**: `indepyear`가 NULL(값 없음)인 나라는 `indepyear < 1901`에 걸리지 않는다. NULL은 "모른다"는 뜻이라 비교 결과가 참도 거짓도 아니고, `WHERE`는 **참인 행만** 남기기 때문이다. NULL을 찾으려면 `= NULL`이 아니라 **`IS NULL`** 을 쓴다.
> - **조인이 필요 없을 수도 있다(2번)**: `city.countrycode`에 이미 `'KOR'`이 들어 있으므로 `SELECT name, countrycode FROM city WHERE countrycode = 'KOR';`로 충분하다. `country`를 엮는 것은 국가 이름처럼 `country`의 정보를 같이 보고 싶을 때 의미가 있다. 테이블을 쉼표로 나열하는 방식은 `WHERE`로 연결 조건을 걸지 않으면 **모든 조합**이 나오는 오래된 조인 방식이다.
> - **`cr`은 별칭(alias)이다(2, 3번)**: 긴 테이블 이름을 줄이거나, 여러 테이블을 엮을 때 어느 테이블인지 구분하려고 쓴다. 테이블이 하나일 때는 안 써도 결과가 같다.
> - **주석 처리한 `continent` 조건(1번)**: `city` 테이블에는 `continent`가 없고 `country`에 있어서 그대로 쓰면 오류가 난다. 대륙으로 거르려면 `country`와 엮어야 한다.
> - **`LIKE`는 대소문자를 구분한다(4번)**: `'san%'`로 쓰면 `San Francisco`가 안 나온다. 대소문자 구분 없이 찾는 것은 PostgreSQL의 **`ILIKE`** 로 알고 있다.
> - **`>`/`<`와 `BETWEEN`(6번)**: `>`/`<`는 양 끝값을 포함하지 않고, `BETWEEN 1000000 AND 2000000`은 **양 끝값을 포함**한다. 문제에 "이상/이하"가 나오면 `>=`, `<=`나 `BETWEEN`을 쓴다. (`BETWEEN`은 DDL 수업의 `CHECK (grade BETWEEN 1 AND 4)`에도 나왔다.)
> - **`IN`(7번)**: `countrycode IN ('KOR','JPN','CHN')`은 `countrycode = 'KOR' OR countrycode = 'JPN' OR countrycode = 'CHN'`을 짧게 쓴 것이다.

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
>
> - **`LIKE`와 와일드카드**: `%` = 길이와 상관없이 아무 문자열, `_` = 아무 한 글자. 예: `'San%'`은 San으로 시작, `'S_n'`은 S와 n 사이에 한 글자.
> - **`IN`, `BETWEEN`**: 기본 조건 연산자이며 `BETWEEN`은 양 끝값을 포함한다.
> - **`AND`와 `OR`의 우선순위**: **`AND`가 `OR`보다 먼저** 계산된다. 괄호를 쓰면 의도가 분명해진다.
> - **NULL 비교**: `= NULL`이 아니라 **`IS NULL` / `IS NOT NULL`**.

### 8-3. SQL의 처리 순서

> ➕ **더 알아두기 — 쓰는 순서와 처리하는 순서가 다르다**
> (수업에서 "SELECT가 제일 마지막"이라는 이야기가 나왔다고 하며, 아래는 대화 중 정리한 일반 지식이다. 수업 원문은 확인하지 못했다.)
>
> | | 순서 |
> |---|---|
> | 쓰는 순서 | `SELECT` → `FROM` → `WHERE` → `GROUP BY` → `HAVING` → `ORDER BY` |
> | 처리하는 순서 (논리적) | `FROM` → `WHERE` → `GROUP BY` → `HAVING` → **`SELECT`** → `ORDER BY` |
>
> - 쓰는 순서는 `select name from city where ...`이 **영어 문장처럼 읽히도록** 만든 것이다. (SEQUEL은 Structured **English** Query Language의 약자였다.)
> - 처리 순서는 "테이블 정하기(`FROM`) → 행 거르기(`WHERE`) → 컬럼 고르기(`SELECT`)"라는 **논리 순서**이다. 컬럼을 고르려면 먼저 대상이 정해져 있어야 한다. 실제 DBMS는 더 빠르게 하려고 내부에서 순서를 바꿔 실행할 수 있다.
> - 이 이유 설명은 일반적인 해석이며, 누군가 이 순서를 일부러 정했다는 기록을 확인한 것은 아니다.
> - **실습에서 바로 쓰이는 결과**: `WHERE`는 `SELECT`보다 먼저 처리되므로, `SELECT`에서 만든 별칭(`AS k`)을 `WHERE`에서 쓸 수 없다. `ORDER BY`는 `SELECT` 뒤에 처리되므로 별칭을 쓸 수 있다. (PostgreSQL 동작으로 알고 있으며 직접 실행해서 확인한다.)

> 📝 **기사 시험 포인트** (정보처리기사)
>
> - **관계대수의 기본 연산자**: **셀렉션 σ**(조건에 맞는 **행** 선택), **프로젝션 π**(원하는 **열** 선택), **조인 ⋈**, **디비전 ÷** 등이 시험에 나오는 것으로 알고 있다. (확실도: 높음)
> - **SQL의 `SELECT`는 프로젝션(열 선택), `WHERE`는 셀렉션(행 선택)** 에 해당한다. 이름이 서로 바뀐 것처럼 보여 가장 헷갈리는 포인트이다.
> - SQL 처리 순서(`FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY`)를 묻는 문제의 출제 여부는 확인하지 못했다. (확실도: 중간) `GROUP BY` 등은 수업에서 나오면 정리한다.

### 8-4. SQL은 누가 만들었는가

> ➕ **더 알아두기 — SQL의 역사**
> (일반 지식이며 연도 등 세부는 자료로 한 번 더 확인한다.)
>
> | 사람/조직 | 한 일 |
> |---|---|
> | 에드거 F. 커드(Edgar F. Codd) | 1970년 IBM에서 **관계형 모델**(데이터를 테이블로 다루는 이론)을 논문으로 제안 |
> | 도널드 챔벌린, 레이먼드 보이스 | 커드의 이론을 바탕으로 IBM에서 **SEQUEL**(나중에 SQL로 줄임)을 만듦 (1970년대 중반) |
> | 오라클(Oracle) | 이 아이디어로 **상용 제품**을 처음 내놓은 것으로 알려진 회사 (1979년경) |
> | ANSI, ISO | 1980년대에 **표준화** |
>
> "처음 만든 한 사람"이라기보다 **커드가 이론을 만들고 챔벌린과 보이스가 언어를 만들었다**고 보는 것이 정확하다.

### 8-5. VS Code 자동완성

> ➕ **더 알아두기 — `se` + `Tab`**
> (VS Code와 Microsoft PostgreSQL 확장의 일반 사용법이며, 아래 출처의 공식 문서를 확인했다.)
>
> - 실습 환경(VS Code에서 PostgreSQL 사용)에서 SQL 파일에 **`se`를 치고 `Tab`** 을 누르면 `select * from`까지 자동으로 입력되었다. `sele`까지 치면 `select`만 나왔다. 글자를 더 칠수록 추천 목록이 좁혀지기 때문이다.
> - Microsoft PostgreSQL 확장의 **IntelliSense**는 타이핑하면 자동으로 추천이 뜨고, 수동으로는 **`Ctrl + Space`**(맥은 `Cmd + Space`)로 띄운다. 연결된 DB를 분석해서 **키워드(SELECT, FROM, WHERE), 테이블, 컬럼, 함수, 스키마**를 추천한다.
> - 같은 확장에는 `pgSelectAll`, `pgCreateTable`, `pgInsertData` 같은 **내장 스니펫**(짧은 이름 + `Tab`으로 펼침)도 있다. `se`로 `select * from`이 나오는 정확한 원리는 확인하지 못했다.
> - 직접 스니펫을 만들려면 `Ctrl + Shift + P` → `Snippets: Configure Snippets` → `sql`에서 `prefix`와 `body`를 지정한다.
> - 설정 화면은 `Ctrl + ,` 이며, `Editor: Quick Suggestions`(자동 추천 표시), `Editor: Snippet Suggestions`(추천 목록에 스니펫 표시), `Editor: Inline Suggest: Enabled`(Copilot 같은 회색 인라인 추천)가 관련 설정이다.
> - 안 될 때는 설정을 건드리기 전에 **`Tab`, `Enter`, `Ctrl + Space`** 를 먼저 눌러 보고, 화면 오른쪽 아래 상태 표시줄에 파일 언어가 `SQL`(또는 `PostgreSQL`)로 되어 있는지 확인한다.
>
> 출처: [Query editor and IntelliSense - PostgreSQL extension for VS Code](https://learn.microsoft.com/en-us/azure/postgresql/development/vs-code-extension/query-editor-intellisense)

---

## 정리

- **DML**은 데이터를 추가·조회·수정·삭제하는 명령어로 `INSERT`, `SELECT`, `UPDATE`, `DELETE`이다.
- **INSERT**: `INSERT INTO 테이블명 (컬럼1, 컬럼2) VALUES (값1, 값2);` 모든 컬럼 값을 순서대로 다 적으면 컬럼 생략 가능, `VALUES` 뒤에 여러 행을 한 번에 넣을 수 있다. (컬럼 생략은 순서에 의존하므로 실무에서는 명시를 권장)
- **SELECT**: `SELECT * FROM 테이블명;`(전체), `SELECT 컬럼1, 컬럼2 FROM 테이블명;`(특정 컬럼). 같은 이름의 컬럼이 여러 테이블에 있으면 `테이블명.컬럼명`으로 구분한다.
- **WHERE**: 조건으로 행을 필터링하는 절. `SELECT * FROM student WHERE grade = 2;`
- **UPDATE**: `UPDATE 테이블명 SET 컬럼 = 새값 WHERE 조건;` `SET`은 무엇을 바꿀지, `WHERE`는 어느 행을 바꿀지이다. 컬럼이 여러 개면 쉼표로 잇는다. `SET` 안의 `=`는 대입, `WHERE` 안의 `=`는 비교이다.
- **DELETE**: `DELETE FROM 테이블명 WHERE 조건;`
- **⚠️ `UPDATE`와 `DELETE`에서 `WHERE`를 생략하면 모든 행이 수정·삭제된다.** 트랜잭션(`BEGIN`/`ROLLBACK`)이 안전장치가 된다.
- (수업 외) `DELETE`(DML, 조건 행) / `TRUNCATE`(DDL, 전체 행) / `DROP`(DDL, 구조까지) 비교. PostgreSQL에는 `USE`가 없고 연결 자체를 `demodb`로 바꿔야 한다.
- (수업 외 · 실습) **값은 작은따옴표, 이름은 큰따옴표**(표준 SQL). `"Europe"`처럼 값을 큰따옴표로 쓰면 컬럼 이름으로 인식되어 오류가 난다. 쿼리 끝에는 `;`를 붙이고, NULL은 `IS NULL`로 찾으며, `BETWEEN`은 양 끝값을 포함한다. SQL은 `FROM → WHERE → … → SELECT → ORDER BY` 순서로 처리되어 `WHERE`에서는 `SELECT`의 별칭을 쓸 수 없다. VS Code에서는 `se` + `Tab`으로 `select * from`이 자동완성된다.

---

## ✅ 확인 질문

1. DML의 네 가지 명령어는 무엇이며, 각각 무엇을 하는가?
2. `INSERT`에서 컬럼 이름을 생략할 수 있는 조건은? 이 방식의 주의점은?
3. `VALUES` 뒤에 여러 행을 한 번에 넣는 문법을 쓰라.
4. `INSERT` 예제에서 문자열과 숫자는 따옴표를 어떻게 다르게 쓰는가?
5. `SELECT *`와 `SELECT id, name`의 차이는? `테이블명.컬럼명` 형태는 언제 필요한가?
6. `SELECT * FROM student WHERE grade = 2;`를 한국어로 풀어 읽어 보라.
7. `UPDATE`에서 `SET`과 `WHERE`는 각각 무엇을 정하는가? 바꿀 컬럼이 여러 개일 때는 어떻게 쓰는가?
8. `SET grade = 2`의 `=`와 `WHERE id = '2024001'`의 `=`는 각각 어떤 뜻인가?
9. `UPDATE student SET grade = 3;`을 `WHERE` 없이 실행하면 어떻게 되는가?
10. `DELETE FROM student;`를 실수로 실행했을 때 되돌리려면 어떤 방법이 필요하며, 그 방법은 어떻게 써야 하는가?
11. `DELETE`, `TRUNCATE`, `DROP`은 각각 무엇을 지우며 어떤 분류(DML/DDL)인가? `WHERE`는 어느 것에서 쓸 수 있는가?
12. (기사 대비) `UPDATE ... SET ... WHERE ...`에서 `WHERE`를 빼면 어떤 결과가 나오는가?
13. (정보보안기사 대비) SQL 인젝션이 `WHERE` 조건과 작은따옴표 규칙을 이용하는 원리를 설명하라. 대응 방법은?
14. PostgreSQL에서 `USE demodb;`가 안 되는 이유는? `demodb`를 사용하려면 어떻게 해야 하는가?
15. `WHERE continent = "Europe"`이 오류가 나는 이유는? 작은따옴표와 큰따옴표는 각각 무엇을 감싸는가?
16. 값(리터럴)과 이름(식별자)의 따옴표 규칙 중 표준 SQL인 것과 PostgreSQL만의 특징인 것을 구분하라.
17. 독립 연도가 NULL인 나라가 `WHERE indepyear < 1901`의 결과에 나오지 않는 이유는? NULL을 찾는 올바른 조건은?
18. `population BETWEEN 1000000 AND 2000000`과 `population > 1000000 AND population < 2000000`의 차이는?
19. `LIKE 'San%'`와 `LIKE 'S_n'`은 각각 어떤 값에 일치하는가? `LIKE`는 대소문자를 구분하는가?
20. `AND`와 `OR`가 섞인 조건에서 어느 것이 먼저 계산되는가? 괄호는 왜 쓰는가?
21. SQL의 쓰는 순서와 처리하는 순서를 각각 쓰고, 처리 순서 때문에 `WHERE`에서 `SELECT`의 별칭을 쓸 수 없는 이유를 설명하라.
22. (기사 대비) 관계대수의 셀렉션과 프로젝션은 각각 SQL의 무엇에 해당하며 행/열 중 무엇을 선택하는가?
23. SQL을 만든 사람과 조직은? 이름 SEQUEL의 뜻은?
24. VS Code에서 SQL 키워드 자동완성을 수동으로 띄우는 단축키는? 추천이 안 뜰 때 설정을 보기 전에 확인할 것 3가지는?
