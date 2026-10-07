---
tags: [sql, join, group-by, subquery, practice]
til: v2 2026-10-06
---

# 2026-10-06_JOIN_실습과_GROUP_BY_규칙(강사님_설명,_서브쿼리)
> 작성일: 2026-10-06

## 🔗 관련 글

- [JOIN(INNER, LEFT, FULL, ON과 WHERE)](2026-10-06%20(2)JOIN(INNER,_LEFT,_FULL,_ON과_WHERE).md) — 이 글의 실습은 거기서 배운 JOIN 체인(`ON`으로 FK와 PK 잇기)과 별칭 규칙을 그대로 쓴다.
- [SELECT advanced(GROUP BY, 집계 함수, HAVING)](2026-10-02%20(6)SELECT_advanced(GROUP_BY,_집계함수,_HAVING).md) — `GROUP BY` 기본 개념과 "SELECT의 일반 컬럼은 GROUP BY에 있어야 한다"는 규칙이 거기에 있다. 이 글에서는 **강사님의 설명**으로 그 이유를 보완한다.
- [SELECT 기본](2026-10-06%20(1)SELECT_기본(DISTINCT,_WHERE,_ORDER_BY,_LIMIT_OFFSET).md) — `DISTINCT`, `ORDER BY`, `LIMIT`.

## ❓ 아직 모르겠는 것

- 이 글의 실습 쿼리는 **다른 세션에서 실행했거나 실행 전**이라 결과를 이 글에서 확인하지 못했다. (특히 1-2, 1-3)
- 1-1 "배우들이 출연한 영화의 등급을 중복 없이"의 **의도가 ①등급 목록인지 ②배우별 등급인지** 확정되지 않았다. 정답 쿼리나 기대 결과를 확인해야 한다.
- 1-3의 정답 쿼리를 **강사님이 어떤 코드로 푸셨는지** 직접 확인하지 못했다. (아래 "강사님 방식"은 설명을 듣고 추정한 예시이다.)
- **서브쿼리** 문법(`WHERE … = (SELECT …)`, `JOIN (SELECT …) t`, `WITH`)은 수업에서 정식으로 나오면 보완한다.

---

## 주제 1: JOIN 실습 문제 (dvdrental)

### 1-1. 배우들이 출연한 영화의 등급을 중복 없이 조회하시오

**쓴 쿼리** (원래 맨 위 설명 줄을 `#`로 적었으나 PostgreSQL의 주석은 `--`이다)

```sql
SELECT DISTINCT
    f.title,
    f.rating,
    a.first_name,
    a.last_name
FROM film f
JOIN film_actor fa ON f.film_id = fa.film_id
JOIN actor a ON fa.actor_id = a.actor_id;
```

JOIN 구조와 `ON` 조건은 모두 맞았다. **틀린 점은 `SELECT`에 `f.title`을 같이 고른 것**이다.

- `DISTINCT`는 **`SELECT`에 쓴 컬럼 전체의 조합**이 같은 행만 하나로 합친다. (수업: 여러 컬럼을 지정하면 컬럼 값의 조합이 같은 행의 중복을 제거한다.)
- `title`이 있으면 영화마다 줄이 갈라져서 "중복 없이"가 안 된다. 같은 배우가 같은 등급의 영화 두 편에 나오면 `(관상, R, 송강호)`, `(기생충, R, 송강호)`가 서로 다른 조합이라 2줄이 되고, `title`을 빼면 `(R, 송강호)` 1줄로 합쳐진다.

> ♻️ 처음엔 **`DISTINCT`를 걸면 "등급 같은 것"이 중복 제거된다**고 생각해 `title`까지 같이 골랐으나, **`DISTINCT`는 SELECT에 쓴 컬럼 전체의 조합**을 기준으로 하므로 중복 없이 보려는 대상보다 세밀한 컬럼(영화 제목)을 같이 고르면 그 단위로 줄이 늘어난다.

