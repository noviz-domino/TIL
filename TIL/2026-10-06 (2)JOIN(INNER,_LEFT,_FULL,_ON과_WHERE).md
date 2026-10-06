---
tags: [sql, join, inner-join, left-join, on-where]
til: v2 2026-10-06
---

# 2026-10-06_JOIN(INNER,_LEFT,_FULL,_ON과_WHERE)
> 작성일: 2026-10-06

## 🔗 관련 글

- [데이터베이스 기초와 관계(RDB, PK/FK, 1:N, M:N)](2026-10-02%20(1)데이터베이스_기초와_관계(RDB,_PK_FK,_1대N,_M대N).md) — JOIN은 거기서 배운 **FK와 PK의 연결**을 SQL로 쓰는 것이다. 1:N, M:N(연관 테이블), 1:1 관계가 모두 예제로 나온다.
- [SELECT 기본](2026-10-06%20(1)SELECT_기본(DISTINCT,_WHERE,_ORDER_BY,_LIMIT_OFFSET).md) — `SELECT`의 구문 틀과 별칭, `WHERE`, `ORDER BY`. JOIN은 그 틀의 `FROM` 부분에 들어간다.
- [SELECT advanced(GROUP BY, 집계 함수, HAVING)](2026-10-02%20(6)SELECT_advanced(GROUP_BY,_집계함수,_HAVING).md) — 이 글의 "도시별 고객 수", "출연 영화 30편 이상 배우" 예제가 JOIN과 `GROUP BY`·`HAVING`을 함께 쓴다.

## ❓ 아직 모르겠는 것

- 수제비(정보처리기사 교재)에 나오는 **SELECT 절 순서 두음 암기법의 정확한 문구**를 확인하지 못했다. 교재 표현을 우선한다.
- 연습 문제를 풀지 않았다.
  - 고객·주문 예시의 INNER JOIN 결과에서 **박지호가 안 나오는 이유**, 김민준이 주문을 5건 했다면 결과는 몇 줄인가?
  - LEFT JOIN 결과에서 **주문을 한 번도 안 한 고객**만 찾으려면 `WHERE`에 어떤 조건을 쓰는가?
  - LEFT JOIN에서 오른쪽 테이블 조건을 `ON`에 둘 때와 `WHERE`에 둘 때, 왼쪽 행이 남는 쪽은 어느 쪽이며 처리 순서로 어떻게 설명하는가?
  - 조건을 `WHERE`에 두었을 때 **이서연이 결과에서 사라지는 이유**는?
  - 배우 예제에서 두 번째 `ON`을 `ON a.actor_id = f.film_id`로 잘못 쓰면 어떻게 되는가?
  - 부서·직원 FULL JOIN(4줄)과 같은 데이터에서 INNER JOIN, LEFT JOIN은 각각 몇 줄인가?
  - 쉼표 조인에서 `WHERE`를 빼면 결과는 몇 줄이며 오류가 나는가?
  - `WHERE cname = '김민준'`처럼 컬럼 별칭을 `WHERE`에서 쓰면 왜 오류인가?
  - `JOIN city ci ON a.city_id = ci.city_id`에서 `a.city_id`와 `ci.city_id` 중 FK와 PK는?
  - `customer`→`address`→`city` 쿼리 끝에 `WHERE ci.city = 'London'`을 붙이면 `ci`를 `WHERE`에서 쓸 수 있는 이유는?
- 실습은 **다른 세션에서** 하므로 이 글에는 실행 결과가 없다. 예제에 쓰인 DB는 `world`(`city`, `country`, `countrylanguage`)와 `dvdrental`이다.

---

## 주제 1: JOIN이란

**JOIN은 두 개 이상의 테이블을 연결하여 데이터를 조회하는 SQL 구문**이다.

```sql
SELECT 컬럼명
FROM 테이블1
JOIN유형 테이블2 ON 조인조건;
```

> ➕ **더 알아두기 — JOIN은 왜 필요한가**
> (수업에서 나온 설명이 아니라 대화 중 정리한 내용이다. 표는 **설명용 가짜 데이터**이다.)
>
> 테이블을 나눠서 저장하면(중복을 줄이려고) 조회할 때는 다시 합쳐서 봐야 한다. 예를 들어 주문 테이블에는 `C001` 같은 고객 ID만 있고 이름은 고객 테이블에 있다.
>
> | 고객ID | 이름 | | 주문ID | 고객ID | 상품명 |
> |---|---|---|---|---|---|
> | C001 | 김민준 | | O001 | C001 | 노트북 |
> | C002 | 이서연 | | O002 | C002 | 핸드폰 |
> | C003 | 박지호 | | O003 | C001 | 태블릿 |
>
> "누가 무엇을 주문했나?"를 알려면 두 표를 이어 붙여야 하고, 그 일을 JOIN이 한다.
>
> **JOIN 문장의 구조**
>
> ```sql
> FROM customer c                    -- ① 첫 번째 테이블 (기준)
> JOIN orders o                      -- ② 이어 붙일 두 번째 테이블 + 별칭
> ON c.고객ID = o.고객ID             -- ③ 어떤 행끼리 이을지 (연결 조건)
> ```
>
> | 자리 | 내용 |
> |---|---|
> | `FROM` 뒤 | 첫 번째 테이블 |
> | **`JOIN` 뒤(ON 앞)** | **이어 붙일 테이블 이름**(+ 별칭) |
> | `ON` 뒤 | **연결 조건** |
>
> "customer 테이블에, orders 테이블을 붙이는데, 고객ID가 같은 행끼리 붙여라"로 읽는다. 앞에 오는 `INNER`, `LEFT`는 JOIN의 **유형**이라서 `JOIN` 앞에 온다. (`유형 JOIN 테이블 ON 조건`)
>
> **`ON`은 "어떤 행끼리 이을지" 정하는 조건**이다. 수업의 설명으로는 `ON`은 두 테이블의 행을 연결하는 조건이다. `ON`이 없으면 모든 행이 모든 행과 이어져서 고객 3명 × 주문 3건 = **9줄**이 나오고 "이서연이 노트북을 주문했다" 같은 틀린 조합도 섞인다. `ON c.고객ID = o.고객ID`로 같은 고객ID끼리만 이으면 맞는 조합 3줄만 남는다. 이때 보통 **FK = PK**를 잇는다. (`orders.고객ID`는 FK, `customer.고객ID`는 PK)
>
> - `DISTINCT ON`의 `ON`과 JOIN의 `ON`은 글자만 같고 **다른 문법**이다.
> - 조인 조건에 **꼭 `=`만 쓰는 것은 아니다.** `=`가 가장 흔하고(**동등 조인, equi join**), 수업 예제에는 `ON` 안에 `AND`로 조건을 더 붙인 것도 있다. `>`, `<`를 쓰는 조인도 문법상 가능하지만 실무에서는 `=`가 대부분이다.
> - JOIN의 결과는 저장되는 새 테이블이 아니라 **조회할 때 만든 임시 결과**이다. (`GROUP BY` 결과와 같은 성격)

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 중간)
> `=`로 연결하는 조인을 **동등 조인(EQUI JOIN)** 이라고 하는 것으로 알고 있다. 조인 조건 없이 모든 조합을 만드는 **CROSS JOIN**(교차 조인)과 **자연 조인(NATURAL JOIN)** 도 시험에 나오는 것으로 알고 있지만 수업에는 아직 나오지 않았다. 시험 범위의 정확한 표현은 확인하지 못했다.

