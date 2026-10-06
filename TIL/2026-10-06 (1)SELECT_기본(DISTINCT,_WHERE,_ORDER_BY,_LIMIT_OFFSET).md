---
tags: [sql, select, where, order-by, limit-offset]
til: v2 2026-10-06
---

# 2026-10-06_SELECT_기본(DISTINCT,_WHERE,_ORDER_BY,_LIMIT_OFFSET)
> 작성일: 2026-10-06

## 🔗 관련 글

- [DML(INSERT, SELECT, WHERE, UPDATE, DELETE)](2026-10-02%20(5)DML(INSERT,_SELECT,_WHERE,_UPDATE,_DELETE).md) — `SELECT`와 `WHERE`의 가장 기본형을 먼저 다뤘다. 여기서는 `SELECT` 전체를 체계적으로 배운다.
- [SELECT advanced(GROUP BY, 집계 함수, HAVING)](2026-10-02%20(6)SELECT_advanced(GROUP_BY,_집계함수,_HAVING).md) — 아래 `SELECT` 구문 틀에 나오는 `GROUP BY`, `HAVING`은 거기서 자세히 다뤘다.

## ❓ 아직 모르겠는 것

- 이 글의 실습 문제(1249, 1251 레슨)는 **다른 세션에서 풀었기 때문에** 풀이 쿼리와 결과가 이 글에 없다.

---

## 주제 1: SELECT 문의 전체 구문

`SELECT`문은 아래와 같은 구문들을 포함한다.

- `[ ]`는 **생략 가능한 부분**, `|`는 **선택**을 나타내며 실제 SQL에는 입력하지 않는다.
- `...`는 같은 형식의 항목을 **더 나열할 수 있음**을 나타낸다.

```sql
SELECT [DISTINCT] column1 [, column2 ...]
FROM table_name [AS alias]
[WHERE condition]
[GROUP BY column1]
[HAVING group_condition]
[ORDER BY column1 [ASC | DESC]]
[LIMIT N] [OFFSET M];
```

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
> `SELECT … FROM … WHERE … GROUP BY … HAVING … ORDER BY`의 **절 순서**는 시험에서 자주 다루는 SELECT 문의 기본 구조로 알고 있다. 순서를 틀리면 문법 오류이다. (`LIMIT`/`OFFSET`은 표준 SQL이 아니라 PostgreSQL·MySQL 등에서 쓰는 형태로 알고 있으며 시험 범위인지는 확인하지 못했다.)

---

## 주제 2: 기본 SELECT

### 모든 필드 조회

```sql
SELECT * FROM 테이블명;
```

country table에서 모든 필드 조회:

```sql
SELECT * FROM country;
```

### 특정 컬럼 조회

컬럼이 **띄어쓰기가 있는 경우**, `""`(큰따옴표)로 감싸서 하나의 컬럼명으로 인식하게 한다.

```sql
SELECT 테이블명.컬럼명1, 테이블명.컬럼명2 FROM 테이블명;
```

- 컬럼 앞에 **테이블명을 붙이면 컬럼의 소속을 명확히** 하고, 여러 테이블에 같은 이름의 컬럼이 있을 때 **모호함을 방지**할 수 있다.
- **단일 테이블 조회에서는 컬럼 앞의 테이블명을 생략**할 수 있다.

```sql
SELECT 컬럼명1, 컬럼명2 FROM 테이블명;
```

country table에서 name, continent 필드 조회:

```sql
SELECT name, continent FROM country;
```

> ➕ **더 알아두기 — 큰따옴표는 "이름"을 감싼다**
> 띄어쓰기가 있는 컬럼명을 큰따옴표로 감싸는 것은 앞에서 배운 **"값은 작은따옴표, 이름은 큰따옴표"** 규칙과 같다. 띄어쓰기가 있으면 SQL이 이름 하나로 인식하지 못하므로 큰따옴표로 하나의 이름임을 알려 준다.

---

## 주제 3: 별칭(alias)

### 테이블 별칭

테이블명을 **축약**하여 사용할 수 있다.

```sql
SELECT t.컬럼명1, t.컬럼명2 FROM 테이블명 AS t;
SELECT t.컬럼명1, t.컬럼명2 FROM 테이블명 t;
```