> ➕ **더 알아두기 — 문제의 두 가지 해석과 쿼리**
> (대화 중 정리한 내용이며, 어느 해석이 정답인지는 확인하지 못했다.)
>
> | 해석 | 원하는 결과 | 쿼리 |
> |---|---|---|
> | ① 출연한 영화의 등급 **목록** | 등급 종류 (보통 몇 줄) | `SELECT DISTINCT f.rating FROM film f JOIN film_actor fa ON f.film_id = fa.film_id;` |
> | ② **각 배우**가 출연한 영화의 등급 | (배우, 등급) 조합 | `SELECT DISTINCT a.actor_id, a.first_name, a.last_name, f.rating FROM film f JOIN film_actor fa ON … JOIN actor a ON …;` |
>
> - 영화에는 보통 배우가 있어서 "배우들이 출연한 영화"가 한정이 아니라 "배우별로"일 가능성이 있고, 그러면 ②가 JOIN 문제로서 더 자연스럽다. (3개 테이블을 모두 쓴다.) 어느 해석이든 **`title`은 빼야 한다.**
> - **①에서 JOIN의 역할은 "대상을 좁히는 조건"** 이다. `SELECT DISTINCT rating FROM film;`은 모든 영화가 대상이고, `INNER JOIN film_actor`를 붙이면 **출연 기록이 있는 영화만** 대상이 된다. 모든 영화에 배우가 있다면 결과는 같다. 확인하려면 아래 쿼리가 0줄인지 본다.
>
> ```sql
> SELECT f.title
> FROM film f
> LEFT JOIN film_actor fa ON f.film_id = fa.film_id
> WHERE fa.film_id IS NULL;
> ```
>
> - ①에서는 `actor` JOIN이 필요 없다. `film_actor`가 이미 "출연 기록이 있는지"를 알려 준다.
> - ②에서 `a.actor_id`를 같이 고르는 것은 **동명이인**이 한 사람으로 합쳐지는 것을 막기 위한 습관이다.
> - 결과 표가 어떤 모양이어야 하는지(열이 하나인지 셋인지)를 먼저 그리면 해석이 정해진다.

### 1-2. 카테고리별 평균 대여료를 조회하시오

필요한 정보가 있는 테이블을 먼저 찾는다.

| 필요한 것 | 어디에 있나 |
|---|---|
| 카테고리 이름 | `category` (`name`) |
| 대여료 | `film` (`rental_rate`) |
| 영화와 카테고리의 연결 | `film_category` (연관 테이블, M:N) |

```
category ──category_id── film_category ──film_id── film
 (이름)                    (연결)                (대여료)
```

수업의 "카테고리별 영화 수" 예제에서 `COUNT(f.film_id)`만 **`AVG(f.rental_rate)`** 로 바꾸면 된다.

```sql
SELECT c.name AS category,
       AVG(f.rental_rate) AS avg_rental_rate
FROM category c
JOIN film_category fc ON c.category_id = fc.category_id
JOIN film f ON fc.film_id = f.film_id
GROUP BY c.name;
```

> ➕ **더 알아두기 — 이 문제의 포인트**
> JOIN으로 이어 붙인 **결과에 `GROUP BY`와 집계 함수**를 적용하는 패턴이다. 처리 순서는 `FROM`+`JOIN`(이어 붙이기) → `GROUP BY`(카테고리별 묶기) → `SELECT`(평균 계산)이다. 평균의 소수점이 길면 `ROUND(AVG(f.rental_rate), 2)`로 소수 둘째 자리까지 볼 수 있는 것으로 알고 있다. (`ROUND`는 수업에 없다.) `LEFT JOIN`으로 바꾸면 영화가 하나도 없는 카테고리도 남고 평균은 NULL이 된다.

### 1-3. 고객 ID가 1인 고객이 가장 많이 대여한 영화의 카테고리를 조회하시오

필요한 테이블의 길은 다음과 같다.