---

## 주제 2: INNER JOIN

- **두 테이블에서 조인 조건이 일치하는 레코드만 반환**한다.
- `INNER` 키워드를 제외하여 **`JOIN`만 명시하여도 INNER JOIN으로 동작**한다.

### 도시와 국가 정보 연결 (world, 1:N 관계)

```sql
SELECT city.name as cityname,
       country.name as countryname,
       country.continent
FROM city
INNER JOIN country ON city.countrycode = country.code;
```

INNER 생략 가능:

```sql
SELECT city.name as cityname,
       country.name as countryname,
       country.continent
FROM city
JOIN country ON city.countrycode = country.code;
```

### 배우와 출연 영화 목록 (dvdrental, M:N 관계)

```sql
SELECT actor.first_name,
       actor.last_name,
       film.title
FROM actor
INNER JOIN film_actor ON actor.actor_id = film_actor.actor_id
INNER JOIN film ON film_actor.film_id = film.film_id;
```

> ➕ **더 알아두기 — INNER JOIN 결과 읽기**
> (대화 중 정리한 설명이며 표는 설명용 가짜 데이터이다.)
>
> 위 고객·주문 표에 `JOIN orders o ON c.고객ID = o.고객ID`를 하면 다음처럼 **짝이 맞는 행만** 나온다.
>
> | 이름 | 상품명 |
> |---|---|
> | 김민준 | 노트북 |
> | 이서연 | 핸드폰 |
> | 김민준 | 태블릿 |
>
> - **박지호는 안 나온다.** 주문이 없어서 `ON` 조건에 맞는 행이 없기 때문이다. INNER JOIN은 양쪽에 짝이 있는 행만 남긴다.
> - **김민준이 두 번 나온다.** 주문이 2건이라 주문 한 건마다 한 줄이 만들어진다. (1:N)
> - `c`, `o`는 **테이블 별칭**이다.
> - **JOIN만 쓰면 INNER JOIN**이다. 이 "생략하면 기본값"은 `ORDER BY`의 기본 `ASC`, `AS` 생략과 같은 패턴이다. `JOIN`만 보이면 INNER로 읽고, `LEFT`/`RIGHT`/`FULL`이 붙어 있으면 "짝이 없는 행이 일부러 남아 있다"는 신호로 읽는다.
> - `LEFT JOIN`의 정식 이름은 `LEFT OUTER JOIN`이고 `OUTER`도 생략할 수 있는 것으로 알고 있다.

> ➕ **더 알아두기 — 여러 테이블 JOIN은 "덩어리를 이어 붙이기"**
> (대화 중 정리한 설명이다.)
>
> 배우–영화 예제에는 `JOIN ... ON ...`이 두 번 나온다. **한 덩어리 = `JOIN 테이블 ON 조건`** 이고, 테이블이 많아도 이 덩어리를 이어 쓰면 된다. 설명용 데이터(`actor`: 송강호 A001, 이정재 A002 / `film`: M001 관상, M002 기생충, M003 신과함께 / `film_actor`: (A001,M001), (A002,M001), (A001,M002), (A002,M003))로 보면 이렇다.
>
> 1. `actor a JOIN film_actor fa ON a.actor_id = fa.actor_id`: 배우에 출연 기록을 붙인다. 결과에는 영화 제목이 아직 없고 `film_id` 번호만 있다.
> 2. `JOIN film f ON fa.film_id = f.film_id`: **1단계 결과**의 `film_id`와 `film`의 `film_id`가 같은 행을 이어 영화 제목을 붙인다.
>
> 결과는 (송강호-관상, 송강호-기생충, 이정재-관상, 이정재-신과함께) 4줄이다. 배우는 2명인데 결과가 4줄인 것은 배우 한 명이 영화 여러 편에 나오기 때문이다.
>
> - **각 `ON`은 "새로 붙이는 테이블"과 "이미 붙어 있는 테이블 중 하나"를 연결**한다. 두 번째 `ON`의 왼쪽이 `actor`가 아니라 `film_actor`인 것을 보라. **연관 테이블이 다리 역할**을 하는 구조이며, `actor`(PK) ← `film_actor`(FK 두 개) → `film`(PK)의 FK→PK 체인이다.
> - 수업의 고객 정보 예제(주제 6)는 `customer → address → city → country`로 **ID를 따라 한 칸씩** 이동한다. 고객에는 `address_id`만, 주소에는 `city_id`만, 도시에는 `country_id`만 있어서 국가 이름까지 가려면 `JOIN`을 연달아 써야 한다. 이는 테이블을 나눠 저장한 결과이다.
> - 각 `ON`은 FK와 PK의 짝이다. (`c.address_id`(FK) = `a.address_id`(PK), `a.city_id`(FK) = `ci.city_id`(PK))
> - `JOIN address a`는 **`address` 테이블에 `a`라는 별칭**을 붙인 것이다. 이때 `a.address`는 "`a` 테이블의 `address` 컬럼"으로, 테이블 이름과 컬럼 이름이 우연히 같은 경우이다. 별칭이 있으면 헷갈리지 않는다.

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 중간)
> 시험의 SQL 문제에서 **3개 이상의 테이블을 JOIN하는 쿼리를 읽거나 결과를 예측**하는 형태가 나올 수 있는 것으로 알고 있다. 한 덩어리씩 붙이며 중간 결과를 그려 보는 방식이 정확하다. (출제 여부와 형태는 확인하지 못했다.) 또 **내부 조인(INNER)은 일치하는 행만, 외부 조인(OUTER)은 일치하지 않는 행도 포함**한다는 구분이 핵심이다.

---

## 주제 3: LEFT JOIN

- **왼쪽 테이블의 모든 레코드**와 오른쪽 테이블의 일치하는 레코드를 반환한다.

