---
tags: [sql, group-by, having, aggregate-function, select]
til: v2 2026-10-02
---

# 2026-10-02_SELECT_advanced(GROUP_BY,_집계함수,_HAVING)
> 작성일: 2026-10-02

## 🔗 관련 글

- [DML(INSERT, SELECT, WHERE, UPDATE, DELETE)](2026-10-02%20(5)DML(INSERT,_SELECT,_WHERE,_UPDATE,_DELETE).md) — `SELECT`, `WHERE`의 기본형. 이 글은 그 위에 `GROUP BY`, `HAVING`을 얹는다. SQL 처리 순서와 `ORDER BY`의 별칭 규칙도 거기서 먼저 다뤘다.
- [SQL 기초와 DDL](2026-10-02%20(4)SQL_기초와_DDL(명령어_분류,_트랜잭션,_제약조건).md) — `NULL`의 뜻, 키워드·이름·값 구분, 별칭(`AS`) 개념이 거기에 있다.

## ❓ 아직 모르겠는 것

- **정보처리기사·정보보안기사의 최신 출제기준**을 확인하지 않았다. 이 글의 `📝` 블록은 일반 지식에 따른 것이라 시험 범위인지 확정되지 않았다. (공식 출제기준을 확인해 `📝`를 정리하기로 했으나 보류 중이다.)
- 연습 문제를 풀지 않았다.
  - 5행 나라 표에서 `WHERE continent = 'Europe'`를 `GROUP BY`와 함께 쓴 쿼리의 결과는 몇 행이고 어떤 값인가?
  - "평균 인구가 1000만이 넘는 지역(region)의 이름과 평균 인구"는 `WHERE`와 `HAVING` 중 어디에 조건을 쓰는가?
  - `HAVING country_count > 20`이 오류인 이유는?
  - NULL이 섞인 그룹에서 `COUNT(*)`와 `COUNT(컬럼)`은 각각 얼마이며 "대륙별 나라 수"에 더 안전한 쪽은?
  - "전체 도시 중 인구 1위 1개"와 "나라마다 인구 1위 도시 1개씩" 중 `LIMIT 1`로 풀 수 있는 것은?
  - 수업의 `GROUP BY` 실습 문제 중 5번만 풀었다. 나머지와 `HAVING` 실습 문제는 풀지 않았다.

---

## 주제 1: GROUP BY

- `GROUP BY`는 데이터를 **지정된 컬럼 기준으로 그룹화**하고, **각 그룹에 대한 집계 함수**를 사용할 수 있다.
- 그룹별 결과를 조회할 때는 `GROUP BY`에 지정한 컬럼과 집계 함수를 사용하며, 이를 이용한 **계산식**도 사용할 수 있다.

### 집계 함수

**집계 함수(aggregation function)** 는 여러 행의 데이터를 요약하여 **하나의 결과값**을 반환하는 함수이다.

| 함수 | 뜻 |
|---|---|
| `COUNT(*)` | 전체 **행 개수** |
| `COUNT(컬럼)` | 해당 컬럼이 **NULL이 아닌** 행 개수 |
| `SUM(컬럼)` | 합계 |
| `AVG(컬럼)` | 평균 |
| `MIN(컬럼)` | 최솟값 |
| `MAX(컬럼)` | 최댓값 |

- `SUM`, `AVG`, `MIN`, `MAX`는 **NULL을 제외하고** 계산한다.

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
> - **집계 함수**: COUNT, SUM, AVG, MAX, MIN. 시험에서는 **그룹 함수**라고도 부르는 것으로 알고 있다.
> - **`COUNT(*)` vs `COUNT(컬럼)`**: 전자는 NULL을 포함한 전체 행, 후자는 **NULL이 아닌 행만** 센다.

### 대륙별 국가 수

```sql
SELECT continent, COUNT(*) AS country_count
FROM country
GROUP BY continent;
```

### Region별 국가 평균 인구

```sql
SELECT region, AVG(population) AS avg_pop
FROM country
GROUP BY region;
```

### 대륙별 최소 / 최대 인구

```sql
SELECT continent,
       MIN(population) AS min_pop,
       MAX(population) AS max_pop
FROM country
GROUP BY continent;
```

### 아시아 국가들의 Region별 국가 수

```sql
SELECT region, COUNT(*) AS country_count
FROM country
WHERE continent = 'Asia'
GROUP BY region;
```

### (참고) 대륙별 governmentform에 따른 개수

`GROUP BY`에 **여러 컬럼을 지정하면 각 값의 조합별로 그룹화**한다.

```sql
SELECT continent, governmentform, COUNT(*)
FROM country
GROUP BY continent, governmentform
ORDER BY continent, governmentform;
```

### 실습 (수업에서 제시된 문제)

