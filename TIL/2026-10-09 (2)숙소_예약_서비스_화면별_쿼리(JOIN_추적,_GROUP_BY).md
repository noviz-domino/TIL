---
tags: [database, postgresql, join, group-by, practice]
til: v2 2026-10-09
---

# 2026-10-09_숙소_예약_서비스_화면별_쿼리(JOIN_추적,_GROUP_BY)
> 작성일: 2026-10-09

## 🔗 관련 글

- [숙소 예약 서비스 DDL](2026-10-09%20(1)숙소_예약_서비스_DDL(PostgreSQL,_제약조건,_타입).md) — 이 쿼리가 쓰는 테이블 7개.
- [숙소 예약 서비스 ERD 설계](2026-10-08%20(1)숙소_예약_서비스_ERD_설계(기능_정의,_ERD_점검,_설계_결정).md) — 화면 ①~⑥과 기능 정의.
- [JOIN(INNER, LEFT, FULL, ON과 WHERE)](2026-10-06%20(2)JOIN(INNER,_LEFT,_FULL,_ON과_WHERE).md) — INNER/LEFT JOIN, `ON` vs `WHERE`.
- [JOIN 실습과 GROUP BY 규칙](2026-10-06%20(3)JOIN_실습과_GROUP_BY_규칙(강사님_설명,_서브쿼리).md) — GROUP BY 규칙, 강사님 설명.
- [Self JOIN](2026-10-07%20(1)Self_JOIN(같은_테이블_조인,_별칭,_댓글_대댓글).md) — 별칭, 처리 순서.

## ❓ 아직 모르겠는 것

- 아래 쿼리들은 **AI(Claude)가 작성한 것**이고, 샘플 데이터도 **새로 지어낸 값**이다. 이 글 시점에서는 **화면 ①만 직접 따라가며 확인**했고, 화면 ②~⑥의 실행 결과는 확인하지 못했다.
- 화면 ①의 `ORDER BY ... LIMIT 3`까지 직접 실행한 결과를 확인해야 한다.
- 호스트 페이지의 "숙소별 취소 제외 예약 집계" 쿼리를 직접 짜 보지 못했다.
- 한 번에 쿼리를 보내는 방식(JSON으로 내려 주는 구조)이 **실무에서 어떻게 나뉘는지**는 강사님의 설명을 짧게만 들었다.

---

## 주제 1: 샘플 데이터 넣기

DDL로 만든 테이블에 넣는 샘플 데이터이다. (**이름·이메일 등은 지어낸 값**이다.) 부모 테이블부터 넣어야 FK 오류가 나지 않는다.