### 모든 국가와 수도 정보 (world, 1:1 관계)

```sql
SELECT country.name as country_name,
       city.name as capital_city
FROM country
LEFT JOIN city ON country.capital = city.id;
```

join한 테이블에 데이터가 없는 경우, **NULL이 포함될 수 있다.**

```sql
SELECT country.name as country_name,
       city.name as capital_city
FROM country
LEFT JOIN city ON country.capital = city.id
WHERE city.id IS NULL;
```

**INNER JOIN을 한 경우, 수도에 해당하는 도시 데이터가 없는 국가는 조회되지 않는다.**

```sql
SELECT country.name AS country_name,
       city.name AS capital_city
FROM country
INNER JOIN city ON country.capital = city.id
WHERE country.name = 'Antarctica';
```

### 모든 영화의 재고 조회 (dvdrental, 1:N 관계)

영화 하나에 재고가 여러 개 있는 경우, **재고마다 영화 정보가 반복해서 조회**된다.

```sql
SELECT f.film_id, f.title, i.inventory_id
FROM film f
LEFT JOIN inventory i ON f.film_id = i.film_id;
```

join한 테이블에 데이터가 없는 경우, NULL이 포함될 수 있다.

```sql
SELECT f.film_id, f.title, i.inventory_id
FROM film f
LEFT JOIN inventory i ON f.film_id = i.film_id
WHERE i.inventory_id IS NULL;
```

### 모든 영화와 대여 고객 조회 (dvdrental, M:N 관계)

```sql
SELECT
    f.title,
    c.customer_id,
    c.first_name,
    c.last_name,
    r.rental_date
FROM film f
LEFT JOIN inventory i ON f.film_id = i.film_id
LEFT JOIN rental r ON i.inventory_id = r.inventory_id
LEFT JOIN customer c ON r.customer_id = c.customer_id;
```

- 같은 고객이 같은 영화를 여러 번 대여한 경우, **대여 기록마다 조회**된다.
- 재고나 해당 재고의 대여 기록이 없는 경우, **고객 정보와 대여일은 NULL**로 표시된다.

> ♻️ 처음엔 **LEFT JOIN이 "NULL을 만드는 것"** 으로 알았으나, **LEFT JOIN의 목적은 왼쪽 테이블의 행을 하나도 빠뜨리지 않고 남기는 것**이다. 오른쪽에 짝이 없으면 채울 값이 없어서 그 칸이 **NULL**이 되는 것이며, NULL은 목적이 아니라 **"짝이 없다"는 표시**이다.

> ➕ **더 알아두기 — LEFT JOIN 목적, 위치의 뜻**
> (대화 중 정리한 설명이다.)
>
> 고객·주문 표(박지호는 주문 없음)에서 **INNER JOIN은 박지호가 사라지지만 LEFT JOIN은 박지호가 남고 상품명이 NULL**이 된다.
>
> | 이름 | 상품명 |
> |---|---|
> | 김민준 | 노트북 |
> | 이서연 | 핸드폰 |
> | 김민준 | 태블릿 |
> | 박지호 | NULL |
>
> 출석부(왼쪽)를 기준으로 제출한 과제(오른쪽)를 붙이는 것에 비유할 수 있다. 과제를 안 낸 학생도 출석부에는 남고 과제 칸만 비는 것이 NULL이다. 수업의 "수도 정보가 없는 국가 찾기" 예제처럼 **LEFT JOIN으로 모두 남기고 `IS NULL`로 짝 없는 행만 골라내는 활용**도 있다. LEFT JOIN의 NULL은 "짝이 없다"는 신호이다.
>
> **"왼쪽"의 뜻**: LEFT JOIN의 왼쪽은 **SQL 문장에서 JOIN 키워드 앞에 쓴 테이블, 즉 `FROM`에 쓴 테이블**이다. `JOIN` 뒤에 지정한 테이블은 **오른쪽**이다. 결과 표에서 열이 어느 쪽에 보이는지와는 상관없다.
>
> ```sql
> FROM country LEFT JOIN city ON country.capital = city.id
> --   (왼쪽: 전부 남김)       (오른쪽: 짝이 있는 것만)
> ```
>
> "country(왼쪽)의 행은 전부 남기고, city(오른쪽)는 짝이 있는 것만 붙여라"로 읽는다. 그래서 수업의 "연결되는 데이터가 없어도 모든 행을 유지할 테이블을 FROM에 둔다"가 중요하다.

> ♻️ 처음엔 **`JOIN` 뒤에 지정한 테이블(`city`)이 왼쪽**이라고 이해했으나, **왼쪽은 `FROM`에 먼저 쓴 테이블(`country`)** 이고 `JOIN` 뒤에 지정한 테이블이 오른쪽이다.

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
> **외부 조인(OUTER JOIN)** 은 조인 조건에 맞지 않는 행도 결과에 포함하는 조인이고, 짝이 없는 쪽의 열은 NULL로 채워진다. LEFT/RIGHT/FULL이 모두 외부 조인에 속한다.

---

## 주제 4: FROM 테이블 선택 기준, ON과 WHERE의 차이

### FROM에 어떤 테이블을 두는가

- **INNER JOIN**은 조회하려는 **대상의 기준이 되는 테이블**을 `FROM`에 두면 읽기 쉽다.
- **LEFT JOIN**은 연결되는 데이터가 없어도 **모든 행을 유지할 테이블**을 `FROM`에 둔다.

### ON과 WHERE의 차이

- **`ON`**: 두 테이블의 행을 **연결하는 조건**
- **`WHERE`**: 조인 결과에서 **남길 행을 선택하는 조건**
- LEFT JOIN에서는 **오른쪽 테이블에 대한 조건을 어디에 작성하느냐에 따라 결과가 달라진다.**

**모든 영화를 유지하고, 1번 매장의 재고만 연결** (dvdrental)

- `ON`에 매장 조건을 추가하면 **1번 매장에 재고가 없는 영화도 조회**된다.
- 1번 매장에 재고가 없는 영화의 재고 정보는 NULL로 표시된다.

```sql
SELECT f.film_id, f.title, i.inventory_id, i.store_id
FROM film f
LEFT JOIN inventory i ON f.film_id = i.film_id
AND i.store_id = 1;
```

**1번 매장에 재고가 있는 영화만 조회** (dvdrental)

- `WHERE`에 매장 조건을 추가하면 **조인 결과에서 1번 매장의 재고만 남는다.**
- 재고 정보가 NULL인 행도 조건을 만족하지 않아 **제외**된다.