### 컬럼 별칭

컬럼에 별칭을 지정하면 **조회 결과에 표시되는 이름**을 바꿀 수 있다. **원본 테이블의 컬럼명은 변경되지 않는다.**

```sql
SELECT t.컬럼명1 AS first, t.컬럼명2 AS second FROM 테이블명 t;
SELECT t.컬럼명1 first, t.컬럼명2 second FROM 테이블명 t;
```

- **`AS`는 생략이 가능하다.**

```sql
SELECT c.name 국가, c.population 인구 FROM country c;
```

> ➕ **더 알아두기 — AS를 생략할 수 있어서 생기는 실수**
> `AS`를 생략할 수 있다는 것은 **컬럼 뒤에 쉼표 없이 단어가 오면 그 단어를 별칭으로 본다**는 뜻이다. 예를 들어 `SELECT name population FROM city;`처럼 쉼표를 빠뜨리면 오류가 아니라 **`name` 컬럼에 `population`이라는 별칭이 붙은 결과**가 나올 수 있다. 컬럼이 하나만 나오면 쉼표를 빠뜨렸는지 확인한다. (일반 지식이며 직접 실행해서 확인한다.)

---

## 주제 4: 중복 제거 — DISTINCT

```sql
SELECT DISTINCT 컬럼명 FROM 테이블명;
```

country 테이블에서 Continent의 종류 조회:

```sql
SELECT DISTINCT continent FROM country;
```

여러 컬럼을 지정하면 **컬럼 값의 조합이 같은 행의 중복을 제거**한다. 대륙 이름은 반복될 수 있지만, **같은 대륙과 지역의 조합은 한 번만** 나온다.

```sql
SELECT DISTINCT continent, region
FROM country;
```

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
> `DISTINCT`는 SELECT 결과의 **중복 행을 제거**하는 키워드로 시험에 나온다. 선택한 열 **전체의 조합**이 같은 행만 중복으로 본다.

---

## 주제 5: WHERE

- `WHERE` 절은 데이터를 **필터링**할 때 사용하는 조건절이다.
- **문자의 경우 `''`(홑따옴표)** 를 사용한다.

### 비교 연산자

```sql
WHERE population > 1000000
WHERE population >= 1000000
WHERE population < 1000000
WHERE population <= 1000000
WHERE population = 1000000
WHERE population != 1000000 -- 또는 <>
```

### 논리 연산자

```sql
WHERE population > 1000000 AND continent = 'Asia'
WHERE population > 1000000 OR continent = 'Asia'
WHERE NOT population < 1000000
```

- **`AND`는 `OR`보다 먼저 계산된다.** 조건을 묶어 먼저 계산하려면 **괄호**를 사용한다.

인구가 100만을 초과하면서 아시아 또는 유럽에 속한 국가 조회:

```sql
SELECT name, continent, population
FROM country
WHERE population > 1000000
AND (continent = 'Asia' OR continent = 'Europe');
```

### 범위

**`BETWEEN`은 시작값과 끝값을 모두 포함**한다.

```sql
WHERE population BETWEEN 1000000 AND 2000000
WHERE population NOT BETWEEN 1000000 AND 2000000
```

### 포함

```sql
WHERE code IN ('KOR', 'JPN', 'CHN')
WHERE code NOT IN ('KOR', 'JPN', 'CHN')
```

### NULL 여부

- **NULL은 값이 없거나 알려지지 않았음**을 나타낸다.
- `= NULL`, `!= NULL`로 비교하지 않고, **`IS NULL`, `IS NOT NULL`** 로 확인한다.

```sql
WHERE LifeExpectancy IS NULL
WHERE LifeExpectancy IS NOT NULL
```

### 패턴 매칭

- `%` : **0개 이상**의 문자
- `_` : **1개**의 문자
- **`LIKE` 대신 `ILIKE`를 활용하면 대소문자를 구분하지 않는다.**

```sql
WHERE Name LIKE 'S%' -- S로 시작
WHERE Name LIKE '%on' -- on으로 끝남
WHERE Name LIKE '%on%' -- on이 포함
WHERE Name NOT LIKE 'S%' -- S로 시작하지 않음
```