1. 대륙별 총 인구수를 구하시오.
2. 대륙별 평균 GNP와 평균 인구를 구하시오.
3. 인구가 50만 이상 100만 이하인 도시들을 대상으로, CountryCode와 District별 도시 수를 구하시오.
4. 아시아 대륙 국가들의 Region별 총 GNP를 구하시오.
5. 대륙별 국가 수가 많은 순서대로 Continent, 국가 수를 조회하시오.
6. 독립년도가 있는 국가들의 대륙별 평균 기대수명이 높은 순서대로 Continent, 평균 기대수명을 조회하시오.
7. Region별 총 GNP를 구하고, 총 GNP가 가장 높은 Region을 조회하시오. (GNP: 국민 총생산)

> 실습 5번은 다음과 같이 풀었다.
>
> ```sql
> SELECT continent, COUNT(*) AS country_count
> FROM country
> GROUP BY continent
> ORDER BY count(name) DESC;
> ```
>
> 문법과 결과는 맞다. 다만 `SELECT`에는 `COUNT(*)`, `ORDER BY`에는 `COUNT(name)`을 써서 의미가 조금 다르므로(아래 참고), **`ORDER BY country_count DESC`처럼 별칭으로 통일**하는 것이 좋다. 같은 숫자의 국가 수가 있으면 그 사이 순서는 정해져 있지 않으므로 `ORDER BY country_count DESC, continent`처럼 기준을 하나 더 붙이면 고정된다.

---

## 주제 2: GROUP BY를 이해하기 (수업 외 · 대화 중 보충)

이 주제는 수업 본문이 아니라 `GROUP BY`가 처음에 이해되지 않아 질문하며 정리한 내용이다. 설명에 쓴 나라 표는 **설명용으로 만든 가짜 데이터**이다.

> ➕ **더 알아두기 — GROUP BY는 왜 쓰는가**
>
> | name | continent |
> |---|---|
> | Korea | Asia |
> | Japan | Asia |
> | France | Europe |
> | Germany | Europe |
> | Italy | Europe |
>
> "대륙별로 나라가 몇 개씩 있나?"를 `GROUP BY` 없이 하려면 대륙마다 `WHERE continent = 'Asia'`, `'Europe'`, …를 따로 써야 하고, **대륙 이름을 미리 알아야** 한다. `GROUP BY continent`는 이를 한 번에 해 준다. **"~별로 요약"** 이라는 질문(대륙별, 지역별, 국가별)에 쓴다.
>
> - **"안의 내용물을 모를 때 쓴다"** 는 절반만 맞다. 값을 몰라도 알아서 묶는 것은 맞지만, 값을 이미 알아도 쓴다. 진짜 이유는 **그룹별로 계산**하기 위해서이다.
> - 목록만 궁금하면 `SELECT DISTINCT 컬럼`이 어울리고, 값별로 개수·합계·평균이 궁금하면 `GROUP BY` + 집계 함수이다. (`DISTINCT`는 수업에 나오지 않은 일반 지식이다.)

> ♻️ 처음엔 **`GROUP BY`를 "안에 어떤 값이 있는지 모를 때 쓰는 것"** 으로 알았으나, 값을 알아도 쓰며 **핵심은 그룹별로 계산(요약)하는 것**이다.

> ➕ **더 알아두기 — 동작 3단계와 "그룹별 작은 표" 모델**
>
> | 단계 | 하는 일 | 예 |
> |---|---|---|
> | ① 묶기 | `GROUP BY continent` 기준으로 같은 값끼리 묶음 | Asia(Korea, Japan) / Europe(France, Germany, Italy) |
> | ② 계산 | 각 묶음마다 집계 함수를 한 번씩 적용 | Asia → 2, Europe → 3 |
> | ③ 결과 | **그룹 하나가 결과 행 하나**가 됨 | 2행 |
>
> 결과의 행 수는 원래 행 수(5개)가 아니라 **그룹의 수(2개)** 이다.
>
> **이해용 모델**: 그룹은 `country`의 행들을 묶은 **그룹별 작은 표**(Asia 표, Europe 표)이고, 그 안에는 **원본 행이 통째로**(모든 컬럼) 들어 있다고 생각하면 된다. 그래서 집계 함수가 `COUNT(name)`, `AVG(population)`처럼 어느 컬럼이든 들여다볼 수 있다. 각 작은 표에서 집계 함수가 계산한 값을 모은 것이 쿼리의 **결과 표(결과 집합, result set)** 이다.
>
> | 질문 | 답 |
> |---|---|
> | 그룹별 작은 표가 쿼리 실행 중에 생기는가? | 논리적으로 그렇게 생각하면 된다 |
> | 진짜 테이블로 **저장**되는가? | ❌ 쿼리가 끝나면 사라진다. 원본 `country`는 그대로(5행)이다 |
> | 그 작은 표에 이름을 붙여 다시 조회할 수 있는가? | ❌ |
>
> 이 모델은 이해를 돕기 위한 것이다. **실제 DBMS는 그룹별로 행을 통째로 모아 두지 않고, 계산에 필요한 값만 들고 있는 경우가 많다.** (`COUNT(*)`면 개수 하나, `SUM`이면 합계 하나, `AVG`면 합계와 개수 두 개, `MAX`면 지금까지의 최댓값 하나.) PostgreSQL의 `EXPLAIN`으로 처리 방식(`HashAggregate`, `GroupAggregate` 등)을 볼 수 있는 것으로 알고 있다. 결과를 진짜로 저장하려면 `CREATE TABLE 새테이블 AS SELECT ...`나 뷰(VIEW)를 쓴다.
> 이는 **연관 테이블은 연산된 결과가 아니라 직접 저장한 원본**이라는 앞의 정리의 반대편이다. `GROUP BY`나 `JOIN`의 결과는 저장되지 않는 연산 결과이다.