```sql
SELECT f.film_id, f.title, i.inventory_id, i.store_id
FROM film f
LEFT JOIN inventory i ON f.film_id = i.film_id
WHERE i.store_id = 1;
```

> ➕ **더 알아두기 — `AND i.store_id = 1`은 무엇인가, `ON`과 `WHERE`를 작은 예로**
> (대화 중 정리한 설명이며 표는 설명용 가짜 데이터이다.)
>
> `AND i.store_id = 1`은 **`ON` 안에 붙인 추가 조건**으로, 연결 조건(`f.film_id = i.film_id`)에 더해 "재고는 1번 매장 것만 연결 대상으로 삼는다"는 뜻이다. 고객·주문 표(O001 C001 노트북, O002 C002 핸드폰, O003 C001 태블릿, 박지호는 주문 없음)로 보면 다음과 같다.
>
> **① 조건을 `ON`에 둠**
>
> ```sql
> SELECT c.이름, o.상품명
> FROM customer c
> LEFT JOIN orders o ON c.고객ID = o.고객ID AND o.상품명 = '노트북';
> ```
>
> | 이름 | 상품명 |
> |---|---|
> | 김민준 | 노트북 |
> | 이서연 | NULL |
> | 박지호 | NULL |
>
> "노트북인 주문만 이어라"는 **연결 대상을 정하는 조건**이라 이서연, 박지호도 LEFT JOIN이므로 줄이 남고 상품명만 NULL이다. (김민준의 태블릿 주문은 연결 대상이 아니라 빠진다.)
>
> **② 같은 조건을 `WHERE`에 둠**
>
> ```sql
> SELECT c.이름, o.상품명
> FROM customer c
> LEFT JOIN orders o ON c.고객ID = o.고객ID
> WHERE o.상품명 = '노트북';
> ```
>
> | 이름 | 상품명 |
> |---|---|
> | 김민준 | 노트북 |
>
> 먼저 이어 붙인 결과(김민준-노트북, 김민준-태블릿, 이서연-핸드폰, 박지호-NULL)를 만들고 그 **결과에서 줄을 거르므로**, NULL 행(박지호)은 `NULL = '노트북'`이 참이 아니라서 사라지고 이서연도 핸드폰이라서 빠진다. **`WHERE`에 두면 INNER JOIN처럼 동작한다.**
>
> | 조건 위치 | 결과 |
> |---|---|
> | `ON` | 왼쪽 행(이서연·박지호)이 **남음** (NULL) |
> | `WHERE` | 조건에 맞는 줄만 (왼쪽 행도 사라짐) |
>
> 한 줄로 외우면 **`ON`은 "이을 때 걸러내는" 조건, `WHERE`는 "이은 뒤에 걸러내는" 조건**이다. LEFT JOIN에서 왼쪽 행을 모두 남기려면 오른쪽 테이블에 대한 조건은 `ON`에 둔다. INNER JOIN에서는 어느 쪽에 써도 결과가 같은 경우가 많아서 차이가 LEFT JOIN에서 중요해진다.

> ➕ **더 알아두기 — JOIN의 처리 순서: FROM 단계 안에서 왼쪽부터**
> (대화 중 정리한 설명이다.)
>
> 수업의 처리 순서(`FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT`)에는 JOIN이 따로 없다. **JOIN은 `FROM` 단계의 일부**이며, `FROM` 안에서 첫 테이블을 가져온 뒤 각 `JOIN`이 **왼쪽부터 차례로** 붙고 `ON`으로 연결된다. 그 이어 붙인 결과가 `WHERE`의 입력이 된다.
>
> - 그래서 **`ON`은 JOIN할 때(`FROM` 단계 안), `WHERE`는 JOIN이 끝난 뒤** 적용된다. LEFT JOIN에서 `ON`에 조건을 두면 왼쪽 행이 남고 `WHERE`에 두면 NULL 행도 걸러져 사라질 수 있는 이유이다.
> - "동시"가 아니라 FROM 단계 안에서 순서대로이다. 실제 DBMS는 더 빠르게 하려고 내부에서 순서를 바꿔 실행할 수 있다. 위는 결과를 이해하기 위한 논리적 순서이다.
> - 이 순서 때문에 **별칭**은 **정의된 위치 이후**에서만 쓸 수 있다.
>
> | 별칭 | 만들어지는 곳 | 쓸 수 있는 곳 |
> |---|---|---|
> | 테이블 별칭 (`c`, `o`) | `FROM`, `JOIN` 뒤 | 정의된 이후 **어디서든** (`ON`, `WHERE`, `SELECT` 등) |
> | 컬럼 별칭 (`country_count`) | `SELECT` | `ORDER BY`만 (`WHERE`, `HAVING`은 `SELECT`보다 앞이라 못 씀) |
>
> - `FROM`에서 정의한 별칭을 `JOIN`의 `ON`에서 쓸 수 있다. 단 **`ON`에서는 자기보다 앞에 나온 테이블과 지금 붙이는 테이블의 별칭만** 쓸 수 있다. 첫 `ON`에서 아직 나오지 않은 뒤의 `JOIN` 별칭을 쓰면 오류가 난다. (PostgreSQL 동작으로 알고 있으며 직접 확인한다.)
> - **JOIN에서 만든 별칭을 그보다 앞쪽(`FROM`이나 앞의 `ON`)에서는 쓸 수 없다.** 별칭은 "위에서 아래로, 앞에서 뒤로 정의된 것만" 쓸 수 있다.
> - PostgreSQL에서는 컬럼 별칭을 `GROUP BY`에서도 쓸 수 있는 것으로 알고 있으나(확실도 중간), 수업에는 없고 다른 DBMS에서는 안 될 수 있다.

---

## 주제 5: RIGHT JOIN, FULL JOIN, Self JOIN

### RIGHT JOIN

- **오른쪽 테이블의 모든 레코드**와 왼쪽 테이블의 일치하는 레코드를 반환한다.
- LEFT JOIN에서 **순서를 바꾸면 되기 때문에 잘 사용하지 않는다.**

### FULL JOIN

- 조인 조건이 일치하는 행을 연결하고, **양쪽 테이블의 일치하지 않는 행도 포함**한다.
- 일치하는 행이 없는 쪽의 컬럼은 **NULL**로 표시된다.

### Self JOIN

- JOIN의 한 형태는 아니라 **같은 테이블을 자기 자신과 조인하는 방식**이며, INNER JOIN, LEFT JOIN 등 모두 사용할 수 있다.