> ➕ **더 알아두기 — 이전에 정리한 내용과 연결**
>
> - `WHERE`의 문자열 값은 **대소문자를 구분**하므로 `LIKE 'san%'`로는 `San Francisco`가 나오지 않고, 구분하지 않으려면 `ILIKE`를 쓴다. [DML 글](2026-10-02%20(5)DML(INSERT,_SELECT,_WHERE,_UPDATE,_DELETE).md)에서 일반 지식으로 적어 둔 내용이 수업에서 확인되었다.
> - NULL은 "모른다"는 뜻이라 `= NULL` 비교가 참도 거짓도 아니어서, `WHERE`가 **참인 행만** 남기므로 아무 행도 안 나온다. 그래서 `IS NULL`을 쓴다.
> - `!=`와 `<>`는 둘 다 "같지 않다"이다. 이 중 `<>`가 표준 SQL이고 `!=`는 많은 DBMS가 지원하는 형태로 알고 있다.

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
>
> - **`AND`가 `OR`보다 먼저** 계산된다. 괄호를 쓰면 의도가 분명해진다.
> - **`BETWEEN`은 양 끝값을 포함**한다. (`>=` AND `<=`와 같다.)
> - **`IN`** 은 `OR` 여러 개를 짧게 쓴 것이다.
> - **NULL 비교는 `IS NULL` / `IS NOT NULL`** 을 쓴다. `= NULL`은 쓸 수 없다.
> - **`LIKE`의 와일드카드**: `%` = 길이와 상관없이 아무 문자열, `_` = 아무 한 글자. `ILIKE`는 PostgreSQL의 대소문자 무시 `LIKE`로 알고 있어 표준 SQL이 아니다. (확실도: ILIKE의 표준 여부는 중간)

### 실습 (수업 1249에서 제시된 문제)

1. 인구가 800만 이상인 도시의 name, population을 조회하시오.
2. 한국(KOR)에 있는 도시의 name, code를 조회하시오.
3. 유럽 대륙에 속한 나라들의 name과 region을 조회하시오.
4. 이름이 'San'으로 시작하는 도시의 name을 조회하시오.
5. 독립 연도(IndepYear)가 1901년 이상인 나라의 name, indepyear를 조회하시오.
6. 인구가 100만에서 200만 사이인 한국 도시의 name을 조회하시오.
7. 인구가 500만 이상인 한국, 일본, 중국의 도시의 name, code, population을 조회하시오.
8. 도시 이름이 'A'로 시작하고 'a'로 끝나는 도시의 name을 조회하시오.
9. 동남아시아(Southeast Asia) 지역(Region)에 속하지 않는 아시아(Asia) 대륙 나라들의 name, region을 조회하시오.
10. 오세아니아 대륙에서 기대수명의 데이터가 없는 나라의 name, lifeexpectancy, continent를 조회하시오.

---

## 주제 6: ORDER BY

`ORDER BY`는 결과를 **정렬**하는 구문이다.

```sql
ORDER BY 컬럼명 [ASC | DESC]
```

- `ASC`: **오름차순** (기본값, 생략 가능)
- `DESC`: **내림차순**

인구 많은 순으로 정렬:

```sql
SELECT name, population
FROM city
ORDER BY population DESC;
```

국가순으로 정렬 후, **같은 국가 내에서는 인구순** 정렬. 정렬을 **여러 컬럼에 순차적으로** 적용할 수 있다.

```sql
SELECT name, code, population
FROM city
ORDER BY code , population DESC;
```

### NULL의 정렬 위치

PostgreSQL에서는 기본적으로 **`ASC` 정렬 시 NULL이 마지막에, `DESC` 정렬 시 처음에** 나온다.

```sql
SELECT name, indepyear FROM country
ORDER BY indepyear DESC;
```

**`NULLS FIRST`, `NULLS LAST`** 로 위치를 지정할 수 있다.

```sql
SELECT name, indepyear FROM country
ORDER BY indepyear DESC NULLS LAST;
```

```sql
SELECT name, indepyear FROM country
ORDER BY indepyear ASC NULLS FIRST;
```