```sql
INSERT INTO users (user_name, phone, email) VALUES
('김민수', '010-1111-1111', 'minsu@example.com'),
('이하은', '010-2222-2222', 'haeun@example.com'),
('박지영', '010-3333-3333', 'jiyoung@example.com'),
('정유진', '010-4444-4444', 'yujin@example.com'),
('최서연', '010-0000-0001', 'seoyeon@example.com'),
('한도윤', '010-5555-5555', 'doyoon@example.com');

INSERT INTO host (user_id, introduction) VALUES
(2, '제주에서 숙소를 운영합니다.'),
(1, '부산 바닷가 숙소를 운영합니다.');

INSERT INTO accommodation (host_id, name, region, address, description, phone, amenities, registered_at) VALUES
(1, '제주 바람집',   '제주', '제주특별자치도 제주시 (가상 주소)', '바람이 좋은 숙소', '064-111-1111', '무선 인터넷, 무료 주차, 에어컨, 주방', '2025-06-06'),
(2, '부산 파도하우스', '부산', '부산광역시 해운대구 (가상 주소)', '바다가 보이는 숙소', '051-222-2222', '무선 인터넷, 에어컨', '2025-06-05'),
(1, '제주 돌담스테이', '제주', '제주특별자치도 서귀포시 (가상 주소)', '돌담이 예쁜 숙소', '064-333-3333', '무선 인터넷, 주방', '2025-05-20');

INSERT INTO room (accommodation_id, room_name, max_guests, price_per_night, size, room_count, description) VALUES
(1, '바람 101호', 2, 85000, 20, 1, '더블침대 1, 정원 전망'),
(1, '바다 201호', 4, 130000, 30, 2, '바다 전망'),
(2, '파도 101호', 2, 95000, 22, 1, '아늑한 방'),
(2, '파도 201호', 4, 150000, 33, 2, '큰 방'),
(3, '돌담 1호',   3, 110000, 25, 1, '돌담 정원');

INSERT INTO booking (booking_no, user_id, room_id, check_in, check_out, guests, price_per_night, status, booked_at) VALUES
('BK-20250607-001', 1, 1, '2025-06-20', '2025-06-22', 2,  85000, '확정',     '2025-06-07'),
('BK-20250515-002', 1, 5, '2025-05-25', '2025-05-27', 2, 110000, '이용 완료', '2025-05-15'),
('BK-20250606-003', 5, 3, '2025-06-15', '2025-06-17', 2,  95000, '확정',     '2025-06-06'),
('BK-20250501-004', 3, 2, '2025-05-10', '2025-05-12', 3, 130000, '이용 완료', '2025-05-01'),
('BK-20250502-005', 4, 1, '2025-05-12', '2025-05-14', 2,  85000, '이용 완료', '2025-05-02');

INSERT INTO review (booking_id, rating, content) VALUES
(4, 5, '정말 좋았어요.'),
(5, 4, '깔끔하고 조용했어요.'),
(2, 4, '돌담이 예뻤어요.');

INSERT INTO wishlist (user_id, accommodation_id) VALUES (1, 1);
```

> ➕ **더 알아두기 — 이 데이터의 의도**
> 부산 파도하우스는 **후기가 하나도 없게** 만들었다. 평균 평점이 `NULL`로 나오는지(= 화면의 "아직 후기가 없습니다") 확인하기 위해서이다. 또 예약이 없는 객실(`파도 201호`)도 있어서 `LEFT JOIN`의 동작을 확인하기 좋다.

---

## 주제 2: 화면별 쿼리

### 화면 ①: 메인 페이지 (최근 등록한 숙소 3개)

```sql
SELECT a.name, a.address,
       MIN(r.price_per_night)    AS min_price,
       ROUND(AVG(rv.rating), 1)  AS avg_rating,
       a.registered_at
FROM accommodation a
JOIN room r ON r.accommodation_id = a.accommodation_id
LEFT JOIN booking b ON b.room_id = r.room_id
LEFT JOIN review rv ON rv.booking_id = b.booking_id
GROUP BY a.accommodation_id, a.name, a.address, a.registered_at
ORDER BY a.registered_at DESC
LIMIT 3;
```

**기대 결과**

| name | min_price | avg_rating | registered_at |
|---|---|---|---|
| 제주 바람집 | 85000 | 4.5 | 2025-06-06 |
| 부산 파도하우스 | 95000 | NULL | 2025-06-05 |
| 제주 돌담스테이 | 110000 | 4.0 | 2025-05-20 |

- `MIN(r.price_per_night)` — 숙소의 **객실 중 최저 요금**("1박 ○○원부터").
- `ROUND(AVG(rv.rating), 1)` — 평균 평점을 **소수 첫째 자리**까지.
- 후기가 없으면 평균이 `NULL`이므로 화면에서 "아직 후기가 없습니다"로 바꿔 보여 주거나, 쿼리에서 `COALESCE`로 바꿀 수 있다.

### 화면 ②: 숙소 목록 (지역별)