> ➕ **더 알아두기 — 기준 컬럼이 여러 개일 때**
>
> `GROUP BY continent, governmentform`은 **(대륙, 정부 형태) 조합이 같은 행끼리** 한 그룹이 된다. 두 컬럼의 값이 **모두** 같아야 같은 그룹이다.
>
> - 조합이 다른 행은 **버려지는 것이 아니라 자기만의 별도 그룹**이 된다. (`WHERE`로 먼저 걸러진 행은 제외)
> - 기준 컬럼을 늘리면 그룹이 잘게 쪼개지고, 줄이면 합쳐진다. 예를 들어 `GROUP BY governmentform`만 쓰면 (Asia, Republic)과 (Europe, Republic)이 한 그룹으로 합쳐진다.
> - 문자열 값은 대소문자를 구분하므로 `'Republic'`과 `'republic'`은 서로 다른 그룹이 된다.
> - NULL 값끼리는 한 그룹으로 묶이는 것으로 알고 있다. (직접 실행해서 확인한다.)

> ♻️ 처음엔 **`GROUP BY`가 기준 컬럼을 딱 하나만 정하는 것**으로 알았으나, 수업의 (참고) 예제처럼 **여러 컬럼을 지정하면 값의 조합별로 그룹화**한다.

---

## 주제 3: SELECT에 무엇을 쓸 수 있는가

> ➕ **더 알아두기 — 컬럼과 집계 함수를 같이 쓰는 이유와 규칙**
>
> `GROUP BY`의 결과는 **그룹마다 한 줄**이고, 그 줄에는 두 가지 정보가 필요하다.
>
> | 필요한 정보 | 쓰는 것 |
> |---|---|
> | "어느 그룹이야?" (이름표) | **묶는 컬럼** (`continent`) |
> | "그 그룹의 값은?" (요약 값) | **집계 함수** (`COUNT(*)`) |
>
> 집계 함수만 쓰면 숫자만 나와서 어느 그룹의 숫자인지 알 수 없다. 그래서 `SELECT continent, COUNT(*)`처럼 쓴다. 레슨 본문은 "그룹별 결과를 조회할 때는 `GROUP BY`에 지정한 컬럼과 집계 함수를 사용한다"이다.
>
> - **`GROUP BY`에 적은 컬럼**은 SELECT에 쓸 수 있다. 그룹 안에서 값이 모두 같아서 한 줄에 하나만 적을 수 있기 때문이다.
> - **`GROUP BY`에 없는 컬럼**(예: `name`)은 쓸 수 없다. 그룹 안에 값이 여러 개(Korea, Japan)라서 한 줄에 하나만 적을 수 없고, PostgreSQL이 `column "name" must appear in the GROUP BY clause or be used in an aggregate function` 같은 오류를 낸다. (오류 문구는 직접 확인한다.)
> - **`GROUP BY` 없이** 집계 함수와 일반 컬럼을 같이 쓰면(`SELECT name, COUNT(*) FROM city;`) 오류가 난다. 집계 함수는 **전체를 한 줄로 요약**하는데 `name`은 행마다 값이 달라서 한 줄에 적을 수 없다. 결과 표는 줄마다 칸 수가 같은 직사각형이어야 하는데 줄 수가 다른 둘을 섞을 수 없기 때문이다.
> - `SUM(*)`은 안 되며 **`SUM(컬럼)`** 으로 쓴다. `*`가 되는 것은 `COUNT(*)`뿐이다.
>
> "인구도 구하고 name도 궁금하다"는 상황의 해결책은 다음과 같다.
>
> | 원하는 것 | 방법 |
> |---|---|
> | 가장 큰/작은 행의 **상세 정보** | `ORDER BY population DESC LIMIT 1` |
> | 각 행 옆에 **전체 합계** | 윈도우 함수 `SUM(population) OVER ()` (나중에) |
> | **그룹별** 합계 | `GROUP BY` |

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
> **`GROUP BY` 없이 집계 함수와 일반 컬럼을 같이 쓸 수 없다**, 그리고 **`GROUP BY`를 쓰면 SELECT에는 그룹 기준 컬럼과 집계 함수만 올 수 있다**는 내용은 SQL 문제에서 오류를 찾는 형태로 나오는 것으로 알고 있다.

### COUNT(*)와 COUNT(컬럼)

