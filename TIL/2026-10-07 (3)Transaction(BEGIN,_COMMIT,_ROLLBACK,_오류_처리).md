---
tags: [sql, transaction, commit, rollback, error-handling]
til: v2 2026-10-07
---

# 2026-10-07_Transaction(BEGIN,_COMMIT,_ROLLBACK,_오류_처리)
> 작성일: 2026-10-07

## 🔗 관련 글

- [SQL 기초와 DDL](2026-10-02%20(4)SQL_기초와_DDL(명령어_분류,_트랜잭션,_제약조건).md) — 10월 2일에 트랜잭션의 **기본 개념**(송금 예시, `BEGIN`/`COMMIT`/`ROLLBACK`, ACID, TCL 분류)을 먼저 배웠다. 이 글은 그 개념을 **실제 SQL로 직접 실행**한다. 거기의 "하나씩 입력 vs 묶어서 보내기"(`♻️`)와 PostgreSQL의 오류 동작 설명이 이 글의 "오류가 발생한 경우"와 이어진다.
- [DML(INSERT, SELECT, WHERE, UPDATE, DELETE)](2026-10-02%20(5)DML(INSERT,_SELECT,_WHERE,_UPDATE,_DELETE).md) — 아래 실습의 `UPDATE … SET … WHERE …`는 거기서 배운 문법이다. `WHERE`를 빼면 전체 행이 바뀌는 위험의 안전장치가 트랜잭션이다.

## ❓ 아직 모르겠는 것

- 이 글의 SQL은 **직접 실행해 결과를 확인하지 않았다.** 결과 값(70,000원, 80,000원 등)은 수업 설명을 옮긴 것이다.
- 10월 2일에 낸 연습 문제를 풀지 않았다: **송금 중 서버가 꺼졌고 출금까지 했으나 `COMMIT`은 하지 않았다면, DB를 다시 켰을 때 A의 잔액은 얼마인가?** (이 레슨에는 직접 나오지 않았다.)
- **SQL 인젝션** 레슨은 별도 글로 정리할 예정이다. (링크 미확인)

---

## 주제 1: 트랜잭션이란

- **하나 이상의 SQL 작업을 하나로 묶어 처리하는 논리적 작업 단위**이다.
- 예: 김철수가 이영희에게 30,000원을 송금하려면 **김철수의 계좌에서 출금**하고 **이영희의 계좌에 입금**해야 한다.
- 출금과 입금을 하나의 트랜잭션으로 묶으면 두 작업의 변경을 **모두 반영하거나 모두 취소**할 수 있다.
- 이를 통해 **출금만 반영되고 입금은 반영되지 않는 상태를 방지**한다.

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 높음)
> 트랜잭션의 4가지 성질 **ACID**(원자성·일관성·격리성·지속성)는 [SQL 기초와 DDL](2026-10-02%20(4)SQL_기초와_DDL(명령어_분류,_트랜잭션,_제약조건).md)에 표로 정리했다. 위 "모두 반영하거나 모두 취소"가 **원자성**(Atomicity)이다.

---

## 주제 2: 주요 명령어

| 명령어 | 역할 |
|---|---|
| `BEGIN` | 트랜잭션을 **시작**한다 |
| `COMMIT` | 현재 트랜잭션의 변경을 **확정**하고 종료한다 |
| `ROLLBACK` | 현재 트랜잭션의 변경을 **취소**하고 종료한다 |

- `BEGIN`으로 시작한 트랜잭션은 작업이 **성공하면 `COMMIT`**, **실패하거나 취소하려면 `ROLLBACK`** 으로 종료한다.
- **트랜잭션을 종료하지 않으면 변경이 미확정 상태로 남고, 잠금 때문에 다른 작업이 기다릴 수 있다.**
- **`COMMIT`으로 확정한 변경은 이후 `ROLLBACK`으로 취소할 수 없다.**

> ➕ **더 알아두기 — 잠금과 종료 습관**
> (일반 지식이며 수업에서 나온 짧은 설명을 풀어 쓴 것이다.) 트랜잭션 안에서 바꾼 행은 다른 사용자가 같은 행을 고치지 못하도록 **잠금(lock)** 이 걸리고, 트랜잭션이 끝나야 풀린다. 그래서 `BEGIN`만 하고 `COMMIT`/`ROLLBACK`을 안 하면 **다른 작업이 계속 기다리게** 된다. 수업의 "시작한 트랜잭션은 작업이 끝나면 바로 종료한다"가 이 이유이다.