```sql
SELECT a.name, a.address,
       MIN(r.price_per_night)   AS min_price,
       ROUND(AVG(rv.rating), 1) AS avg_rating
FROM accommodation a
JOIN room r ON r.accommodation_id = a.accommodation_id
LEFT JOIN booking b ON b.room_id = r.room_id
LEFT JOIN review rv ON rv.booking_id = b.booking_id
WHERE a.region = '제주'
GROUP BY a.accommodation_id, a.name, a.address;
```

"제주 지역 · 숙소 2개"의 숫자는 위 결과의 **행 수**이다. (필요하면 별도 `COUNT` 쿼리로 구한다.)

### 화면 ③: 숙소 상세 (여러 쿼리)

**한 화면에 SQL이 여러 개**여도 된다. 숙소 정보, 객실 목록, 후기를 따로 가져온다.

```sql
-- 숙소 + 호스트 정보
SELECT a.name, a.address, a.description, a.amenities,
       u.user_name AS host_name, h.introduction
FROM accommodation a
JOIN host h ON h.host_id = a.host_id
JOIN users u ON u.user_id = h.user_id
WHERE a.accommodation_id = 1;

-- 객실 및 요금
SELECT room_name, description, max_guests, price_per_night
FROM room
WHERE accommodation_id = 1;

-- 후기 (작성자, 객실, 평점, 내용, 작성일)
SELECT u.user_name, r.room_name, rv.rating, rv.content, rv.created_at
FROM review rv
JOIN booking b ON b.booking_id = rv.booking_id
JOIN users u   ON u.user_id = b.user_id
JOIN room r    ON r.room_id = b.room_id
WHERE r.accommodation_id = 1;

-- 후기 수와 평균 평점
SELECT COUNT(*) AS review_count, ROUND(AVG(rv.rating), 1) AS avg_rating
FROM review rv
JOIN booking b ON b.booking_id = rv.booking_id
JOIN room r    ON r.room_id = b.room_id
WHERE r.accommodation_id = 1;
```

후기는 **`review → booking → users / room`** 경로로 작성자와 객실을 찾는다. (후기에 유저·객실 컬럼을 두지 않은 설계의 결과이다.)

### 화면 ④: 예약 확인

```sql
SELECT b.booking_no, a.name, a.address, r.room_name,
       b.check_in, b.check_out,
       (b.check_out - b.check_in)                AS nights,
       b.guests, u.user_name,
       b.price_per_night * (b.check_out - b.check_in) AS total_price,
       b.status, b.booked_at
FROM booking b
JOIN users u         ON u.user_id = b.user_id
JOIN room r          ON r.room_id = b.room_id
JOIN accommodation a ON a.accommodation_id = r.accommodation_id
WHERE b.booking_no = 'BK-20250607-001';
```

박 수는 `check_out - check_in`(날짜끼리 빼면 일수), 총액은 **예약 당시 1박 요금 × 박 수**로 계산한다. 예시의 85,000 × 2 = 170,000.

### 화면 ⑤: 마이페이지

```sql
-- 내 예약
SELECT b.booking_no, a.name, r.room_name, b.check_in, b.check_out,
       (b.check_out - b.check_in) AS nights, b.guests,
       b.price_per_night * (b.check_out - b.check_in) AS total_price, b.status
FROM booking b
JOIN room r          ON r.room_id = b.room_id
JOIN accommodation a ON a.accommodation_id = r.accommodation_id
WHERE b.user_id = 1;

-- 찜한 숙소
SELECT a.name, a.address, MIN(r.price_per_night) AS min_price,
       ROUND(AVG(rv.rating), 1) AS avg_rating
FROM wishlist w
JOIN accommodation a ON a.accommodation_id = w.accommodation_id
JOIN room r          ON r.accommodation_id = a.accommodation_id
LEFT JOIN booking b  ON b.room_id = r.room_id
LEFT JOIN review rv  ON rv.booking_id = b.booking_id
WHERE w.user_id = 1
GROUP BY a.accommodation_id, a.name, a.address;

-- 내가 쓴 후기
SELECT a.name, r.room_name, rv.rating, rv.content, rv.created_at, b.booking_no
FROM review rv
JOIN booking b       ON b.booking_id = rv.booking_id
JOIN room r          ON r.room_id = b.room_id
JOIN accommodation a ON a.accommodation_id = r.accommodation_id
WHERE b.user_id = 1;
```

