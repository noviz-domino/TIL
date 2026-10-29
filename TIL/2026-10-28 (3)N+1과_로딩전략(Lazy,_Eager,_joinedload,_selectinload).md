---
tags: [sqlalchemy, n-plus-one, lazy-loading, eager-loading, query-optimization]
til: v2 2026-10-28
---

# 2026-10-28_N+1과_로딩전략(Lazy,_Eager,_joinedload,_selectinload)
> 작성일: 2026-10-28

## 🔗 관련 글

- [ORM 관계와 연결 테이블](2026-10-28%20%282%29ORM_%EA%B4%80%EA%B3%84%EC%99%80_%EC%97%B0%EA%B2%B0%ED%85%8C%EC%9D%B4%EB%B8%94%281%EB%8C%80N%2C_M%EB%8C%80N%2C_relationship%2C_association_proxy%29.md) — 연관 객체 접근이 추가 SELECT를 일으키는 출발점이다.
- [DTO 응답 변환](2026-10-28%20%281%29FastAPI_%EC%9A%94%EC%B2%AD%ED%9D%90%EB%A6%84%EA%B3%BC_DTO%28Depends%2C_Annotated%2C_yield%2C_CRUD%29.md) — 객체 속성을 읽는 응답 과정과 연결한다.

## ❓ 아직 모르겠는 것

대화에서 풀렸지만 실제 질문한 내용들을 복습 때 다시 설명한다.

- 지연 로딩은 get_db의 yield처럼 실행을 멈추는 것인가? (주제 1)
- post.comments는 프런트엔드의 댓글 버튼을 눌러 비동기 요청을 보내는 것 아닌가? (주제 2)
- 지연 로딩은 SQL을 미리 만들어 놓고 실행만 미루는 것인가? (주제 2)
- 즉시 로딩과 지연 로딩은 결국 둘 다 미리 가져오는 것 아닌가? (주제 2)
- JOIN으로 편하게 가져오는 것인가, 메모리 최적화인가? (주제 3)
- 즉시 로딩은 좋고 지연 로딩은 나쁜 것인가? (주제 3)
- JOIN 한 번과 조회 두 번 중 무엇을 선택하고 unique는 왜 필요한가? (주제 4)

## 주제 1: N+1 문제와 yield의 차이

원문은 [N+1 문제](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1451/)이다. 게시글 목록을 한 번 읽은 뒤 각 게시글의 댓글을 개별 조회하면 게시글 N개에 대해 1+N회의 SQL이 실행된다. N은 댓글 수가 아니라 처음 조회한 게시글 수이다. DB와 반복 통신하고 쿼리를 처리하는 비용이 누적되어 응답이 느려질 수 있다.

| 항목 | 하는 일 | 진행 계기 |
|---|---|---|
| ORM 지연 로딩 | 아직 읽지 않은 연관 데이터의 DB 조회를 미룸 | 관계 속성에 접근 |
| get_db의 yield | 실행 위치를 보존하며 Session을 전달 | FastAPI의 제너레이터 진행·정리 |

둘은 ‘미룬다’는 느낌만 비슷하다. 관계 지연 로딩이 우리가 작성한 yield 때문에 일어나는 것은 아니며 비동기 실행과도 동의어가 아니다. 미조회 관계를 읽으려면 Session이 필요하다는 점에서 Session 수명과 관련은 있다.

## 주제 2: 지연 로딩과 즉시 로딩

지연 로딩(Lazy Loading)은 연관 데이터를 사용하는 시점까지 조회하지 않는 방식이다. 강의의 relationship 기본 전략이 이에 해당한다. 게시글 조회 후 comments가 미조회라면 post.comments를 읽을 때 SELECT가 추가된다. 끝까지 댓글을 사용하지 않으면 댓글 쿼리가 실행되지 않는다. 관계 정의가 미리 있다는 것과 객체·SQL 결과를 미리 가져왔다는 것은 다르다.

즉시 로딩(Eager Loading)은 조회 결과를 구성하는 과정에서 필요한 연관 데이터도 준비하는 방식이다. 이후 post.comments를 읽을 때 이미 준비된 값을 사용한다. 무조건 SQL 한 번이라는 뜻은 아니며 JOIN으로 합치거나 게시글 뒤에 댓글을 묶음 조회할 수 있다.

| 시점 | 지연 로딩 | 즉시 로딩 |
|---|---|---|
| 게시글 조회 결과 준비 | 게시글만 읽고 댓글은 미조회일 수 있음 | 지정한 댓글까지 준비 |
| 첫 미조회 comments 접근 | 추가 SELECT | 준비된 객체 사용 |
| 댓글을 전혀 사용하지 않음 | 조회 생략 가능 | 이미 가져온 비용 발생 |

> ♻️ 처음엔 댓글 버튼을 누른 뒤 JS가 서버에 요청하는 기능과 지연 로딩을 같은 것으로 생각했으나, 전자는 HTTP 요청 설계이고 후자는 한 HTTP 요청을 처리하는 서버 내부의 DB 접근 전략이다. 프런트엔드 요청과 최종 응답이 같아도 DB 조회 방식은 다를 수 있다.