> 📝 **기사 시험 포인트** (정보처리기사, 확실도: 중간)
> `COMMIT`, `ROLLBACK`은 **TCL(트랜잭션 제어어)** 이고, 표준 SQL의 TCL에는 `SAVEPOINT`가 같이 거론되는 것으로 알고 있다. 트랜잭션 시작은 표준 SQL에서 `START TRANSACTION`이고 `BEGIN`은 PostgreSQL 등에서 쓰는 형태로 알고 있다. **잠금(Lock)과 동시성 제어**는 격리성과 이어지는 개념으로 알고 있으나 수업에서는 이름만 나왔다. (확실도: 중간)

---

## 주제 3: 실습 데이터 준비

- **같은 연결에서 위에서부터 순서대로 실행**한다.
- **자동 커밋이 켜진 상태**에서 준비 SQL을 실행한다. 자동 커밋은 **명시적으로 트랜잭션을 시작하지 않았을 때 각 SQL 문이 성공하면 변경을 자동으로 확정**하는 방식이다.

```sql
CREATE TABLE transaction_account (
    account_id INTEGER PRIMARY KEY,
    owner_name VARCHAR(20) NOT NULL,
    balance INTEGER NOT NULL CHECK (balance >= 0)
);

INSERT INTO transaction_account (account_id, owner_name, balance)
VALUES
(1, '김철수', 100000),
(2, '이영희', 50000);
```

> ➕ **더 알아두기 — 테이블 정의 읽기**
> (DDL 글에서 배운 제약조건이 쓰인 예이다.) `account_id`는 `PRIMARY KEY`, `owner_name`과 `balance`는 `NOT NULL`이다. **`balance`의 `CHECK (balance >= 0)`은 잔액이 음수가 되는 것을 막는 제약조건**이고, 아래 "오류가 발생한 경우"에서 일부러 이 제약을 위반한다. **자동 커밋**이 켜져 있으면 `BEGIN` 없이 실행한 `INSERT`는 문장마다 바로 확정된다.

---

## 주제 4: 변경 확정하기

김철수의 계좌에서 이영희의 계좌로 30,000원을 송금한다. `BEGIN`부터 `COMMIT`까지 **두 번의 `UPDATE`를 하나의 트랜잭션으로 묶는다.**

```sql
BEGIN;

UPDATE transaction_account
SET balance = balance - 30000
WHERE account_id = 1;

UPDATE transaction_account
SET balance = balance + 30000
WHERE account_id = 2;

COMMIT;

SELECT * FROM transaction_account ORDER BY account_id;
```

- 확정 후 잔액은 **김철수 70,000원, 이영희 80,000원**이다.

> ➕ **더 알아두기 — 코드 읽기**
> `SET balance = balance - 30000`은 "balance를 (지금 balance에서 30000을 뺀 값)으로 설정한다"는 뜻이다. [DML 글](2026-10-02%20(5)DML(INSERT,_SELECT,_WHERE,_UPDATE,_DELETE).md)에서 배운 대로 `SET`의 `=`는 대입이고, `SET` 오른쪽에는 기존 값을 이용한 계산식을 쓸 수 있다. `WHERE account_id = 1`이 있어서 해당 계좌 한 행만 바뀐다. (`WHERE`가 없으면 모든 계좌가 바뀐다.)

---

## 주제 5: 변경 취소하기

김철수의 잔액을 10,000원 줄인 뒤 변경을 취소한다. **같은 트랜잭션 안에서는 아직 확정하지 않은 자신의 변경을 조회**할 수 있다.

```sql
BEGIN;

UPDATE transaction_account
SET balance = balance - 10000
WHERE account_id = 1;

SELECT * FROM transaction_account ORDER BY account_id;

ROLLBACK;

SELECT * FROM transaction_account ORDER BY account_id;
```

- `ROLLBACK` 전에는 김철수의 잔액이 **60,000원**으로 조회된다.
- `ROLLBACK` 후에는 김철수의 잔액이 **70,000원**으로 돌아온다. 이전 트랜잭션에서 **확정한 송금은 유지**된다.

> ➕ **더 알아두기 — `COMMIT` 전의 변경은 "임시 상태"**
> `COMMIT` 전의 변경은 확정이 아닌 **임시 상태**라서 `ROLLBACK`하면 없던 일이 되고, `COMMIT`하면 그때서야 진짜로 저장된다. 장바구니에 담는 동안은 취소할 수 있고(`ROLLBACK`) 결제 버튼을 누르면 확정(`COMMIT`)인 것과 같다. 이 임시 상태는 **같은 트랜잭션 안에서만 보인다.** 다른 연결에서 조회하면 어떻게 보이는지는 격리성 수준과 이어지는 내용으로 알고 있으며 수업에서 확인하지 못했다.
>
> **서버가 꺼지면?** (미해결 연습 문제와 관련) `COMMIT` 전에 서버가 꺼지면 미확정 변경은 **취소된 상태**로 DB가 다시 시작되는 것으로 알고 있다. 이는 원자성(전부 반영이거나 전부 취소)과 지속성(확정된 것은 유지)이 지켜지는 방식이다. (일반 지식이며 확실도 높음~중간)