### 화면 ⑥: 호스트 페이지

```sql
-- 내가 등록한 숙소와 객실
SELECT a.name, a.address, a.registered_at, r.room_name, r.price_per_night
FROM host h
JOIN accommodation a ON a.host_id = h.host_id
JOIN room r          ON r.accommodation_id = a.accommodation_id
WHERE h.user_id = 1;

-- 예약 현황과 예약자 정보
SELECT b.booking_no, a.name, r.room_name,
       u.user_name, u.email, u.phone,
       b.check_in, b.check_out, b.guests,
       b.price_per_night * (b.check_out - b.check_in) AS total_price, b.status
FROM host h
JOIN accommodation a ON a.host_id = h.host_id
JOIN room r          ON r.accommodation_id = a.accommodation_id
JOIN booking b       ON b.room_id = r.room_id
JOIN users u         ON u.user_id = b.user_id
WHERE h.user_id = 1;
```

### 화면 ⑥-4: 숙소별 예약 집계 (취소 제외)

```sql
SELECT a.name,
       COUNT(b.booking_id) AS booking_count,
       COALESCE(SUM(b.price_per_night * (b.check_out - b.check_in)), 0) AS total_amount
FROM host h
JOIN accommodation a ON a.host_id = h.host_id
JOIN room r          ON r.accommodation_id = a.accommodation_id
LEFT JOIN booking b  ON b.room_id = r.room_id AND b.status <> '취소'
WHERE h.user_id = 1
GROUP BY a.accommodation_id, a.name;
```

> ➕ **더 알아두기 — 취소 조건을 `ON`에 쓴 이유**
> `b.status <> '취소'`를 `WHERE`에 쓰면 **LEFT JOIN으로 살려 둔 "예약 없는 숙소" 행이 `NULL <> '취소'` → 거짓으로 걸러져서** 사라진다. **`ON`에 쓰면** 취소가 아닌 예약만 붙이고, 짝이 없는 숙소는 `NULL`로 남아서 **예약 0건인 숙소도 결과에 나온다.** `ON`과 `WHERE`의 차이는 `LEFT JOIN`에서 가장 크게 드러난다.
> `SUM`이 `NULL`이 되는 경우를 위해 `COALESCE(…, 0)`로 0으로 바꿨다.

---

## 주제 3: 화면 ① 한 단계씩 따라가기 (행 수 추적)

중간 테이블이 어떻게 변하는지 직접 `SELECT *`로 확인하며 공부했다. 처음에는 결과를 `COUNT`로만 보여 줬는데 **"중간 테이블의 구성이 어떻게 변하는지 하나씩 추적"** 하고 싶다는 요구가 맞았다. 단계마다 `SELECT *`로 본다.

```sql
-- 1단계: accommodation JOIN room
SELECT * FROM accommodation a
JOIN room r ON r.accommodation_id = a.accommodation_id;

-- 2단계: + LEFT JOIN booking
SELECT * FROM accommodation a
JOIN room r ON r.accommodation_id = a.accommodation_id
LEFT JOIN booking b ON b.room_id = r.room_id;

-- 3단계: + LEFT JOIN review
SELECT * FROM accommodation a
JOIN room r ON r.accommodation_id = a.accommodation_id
LEFT JOIN booking b ON b.room_id = r.room_id
LEFT JOIN review rv ON rv.booking_id = b.booking_id;
```