| 필요한 것 | 어디에 있나 |
|---|---|
| 고객의 대여 기록 | `rental` (`customer_id`, `inventory_id`) |
| 재고 → 영화 연결 | `inventory` (`inventory_id`, `film_id`) |
| 영화 | `film` |
| 영화 → 카테고리 연결 | `film_category` |
| 카테고리 이름 | `category` |

```
rental ─inventory_id─ inventory ─film_id─ film ─film_id─ film_category ─category_id─ category
```

**문제 해석이 두 가지**이다. (강사님이 짚어 주신 내용)

```sql
--고객 ID가 1인 고객이 (가장 많이 대여한 영화의) 카테고리를 조회하시오.
--고객 ID가 1인 고객이 가장 많이 대여한 (영화의 카테고리)를 조회하시오.
```

| | 끊는 방법 | 뜻 |
|---|---|---|
| (a) | (가장 많이 대여한 **영화**)의 카테고리 | 가장 많이 대여한 **영화 한 편**을 먼저 정하고 그 영화의 카테고리 |
| (b) | 가장 많이 대여한 (**영화의 카테고리**) | 가장 많이 대여된 **카테고리** (카테고리 기준 1등) |

- **강사에 따르면 이런 문제는 서브쿼리로 먼저 "가장 많이 대여한 영화"를 구하는 쿼리를 만들고, 이후 별도의 쿼리로 카테고리를 조회**하면 된다.

> ➕ **더 알아두기 — 해석에 따라 정답이 달라지는 예와 쿼리**
> (대화 중 정리한 내용이며 강사님의 실제 코드는 확인하지 못했다. 서브쿼리는 수업에 아직 정식으로 나오지 않았다.)
>
> | 대여 기록 | 영화 | 카테고리 |
> |---|---|---|
> | 3번 | 관상 | Drama |
> | 2번 | 기생충 | Comedy |
> | 2번 | 웃음의 대학 | Comedy |
>
> (a)는 영화 기준 1등이 관상(3번)이라 **Drama**이고, (b)는 카테고리 합계가 Drama 3, Comedy 4(2+2)라 **Comedy**이다. 영화 한 편으로는 Drama가 1등이어도 카테고리로 묶으면 Comedy가 1등이 될 수 있다.
>
> **① 가장 많이 대여한 영화 구하기** (서브쿼리가 될 부분)
>
> ```sql
> SELECT i.film_id
> FROM rental r
> JOIN inventory i ON r.inventory_id = i.inventory_id
> WHERE r.customer_id = 1
> GROUP BY i.film_id
> ORDER BY COUNT(*) DESC
> LIMIT 1;
> ```
>
> **② (a) 그 영화의 카테고리 구하기** (①을 안에 넣음)
>
> ```sql
> SELECT c.name AS category
> FROM film_category fc
> JOIN category c ON fc.category_id = c.category_id
> WHERE fc.film_id = (
>     SELECT i.film_id
>     FROM rental r
>     JOIN inventory i ON r.inventory_id = i.inventory_id
>     WHERE r.customer_id = 1
>     GROUP BY i.film_id
>     ORDER BY COUNT(*) DESC
>     LIMIT 1
> );
> ```
>
> **(b) 카테고리 기준 1등**은 카테고리로 묶는다.
>
> ```sql
> SELECT c.name AS category, COUNT(*) AS rental_count
> FROM rental r
> JOIN inventory i ON r.inventory_id = i.inventory_id
> JOIN film_category fc ON i.film_id = fc.film_id
> JOIN category c ON fc.category_id = c.category_id
> WHERE r.customer_id = 1
> GROUP BY c.name
> ORDER BY rental_count DESC
> LIMIT 1;
> ```
>
> - `WHERE fc.film_id = ( … )`는 괄호 안 쿼리의 결과와 같은 `film_id`만이라는 뜻이다. `=`로 비교하려면 안쪽 쿼리가 **딱 한 줄(한 값)** 을 돌려줘야 하므로 `LIMIT 1`이 중요하다.
> - **동점**(대여 횟수가 같은 영화·카테고리가 여럿)이면 `LIMIT 1`은 임의의 하나만 고른다. 문제가 동점을 어떻게 다루는지는 의도를 확인한다.
> - 해석이 애매한 문제는 한 번에 쓰면 어느 해석인지 쿼리 안에서 섞이기 쉬워서, **먼저 "가장 많이 대여한 영화"만 구하는 쿼리를 만들어 결과를 확인하고 카테고리 조회를 붙이면 각 단계를 검증**할 수 있다.

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
> **서브쿼리(부속 질의)** 는 `WHERE`절에서 **단일 행 서브쿼리**(`=`, `>` 등으로 비교, 결과 한 줄)와 **다중 행 서브쿼리**(`IN` 등, 결과 여러 줄)로 구분하는 것으로 알고 있다. 위 ②의 `= (SELECT … LIMIT 1)`은 단일 행 서브쿼리이다. 서브쿼리를 `FROM`/`JOIN` 자리에 쓰는 경우(인라인 뷰)도 시험에서 구분하는 것으로 알고 있다.