> ➕ **더 알아두기 — Self JOIN을 예시로 이해하기**
> (수업에는 설명 문장만 있고 예제 코드가 없어 대화 중 만든 예시이다. 설명용 가짜 데이터이다.)
>
> **Self JOIN은 "SELF JOIN"이라는 별도 문법이 아니라, 같은 테이블을 두 번 쓰면서 서로 다른 별칭을 붙여 조인하는 방식**이다. 한 테이블 안에 **자기 자신을 가리키는 FK**가 있을 때 쓴다. 직원 테이블에 상사의 직원ID(`상사ID`)가 들어 있는 경우가 대표적이다.
>
> **`employee`**
>
> | 직원ID | 이름 | 상사ID |
> |---|---|---|
> | E1 | 김대표 | *(NULL, 상사 없음)* |
> | E2 | 이서연 | E1 |
> | E3 | 박지호 | E2 |
> | E4 | 최하늘 | E2 |
>
> 직원 테이블에는 상사의 **이름이 아니라 ID**만 있으므로, 상사 이름을 보려면 **같은 테이블을 한 번 더** 이어야 한다.
>
> ```sql
> SELECT e.이름 AS 직원, m.이름 AS 상사
> FROM employee e
> LEFT JOIN employee m ON e.상사ID = m.직원ID;
> ```
>
> - `employee e`: **직원** 역할로 쓰는 테이블
> - `employee m`: **상사** 역할로 쓰는 같은 테이블 (이름만 다른 별칭)
> - `ON e.상사ID = m.직원ID`: 직원의 `상사ID`가 상사 쪽의 `직원ID`와 같은 행끼리 연결
>
> | 직원 | 상사 |
> |---|---|
> | 김대표 | NULL |
> | 이서연 | 김대표 |
> | 박지호 | 이서연 |
> | 최하늘 | 이서연 |
>
> - **같은 테이블을 두 번 쓰므로 별칭이 필수**이다. 별칭이 없으면 `이름`이 어느 쪽(직원인지 상사인지)인지 구분할 수 없다.
> - **LEFT JOIN을 쓴 이유**: 김대표는 상사가 없어서 `INNER JOIN`을 쓰면 결과에서 빠진다. 모든 직원을 남기려면 LEFT JOIN을 쓴다. 이렇게 Self JOIN에도 INNER, LEFT 등을 모두 쓸 수 있다.
> - 같은 테이블이지만 **두 개의 서로 다른 테이블처럼** 읽으면 쉽다. (직원 표와 상사 표를 따로 있다고 상상)

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 중간)
> **셀프 조인(자기 조인, Self JOIN)** 은 한 테이블을 자기 자신과 조인하는 것으로, **같은 테이블에 서로 다른 별칭을 붙여 사용**한다고 알고 있다. 직원–상사처럼 자기 자신을 참조하는 계층 구조가 대표 예이다. 시험 범위의 정확한 표현은 확인하지 못했다.

> ➕ **더 알아두기 — LEFT와 RIGHT는 서로 바꿀 수 있고, FULL은 별개이다**
> (대화 중 정리한 설명이며 일반 지식이다.)
>
> ```sql
> FROM city    LEFT  JOIN country ON ...   -- city 전부 남김
> FROM country RIGHT JOIN city    ON ...   -- 위와 똑같음 (순서와 방향을 같이 바꿈)
> ```
>
> `LEFT`/`RIGHT`는 "어느 쪽을 지키느냐"를 알려 주는 단어이고, 순서만 바꾸면 서로 같은 기능이다. 그래서 실무에서는 **`LEFT`로 통일하고 `FROM`에 남기고 싶은 테이블을 둔다.** 다만 남이 쓴 `RIGHT JOIN`을 읽을 줄은 알아야 한다. 표준에 둘 다 있는 것은 대칭 설계 때문으로 알고 있다.
>
> | 종류 | 다른 것으로 대체 가능? |
> |---|---|
> | `LEFT` ↔ `RIGHT` | ✅ 순서만 바꾸면 같음 |
> | `INNER` | ❌ 별개 (남기는 쪽 없음) |
> | `FULL` | ❌ 별개 (양쪽 모두 남김) |
>
> 한 방향으로 통일하면 긴 쿼리에서 "어느 쪽이 남는지" 추적하기 쉽다.

> ➕ **더 알아두기 — FULL JOIN 예시 (부서와 직원)**
> (수업에는 FULL JOIN 예제 코드가 없어 대화 중 만든 예시이다. 설명용 데이터이다.)
>
> `department`: D1 개발, D2 영업, D3 인사(소속 직원 없음) / `employee`: E1 김민준(D1), E2 이서연(D2), E3 박지호(부서 NULL)
>
> | JOIN 종류 | 결과 | 남는 행 |
> |---|---|---|
> | INNER | 김민준-개발, 이서연-영업 | 짝이 있는 것만 (2줄) |
> | LEFT (`employee` 기준) | 위 + 박지호-NULL | 직원은 모두 (3줄) |
> | RIGHT (`department` 기준) | 위 INNER + NULL-인사 | 부서는 모두 (3줄) |
> | **FULL** | 김민준-개발, 이서연-영업, 박지호-NULL, NULL-인사 | **양쪽 모두** (4줄) |
>
> ```sql
> SELECT e.이름, d.부서명
> FROM employee e
> FULL JOIN department d ON e.부서ID = d.부서ID;
> ```
>
> FULL JOIN은 **LEFT JOIN + RIGHT JOIN을 합친 것**이다. 주로 **두 테이블을 비교해 한쪽에만 있는 것을 찾을 때** 쓴다. `WHERE e.직원ID IS NULL OR d.부서ID IS NULL`을 붙이면 짝이 없는 행(박지호, 인사)만 남는다. PostgreSQL에서는 쓸 수 있지만 **MySQL에는 없는 것으로 알고 있어** DBMS마다 지원이 다르다. 실무에서는 INNER와 LEFT가 대부분이고 FULL은 가장 적게 쓰는 편으로 알고 있다.

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 중간)
> 외부 조인(**LEFT / RIGHT / FULL OUTER**)은 조인 조건에 맞지 않는 행도 포함하며, FULL OUTER JOIN은 양쪽 모두의 불일치 행을 포함하고 없는 쪽의 열은 NULL이다. 실무에서 `RIGHT`를 안 쓰더라도 의미는 알아 둔다. 표준에서 `OUTER` 키워드는 생략할 수 있는 것으로 알고 있다.

---

## 주제 6: 예제

### world database

**어느 나라에 속한 도시인지**

```sql
SELECT
    co.name AS country_name,
    ci.name AS city_name
FROM city ci
JOIN country co ON ci.countrycode = co.code;
```