> ➕ **더 알아두기 — 다중 정렬 읽는 법**
> `ORDER BY code, population DESC`는 **1차 기준(`code`)으로 먼저 줄 세우고, 같은 `code` 안에서만 2차 기준(`population`)으로 다시 줄 세운다.** 방향을 안 쓴 `code`는 기본인 오름차순이고, **`DESC`는 바로 앞의 `population` 하나에만 적용**된다. `code`도 내림차순으로 하려면 `ORDER BY code DESC, population DESC`처럼 각각 쓴다. (GROUP BY 글의 ♻️와 같은 내용이다.)

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
> `ORDER BY`의 **기본 정렬은 오름차순(ASC)** 이고 내림차순은 `DESC`이다. 여러 열을 쓰면 앞의 열이 우선 기준이다.

### 실습 (수업 1249에서 제시된 문제)

1. country 테이블에서 대륙별로 정렬하고, 같은 대륙 내에서는 GNP가 높은 순으로 정렬하여 name, continent, GNP를 조회하시오.
2. country 테이블에서 기대수명이 높은 순으로 정렬하되, NULL값은 마지막에 나오도록 정렬하여 name, lifeexpectancy를 조회하시오.

---

## 주제 7: LIMIT, OFFSET

- **`LIMIT`** 은 반환할 **행 수를 제한**하고, **`OFFSET`** 은 앞에서 **건너뛸 행 수**를 지정한다.
- 원하는 순서로 정렬한 결과의 일부를 조회하려면 **`ORDER BY`와 함께** 사용한다.
- `LIMIT`: 개수 (한 페이지에 보여줄 개수)
- `OFFSET`: 앞에서 건너뛸 행 수 = **(페이지 번호 − 1) × 페이지당 개수**

인구수 상위 1위 ~ 5위 (1페이지):

```sql
SELECT name, population
FROM city
ORDER BY population DESC
LIMIT 5; -- OFFSET 0 생략
```

인구수 상위 11위 ~ 15위 조회 (3페이지):

```sql
SELECT name, population
FROM city
ORDER BY population DESC
LIMIT 5 OFFSET 10;
```

> ➕ **더 알아두기 — 페이지 계산 확인**
> 3페이지, 페이지당 5개이면 `OFFSET = (3 − 1) × 5 = 10`이다. 앞의 10개를 건너뛰고 5개를 가져오므로 11위부터 15위가 된다. `ORDER BY` 없이 `LIMIT`만 쓰면 어떤 행이 나올지 정해져 있지 않으므로, 일부를 뽑을 때는 항상 `ORDER BY`와 함께 쓴다. `LIMIT`은 **전체 결과에서 위 n줄**이라서 "그룹마다 1등"은 `LIMIT`으로 못 푼다. ([GROUP BY 글](2026-10-02%20(6)SELECT_advanced(GROUP_BY,_집계함수,_HAVING).md) 참고)

### 실습 (수업 1249에서 제시된 문제)

1. city 테이블에서 인구수가 가장 적은 도시 5개를 조회하시오.
2. country 테이블에서 면적(surfacearea)이 가장 넓은 순서대로 11위부터 20위까지의 국가를 조회하시오.
3. country 테이블에서 기대수명이 높은 순서대로 1위부터 5위까지의 국가를 조회하시오.

---

## 주제 8: SELECT practice (수업 1251)

### 실습 환경

- 모든 문제는 **dvdrental 데이터베이스의 `film` 테이블**을 사용한다.
- 상단의 연결 database가 `dvdrental`로 연결되어 있는지 확인한다.

| 컬럼 | 뜻 |
|---|---|
| `title` | 영화 제목 |
| `rental_rate` | 영화 대여료 (금액) |
| `rating` | 영화 관람 등급 (G, PG, R 등) |
| `length` | 영화 상영 시간 (분 단위) |
| `rental_duration` | 대여 가능 기간 (일 단위) |
| `replacement_cost` | 영화 분실 시 변상 비용 |