---

## 주제 2: GROUP BY 규칙 — 강사님 설명과 기준 정하기

### SELECT와 GROUP BY의 양방향 규칙 (강사님 설명)

강사에게 질문해 정리한 내용은 다음과 같다.

| 방향 | 규칙 | 이유 |
|---|---|---|
| **SELECT → GROUP BY** | `SELECT`에 쓴 **일반 컬럼**(집계 함수 안에 들지 않은 컬럼)은 **반드시 `GROUP BY`에도 있어야 한다** | 출력할 값의 **근간 데이터**가 그룹 기준에 있어야 그룹 안에서 값이 하나로 정해진다 |
| **GROUP BY → SELECT** | `GROUP BY`에 쓴 컬럼이 **`SELECT`에 없어도 오류가 나지 않는다** | `GROUP BY`가 **`SELECT`보다 먼저 처리**되어, 그 데이터를 바탕으로 그룹이 이미 만들어져 있고 `SELECT`는 거기서 **필요한 컬럼만 골라 출력**하기 때문이다 |

- 강사님의 "백그라운드에 테이블이 생성된다"는 [GROUP BY 글](2026-10-02%20(6)SELECT_advanced(GROUP_BY,_집계함수,_HAVING).md)의 "그룹별 작은 표(이해용 모델)"와 같은 말이다.
- 정리하면 **`SELECT`의 일반 컬럼 ⊂ `GROUP BY`**(부분집합)이고, `GROUP BY`의 컬럼이 `SELECT`에 없어도 된다.

```sql
-- ✅ GROUP BY의 film_id가 SELECT에 없어도 됨
SELECT f.title, COUNT(*)
FROM ...
GROUP BY f.film_id, f.title;

-- ❌ SELECT의 f.title이 GROUP BY에 없으면 오류 (그룹 안에서 값이 하나로 안 정해짐)
SELECT f.title, COUNT(*)
FROM ...
GROUP BY f.film_id;
```

> ➕ **더 알아두기 — 한 줄에는 칸마다 값 하나, 같은 규칙의 다른 설명**
> 그룹 하나는 결과 한 줄이 되고 한 칸에는 값 하나만 적을 수 있다. `GROUP BY`에 적은 컬럼은 그룹 안에서 값이 전부 같고, 집계 함수는 여러 값을 하나로 줄이지만, 그 외 일반 컬럼은 값이 여럿(Korea, Japan)이라 한 칸에 적을 수 없다. 강사님 설명의 첫 번째 방향(SELECT → GROUP BY)과 같은 이야기이다.
> 단 같은 테이블의 **PK**를 `GROUP BY`에 쓰면 그 테이블의 다른 컬럼은 적지 않아도 되는 경우가 있는 것으로 알고 있다. (PostgreSQL, 확실도 중간) 다른 테이블 컬럼은 적어야 하므로 처음에는 모두 적는 방식이 안전하다.