---

## 주제 6: 오류가 발생한 경우

- 아래 SQL은 **명령문별로 실행**한다. 두 번째 `UPDATE`에서 **의도적으로 오류**가 발생한다.
- 첫 번째 `UPDATE`는 성공하지만, 두 번째 `UPDATE`는 잔액 70,000원에서 100만 원을 출금하려 하므로 **잔액이 음수가 되어 `CHECK` 제약조건을 위반**한다.

```sql
BEGIN;

UPDATE transaction_account
SET balance = balance + 10000
WHERE account_id = 2;

UPDATE transaction_account
SET balance = balance - 1000000
WHERE account_id = 1;
```

다음 명령을 **별도로 실행**해 트랜잭션 전체를 취소한다.

```sql
ROLLBACK;

SELECT * FROM transaction_account ORDER BY account_id;
```

- **먼저 성공했던 이영희의 잔액 증가도 취소**되어 김철수 70,000원, 이영희 80,000원이 유지된다.

> ➕ **더 알아두기 — 오류 후의 PostgreSQL 동작**
> PostgreSQL은 트랜잭션 안에서 SQL이 오류가 나면 그 트랜잭션을 **"실패 상태"** 로 표시하고, 이후 명령은 `ROLLBACK`으로 끝낼 때까지 **무시**한다. 이때 `COMMIT`을 보내도 실제로는 `ROLLBACK`처럼 처리되는 것으로 알고 있다. 그래서 수업의 두 번째 `UPDATE`가 실패한 뒤에는 **`ROLLBACK`으로 트랜잭션 전체를 끝내야** 한다. ([SQL 기초와 DDL](2026-10-02%20(4)SQL_기초와_DDL(명령어_분류,_트랜잭션,_제약조건).md)의 "프로그램에서의 트랜잭션과 PostgreSQL의 오류 동작"과 같은 내용) 직접 실행해서 확인한다.
> 이 예제는 **`CHECK` 제약조건**(DDL 글)이 실제로 데이터를 지키는 모습과, 그 위반이 트랜잭션 전체 취소로 이어지는 모습을 함께 보여 준다.

---

## 주제 7: 백엔드에서 오류 처리하기

- 백엔드에서는 작업이 **모두 성공하면 `COMMIT`**, **도중에 예외가 발생하면 `ROLLBACK`** 하도록 처리한다.
- 다음은 Python 코드 예시이다. `connection`은 연결된 DB 객체이며, **진행 중인 트랜잭션이 없는 상태에서 시작**한다.

```python
cursor = connection.cursor()

try:
    cursor.execute("BEGIN")

    cursor.execute("""
        UPDATE transaction_account
        SET balance = balance - 30000
        WHERE account_id = 1
    """)

    cursor.execute("""
        UPDATE transaction_account
        SET balance = balance + 30000
        WHERE account_id = 2
    """)

    connection.commit()
except Exception:
    connection.rollback()
    raise
finally:
    cursor.close()
```

- SQL 실행 중 오류가 발생하면, **앞서 실행한 변경도 함께 취소**한다.
- **`raise`는 예외를 다시 전달**하여 요청을 처리하는 쪽에서 실패를 알 수 있게 한다.

> ➕ **더 알아두기 — 코드 읽기**
> (대화 중 정리한 설명이다. [SQL 기초와 DDL](2026-10-02%20(4)SQL_기초와_DDL(명령어_분류,_트랜잭션,_제약조건).md)의 "프로그램에서의 트랜잭션"에 의사 코드로 적어 둔 모양이 실제 코드로 나온 것이다.)
>
> | 부분 | 하는 일 |
> |---|---|
> | `try:` 안 | 출금, 입금 두 `UPDATE`를 실행하고 모두 성공하면 `connection.commit()`으로 **확정** |
> | `except Exception:` | 예외가 나면 `connection.rollback()`으로 **전부 취소** |
> | `raise` | 예외를 **다시 던져** 요청을 처리하는 쪽에서 실패를 알게 함 |
> | `finally:` | 성공이든 실패든 **항상** `cursor.close()`로 커서를 닫음 |
>
> 이전에 "`BEGIN`부터 `ROLLBACK`까지를 사람이 하나씩 입력해야 하는가?"를 물었는데, **실제 서비스에서는 이처럼 프로그램이 오류를 감지해 자동으로 `ROLLBACK`** 한다. `ROLLBACK`은 "사람이 알아챈 다음 치는 명령"이 아니라 **"실패했을 때 실행하도록 정해 둔 명령"** 이다. (드라이버에 따라 `cursor.execute("BEGIN")`을 직접 쓰지 않아도 트랜잭션이 암묵적으로 시작되는 경우가 있는 것으로 알고 있다. 일반 지식이며 확실도 중간.)