> ➕ **더 알아두기 — 무엇을 세는가**
>
> | 쓴 것 | 그룹 안에서 세는 대상 |
> |---|---|
> | `COUNT(*)` | **행 자체** (컬럼은 안 봄, NULL 무관) |
> | `COUNT(name)` | 행 중에서 **`name`이 NULL이 아닌 것** |
>
> - **`COUNT(*)`의 `*`는 "행 전체"** 를 뜻한다. 아무것도 안 정한 것이 아니라 센 대상이 컬럼이 아니라 행이다.
> - **괄호 안의 컬럼은 "무엇을 셀지(어느 컬럼의 값을 볼지)"** 이고, **묶는 기준은 `GROUP BY`** 뿐이다. 그룹 나누는 설정은 `GROUP BY`가 하고, 집계 함수는 이미 나뉜 각 그룹 안에서 알아서 한 번씩 계산한다. `GROUP BY` 없이 쓰면 테이블 **전체를 하나의 그룹**으로 본다.
> - Asia 그룹에 `name`이 NULL인 행이 하나 섞이면(Korea, Japan, NULL) `COUNT(*)`는 **3**, `COUNT(name)`은 **2**이다.
> - 대륙 값이 NULL인 행이 있으면 NULL끼리 한 그룹이 되는데, `COUNT(continent)`는 그 그룹에서 **0**이 나오고 `COUNT(*)`는 실제 행 수가 나온다. 그래서 **"행이 몇 개냐"를 셀 때는 `COUNT(*)`가 안전**하다. NULL이 없는 데이터에서는 어느 쪽을 써도 같은 숫자가 나와 차이를 못 느끼기 쉽다.
> - 정말 "서로 다른 값이 몇 종류냐"를 세려면 **`COUNT(DISTINCT name)`** 을 쓰는 것으로 알고 있다. (수업에 나오지 않았다.)

> ♻️ 처음엔 **`COUNT(name)`을 "name의 같은 값을 센다"** 로 알았으나, **그룹 안에서 `name`이 NULL이 아닌 행의 개수**가 맞다. 같은 값끼리 묶는 것은 `GROUP BY name`이 하는 일이다.

> ♻️ 처음엔 **`COUNT`를 키워드(명령어)** 로 알았으나, **`COUNT`는 키워드가 아니라 함수**이다. 키워드는 SQL 문장의 뼈대(`SELECT`, `FROM`)이고, 함수는 괄호를 쓰고 값을 계산해 돌려준다.

> ➕ **더 알아두기 — 결과 열의 이름**
>
> - `SELECT`에 적은 항목은 **저장된 컬럼이든 계산한 값이든 각각 결과의 열이 된다.** `COUNT(*)`의 결과도 원본 테이블에 없는, **결과 표에만 임시로 생기는 새 열**이다.
> - 이름을 안 붙이면 PostgreSQL은 **함수 이름을 그대로** 열 이름으로 쓴다. `COUNT(*)` → `count`, `SUM(...)` → `sum`, 계산식(`population / 1000`)은 단서가 없어 `?column?`이 되는 것으로 알고 있다. MySQL 같은 곳은 식 전체가 이름이 되는 것으로 알고 있어 DBMS마다 다르다.
> - 그래서 `AS`로 읽기 좋은 이름(`AS country_count`)을 붙이는 것이 좋고, 어느 DBMS에서나 안전하다. 자동 이름은 결과 표의 머리글일 뿐이라 예약어 규칙과 무관하다. 다만 `count`를 컬럼명으로 짓는 것은 피한다.
> - **원본 컬럼**(`CREATE TABLE`로 정의, 저장됨)과 **결과 열**(`SELECT`마다 만들어짐, 저장 안 됨)은 다른 개념이다.

> ➕ **더 알아두기 — 테이블의 컬럼 이름을 확인하는 법**
> 그룹을 만들어도 그룹은 원본 테이블의 행을 나눠 놓은 것이라 원본의 모든 컬럼을 계산에 쓸 수 있다. 컬럼 이름은 테이블 구조(스키마)를 확인해서 안다.
>
> - `SELECT * FROM country LIMIT 5;` 로 결과의 열 머리글을 본다.
> - VS Code의 PostgreSQL 확장 왼쪽 DB 탐색 패널에서 테이블을 펼친다.
> - 아래 쿼리로 `information_schema`(DB가 테이블·컬럼 정보를 정리해 둔 곳)를 조회한다.
>
> ```sql
> SELECT column_name, data_type
> FROM information_schema.columns
> WHERE table_name = 'country';
> ```
>
> `psql` 터미널에서는 `\d country`로도 볼 수 있는 것으로 알고 있다.

---

## 주제 4: HAVING

- `HAVING`은 **`GROUP BY`의 결과에 조건을 거는 절**이다.
- **`WHERE`는 개별 행 데이터, `HAVING`은 그룹화된 결과**를 필터링한다.
- 즉 `HAVING`은 **집계 함수와 함께** 쓰이는 경우가 많다.

### SELECT문의 논리적 처리 순서