### 기준 컬럼을 정하는 법

> ➕ **더 알아두기 — 기준은 "~별", 나머지는 SELECT의 일반 컬럼**
> (대화 중 정리한 요령이며, 위 강사님 규칙을 실습에 적용한 것이다.)
>
> 1. **진짜 기준**: 문제에서 "~별"에 해당하는 컬럼. 1-3에서는 "가장 많이 대여한 **영화**"가 영화마다 횟수를 비교하는 것이므로 영화별(`f.film_id`)이다.
>
> | 문제의 표현 | 그룹 기준 |
> |---|---|
> | "카테고리**별** 평균 대여료" | 카테고리 |
> | "도시**별** 고객 수" | 도시 |
> | "가장 많이 대여한 영화"(영화마다 횟수 비교) | 영화 |
>
> 2. **SELECT에 쓴 일반 컬럼을 `GROUP BY`에 덧붙인다.** (집계 함수는 제외)
>
> ```sql
> SELECT c.name AS category, f.title, COUNT(*) AS rental_count
> ...
> GROUP BY f.film_id, f.title, c.name
> ```
>
> **컬럼 수와 그룹 수는 별개**이다. `GROUP BY`에 컬럼이 3개여도 그룹 수는 **값의 조합이 몇 가지냐**로 정해지고, 영화가 정해지면 제목과 카테고리가 저절로 정해지므로 `film_id`만 쓸 때와 **그룹 수가 같다**(영화 수). 반대로 **영화가 같아도 값이 따로 도는 컬럼**(예: 대여 기록 번호 `rental_id`)을 추가하면 조합이 매번 달라져서 그룹이 대여 기록 수만큼 쪼개진다. 그래서 컬럼을 추가할 때는 "영화가 정해지면 이 값도 정해지나?"를 물어본다.
> 단 영화 하나에 카테고리가 여럿이면 `c.name` 때문에 그룹이 쪼개진다. (영화당 카테고리 수는 확인하지 못했다.)
>
> **그룹이 만들어진 뒤에는** `ORDER BY 횟수 DESC LIMIT 1`이 **그룹을 요약한 줄들 중 맨 위 하나**를 고른다. "전체에서 1등 하나"는 `LIMIT 1`로 풀 수 있다.

---

## 주제 3: 강사님의 풀이 습관

- 강사에 따르면 **`GROUP BY`의 컬럼은 되도록 하나만** 쓰려고 하고, 대신 **`JOIN`으로 비슷한 효과**를 낸다.
- 강사에 따르면 **코드를 처리 순서대로** 작성한다. 즉 **`FROM`부터** 쓴다.

> ➕ **더 알아두기 — 두 습관의 의미 (추정)**
> (강사님 코드를 직접 보지 못해 설명을 듣고 추정한 내용이다.)
>
> **GROUP BY 하나 + JOIN으로 보완**: 먼저 집계만 하고 그 결과에 JOIN으로 필요한 정보를 붙이는 방식으로 보인다.
>
> ```sql
> SELECT c.name AS category, f.title, t.rental_count
> FROM (
>     SELECT i.film_id, COUNT(*) AS rental_count     -- ① GROUP BY 컬럼 하나만
>     FROM rental r
>     JOIN inventory i ON r.inventory_id = i.inventory_id
>     WHERE r.customer_id = 1
>     GROUP BY i.film_id
>     ORDER BY rental_count DESC
>     LIMIT 1
> ) t
> JOIN film f ON t.film_id = f.film_id               -- ② 제목을 JOIN으로 붙임
> JOIN film_category fc ON f.film_id = fc.film_id
> JOIN category c ON fc.category_id = c.category_id;  -- ③ 카테고리도 JOIN으로 붙임
> ```
>
> 이렇게 하면 `GROUP BY`의 의도가 "영화별로 센다" 하나로 분명하고, "SELECT의 일반 컬럼을 GROUP BY에 모두 적는" 규칙을 신경 쓸 일이 줄고, 카테고리처럼 그룹을 쪼갤 수 있는 컬럼을 넣을 필요가 없어 결과가 안전하다.
>
> **처리 순서대로 쓰기**: `FROM`(테이블 정하기) → `JOIN … ON`(이어 붙이기) → `WHERE`(행 거르기) → `GROUP BY`(그룹) → `SELECT`(마지막에 보여줄 컬럼) → `ORDER BY`·`LIMIT`. `SELECT`를 맨 마지막에 쓴다는 점이 핵심이고, 앞에서 배운 처리 순서와 일치한다. `FROM`·`JOIN`에서 **별칭이 먼저 정의**되고 이후 단계에서 쓰므로 "별칭은 정의된 위치 이후에서만 쓴다"는 규칙과도 잘 맞는다.
> 글자를 쓰는 순서("셀프웨 그해오")와 생각하는 순서(처리 순서)는 다르다는 점을 기억한다.