**국가와 그 국가의 공식 수도를 매칭**

```sql
SELECT
    co.name AS country_name,
    ci.name AS capital_city
FROM country co
JOIN city ci ON co.capital = ci.id;
```

**특정 대륙에 속한 도시들 목록**

```sql
SELECT
    co.continent,
    co.name AS country_name,
    ci.name AS city_name
FROM country co
JOIN city ci ON co.code = ci.countrycode
WHERE co.continent = 'Asia'
ORDER BY co.name, ci.name;
```

**특정 대륙에서 인구가 500만 명 이상인 도시만 조회**

```sql
SELECT
    co.continent,
    co.name AS country,
    ci.name AS city,
    ci.population
FROM country co
JOIN city ci ON co.code = ci.countrycode
WHERE co.continent = 'Asia'
AND ci.population >= 5000000
ORDER BY ci.population DESC;
```

**국가와 수도, 공식언어 가져오기**

```sql
SELECT
    co.name AS country_name,
    ci.name AS capital_city,
    cl."Language"
FROM country co
JOIN city ci ON co.capital = ci.id
JOIN countrylanguage cl ON co.code = cl.countrycode
WHERE cl.isofficial = 'T';
```

- `cl."Language"`처럼 큰따옴표로 감싼 것은 **이름(컬럼명)** 을 대소문자 구분하여 가리키는 것이다.

### dvdrental — 고객 정보 조회

고객 정보를 **한 테이블씩 추가**하며 단계적으로 늘려 간다.

**고객의 이름, 이메일 조회**

```sql
SELECT
    c.first_name,
    c.last_name,
    c.email
FROM customer c;
```

**고객의 이름, 이메일, 주소 조회**

```sql
SELECT
    c.first_name,
    c.last_name,
    c.email,
    a.address
FROM customer c
JOIN address a ON c.address_id = a.address_id;
```

**고객의 이름, 이메일, 주소, 도시 조회**

```sql
SELECT
    c.first_name,
    c.last_name,
    c.email,
    a.address,
    ci.city
FROM customer c
JOIN address a ON c.address_id = a.address_id
JOIN city ci ON a.city_id = ci.city_id;
```

**고객의 이름, 이메일, 주소, 도시, 국가 조회**

```sql
SELECT
    c.first_name,
    c.last_name,
    c.email,
    a.address,
    ci.city,
    co.country
FROM customer c
JOIN address a ON c.address_id = a.address_id
JOIN city ci ON a.city_id = ci.city_id
JOIN country co ON ci.country_id = co.country_id;
```

**London(city)에 사는 고객의 이름, 이메일, 주소, 도시 조회**

```sql
SELECT
    c.first_name,
    c.last_name,
    c.email,
    a.address,
    ci.city
FROM customer c
JOIN address a ON c.address_id = a.address_id
JOIN city ci ON a.city_id = ci.city_id
WHERE ci.city = 'London';
```

**도시별 고객 수 조회**

```sql
SELECT
    ci.city, COUNT(*) AS customer_count
FROM customer c
JOIN address a ON c.address_id = a.address_id
JOIN city ci ON a.city_id = ci.city_id
GROUP BY ci.city_id, ci.city
ORDER BY COUNT(*) DESC;
```

> ➕ **더 알아두기 — customer → address → city 쿼리를 단계별로 읽기**
> (대화 중 정리한 설명이며 표는 설명용 가짜 데이터이다.)
>
> "고객의 이름, 이메일, 주소, 도시 조회" 쿼리를 따라가 본다. 고객에는 **주소의 번호(`address_id`)** 만, 주소에는 **도시의 번호(`city_id`)** 만 있어서 도시 이름까지 가려면 번호를 따라 두 번 이동해야 한다.
>
> ```
> customer ──address_id──▶ address ──city_id──▶ city
> ```
>
> | 단계 | 하는 일 |
> |---|---|
> | ① `FROM customer c` | 고객 테이블을 가져온다 |
> | ② `JOIN address a ON c.address_id = a.address_id` | 고객의 `address_id`와 주소의 `address_id`가 같은 행을 이어 **주소 문자열**을 붙인다. 도시는 아직 번호(`city_id`)뿐이다 |
> | ③ `JOIN city ci ON a.city_id = ci.city_id` | ②의 결과의 `city_id`와 도시의 `city_id`가 같은 행을 이어 **도시 이름**을 붙인다 |
> | ④ `SELECT c.first_name, c.last_name, c.email, a.address, ci.city` | 이어 붙인 결과에서 **원하는 5개 열만** 보여 준다 |
>
> - 각 `ON`은 FK와 PK를 잇는다. `c.address_id = a.address_id`는 `customer.address_id`(FK) = `address.address_id`(PK), `a.city_id = ci.city_id`는 `address.city_id`(FK) = `city.city_id`(PK)이다.
> - `JOIN`만 썼으므로 **INNER JOIN**이다. 주소가 없는 고객이 있다면 결과에서 빠지고, 그런 고객까지 보려면 `LEFT JOIN`을 써야 한다.
> - 다음 단계(국가까지)는 `JOIN country co ON ci.country_id = co.country_id`를 **한 줄 더 붙이면** 되는 같은 패턴이다.
> - `WHERE ci.city = 'London'`에서 `ci`를 쓸 수 있는 것은 **테이블 별칭이 `FROM` 단계에서 만들어져 이후 어디서든 쓸 수 있기** 때문이다.

### dvdrental — 배우와 영화 정보

**배우가 출연한 영화 조회**

```sql
SELECT a.first_name,
       a.last_name,
       f.title
FROM actor a
JOIN film_actor fa ON a.actor_id = fa.actor_id
JOIN film f ON fa.film_id = f.film_id;
```

**배우별 출연 영화 수**

```sql
SELECT a.first_name,
       a.last_name,
       COUNT(fa.film_id) AS num_of_films
FROM actor a
JOIN film_actor fa ON a.actor_id = fa.actor_id
GROUP BY a.actor_id, a.first_name, a.last_name
ORDER BY num_of_films;
```

**영화별 출연 배우 수**

```sql
SELECT f.title, COUNT(fa.actor_id) AS actor_count
FROM film f
JOIN film_actor fa ON f.film_id = fa.film_id
GROUP BY f.film_id;
```

**영화의 카테고리 정보**

```sql
SELECT
    f.title,
    c.name as category
FROM film f
JOIN film_category fc ON f.film_id = fc.film_id
JOIN category c ON fc.category_id = c.category_id
ORDER BY c.name;
```

**카테고리별 영화 수**