> ➕ **더 알아두기 — 연결 DB 확인**
> PostgreSQL은 한 연결이 한 DB에 붙으므로(`USE`가 없다) 문제의 DB가 `world`(`city`, `country`)에서 `dvdrental`(`film`)로 바뀌면 **연결 설정에서 Database를 바꿔야** 한다. (자세한 내용은 [DML 글](2026-10-02%20(5)DML(INSERT,_SELECT,_WHERE,_UPDATE,_DELETE).md)의 주제 7)

### 실습 문제

1. 모든 영화의 제목과 대여료를 조회하시오.
2. 대여료가 4달러 이상인 영화의 제목과 대여료를 조회하시오.
3. 제목에 대소문자를 구분하지 않고 'angel'이 포함된 영화의 제목을 조회하시오.
4. 대여 기간이 5일 이상이고 대여료가 4달러 이상인 영화의 제목, 대여 기간, 대여료를 조회하시오.
5. 등급이 'PG' 또는 'G'이면서 대여 기간이 4일 이하인 영화의 제목, 등급, 대여 기간을 조회하시오.
6. 상영 시간이 60분 이상 120분 이하인 영화의 제목과 상영 시간을 상영 시간 오름차순으로 조회하시오.
7. 등급이 'R'인 영화를 제목 오름차순으로 정렬한 뒤, 처음 10개의 제목과 등급을 조회하시오.
8. 영화의 상영 시간을 중복 없이 긴 순서로 10개 조회하시오.
9. 상영 시간이 긴 순서로 10번째부터 15번째까지 영화의 제목과 상영 시간을 조회하시오. (상영 시간이 같으면 제목 오름차순)
10. 등급별 영화 수를 조회하시오.
11. 대여 기간별 영화 수를 대여 기간 내림차순으로 정렬하여 조회하시오.
12. 등급별 영화 수와 평균 상영 시간을 조회하시오.
13. 변상 비용별 영화 수와 평균 대여료를 변상 비용 오름차순으로 정렬하여 조회하시오.
14. 대여료별 영화 수를 영화 수 내림차순으로 정렬하여 조회하시오.
15. 등급별 평균 대여료가 3달러 미만인 등급과 평균 대여료를 조회하시오.

> ➕ **더 알아두기 — 문제와 사용 문법의 대응**
> (풀이 힌트이며 정답이 아니다.)
>
> | 문제 | 쓰는 문법 |
> |---|---|
> | 1~2 | `SELECT`, `WHERE` 비교 |
> | 3 | `ILIKE '%angel%'` (대소문자 무시 포함 검색) |
> | 4~5 | `AND`, `OR`와 괄호, `IN` |
> | 6 | `BETWEEN` + `ORDER BY` |
> | 7 | `ORDER BY` + `LIMIT 10` |
> | 8 | `DISTINCT` + `ORDER BY ... DESC` + `LIMIT 10` |
> | 9 | `ORDER BY 상영시간 DESC, 제목` + `LIMIT 6 OFFSET 9` (10번째부터 15번째는 6개이며 앞 9개를 건너뜀) |
> | 10~14 | `GROUP BY` + 집계 함수 (+ `ORDER BY`) |
> | 15 | `GROUP BY` + `HAVING` |
>
> 9번의 `OFFSET`은 "건너뛸 행 수"이므로 10번째부터이면 **9개**를 건너뛴다. 10~15번째는 `15 − 10 + 1 = 6`개이다.

---

## 정리