```sql
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

- `WHERE`로 행을 필터링한 뒤, `GROUP BY`로 그룹화하고 집계한 결과를 `HAVING`으로 필터링한다.

| | `WHERE` | `HAVING` |
|---|---|---|
| 거르는 대상 | **개별 행** | **그룹** |
| 시점 | 그룹 만들기 **전** | 그룹 만든 **후** |
| 집계 함수 사용 | ❌ | ✅ |

### 대륙별 국가 수가 20개가 넘는 대륙, 국가 수 조회

```sql
SELECT continent, COUNT(*) AS country_count
FROM country
GROUP BY continent
HAVING COUNT(*) > 20;
```

- **`SELECT`에서 지정한 별칭은 `HAVING`에서 사용할 수 없으므로, 집계 함수를 직접 작성한다.**

### Region별 평균 인구가 1000만이 넘는 지역, 평균 인구 조회

```sql
SELECT region, AVG(population) AS avg_pop
FROM country
GROUP BY region
HAVING AVG(population) > 10000000;
```

### 인구가 1000만 이상인 국가의 수가 10개가 넘는 대륙의 이름과 국가 수 조회

```sql
SELECT continent, COUNT(*) AS big_countries
FROM country
WHERE population >= 10000000
GROUP BY continent
HAVING COUNT(*) > 10;
```

- 문장이 `WHERE`(행: 인구 1000만 이상인 나라) → `GROUP BY`(대륙별로 묶기) → `HAVING`(그룹: 그런 나라가 10개 넘는 대륙) 순서 그대로 나뉜다.

### 평균 인구수가 1000만이 넘는 대륙의 국가 수

- **`HAVING`절에는 `SELECT`절에 포함되지 않는 집계 함수가 포함될 수 있다.**

```sql
SELECT continent, COUNT(*) AS country_count
FROM country
GROUP BY continent
HAVING AVG(population) > 10000000;
```

### 실습 (수업에서 제시된 문제)

1. 각 국가별 도시가 10개 이상인 국가의 CountryCode, 도시 수를 조회하시오.
2. CountryCode와 District별로 집계하여, 평균 인구가 100만 이상이면서 도시 수가 3개 이상인 그룹의 CountryCode, District, 도시 수, 총 인구를 구하시오.
3. 아시아 대륙의 국가들 중에서, Region별 평균 GNP가 1000 이상인 Region, 평균 GNP를 조회하시오.
4. 독립년도가 1900년 이후인 국가들 중에서, 대륙별 평균 기대수명이 70세 이상인 Continent, 평균 기대수명을 조회하시오.
5. 도시 평균 인구가 100만 이상이고, 도시 최소 인구가 50만 이상인 국가의 CountryCode, 총 도시 수, 총 인구수를 조회하시오.
6. 인구가 50만 이상인 도시만 대상으로 국가별로 집계하여, 평균 인구가 100만 이상인 국가의 CountryCode, 해당 도시 수, 해당 도시들의 인구 합계를 구하시오.

> ➕ **더 알아두기 — HAVING은 그룹을 통째로 남기거나 뺀다**
>
> `HAVING COUNT(*) > 2`를 걸면 각 그룹이 조건을 통과하느냐 못 하느냐 둘 중 하나로 갈린다. Asia(2행)는 그룹 전체(Korea, Japan)가 사라지고, Europe(3행)은 그룹 전체가 남는다. **그룹 안의 행을 일부만 자르지는 않는다.** 조건은 그룹 하나당 한 번 판단한다.
>
> - `WHERE`는 행 하나하나를, `HAVING`은 그룹 하나씩을 도려낸다.
> - 그래서 `HAVING`에는 **그룹 전체를 대표하는 값**(개수, 평균, 합계)을 쓴다.
> - `WHERE`가 `GROUP BY`보다 앞이라 `COUNT(*)` 같은 그룹 계산값을 쓸 수 없고, `HAVING`이 `SELECT`보다 앞이라 `SELECT`의 별칭을 쓸 수 없다.
> - 행 단위 조건으로 일부 행만 남긴 채 집계하려면 `HAVING`이 아니라 `WHERE`를 `GROUP BY` 앞에 쓴다.

> 📝 **기사 시험 포인트** (정보처리기사, 확실도는 항목마다 표시)
> - **`WHERE`와 `HAVING`의 차이** (확실도: 높음): `WHERE`는 행을 거르고 `GROUP BY` 앞에, `HAVING`은 그룹을 거르고 `GROUP BY` 뒤에 온다. **집계 함수는 `WHERE`에 쓸 수 없고 `HAVING`에 쓴다.** 시험의 단골 비교 포인트로 알고 있다.
> - **SELECT 문의 구조** (확실도: 높음): `SELECT … FROM … WHERE … GROUP BY … HAVING … ORDER BY`. 이제 이 순서의 부품이 거의 나왔다.
> - **`GROUP BY` 없이 `HAVING`만 쓰는 경우** (확실도: 중간): 테이블 전체를 하나의 그룹으로 보는 것으로 알고 있다. 수업에는 없다.

---

## 주제 5: ORDER BY와 정렬 (수업 외 · 대화 중 보충)

> ➕ **더 알아두기 — 다중 정렬과 DESC**
>
> ```sql
> ORDER BY account, dollar DESC
> ```
>
> 1. `account`로 먼저 **오름차순** 정렬
> 2. **같은 `account` 안에서만** `dollar`를 **내림차순** 정렬
>
> - 방향을 안 쓰면 **기본은 오름차순(`ASC`)** 이고, **`DESC`는 바로 앞에 있는 그 기준 하나에만** 적용된다. `continent`에도 내림차순을 주려면 `ORDER BY continent DESC, ...`처럼 각각 쓴다.
> - 이 규칙은 `GROUP BY`가 있든 없든 같고, **정렬하는 줄이 달라질 뿐**이다. `GROUP BY`가 없으면 원본 행을 정렬하고, `GROUP BY`가 있으면 **그룹을 요약한 줄**을 정렬한다.
> - 정렬은 한 그룹(한 줄) 안에서가 아니라 **줄들 사이의 순서**를 정하는 것이다. 합계는 그룹마다 하나씩 나오지만 `GROUP BY continent, region`이면 한 대륙 안에 지역 그룹이 여럿이라 합계 값도 여럿이다. 그 값들을 비교해 줄 세우는 것이다.
> - `GROUP BY`에 쓴 컬럼(어떻게 묶을까)과 `ORDER BY`에 쓴 컬럼(묶인 결과를 어떻게 줄 세울까)은 서로 다른 목록이다.

> ♻️ 처음엔 **`ORDER BY continent, SUM(population) DESC`에서 `DESC`가 앞의 `continent`까지 적용되어 대륙이 내림차순**이라고 알았으나, **`DESC`는 바로 앞 기준 하나에만** 적용되고 `continent`는 기본인 **오름차순**이다.

> ➕ **더 알아두기 — ORDER BY와 SELECT에 없는 집계**
>
> `ORDER BY count(name) DESC`처럼 `SELECT`에 쓰지 않은 집계 함수로도 정렬이 된다. DBMS가 그룹마다 **정렬용 값(`COUNT(name)`)** 도 같이 계산해 두었다가 그 값으로 줄을 세우고, **출력은 `SELECT`에 쓴 열만** 한다. 정렬용 값은 결과에 보이지 않는다.
>
> - `ORDER BY`는 **`SELECT`에서 만든 별칭**도, **그룹에서 계산하는 식**(`SELECT`에 없어도 됨)도 **둘 다** 쓸 수 있다. `HAVING`이 `SELECT`에 없는 집계(`AVG(population)`)를 쓸 수 있는 것(수업)과 같은 원리이다.
> - 처리 순서를 "표가 차례로 바뀌는 과정"으로만 외우면 한계가 있다. `SELECT`가 결과 열을 정한 뒤에도 `HAVING`과 `ORDER BY`는 **그룹 정보**를 쓸 수 있다.
> - `GROUP BY` 없는 쿼리에서도 `SELECT name FROM city ORDER BY population DESC;`처럼 `SELECT`에 없는 컬럼으로 정렬한다. (`SELECT DISTINCT`를 쓸 때는 제한이 생기는 것으로 알고 있다.)

> ♻️ 처음엔 **`ORDER BY`를 "`SELECT` 결과 표만 정렬한다"** 로 이해했으나, **`SELECT`의 별칭과 `SELECT`에 없는 그룹 계산식을 둘 다** 정렬 기준으로 쓸 수 있다.

---

## 주제 6: 그룹 안의 행이나 그룹의 대표 행을 보고 싶을 때 (수업 외 · 대화 중 보충)

`GROUP BY`는 그룹당 한 줄로 **요약**하므로 요약하면 원래 행의 세부 정보는 결과에서 사라진다. 세부를 보려면 다른 방법을 쓴다.

> ➕ **더 알아두기 — 상황별 도구**
> (PostgreSQL 일반 지식이며, 서브쿼리·`DISTINCT ON`·윈도우 함수는 수업에 나오지 않았다.)
>
> | 하고 싶은 것 | 방법 |
> |---|---|
> | **특정 그룹 하나**의 행 (예: Asia의 나라 목록) | `WHERE continent = 'Asia'` (GROUP BY 불필요) |
> | **그룹 조건에 맞는 그룹들**의 행 | 서브쿼리 (`GROUP BY` + `HAVING`을 안쪽에) |
> | 그룹마다 **대표 행** | `DISTINCT ON`, 윈도우 함수 |
> | 그룹별 **요약 숫자** | `GROUP BY` + 집계 함수 |
>
> ```sql
> -- 나라가 10개 넘는 대륙에 속한 나라들 목록
> SELECT name, continent
> FROM country
> WHERE continent IN (
>     SELECT continent
>     FROM country
>     GROUP BY continent
>     HAVING COUNT(*) > 10
> );
> ```
>
> 안쪽 쿼리에서 그룹 조건에 맞는 대륙 이름들을 구하고, 바깥쪽 쿼리에서 그 대륙에 속한 나라 행을 뽑는다.

> ➕ **더 알아두기 — DISTINCT ON**
> (PostgreSQL 전용 문법이며 시험용이 아니라 호기심에서 나온 보충이다.)
>
> - **`DISTINCT ON (열)`은 `DISTINCT`와 `ON`이 하나로 묶인 세트 문법**이다. 그냥 `DISTINCT`는 고른 열 **전체**가 같은 줄의 중복을 없애고, `DISTINCT ON (열)`은 **지정한 열의 값이 같은 줄들 중 첫 줄만** 남기며 다른 열은 자유롭게 고를 수 있다. 여기서 `ON`은 조인의 `ON`과 다른 용도이다.
> - 어느 줄이 "첫 줄"인지는 `ORDER BY`가 정한다. `DISTINCT ON`에 쓴 열은 `ORDER BY`의 **맨 왼쪽**에 있어야 하는 것으로 알고 있다.
>
> ```sql
> SELECT DISTINCT ON (continent) continent, name, population
> FROM country
> ORDER BY continent, population DESC;
> ```
>
> 이 쿼리는 `GROUP BY`가 없어서 `ORDER BY`가 **나라 행**을 정렬하고, 대륙마다 인구가 가장 큰 행이 맨 위에 와서 남는다. 대륙별 인구 1위 나라(이름과 인구)가 나온다.
>
> - **`GROUP BY`와 `DISTINCT ON`의 차이**: 결과가 그룹마다 한 줄인 것은 같지만 그 한 줄이 다르다. `GROUP BY` + `MAX(population)`은 **그룹을 요약한 값**이라 125라는 숫자는 나와도 어느 나라인지 알 수 없다. `DISTINCT ON`은 그룹에서 고른 **실제 행**이라 `name` 등 세부 정보가 그대로 살아 있다.
> - **결합 예시**: `GROUP BY continent, region`으로 지역별 합계 줄을 만들고, `ORDER BY continent, SUM(population) DESC`로 줄 세운 뒤, `DISTINCT ON (continent)`로 대륙마다 합계가 가장 큰 지역 한 줄만 남긴다. 위에서부터 훑으며 같은 `continent`가 이미 나왔으면 버린다고 생각하면 쉽다. (이해용 그림이며 실제 처리와 같다는 뜻은 아니다.)
> - 예전 MySQL은 `SELECT continent, name, MAX(population) ... GROUP BY continent`를 허용하고 `name`에 그룹 안의 **아무 값이나** 넣었던 것으로 알고 있다. 최댓값과 짝이 맞는 이름이 나온다는 보장이 없어 요즘은 기본적으로 막는 쪽이다. PostgreSQL은 처음부터 오류를 낸다.

> ➕ **더 알아두기 — LIMIT으로는 안 되는 이유**
> `LIMIT n`은 **전체 결과에서 위 n줄**이다. "전체에서 1등"(`ORDER BY population DESC LIMIT 1`)은 되지만 **"대륙마다 1등"** 은 그룹별 조건이라 `LIMIT`으로는 못 푼다. 질문에 "~마다", "~별로"가 붙으면 그룹별로 생각한다.
>
> | 질문 | 도구 |
> |---|---|
> | 전체에서 1등 | `ORDER BY ... LIMIT 1` |
> | **그룹마다** 1등 | `DISTINCT ON` 또는 윈도우 함수 (`ROW_NUMBER() OVER (PARTITION BY ...)`) |
>
> 윈도우 함수는 표준 SQL에 가깝게 쓰이는 방법으로 알고 있으나, 수업에서 나오면 다시 정리한다.

---

## 정리

- **`GROUP BY`** 는 지정한 컬럼 기준으로 같은 값끼리 묶어 **그룹별로 요약**한다. 기준 컬럼은 1개 이상이며, 여러 개면 값의 **조합이 같은 행**끼리 한 그룹이 된다. 그룹당 결과 한 줄이다.
- **집계 함수**는 여러 행을 하나의 값으로 요약한다. `COUNT(*)` 전체 행, `COUNT(컬럼)` NULL 아닌 행, `SUM`·`AVG`·`MIN`·`MAX`는 NULL 제외. 괄호 안 컬럼은 "무엇을 셀지"이고 묶는 기준은 `GROUP BY`이다.
- **SELECT**에는 그룹 기준 컬럼과 집계 함수를 쓴다(이름표 + 값). `GROUP BY`에 없는 일반 컬럼은 못 쓴다. 계산한 값도 결과 열이 되며 이름은 `AS`로 정한다.
- **`HAVING`** 은 그룹을 만든 뒤 집계값에 조건을 건다. 조건에 안 맞는 **그룹을 통째로** 뺀다. `WHERE`는 그룹 만들기 전에 행을 거르고 집계 함수를 못 쓴다.
- **처리 순서**: `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT`. `SELECT`의 별칭은 `HAVING`(과 `WHERE`)에서 못 쓰고 `ORDER BY`에서는 쓴다. `HAVING`은 `SELECT`에 없는 집계도 쓸 수 있다.
- **ORDER BY**: 여러 기준이면 1차로 먼저, 같은 값 안에서 2차. `DESC`는 바로 앞 기준에만 적용되고 기본은 오름차순이다. `SELECT`의 별칭과 `SELECT`에 없는 그룹 계산식을 모두 쓸 수 있다.
- `GROUP BY` 결과는 **저장되지 않는 임시 결과 표**이다. 원본 테이블은 그대로이다.
- (수업 외) 그룹 안의 행을 보려면 `WHERE`, 서브쿼리를 쓰고, 그룹별 대표 행은 `DISTINCT ON`이나 윈도우 함수를 쓴다. `LIMIT`은 그룹별로는 못 쓴다.

---

## ✅ 확인 질문

1. `GROUP BY`는 무엇을 하는가? 대륙별 나라 수를 구하는 예로 쓰는 이유를 설명하라. (`WHERE`로 하나씩 하는 방법과 비교)
2. `GROUP BY`는 안의 값을 모를 때만 쓰는 것인가? 쓰는 진짜 이유는?
3. 집계 함수란 무엇이며 수업에 나온 6가지를 말하라. `NULL`은 각각 어떻게 다루는가?
4. `COUNT(*)`와 `COUNT(컬럼)`의 차이는? NULL이 섞인 그룹에서 각각 얼마가 나오는가?
5. `COUNT(*)`의 `*`는 무엇을 뜻하는가? 컬럼을 지정하지 않았는데 어떻게 그룹별로 세는가?
6. `COUNT(name)`의 괄호 안 컬럼과 `GROUP BY`의 기준 컬럼은 각각 어떤 역할인가?
7. `GROUP BY continent`를 실행하면 그룹별로 어떤 일이 단계별로 일어나는가? 결과 행 수는 무엇으로 정해지는가?
8. 그룹을 "그룹별 작은 표"로 상상하는 모델은 어떤 점에서 맞고, 실제 DBMS와는 어떻게 다른가? 그 결과는 저장되는가?
9. `GROUP BY`에 컬럼이 둘 이상이면 그룹은 어떻게 만들어지는가? 조합이 다른 행은 어떻게 되는가?
10. `GROUP BY continent, governmentform`에서 `governmentform`을 빼면 그룹은 어떻게 달라지는가?
11. `SELECT`에 컬럼과 집계 함수를 함께 쓰는 이유는? `GROUP BY`에 없는 일반 컬럼을 쓰면 왜 오류가 나는가?
12. `GROUP BY` 없이 `SELECT name, COUNT(*) FROM city;`가 오류인 이유는?
13. 집계 함수의 결과 열 이름은 어떻게 정해지는가? `AS`를 쓰는 이유는?
14. `HAVING`은 무엇을 걸러내는가? 그룹의 일부 행만 자르는가, 그룹 통째로 남기거나 빼는가?
15. `WHERE`와 `HAVING`의 차이를 거르는 대상, 시점, 집계 함수 사용 여부로 비교하라.
16. "평균 인구가 1000만이 넘는 지역"과 "인구가 1000만 이상인 나라"는 각각 `WHERE`와 `HAVING` 중 어디에 조건을 쓰는가?
17. SELECT문의 논리적 처리 순서를 쓰고, `WHERE`와 `HAVING`에서 `SELECT`의 별칭을 쓸 수 없는 이유를 설명하라.
18. `HAVING country_count > 20`이 오류인 이유는? 어떻게 고치는가?
19. `HAVING`이 `SELECT`에 없는 집계 함수를 쓸 수 있다는 것은 어떤 뜻인가? 같은 원리가 `ORDER BY`에서도 적용되는가?
20. `ORDER BY continent, SUM(population) DESC`에서 `continent`와 `SUM(population)`의 정렬 방향은? `DESC`는 어디까지 적용되는가?
21. 다중 정렬(`ORDER BY account, dollar DESC`)의 규칙은? `GROUP BY`가 있을 때 정렬되는 줄은 무엇인가?
22. 그룹당 합계가 한 개뿐인데도 `ORDER BY SUM(...) DESC`가 의미 있는 경우는 언제인가?
23. `ORDER BY`는 `SELECT`의 별칭만 쓸 수 있는가? `SELECT`에 없는 집계 함수로도 정렬되는 이유는?
24. "특정 그룹 하나의 행", "그룹 조건에 맞는 그룹들의 행", "그룹마다 대표 행"을 각각 보려면 어떤 방법을 쓰는가?
25. `DISTINCT`와 `DISTINCT ON (열)`의 차이는? `GROUP BY` + `MAX`와 `DISTINCT ON`은 결과 한 줄의 정체가 어떻게 다른가?
26. "전체에서 인구 1위"와 "대륙마다 인구 1위"는 각각 `LIMIT`으로 풀 수 있는가? 이유는?
27. (기사 대비) `GROUP BY`가 있을 때 `SELECT`에 쓸 수 있는 것과 집계 함수를 `WHERE`에 쓸 수 없는 이유를 설명하라.