---

## 주제 4: 서브쿼리 맛보기 (수업 외 · 대화 중 보충)

서브쿼리는 **쿼리의 결과를 다른 쿼리의 재료로 쓰는** 문법이다. 아래는 일반 지식이며 수업에서 나오면 정리한다.

> ➕ **더 알아두기 — 서브쿼리를 쓰는 자리**
>
> | 자리 | 예 | 결과 |
> |---|---|---|
> | `WHERE … = (SELECT …)` | 1-3의 ② | 안쪽이 **한 값**을 돌려줘야 한다 (단일 행) |
> | `WHERE … IN (SELECT …)` | 나라가 10개 넘는 대륙의 나라 목록 | 안쪽이 **여러 값** (다중 행) |
> | `JOIN (SELECT …) 별칭 ON …` | 고객별 주문 수를 먼저 구해 이름 붙이기 | 안쪽 결과를 **임시 테이블처럼** 붙임 (별칭 필요) |
> | `SELECT` 안의 서브쿼리 | 상관 서브쿼리 | 안쪽이 **바깥의 별칭**을 사용 |
> | `WITH 이름 AS (SELECT …)` | 결과에 이름을 붙여 재사용 | 한 문장 안에서 이어짐 |
>
> - 세미콜론으로 끝난 **별개의 SELECT 문장**은 서로 독립이라 각자 결과 표를 만들고 별칭도 이어지지 않는다. 서브쿼리를 괄호로 **한 문장 안에** 넣을 때만 안쪽이 바깥의 재료가 된다. (별칭의 유효 범위는 [JOIN 글](2026-10-06%20(2)JOIN(INNER,_LEFT,_FULL,_ON과_WHERE).md) 참고)
> - 서브쿼리 안쪽의 `SELECT` 결과는 따로 출력되지 않고 바깥 쿼리의 **재료**일 뿐이다.
> - 두 SELECT의 결과를 **한 표로 합치려면** `UNION`이라는 다른 문법이 있는 것으로 알고 있다. (수업에 없다.)

> ➕ **더 알아두기 — AS는 생략할 수 있다**
> 한 칸 띄워서 쓴 것도 별칭이다. `FROM rental r`, `COUNT(*) 개수`처럼 `AS`를 생략한 것이다. 다만 컬럼 별칭에서 **쉼표를 빼먹으면 의도치 않게 별칭이 되는** 실수(`SELECT name population FROM city;`)가 생겨서 컬럼 별칭에는 `AS`를 쓰는 것을 권하는 사람이 많다. (일반적인 관례이며 규칙은 아니다.)

---

## 정리