| 단계 | 행 수 | 컬럼 수 | 설명 |
|---|---|---|---|
| 1. 숙소 JOIN 객실 | 5행 | 17열 | 객실이 5개이므로 5행 (숙소 컬럼 9 + 객실 컬럼 8) |
| 2. + LEFT JOIN 예약 | **6행** | 27열 | 객실마다 예약을 붙인다. **바람 101호는 예약 2건**이라 2행, 파도 201호는 **예약이 없지만 LEFT JOIN이라 NULL로 남는다** |
| 3. + LEFT JOIN 후기 | 6행 | 32열 | 후기가 있는 예약에는 후기 컬럼이 채워지고 없으면 `NULL` |
| GROUP BY 후 | **3행** | — | 숙소 단위로 묶는다 |

### LEFT JOIN의 행 수 규칙

> ♻️ 처음엔 **JOIN이 기본으로 LEFT JOIN인 줄** 알았으나, **`JOIN`(= `INNER JOIN`)이 기본**이고, 짝이 없는 행을 살리려면 **`LEFT JOIN`을 명시**해야 한다. 이 실습에서 `accommodation JOIN room`이 INNER, `room LEFT JOIN booking`이 LEFT이다.

LEFT JOIN에서 **왼쪽 행 하나가 결과에 몇 번 나오는가?**

> **max(짝의 수, 1)** — 짝이 0개여도 1행(오른쪽은 `NULL`), 짝이 2개면 2행.

그래서 5행 → 6행이 된 것이다. (바람 101호: 짝 2 → 2행, 바다 201호: 1, 파도 101호: 1, 파도 201호: 0 → 1, 돌담 1호: 1 → 합 2+1+1+1+1 = 6)

> 📝 **기사 시험 포인트** (정보처리기사)
> JOIN 종류: **내부 조인(INNER JOIN)**은 양쪽에 모두 일치하는 행만, **외부 조인(OUTER JOIN)** 은 일치하지 않는 행도 포함(LEFT / RIGHT / FULL)한다. **참조 무결성**을 지키는 FK가 JOIN의 연결 고리이다.

### 마지막 단계: GROUP BY

| 숙소 | 묶이는 행 | MIN(요금) | AVG(평점) |
|---|---|---|---|
| 제주 바람집 | 바람 101호 2행 + 바다 201호 1행 = 3행 | 85,000 | (후기 5점, 4점) → 4.5 |
| 부산 파도하우스 | 파도 101호 1행 + 파도 201호 1행 = 2행 | 95,000 | 후기 없음 → NULL |
| 제주 돌담스테이 | 1행 | 110,000 | 4.0 |

> ➕ **더 알아두기 — NULL은 집계에서 무시된다**
> `AVG`, `MIN`, `SUM`은 **`NULL`을 계산에서 제외**한다. 파도하우스의 모든 후기 컬럼이 `NULL`이라서 `AVG`가 `NULL`이다. (`COUNT(*)`는 행 수를 세고 `COUNT(컬럼)`은 NULL이 아닌 값만 센다.)

### 직접 이해한 것

1. **"중간에 `LEFT JOIN`하면 6줄이 되는 이유"** — 객실 기준 `LEFT JOIN`에서 객실 하나에 예약이 여러 건이면 행이 늘어난다. 행 수가 **줄어드는 게 아니라 늘어날 수 있다.**
2. **`ORDER BY`는 JOIN의 일부가 아니다.** 처음엔 쿼리 끝의 `ORDER BY`가 JOIN과 같이 처리되는 줄 알았으나, **`ORDER BY`는 JOIN과 GROUP BY가 끝난 뒤 결과를 정렬하는 마지막 단계**이다.

   처리 순서: **FROM(JOIN) → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT**

3. "객실 최저 요금을 숙소별로 구하는" 쿼리이므로 **그룹의 기준은 숙소**이다.

### 왜 `GROUP BY a.accommodation_id, a.name, ...`인가?

처음 질문은 "숙소 id와 숙소명으로 그룹했는데, 숙소명이 같으면 어떻게 되는가"였다.