> ➕ **더 알아두기**
> ‘게시글 10개와 댓글도 달라’는 요청 하나를 처리할 때 게시글을 조회하고 응답 변환 중 post.comments를 각각 읽으면 11회가 될 수 있다. Pydantic의 from_attributes도 속성을 읽으므로 응답 변환이 지연 로딩을 유발할 수 있다. 모든 SQL 문자열을 먼저 만들어 대기시키는 것이 아니라 관계 정의·외래키 값으로 필요할 때 구성·실행한다.

## 주제 3: 무엇을 최적화하며 언제 선택하는가

N+1 해결의 일차 목적은 반복 조회와 네트워크 왕복 비용 감소이다. 메모리 사용이 반드시 줄어드는 것은 아니다. JOIN으로 게시글 본문이 댓글 수만큼 반복되면 결과 행의 데이터 양이 늘 수 있다. 불필요한 관계를 즉시 로딩하면 전송·메모리 비용을 추가한다.

게시글 제목만 필요하면 댓글을 지연 상태로 두고 읽지 않는다. 게시글 여러 개의 댓글이 응답에 반드시 필요하면 미리 묶어 준비해 반복 조회를 줄인다. 같은 모델 관계도 조회 기능마다 옵션을 다르게 줄 수 있다. SQL 횟수뿐 아니라 행 수·반복 컬럼·사용 데이터 양을 함께 비교한다.

## 주제 4: joinedload·selectinload·unique

joinedload는 LEFT OUTER JOIN으로 연관 데이터를 같은 SQL에서 가져온다. 단일 작성자처럼 N:1·1:1에서 우선 고려할 수 있다.

```python
stmt = (
    select(PostNPlusOne)
    .options(joinedload(PostNPlusOne.user))
    .order_by(PostNPlusOne.id)
)
posts = db.scalars(stmt).all()
```

댓글처럼 컬렉션을 joinedload하면 댓글 수만큼 부모 컬럼이 반복된다. 이 경우 SQLAlchemy는 unique로 부모 ORM 객체의 중복 제거를 명시하도록 요구한다.

```python
stmt = (
    select(PostNPlusOne)
    .options(joinedload(PostNPlusOne.comments))
    .order_by(PostNPlusOne.id)
)
posts = db.scalars(stmt).unique().all()
```

unique는 DB에서 가져온 행 수를 줄이는 DISTINCT가 아니라 ORM 결과 객체를 중복 없이 받는 처리이다. 해당 컬렉션 joinedload에서 생략하면 예외가 난다.

selectinload는 게시글을 먼저 조회하고 그 ID들을 모아 댓글을 IN 조건으로 읽는다. 일반적인 작은 결과에서는 SQL 두 번이며, 게시글이 없으면 두 번째 조회가 없고 ID가 많으면 분할되어 횟수가 늘 수 있다.

```python
stmt = (
    select(PostNPlusOne)
    .options(selectinload(PostNPlusOne.comments))
    .order_by(PostNPlusOne.id)
)
posts = db.scalars(stmt).all()
```

댓글 객체는 post_id를 기준으로 각 게시글의 comments에 연결된다. 컬렉션인 1:N·M:N에서 우선 고려할 수 있다. 두 로딩 전략 모두 단일 객체와 컬렉션에 사용할 수 있으며 위 구분은 절대 규칙이 아니다.

| 관계 | 강의에서 우선 고려 | 이유 |
|---|---|---|
| N:1·1:1 | joinedload | 연결 객체 하나를 JOIN으로 받기 쉬움 |
| 1:N·M:N | selectinload | 부모 컬럼이 반복되는 결과를 줄임 |

## 주제 5: 관계 조합과 일반 JOIN의 차이

게시글 작성자는 JOIN, 댓글은 묶음 조회로 함께 준비할 수 있다.

```python
stmt = (
    select(PostNPlusOne)
    .options(
        joinedload(PostNPlusOne.user),
        selectinload(PostNPlusOne.comments),
    )
    .order_by(PostNPlusOne.id)
)
posts = db.scalars(stmt).all()
```

게시글→댓글→댓글 작성자 경로는 옵션을 연결한다. 댓글을 IN으로 조회할 때 사용자 테이블까지 JOIN하면 별도의 작성자 반복 조회를 막는다.

```python
stmt = (
    select(PostNPlusOne)
    .options(
        selectinload(PostNPlusOne.comments).joinedload(CommentNPlusOne.user)
    )
    .order_by(PostNPlusOne.id)
)
posts = db.scalars(stmt).all()
```

반면 일반 join은 연관 테이블의 컬럼으로 필터·정렬하는 데 사용한다.

```python
stmt = (
    select(PostNPlusOne)
    .join(PostNPlusOne.user)
    .where(UserNPlusOne.name == "Alice")
)
posts = db.scalars(stmt).all()
```