```sql
SELECT
    c.name AS category,
    COUNT(f.film_id) AS film_count
FROM category c
JOIN film_category fc ON c.category_id = fc.category_id
JOIN film f ON fc.film_id = f.film_id
GROUP BY c.name
ORDER BY film_count DESC;
```

**배우가 출연한 영화를 카테고리를 포함하여 조회**

```sql
SELECT a.first_name,
       a.last_name,
       f.title,
       c.name as category
FROM actor a
JOIN film_actor fa ON a.actor_id = fa.actor_id
JOIN film f ON fa.film_id = f.film_id
JOIN film_category fc ON f.film_id = fc.film_id
JOIN category c ON fc.category_id = c.category_id;
```

**출연 영화가 30편 이상인 배우**

```sql
SELECT
    a.first_name,
    a.last_name,
    COUNT(fa.film_id) AS film_count
FROM actor a
JOIN film_actor fa ON a.actor_id = fa.actor_id
GROUP BY a.actor_id, a.first_name, a.last_name
HAVING COUNT(fa.film_id) >= 30
ORDER BY film_count DESC;
```

**출연 배우가 10명 이상인 영화**

```sql
SELECT
    f.title,
    COUNT(fa.actor_id) AS actor_count
FROM film f
JOIN film_actor fa ON f.film_id = fa.film_id
GROUP BY f.film_id, f.title
HAVING COUNT(fa.actor_id) >= 10
ORDER BY actor_count DESC;
```