- **그룹의 기준은 사실상 `accommodation_id`** 이다. (PK라서 숙소 하나당 유일하다.)
- `a.name`, `a.address`, `a.registered_at`을 `GROUP BY`에 함께 적는 이유는 **SELECT에 쓰는 일반 컬럼은 `GROUP BY`에 있거나 집계 함수 안에 있어야** 하기 때문이다. (강사님이 설명한 규칙.) 숙소명이 같은 두 숙소가 있어도 `accommodation_id`가 다르므로 **서로 다른 그룹**으로 나뉜다.
- 반대로 **`GROUP BY`에 있지만 SELECT에 없는 컬럼은 허용**된다.

> ➕ **더 알아두기 — PostgreSQL의 PK 예외**
> PostgreSQL은 `GROUP BY`에 **PK(`a.accommodation_id`)가 있으면 같은 테이블의 다른 컬럼은 따로 적지 않아도** 허용하는 기능이 있는 것으로 알고 있다. 다만 수업에서는 `a.name`까지 모두 적는 방식이라 그대로 쓴다. (정확한 동작은 버전과 환경에 따라 직접 확인해야 한다.)

---

## 주제 4: 화면마다 쿼리가 다른데 한 번에 요청하는가?

질문: "웹사이트는 레이아웃에 필요한 값이 화면마다 다른데, **한 번에 쿼리를 요청**하는 게 의미가 있는가? 강사님은 JSON으로 내려 준다고 하셨다."

> 이해한 구조: 화면에 따라 필요한 쿼리가 다르고, **한 화면에 쿼리가 여러 개**여도 된다. 쿼리를 한 번에 요청하는 것이 중요한 게 아니라 **화면(API)마다 필요한 데이터를 SQL로 가져오는** 것이 핵심이다.

**데이터가 화면까지 가는 길**

```text
SQL 실행 (DB)
 → 백엔드 (cursor.execute() → fetchall() 로 행 가져오기)
 → JSON으로 직렬화 (직렬화: 데이터를 문자열로 바꿔 전송 가능하게 만드는 것)
 → 프론트엔드 (JSON을 파싱해서 화면에 그림)
```

- **백엔드**가 SQL을 실행한다. 프론트엔드는 SQL을 직접 쓰지 않는다.
- 화면 ③처럼 여러 쿼리가 필요하면 백엔드가 **여러 쿼리를 실행해 하나의 JSON으로 묶어** 내려 줄 수도 있고, **API를 나눌** 수도 있다. (구조는 설계에 달려 있다.)

> ➕ **더 알아두기 — 이 글에서 확인하지 못한 것**
> 쿼리 하나로 모든 것을 가져오는 것이 좋은지, 화면마다 나누는 것이 좋은지는 **성능과 유지보수에 따라 다르다.** 수업에서는 화면별 SQL 작성을 연습하는 것이 목적이다.

---

## 주제 5: 검증 쿼리

데이터가 제대로 들어갔는지 확인하는 쿼리이다.

```sql
SELECT 'users' AS t, COUNT(*) FROM users
UNION ALL SELECT 'host', COUNT(*) FROM host
UNION ALL SELECT 'accommodation', COUNT(*) FROM accommodation
UNION ALL SELECT 'room', COUNT(*) FROM room
UNION ALL SELECT 'booking', COUNT(*) FROM booking
UNION ALL SELECT 'review', COUNT(*) FROM review
UNION ALL SELECT 'wishlist', COUNT(*) FROM wishlist;
```

기대 행 수: users 6, host 2, accommodation 3, room 5, booking 5, review 3, wishlist 1.

> ➕ **더 알아두기 — `UNION ALL`**
> 여러 `SELECT`의 결과를 **위아래로 이어 붙인다.** 컬럼 수와 타입이 맞아야 한다. `UNION`은 중복을 제거하고 `UNION ALL`은 제거하지 않는다. (수업 범위 밖의 보충이다.)