이 조회는 게시글 컬럼만 선택하므로 post.user 객체를 미리 채우는 설정이 아니다. 관계 객체도 필요하면 로딩 옵션을 별도로 지정한다.

## 주제 6: 강의 실습의 데이터·API·조회 수

실습 모델은 사용자 id·name, 게시글 id·title·user_id, 댓글 id·content·post_id·user_id이다. 작성자·댓글·부모 관계는 ForeignKey와 양방향 relationship으로 연결한다. 강의의 mysite_nplusone 예제이고 현재 board-practice에 이 사용자 모델을 추가했다는 뜻은 아니다.

POST /nplusone/setup을 본문 없이 호출하면 실습 데이터 사용자 5명·게시글 25개·댓글 125개를 준비한다. 재호출은 해당 실습 데이터를 삭제한 뒤 다시 만드는 동작이다. 각 게시글에는 사용자 5명의 댓글이 하나씩 있다. 아래 수치는 이 데이터의 예상값이다.

| GET 경로의 /nplusone 뒤 부분 | 내용·전략 | 예상 SQL 수 |
|---|---|---|
| /posts/with-user | 작성자 지연 로딩 | 1+5=6 |
| /posts/with-user/optimized | 작성자 joinedload | 1 |
| /posts/with-comments | 댓글 지연 로딩 | 1+25=26 |
| /posts/with-comments/optimized | 댓글 selectinload | 2 |
| /posts/with-comments/joined | 댓글 joinedload·unique | 1 |
| /posts/full | 게시글 작성자·댓글 지연 로딩 | 1+5+25=31 |
| /posts/full/optimized | 작성자 JOIN·댓글 IN | 2 |
| /posts/with-comment-users | 댓글·댓글 작성자 지연 로딩 | 1+25+5=31 |
| /posts/with-comment-users/optimized | 댓글 IN·댓글 작성자 JOIN | 2 |

사용자 조회가 매 게시글·매 댓글마다 반복되지 않고 5회인 것은 같은 사용자 객체를 Session이 재사용하기 때문이다. 두 최적화 API가 모두 SQL 2회여도 JOIN 대상이 다르다. full은 게시글 작성자, with-comment-users는 댓글 작성자를 연결한다.

응답의 query_count는 SQL 횟수, queries는 실제 구문 목록, posts는 결과 데이터이다. 계측은 ORM 조회뿐 아니라 응답 DTO 변환 중 발생한 조회도 포함한다. echo=True는 별도로 터미널 로그를 보여주는 Engine 설정이며 이 응답 계측에 필수는 아니다. 개선 전후 posts의 내용이 같은지, 반복 SELECT가 JOIN·IN으로 바뀌었는지 함께 확인한다.

> ➕ **더 알아두기**
> 이 TIL 작성에서는 강의의 예상 수치를 읽어 정리했으며 setup이나 실제 N+1 API를 실행하지 않았다. 동작 중인 사용자 DB에 실습 데이터를 초기화하지 않았다. 현재 게시판의 전체 목록 응답은 댓글·태그를 포함하지 않아 그 관계를 통한 목록 N+1은 아직 같은 방식으로 발생하지 않는다.

## 정리

지연·즉시 로딩은 프런트엔드의 두 방법이 아니라 백엔드에서 관계를 언제 읽을지 정하는 전략이다. 필요한 관계만 효율적으로 준비하고, 횟수·전송량·응답 내용을 함께 비교한다. 관계 접근과 DTO 변환이 추가 조회의 시점이 될 수 있다.

## ✅ 확인 질문

1. N+1에서 N은 무엇이며 게시글 10개와 댓글 조회는 어떻게 11회가 되는가?
2. get_db의 yield와 ORM 지연 로딩의 진행 계기는 어떻게 다른가?
3. 댓글 버튼의 HTTP 요청과 post.comments 접근의 DB 조회를 서로 다른 단계로 설명할 수 있는가?
4. 지연 로딩에서 미리 정의된 관계와 실제로 로딩된 객체는 어떻게 다른가?
5. 즉시 로딩이 메모리 절약이나 SQL 한 번을 반드시 뜻하지 않는 이유는 무엇인가?
6. 댓글을 사용하지 않는 화면과 반드시 사용하는 목록에서 어떤 전략을 고려할 것인가?
7. 컬렉션 joinedload의 unique는 왜 필요하며 DB 행 수를 줄이는가?
8. selectinload의 SQL 수가 반드시 두 번으로 고정되지 않는 경우는 무엇인가?
9. 일반 join으로 필터했다고 관계 객체가 미리 준비되지 않을 수 있는 이유는 무엇인가?
10. 게시글 작성자와 댓글 작성자를 미리 읽는 두 옵션 경로는 어떻게 다른가?
11. 사용자 5명·게시글 25개의 작성자 지연 로딩 조회가 26회가 아닌 6회일 수 있는 이유는 무엇인가?
12. SQL 수뿐 아니라 어떤 항목을 비교해야 최적화가 정확한지 판단할 수 있는가?