> ➕ **더 알아두기 — JOIN과 GROUP BY를 같이 쓸 때**
> 이 예제들은 JOIN으로 테이블을 이어 붙인 **그 결과**에 `GROUP BY`와 집계 함수를 적용한다. 처리 순서는 `FROM`(JOIN 포함) → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY`이다. 이어 붙인 결과에서 배우 한 명이 여러 줄이 되므로, 그 줄들을 배우별로 묶어 세면 "배우별 출연 영화 수"가 된다. `GROUP BY`에 쓰는 컬럼과 `SELECT`의 일반 컬럼이 일치해야 하는 규칙은 [GROUP BY 글](2026-10-02%20(6)SELECT_advanced(GROUP_BY,_집계함수,_HAVING).md)과 같다.

---

## 주제 7: 쉼표 조인과 JOIN, SELECT 절 순서 두음 (수업 외 · 대화 중 보충)

### FROM에 테이블을 쉼표로 나열하는 방식

> ➕ **더 알아두기 — "FROM에 2개를 쓰고 WHERE로 연결하면 되지 않나?"**
> (일반 지식이며 확실도는 높음~중간이다.)
>
> ```sql
> -- (가) 쉼표로 나열 + WHERE로 연결 (옛날 방식)
> SELECT c.이름, o.상품명
> FROM customer c, orders o
> WHERE c.고객ID = o.고객ID;
>
> -- (나) JOIN ... ON
> SELECT c.이름, o.상품명
> FROM customer c
> JOIN orders o ON c.고객ID = o.고객ID;
> ```
>
> (가)와 (나)는 **내부 조인으로는 같은 결과**이다. 2026-10-02 실습에서 쓴 `from city ct, country cr where (countrycode = cr.code) ...`가 (가) 방식이다. 그래도 `JOIN ... ON`을 권하는 이유는 다음과 같다.
>
> 1. **연결 조건과 거르는 조건이 분리**된다. (가)는 `WHERE` 하나에 둘이 섞여 테이블이 늘면 알아보기 어렵다. (나)는 `ON`과 `WHERE`가 나뉜다.
> 2. (가)에서 **`WHERE`를 깜빡해도 오류가 안 난다.** 고객 3명 × 주문 3건 = 9줄의 모든 조합이 조용히 나온다. `JOIN`은 `ON`을 쓰는 문법이라 빠뜨리기 어렵다.
> 3. 쉼표 방식은 짝이 맞는 행만 잇는 내부 조인이라 **LEFT/RIGHT/FULL 외부 조인을 표준 방식으로 표현할 수 없다.**
> 4. 테이블이 많을수록 `JOIN`은 "A에 B를 붙이고 그 결과에 C를 붙인다"가 위에서 아래로 보여 읽기 쉽다.
>
> 쉼표 방식은 옛 SQL 문법이라 오래된 코드에서 자주 보이므로 **읽을 줄은 알아야 한다.**

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 중간)
> 시험 문제의 SQL에서도 `FROM 테이블1, 테이블2 WHERE 조건`처럼 쉼표로 나열한 방식이 나오는 경우가 있는 것으로 알고 있어, 두 방식을 모두 읽을 줄 알아 두는 것이 좋다. (출제 형태는 확인하지 못했다.)

### SELECT 절 순서 두음

> ➕ **더 알아두기 — 쓰는 순서와 처리 순서**
> (수제비 교재의 정확한 두음 문구는 확인하지 못했다. 아래 두음은 대화 중 정리한 것이다.)
>
> | 순서 | 절 | 두음 |
> |---|---|---|
> | 1 | **SEL**ECT | 셀 |
> | 2 | **F**ROM | 프 |
> | 3 | **WHE**RE | 웨 |
> | 4 | **G**ROUP BY | 그 |
> | 5 | **H**AVING | 해 |
> | 6 | **O**RDER BY | 오 |
>
> → 쓰는 순서: **"셀프웨 그해오"**
>
> | | 순서 |
> |---|---|
> | **쓰는 순서** | `SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY` |
> | **처리 순서** | `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT` |
>
> 두 순서의 차이는 `SELECT`의 위치뿐이다. 처리 순서는 **"프웨 그해 셀 오"** 로 외울 수 있다. JOIN이 들어가면 `FROM`과 `WHERE` **사이**에 `JOIN ... ON ...`이 온다.
>
> ```
> SELECT → FROM → JOIN … ON … → WHERE → GROUP BY → HAVING → ORDER BY
> ```

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
> **SELECT 문의 절 순서**(`SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY`)는 시험에서 자주 다루는 기본 구조로 알고 있다. 순서를 틀리면 문법 오류이다. "올바른 문장 순서"를 묻는다면 쓰는 순서, SELECT가 언제 실행되는지 묻는다면 처리 순서를 떠올린다.

---

## 정리

- **JOIN**은 두 개 이상의 테이블을 연결해 조회하는 구문이다. `FROM 테이블1 JOIN유형 테이블2 ON 조인조건`. `JOIN` 뒤(ON 앞)는 이어 붙일 테이블이고, `ON`은 어떤 행끼리 이을지 정하는 연결 조건(보통 FK = PK)이다.
- **INNER JOIN**: 조건이 일치하는 행만. `INNER`를 생략하고 `JOIN`만 써도 INNER JOIN이다.
- **LEFT JOIN**: **왼쪽(FROM에 쓴) 테이블의 모든 행**을 남기고, 오른쪽에 짝이 없으면 NULL. NULL은 "짝이 없다"는 표시이며 `IS NULL`로 짝 없는 행을 찾을 수 있다.
- **RIGHT JOIN**은 LEFT에서 순서를 바꾼 것(잘 안 씀), **FULL JOIN**은 양쪽의 불일치 행도 포함(없는 쪽 NULL), **Self JOIN**은 같은 테이블을 자기와 조인하는 방식이다.
- **FROM 테이블 선택**: INNER는 기준 테이블을, LEFT는 모든 행을 유지할 테이블을 `FROM`에 둔다.
- **`ON`과 `WHERE`**: `ON`은 행을 연결하는 조건, `WHERE`는 조인 결과에서 남길 행을 선택하는 조건이다. LEFT JOIN에서는 오른쪽 테이블 조건을 어디에 쓰느냐에 따라 결과가 다르다. (`ON`에 두면 왼쪽 행 유지, `WHERE`에 두면 NULL 행도 제외)
- 여러 테이블을 이으려면 `JOIN … ON …`을 차례로 이어 쓴다. 각 `ON`은 새로 붙이는 테이블과 이미 붙은 테이블을 연결하며, 연관 테이블이 다리 역할을 한다.
- JOIN은 `FROM` 단계의 일부로 왼쪽부터 차례로 처리되며, `ON`은 JOIN할 때, `WHERE`는 JOIN이 끝난 뒤 적용된다. 별칭은 정의된 위치 이후에서만 쓸 수 있다.
- (수업 외) 쉼표 조인(`FROM A, B WHERE …`)은 내부 조인으로 같은 결과지만 조건 누락 시 모든 조합이 나오고 외부 조인을 못 쓴다. SELECT 절의 쓰는 순서는 "셀프웨 그해오"이다.

---

## ✅ 확인 질문

1. JOIN은 왜 필요한가? 테이블을 나눠 저장하는 것과 어떻게 연결되는가?
2. `FROM`, `JOIN`, `ON` 뒤에는 각각 무엇이 오는가? `JOIN` 뒤(ON 앞)에 오는 것은?
3. `ON` 조건이 하는 일은 무엇인가? `ON` 없이 두 테이블을 이으면 고객 3명 × 주문 3건에서 몇 줄이 나오는가?
4. JOIN의 `ON`에서 보통 어떤 열끼리 이으며(FK와 PK), `=`만 쓸 수 있는가? `=`로 잇는 조인을 무엇이라 하는가?
5. INNER JOIN은 어떤 행을 반환하는가? `JOIN`만 쓰면 어떤 JOIN으로 동작하는가?
6. 고객·주문 INNER JOIN 결과에서 주문이 없는 고객이 안 나오는 이유는? 한 고객이 주문을 여러 건 했으면 결과는 어떻게 되는가?
7. 배우–영화(M:N) 예제에서 JOIN이 두 번 쓰이는 이유는? 연관 테이블(`film_actor`)은 어떤 역할을 하는가?
8. `customer → address → city` 쿼리에서 각 `ON`은 어떤 FK와 PK를 잇는가? 도시 이름까지 가려면 왜 JOIN을 연달아 써야 하는가?
9. `JOIN address a`에서 `address`와 `a`는 각각 무엇인가? `a.address`는 무엇을 가리키는가?
10. LEFT JOIN은 어떤 행을 남기는가? "왼쪽"은 `FROM`에 쓴 테이블인가, `JOIN` 뒤에 쓴 테이블인가?
11. LEFT JOIN 결과에 NULL이 생기는 이유는? NULL을 이용해 "짝이 없는 행"을 찾으려면 `WHERE`에 어떻게 쓰는가?
12. "수도 정보가 없는 국가 찾기" 쿼리가 LEFT JOIN과 `IS NULL`을 쓰는 이유는? INNER JOIN으로는 왜 안 되는가?
13. LEFT JOIN으로 영화와 재고를 이으면 재고가 여러 개인 영화는 어떻게 조회되는가?
14. 3개 테이블을 LEFT JOIN으로 이었을 때 중간 단계에서 짝이 없으면 뒤의 열은 어떻게 되는가?
15. INNER JOIN과 LEFT JOIN에서 `FROM`에 어떤 테이블을 두어야 하는가?
16. `ON`과 `WHERE`의 차이를 적용 시점으로 설명하라.
17. LEFT JOIN에서 오른쪽 테이블 조건을 `ON`에 둘 때와 `WHERE`에 둘 때 결과가 어떻게 다른가? 왜 `WHERE`에 두면 NULL 행이 사라지는가?
18. `AND i.store_id = 1`을 `ON`에 둔 쿼리와 `WHERE`에 둔 쿼리의 결과를 각각 설명하라.
19. LEFT JOIN과 RIGHT JOIN은 어떤 관계인가? 실무에서 LEFT로 통일하는 이유는? FULL JOIN은 왜 대체할 수 없는가?
20. FULL JOIN은 어떤 행을 포함하는가? 부서·직원 예시에서 INNER, LEFT, RIGHT, FULL의 결과 줄 수는?
21. Self JOIN은 JOIN의 한 종류인가? 같은 테이블을 두 번 쓸 때 별칭이 반드시 필요한 이유는? 직원–상사 예에서 김대표(상사 없음)까지 결과에 넣으려면 어떤 JOIN을 쓰는가?
22. JOIN은 처리 순서에서 어느 단계에 속하는가? `ON`과 `WHERE`가 적용되는 시점은?
23. 별칭을 쓸 수 있는 위치를 테이블 별칭과 컬럼 별칭으로 나누어 설명하라. JOIN에서 만든 별칭을 `FROM`이나 앞의 `ON`에서 쓸 수 없는 이유는?
24. `FROM A, B WHERE …`(쉼표 조인)과 `JOIN … ON`은 같은 결과인가? `WHERE`를 빼먹으면 어떻게 되며, 쉼표 방식의 한계는?
25. JOIN과 `GROUP BY`를 함께 쓰는 "도시별 고객 수" 쿼리의 처리 순서를 설명하라.
26. (기사 대비) 내부 조인과 외부 조인(LEFT/RIGHT/FULL)의 차이는? 외부 조인에서 짝이 없는 쪽의 열은?
27. (기사 대비) SELECT 문의 절을 쓰는 순서를 쓰고, JOIN이 들어가면 어디에 오는지 설명하라.