- **실습 문제**: 1-1 `DISTINCT`는 SELECT에 쓴 컬럼 전체의 조합 기준이라 중복 없이 보려는 대상만 고른다. 1-2 카테고리별 평균 대여료는 `category → film_category → film` JOIN + `GROUP BY` + `AVG`. 1-3은 해석이 둘(영화 기준 1등 → 카테고리 / 카테고리 기준 1등)이라 **서브쿼리로 단계를 나눠 푼다.**
- **강사님 설명**: `SELECT`의 일반 컬럼은 반드시 `GROUP BY`에 있어야 하고, `GROUP BY`의 컬럼은 `SELECT`에 없어도 된다. `GROUP BY`가 `SELECT`보다 먼저 처리되기 때문이다.
- **GROUP BY 기준**은 문제의 "~별"이고, SELECT의 일반 컬럼을 덧붙인다. 컬럼 수와 그룹 수는 별개이며 그룹 수는 값의 조합 수이다.
- **강사님의 습관**: `GROUP BY` 컬럼은 하나로 두고 JOIN으로 보완, 코드는 처리 순서대로(`FROM`부터) 작성한다.
- (수업 외) 서브쿼리는 쿼리 결과를 재료로 쓰는 문법으로 `WHERE`·`JOIN`·`SELECT`·`WITH`에 쓴다. `AS`는 생략할 수 있다.

---

## ✅ 확인 질문

1. `DISTINCT`는 무엇을 기준으로 중복을 제거하는가? `SELECT DISTINCT f.title, f.rating, a.first_name, a.last_name`이 "등급을 중복 없이"를 만족하지 못하는 이유는?
2. "배우들이 출연한 영화의 등급을 중복 없이"를 ① 등급 목록으로 읽을 때와 ② 배우별 등급으로 읽을 때 쿼리가 어떻게 다른가?
3. ①의 쿼리에서 JOIN(`film_actor`)은 어떤 역할이며, 모든 영화에 배우가 있다면 JOIN이 없을 때와 결과가 어떻게 되는가? `actor` JOIN이 필요 없는 이유는?
4. "카테고리별 평균 대여료"를 구할 때 어떤 테이블들을 어떤 순서로 JOIN하며, `GROUP BY` 기준은 무엇인가?
5. "고객 ID가 1인 고객이 가장 많이 대여한 영화의 카테고리"가 두 가지로 해석되는 이유를 괄호로 나누어 설명하라.
6. (a) 해석과 (b) 해석에서 `GROUP BY` 기준은 각각 무엇이며 결과가 달라질 수 있는 예를 들어 보라.
7. 강사님이 이런 문제를 서브쿼리로 푸라고 한 방식(먼저 가장 많이 대여한 영화를 구하고 이후 카테고리 조회)을 단계별로 설명하라.
8. `WHERE fc.film_id = (SELECT … LIMIT 1)`에서 안쪽 쿼리가 한 줄만 돌려줘야 하는 이유는? `LIMIT 1`이 없으면 어떻게 되는가?
9. `SELECT`와 `GROUP BY`의 양방향 규칙(강사님 설명)을 처리 순서로 설명하라.
10. `GROUP BY` 기준은 어떻게 정하는가? "~별"과 SELECT의 일반 컬럼으로 설명하라.
11. `GROUP BY`에 컬럼이 3개일 때 그룹이 3개인가? 그룹 수는 무엇으로 정해지는가? 영화 기준 그룹에서 `rental_id`를 추가하면 어떻게 되는가?
12. 강사님이 `GROUP BY` 컬럼을 하나만 쓰고 JOIN으로 보완하는 이유를 추정해서 설명하라.
13. 코드를 처리 순서대로(`FROM`부터) 작성하는 이점은? 글자를 쓰는 순서와 생각하는 순서의 차이는?
14. 세미콜론으로 끝난 별개의 SELECT 문장과 서브쿼리는 결과와 별칭 사용에서 어떻게 다른가?
15. `AS`를 생략한 별칭은 어떻게 쓰는가? 컬럼 별칭에서 쉼표를 빼먹으면 어떤 실수가 생기는가?
16. (기사 대비) 단일 행 서브쿼리와 다중 행 서브쿼리의 차이는? `=`와 `IN` 중 각각 어느 것과 어울리는가?