---

## 정리

- 쿼리는 **화면별로** 작성하고, 한 화면에 여러 쿼리를 써도 된다. 화면 ③이 대표적이다.
- 집계(`MIN`, `AVG`, `COUNT`, `SUM`)는 **저장하지 않고 조회 때 계산**한다. 박 수는 `check_out - check_in`, 총액은 `price_per_night × 박 수`.
- `JOIN`은 기본이 **INNER**이다. 짝이 없는 행을 살리려면 **`LEFT JOIN`**. LEFT JOIN에서 왼쪽 행 하나가 나오는 횟수는 **max(짝의 수, 1)** 이다.
- 행이 **줄어드는 것이 아니라 늘어날 수도 있다**(1:N JOIN). 중간 결과는 `SELECT *`로 단계마다 확인하며 추적한다.
- `GROUP BY` 규칙: SELECT의 일반 컬럼은 `GROUP BY`에 있어야 한다. `GROUP BY`에만 있는 컬럼은 가능. 그룹의 실제 기준은 PK(`accommodation_id`).
- 처리 순서: **FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT**. `ORDER BY`는 JOIN의 일부가 아니다.
- `LEFT JOIN`에서 오른쪽 테이블 조건은 **`ON`에** 써야 짝 없는 행이 사라지지 않는다.
- 집계 함수는 **NULL을 무시**한다. 후기가 없으면 `AVG`는 `NULL`.
- 데이터 흐름: **SQL → 백엔드(`execute`/`fetchall`) → JSON → 프론트엔드**.

---

## ✅ 확인 질문

1. 샘플 데이터를 넣을 때 왜 부모 테이블부터 넣어야 하는가? 부산 파도하우스를 후기 없이 만든 이유는?
2. 화면 ①의 쿼리에서 `MIN(r.price_per_night)`와 `ROUND(AVG(rv.rating), 1)`는 각각 화면의 어떤 값인가?
3. 후기가 없는 숙소의 평균 평점은 왜 `NULL`인가? 화면에는 어떻게 보여줄 수 있는가?
4. 화면 ③에서 쿼리를 여러 개 쓰는 이유는? 후기에서 작성자와 객실은 어떤 경로로 찾는가?
5. 박 수와 총 예약금액은 SQL로 어떻게 계산하는가? 왜 컬럼으로 저장하지 않는가?
6. `JOIN`과 `LEFT JOIN`의 차이는? 기본 `JOIN`은 어느 쪽인가?
7. 숙소 JOIN 객실이 5행인데 LEFT JOIN 예약 후에 6행이 된 이유는? 행 수가 줄지 않고 늘어날 수 있는 이유는?
8. LEFT JOIN에서 왼쪽 행 하나가 결과에 나타나는 횟수 규칙은?
9. `ORDER BY`는 JOIN과 같은 단계에서 처리되는가? SQL의 처리 순서를 말하라.
10. `GROUP BY a.accommodation_id, a.name, a.address, a.registered_at`처럼 여러 컬럼을 적는 이유는? 숙소명이 같은 두 숙소는 같은 그룹이 되는가?
11. `GROUP BY`에는 있고 SELECT에는 없는 컬럼은 허용되는가? 반대는?
12. `AVG`, `MIN`, `SUM`은 `NULL`을 어떻게 다루는가? `COUNT(*)`와 `COUNT(컬럼)`의 차이는?
13. 취소가 아닌 예약만 집계할 때 `b.status <> '취소'`를 `ON`에 쓰는 이유는? `WHERE`에 쓰면 어떻게 되는가?
14. 중간 결과를 `COUNT`가 아니라 `SELECT *`로 단계마다 확인해야 하는 이유는?
15. SQL 결과가 화면에 표시되기까지의 흐름을 말하라. 직렬화란?
16. (기사 대비) 내부 조인과 외부 조인의 차이는? 외부 조인의 세 종류는?