---

## 주제 8: 트랜잭션 사용 시 확인할 점

- **함께 성공하거나 취소되어야 하는 작업**을 하나의 트랜잭션으로 묶는다.
- 시작한 트랜잭션은 **작업이 끝나면 바로 종료**한다. 오래 열어 두면 **다른 작업이 잠금 해제를 기다릴 수 있다.**

---

## 정리

- **트랜잭션**은 하나 이상의 SQL 작업을 하나로 묶은 논리적 작업 단위이다. 묶으면 변경을 **모두 반영하거나 모두 취소**해서 송금에서 출금만 반영되는 상태를 막는다.
- **명령어**: `BEGIN`(시작), `COMMIT`(확정·종료), `ROLLBACK`(취소·종료). 종료하지 않으면 미확정 상태로 남고 **잠금** 때문에 다른 작업이 기다릴 수 있다. **`COMMIT`한 변경은 `ROLLBACK`으로 취소할 수 없다.**
- **자동 커밋**: 명시적으로 트랜잭션을 시작하지 않으면 각 SQL 문이 성공할 때마다 자동으로 확정된다.
- **확정 / 취소 / 오류**: 송금 두 `UPDATE`를 `BEGIN`~`COMMIT`으로 묶으면 확정(김철수 70,000, 이영희 80,000). `ROLLBACK` 전에는 자신의 변경이 조회되고(60,000원) 후에는 되돌아온다(70,000원). 두 번째 `UPDATE`가 `CHECK`(잔액 ≥ 0)를 위반해 오류가 나면 `ROLLBACK`으로 앞서 성공한 변경까지 **전체 취소**한다.
- **백엔드**: 모두 성공하면 `commit()`, 예외가 나면 `rollback()`하고 `raise`로 실패를 전달하며, `finally`에서 커서를 닫는다.
- **확인할 점**: 함께 성공하거나 취소되어야 하는 작업을 하나로 묶고, 끝나면 바로 종료한다.

---

## ✅ 확인 질문

1. 트랜잭션이란 무엇인가? 송금 예시로 설명하라.
2. `BEGIN`, `COMMIT`, `ROLLBACK`은 각각 무엇을 하는가? `COMMIT`한 뒤 `ROLLBACK`으로 되돌릴 수 있는가?
3. `BEGIN`을 하고 `COMMIT`/`ROLLBACK`으로 끝내지 않으면 어떤 문제가 생기는가?
4. 자동 커밋이란 무엇이며, 실습 데이터 준비를 자동 커밋이 켜진 상태에서 하는 이유는?
5. `transaction_account`의 `balance`에 걸린 `CHECK (balance >= 0)`은 무엇을 막는가?
6. 송금 트랜잭션의 두 `UPDATE`에서 `SET balance = balance - 30000`과 `WHERE account_id = 1`은 각각 무슨 뜻인가?
7. `ROLLBACK` 전과 후에 김철수의 잔액은 각각 얼마이며, 이전에 확정한 송금은 어떻게 되는가? 같은 트랜잭션 안에서 미확정 변경을 조회할 수 있는 이유는?
8. 두 번째 `UPDATE`에서 `CHECK`를 위반해 오류가 났을 때 왜 `ROLLBACK`이 필요하며, 먼저 성공한 이영희의 잔액 증가는 어떻게 되는가?
9. 오류가 난 트랜잭션에서 PostgreSQL은 이후 명령을 어떻게 처리하는가? `COMMIT`을 보내면 어떻게 되는가?
10. 백엔드에서 트랜잭션을 처리하는 Python 코드에서 `commit`, `rollback`, `raise`, `finally`는 각각 무슨 역할인가?
11. 실제 서비스에서는 누가 `ROLLBACK`을 결정하는가? 사람이 하나씩 입력하는 방식과 어떻게 다른가?
12. 트랜잭션을 사용할 때 확인할 점 두 가지는 무엇인가?
13. 송금 중 서버가 꺼졌고 출금까지 했으나 `COMMIT` 전이었다면 다시 켰을 때 잔액은 어떻게 되는가?
14. (기사 대비) ACID 네 가지 성질을 쓰고, "모두 반영하거나 모두 취소"는 어떤 성질인가? `COMMIT`, `ROLLBACK`은 어떤 분류(DDL/DML/DCL/TCL)에 속하는가?