- **SELECT의 전체 구문**: `SELECT [DISTINCT] … FROM … [WHERE] [GROUP BY] [HAVING] [ORDER BY] [LIMIT] [OFFSET]`. `[ ]`는 생략 가능하다.
- **기본**: `SELECT *`는 모든 컬럼, 컬럼을 나열하면 특정 컬럼. 컬럼 앞에 `테이블명.`을 붙이면 소속이 명확하고 단일 테이블에서는 생략할 수 있다. 띄어쓰기가 있는 컬럼명은 큰따옴표로 감싼다.
- **별칭**: 테이블 별칭(`FROM country c`), 컬럼 별칭(`AS`, 생략 가능). 결과에 표시되는 이름만 바뀌고 원본 컬럼명은 안 바뀐다.
- **DISTINCT**: 중복 제거. 여러 컬럼이면 조합이 같은 행의 중복을 제거한다.
- **WHERE**: 비교(`>`, `>=`, `<`, `<=`, `=`, `!=`/`<>`), 논리(`AND`, `OR`, `NOT`, **`AND`가 먼저**), `BETWEEN`(양 끝 포함), `IN`, `IS NULL`/`IS NOT NULL`, 패턴(`%` 0개 이상, `_` 1개, `LIKE`, `ILIKE`). 문자는 홑따옴표.
- **ORDER BY**: 기본 `ASC`, 내림차순 `DESC`, 여러 컬럼은 순차 적용. PostgreSQL은 `ASC`일 때 NULL이 마지막, `DESC`일 때 처음이며 `NULLS FIRST/LAST`로 지정한다.
- **LIMIT/OFFSET**: `LIMIT` 개수, `OFFSET` 건너뛸 행 수 = (페이지 번호 − 1) × 페이지당 개수. `ORDER BY`와 함께 쓴다.
- **SELECT practice**: dvdrental의 `film` 테이블로 위 문법과 `GROUP BY`/`HAVING`을 섞어 연습한다.

---

## ✅ 확인 질문

1. `SELECT`문의 전체 구문을 절 순서대로 쓰고, `[ ]`, `|`, `...`의 뜻을 설명하라.
2. 컬럼 앞에 `테이블명.`을 붙이는 이유는? 단일 테이블 조회에서는 어떻게 되는가?
3. 컬럼 이름에 띄어쓰기가 있으면 어떻게 써야 하는가? 값을 감싸는 따옴표와 어떻게 다른가?
4. 테이블 별칭과 컬럼 별칭은 각각 무엇을 바꾸는가? 원본 컬럼명은 바뀌는가? `AS`를 생략하면 어떻게 쓰는가?
5. `SELECT name population FROM city;`처럼 쉼표를 빠뜨리면 어떤 일이 생길 수 있는가?
6. `SELECT DISTINCT continent, region`은 무엇의 중복을 제거하는가?
7. `AND`와 `OR`가 섞인 조건에서 계산 순서는? 먼저 계산하려면 어떻게 하는가?
8. `BETWEEN 1000000 AND 2000000`은 양 끝값을 포함하는가? `>`/`<`로 쓴 범위와 어떻게 다른가?
9. `IN ('KOR', 'JPN', 'CHN')`은 `OR`로 어떻게 쓰는 것과 같은가? `NOT IN`은?
10. NULL을 `= NULL`로 비교하면 안 되는 이유는? 올바른 조건 두 가지를 쓰라.
11. `%`와 `_`는 각각 무엇을 뜻하는가? `'S%'`, `'%on'`, `'%on%'`는 각각 어떤 값에 일치하는가?
12. `LIKE`와 `ILIKE`의 차이는? `'angel'`이 들어간 제목을 대소문자 구분 없이 찾는 조건을 쓰라.
13. `ORDER BY`의 기본 정렬 방향은? `ORDER BY code, population DESC`에서 `DESC`는 어디까지 적용되는가?
14. PostgreSQL에서 `ASC`와 `DESC` 정렬 시 NULL은 각각 어디에 오는가? 위치를 바꾸려면?
15. `LIMIT 5 OFFSET 10`은 몇 위부터 몇 위까지인가? 페이지 번호와 페이지당 개수로 `OFFSET`을 구하는 공식은?
16. `ORDER BY` 없이 `LIMIT`만 쓰면 안 되는 이유는?
17. "상영 시간이 긴 순서로 10번째부터 15번째"는 `LIMIT`과 `OFFSET`을 각각 얼마로 써야 하는가?
18. 등급별 영화 수를 구하는 쿼리와, 등급별 평균 대여료가 3달러 미만인 등급을 구하는 쿼리의 차이(`WHERE`/`HAVING`)는?
19. (기사 대비) `SELECT` 문에서 `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY`의 순서를 쓰고, `DISTINCT`는 어떤 역할인가?
20. (기사 대비) `AND`/`OR` 우선순위, `BETWEEN`, `IN`, `IS NULL`을 각각 한 문장으로 설명하라.
