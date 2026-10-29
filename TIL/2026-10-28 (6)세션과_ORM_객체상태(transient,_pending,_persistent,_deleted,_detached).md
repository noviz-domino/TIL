---
tags: [sqlalchemy, session, object-state, transaction, dto]
til: v2 2026-10-28
---

# 2026-10-28_세션과_ORM_객체상태(transient,_pending,_persistent,_deleted,_detached)
> 작성일: 2026-10-28

## 🔗 관련 글

- [Session·flush·commit 선행 질문](2026-10-27_SQLAlchemy%EC%99%80_PostgreSQL_%EA%B2%8C%EC%8B%9C%ED%8C%90_%EC%8B%A4%EC%8A%B5%28Engine%2C_Session%2C_CRUD%2C_Pydantic%29.md) — 반복 질문을 ORM 상태 이름에 연결한다.
- [지연·즉시 로딩](2026-10-28%20%283%29N%2B1%EA%B3%BC_%EB%A1%9C%EB%94%A9%EC%A0%84%EB%9E%B5%28Lazy%2C_Eager%2C_joinedload%2C_selectinload%29.md) — DTO 변환과 미조회 관계의 접근을 비교한다.
- [인증 의존성의 DTO 변환](2026-10-28%20%284%29%EC%BF%A0%ED%82%A4%EC%99%80_%EC%9D%B8%EC%A6%9D%28JWT%2C_%ED%9A%8C%EC%9B%90%EA%B0%80%EC%9E%85%2C_%EB%A1%9C%EA%B7%B8%EC%9D%B8%2C_Bearer%2C_%EC%9D%B8%EC%A6%9D%EC%9D%98%EC%A1%B4%EC%84%B1%29.md) — 열린 Session 안에서 값 변환이 필요한 사례이다.

## ❓ 아직 모르겠는 것

이 파트는 [세션과 ORM 객체 상태](https://ucanlabs.kr/classroom/aicampus-agent-python/lesson/1464/)를 이번 요청으로 읽고, 앞의 대화에서 반복 질문했던 지점을 상태 이름과 연결해 정리했다. 해결한 질문도 복습 대상으로 남긴다.

- Session은 SQL 파일·트랜잭션·스택·힙인가? 안에는 무엇이 있는가? (주제 1)
- add 후와 flush 후는 무엇이 달라지고, 실행했다면 왜 commit을 또 하는가? (주제 2)
- 조회한 객체를 수정하면 왜 다시 add하지 않아도 되는가? (주제 3)
- 관계 속성에 객체를 추가하면 왜 댓글 번호·외래키가 반영되는가? (주제 4)
- delete가 곧바로 SQL DELETE 실행인가? commit 후에도 객체는 남는가? (주제 5)
- DTO 변환은 JSON 검사뿐인데 왜 SELECT가 추가로 일어날 수 있는가? (주제 6)
- begin이 끝나는 것과 Session이 닫히는 것은 같은가? (주제 6)
- refresh는 왜 필요하며 commit이 이미 있는 제목·id를 바꾸는가? (주제 7)

## 주제 1: 강의의 전제와 상태 표

강의는 **expire_on_commit=False인 동기 Session**을 전제로 한다. 모델 관계는 mysite 예시이고 각 코드 블록은 독립 상황이다. import·DB 연결 설정을 생략한 설명용 코드이므로 현재 모델에 그대로 붙여 실행하는 완성 스크립트가 아니다. 예를 들어 현재 Post는 content도 필수이므로 title만 있는 원문 객체 생성은 그대로 실습 파일에 복사하면 안 된다.

Session은 ORM 객체·상태를 관리하고 DB 연결·트랜잭션을 사용하는 도구이다. Engine은 접속·연결 풀, metadata는 테이블 구조, Session은 현재 객체와 작업 상태이다. Session 안에 DB 전체가 복사되거나 특정 테이블 하나만 고정되는 것이 아니다. 같은 객체를 지역변수와 Session이 함께 참조할 수 있다.

| 상태 | 의미 |
|---|---|
| 비영속 transient | 새 객체이며 아직 Session에 등록하지 않음 |
| 대기 pending | Session에 등록했으나 INSERT 전 |
| 영속 persistent | SELECT로 읽거나 INSERT 실행 뒤 Session이 관리함 |
| 삭제 deleted | DELETE SQL 실행 후 아직 commit 전 |
| 분리·준영속 detached | DB 식별 정보는 있으나 Session에서 분리되어 관리받지 않음 |

**영속은 commit 완료와 같은 뜻이 아니다.** 미확정 INSERT 후에도 persistent일 수 있다. 객체의 관리 상태와 DB 트랜잭션 확정은 별도 축이다. Session 하나가 여러 객체·여러 순차 트랜잭션을 관리할 수 있다. commit한다고 Session을 닫는 것은 아니다.

> ♻️ 대화에서 ‘실행은 flush, 확정은 commit’이라는 구분을 이해했지만 이를 ‘Session은 곧 트랜잭션’으로 일반화해서는 안 된다. 트랜잭션은 DB 작업 묶음이고 객체 상태는 Session과 ORM 객체의 관계를 나타낸다.

## 주제 2: 새 객체 add·flush·commit

원문 예제는 다음과 같다.

```python
with SessionLocal() as db:
    with db.begin():
        post = Post(title="첫 번째 글")  # 비영속: 세션에 등록되지 않은 새 객체
        db.add(post)  # 대기: 세션에 등록, 아직 INSERT하지 않음
        db.flush()  # INSERT 실행 → 영속: 세션이 관리하는 객체
    # commit: 저장 확정, post는 영속 유지
# 세션 종료: post는 분리 상태(detached, 준영속)
```

add는 실제 SQL 문장을 metadata에 쌓는 작업이 아니라 새 객체를 pending으로 등록한다. flush는 매핑과 객체 상태로 INSERT를 실행해 persistent로 전환한다. begin 정상 종료는 필요한 flush 후 commit하고, 예외 경로에서는 rollback한다. Session 바깥 with가 끝나면 관리 관계가 끊어져 detached가 된다.

flush를 별도로 쓰지 않아도 commit이 필요한 flush를 수행한다. 이번 실습에서 명시한 flush는 DB가 생성한 번호를 commit 전에 출력하기 위해서였다. 첫 id가 1이라면 flush 후 1, commit 전 1, commit 후에도 1이다. commit이 번호를 변경하는 단계는 아니다. 자동 번호 시퀀스는 실패·삭제 이력으로 건너뛸 수 있다.

> 📝 **기사 시험 포인트** (정보처리기사)
> 트랜잭션의 원자성은 여러 변경을 함께 확정·취소하는 성질이다. flush로 INSERT를 실행했어도 commit 전이면 그 행 변경을 rollback할 수 있다. 지속성은 성공적으로 확정된 변경이 유지되는 성질이다. ORM의 persistent와 DB의 durability는 이름이 비슷해도 같은 판정이 아니다.

## 주제 3: 조회 객체 변경과 더티 체킹

```python
with SessionLocal() as db:
    with db.begin():
        post = db.get(Post, post_id)  # SELECT → 영속
        post.title = "수정된 제목"  # 영속 유지, 변경 사항 추적
        db.flush()  # UPDATE 실행, 영속 유지
    # commit: 수정 확정, post는 영속 유지
# 세션 종료: post는 분리 상태
```

get은 Session 생성 함수가 아니라 기본키 조회 메서드이다. db는 이미 받은 Session, Post는 조회 대상 매핑, post_id는 그 모델의 기본키이다. 존재하지 않으면 None이므로 위 설명 예제는 대상이 있다고 가정한다. 실제 API는 없는 객체를 먼저 검사한다.

조회된 관리 객체의 title만 바꾸면 Session이 변경을 추적한다. 대입 즉시 UPDATE하지 않고 flush에 반영한다. 다시 add하지 않아도 되고 상태는 계속 persistent이다. 변경된 객체는 dirty 추적 대상일 수 있지만 dirty를 이 표의 별도 생명주기 상태로 혼동하지 않는다.

## 주제 4: 관계 append와 save-update cascade

```python
with SessionLocal() as db:
    with db.begin():
        post = db.get(Post, post_id)  # post는 영속, comments는 미조회
        comment = Comment(content="좋은 글입니다")  # comment는 비영속
        post.comments.append(comment)  # 댓글 목록 SELECT 후 추가, comment는 대기
        db.flush()  # 댓글 INSERT → comment도 영속
    # commit: 두 객체 모두 영속 유지
# 세션 종료: 두 객체 모두 분리 상태
```

미조회 post.comments에 접근하면 목록을 조회할 수 있다. 관계의 save-update cascade가 새 댓글도 Session에 등록하므로 별도의 db.add(comment)가 필요 없는 방식이다. flush는 관계를 기준으로 comment.post_id를 설정하고 댓글 INSERT를 실행한다. 관계 연결·관리 등록·SQL 실행·확정이라는 단계를 구분한다.

> ➕ **더 알아두기**
> 앞 실습은 Comment(content=data.content, post=post)를 만든 뒤 명시적으로 add했다. 그 표현에서 오른쪽 post는 조회한 ORM 객체이지 Session이 아니다. 관계 속성을 지정하는 방법과 부모 컬렉션 append는 모두 관계를 구성하지만, 어떤 객체를 Session에 명시 등록했는지까지 함께 봐야 한다. 양방향 속성 동기화와 Session 등록을 모든 상황에서 동일한 것으로 일반화하지 않는다.

## 주제 5: 삭제 대상으로 지정한 것과 deleted 상태

```python
with SessionLocal() as db:
    with db.begin():
        comment = db.get(Comment, comment_id)  # 영속
        db.delete(comment)  # 삭제 대상으로 지정, 아직 영속
        db.flush()  # DELETE 실행 → 삭제 상태
    # commit: 삭제 확정 → comment는 분리 상태
```

delete 호출 시에는 삭제 예정이지만 아직 persistent이다. DELETE SQL이 flush로 실행된 뒤 deleted가 된다. commit으로 행 삭제를 확정하면 객체는 detached가 된다. Python 변수가 참조하는 객체 자체가 즉시 없어지는 것은 아니고 DB 행의 존재와 별개이다.

삭제 전·후 상태와 연결 자식의 cascade를 구분한다. 직접 SQL 대량 삭제까지 모든 ORM 객체 cascade가 동일하게 적용되는 것으로 해석하지 않는다. 현재 실습은 Session.delete에 ORM 객체를 전달하는 방식이다.

## 주제 6: begin 밖의 DTO 변환과 닫힌 Session

PostDTO에 댓글 목록이 있고 게시글·댓글 DTO 모두 from_attributes=True라고 가정한다. 이 설정은 전달된 객체의 속성을 읽도록 하므로 미조회 관계 접근이 SQL을 발생시킬 수 있다.

### 게시글만 조회하고 트랜잭션을 끝낸 경우

```python
with SessionLocal() as db:
    with db.begin():
        post = db.get(Post, post_id)  # post는 영속, comments는 미조회
    # commit: post는 영속 유지, comments는 여전히 미조회

    dto = PostDTO.model_validate(post)  # comments 접근 → 댓글 SELECT → 댓글들도 영속
# 세션 종료: ORM 객체들은 분리 상태, dto의 값은 유지
```

begin은 끝났지만 Session은 열려 있다. ORM 객체는 persistent이고 comments는 미조회이므로 DTO 변환에서 새 트랜잭션을 통해 댓글을 읽을 수 있다. ‘commit 완료=Session 종료=분리’가 아니다.

### 댓글까지 미리 조회한 경우

```python
with SessionLocal() as db:
    with db.begin():
        post = db.scalar(
            select(Post)
            .where(Post.id == post_id)
            .options(selectinload(Post.comments))
        )  # 게시글 SELECT + 댓글 SELECT → 모두 영속
    # commit: 두 객체 모두 영속 유지, 읽은 값도 유지

    dto = PostDTO.model_validate(post)  # 이미 읽은 값 사용, 추가 SELECT 없음
# 세션 종료: ORM 객체들은 분리 상태, dto의 값은 유지
```

expire_on_commit=False이므로 이미 준비한 게시글·댓글 값을 commit 후에도 유지한다. DTO 변환은 그 값을 읽어 추가 조회하지 않는다. 필요한 관계를 미리 준비하면 응답 직렬화의 DB 접근을 예측할 수 있다.

### Session까지 닫은 경우

```python
with SessionLocal() as db:
    post = db.get(Post, post_id)  # 영속, comments는 미조회
# 세션 종료: post는 분리 상태, 관리하는 세션이 없음

dto = PostDTO.model_validate(post)  # comments를 조회할 수 없어 ValidationError
```

이미 읽힌 title은 사용할 수 있지만 미조회 comments를 로딩할 Session이 없다. 관계 접근의 DetachedInstanceError가 Pydantic의 ValidationError 안에 담긴다. DTO 변환을 무조건 Session 종료 뒤에 두면 안 된다. 실제로 읽을 값·만료 여부·관계 로딩 상태·Session 수명을 함께 본다.

## 주제 7: 현재 실습의 기본 만료 설정과 refresh

현재 database.py는 SessionLocal = sessionmaker(bind=engine)이며 expire_on_commit 옵션을 명시하지 않았다. 기본 True에서는 commit 후 ORM 속성 값이 만료될 수 있다. 객체는 Session이 열려 있으면 여전히 persistent이지만, 속성 접근이 다시 SELECT를 발생시킬 수 있다. 이 강의 예제의 False 전제와 혼합하지 않는다.

refresh는 DB SELECT로 기존 객체 값을 다시 채운다. commit은 트랜잭션 확정이고 refresh는 재조회이므로 서로 다르다. 제목은 객체 생성·대입에서 이미 정해진 값이고 refresh가 같은 값을 읽으면 바뀌지 않는다. DB 기본값·트리거 등이 만든 값을 명시적으로 읽어야 할 때 쓸 수 있으며 단순 저장마다 무조건 필요한 단계는 아니다.

| 경우 | 객체의 Session 관계 | 값 접근 |
|---|---|---|
| commit 전 flush 완료 | persistent일 수 있음 | 현재 값 사용, 행 변경은 아직 취소 가능 |
| commit 후 Session 열림·False | persistent | 이미 읽은 값 유지, 미조회 관계는 조회 필요 |
| commit 후 Session 열림·True | persistent | 만료된 값 접근 시 재조회 가능 |
| Session 닫힘 | detached | 이미 로딩되어 남은 값은 접근 가능할 수 있으나 추가 조회 불가 |
| DTO로 값 복사 완료 | 별도 Pydantic 객체 | 필요한 값은 DTO에 보관, ORM Session 관리와 별개 |

> ➕ **더 알아두기**
> Session은 DB 전체나 외부 변경을 감시하지 않는다. 다른 프로그램이 테이블 구조를 바꾸거나 데이터를 바꿨다고 자동 알림을 받지 않는다. 직접 SQL 방식도 트랜잭션을 사용할 수 있다. Session의 추가 역할은 관리 객체·상태·관계 변경을 추적해 SQL 작업을 구성하는 것이다. 순서를 생각할 때 스택·힙보다 관계 의존성과 작업 묶음을 본다.

## 정리

객체 상태와 트랜잭션 확정, 로딩·만료, Session 열림·닫힘을 각각 구분한다. ‘지금 Session이 이 객체를 관리하는가, 필요한 속성을 이미 읽었는가’를 먼저 묻는다. 원문은 False 전제이고 실제 실습은 기본 True이므로 commit 후 접근 결과가 달라질 수 있다.

## ✅ 확인 질문

1. persistent 상태가 DB 저장 확정과 같은 뜻이 아닌 이유는 무엇인가?
2. Session 생성·DB 연결 확보·트랜잭션 시작·Session 종료는 어떻게 구분하는가?
3. 새 객체의 transient→pending→persistent는 어떤 호출에서 바뀌는가?
4. 첫 id가 1일 때 flush·commit 전후 번호와 rollback 가능 여부는 어떻게 다른가?
5. 조회한 객체의 제목을 바꿔도 상태는 persistent인 이유는 무엇인가?
6. 직접 SQL 방식의 commit만으로 Python 객체 변경을 자동 반영할 수 없는 이유는 무엇인가?
7. 부모의 comments에 새 객체를 append하면 어떤 cascade가 관리 등록에 관여하는가?
8. 관계를 연결한 순간과 flush 때 외래키가 저장되는 순간은 어떻게 다른가?
9. delete 호출 직후와 flush 직후 객체 상태는 어떻게 다른가?
10. 행 삭제를 commit한 후에도 Python 객체가 남는다는 것은 무엇을 뜻하는가?
11. begin 블록 밖에서 DTO를 만들면서 새 SELECT가 가능한 조건은 무엇인가?
12. Session을 닫고 미조회 관계가 있는 객체를 model_validate하면 어떤 문제가 생기는가?
13. expire_on_commit=True와 False에서 commit 후 속성 접근은 어떻게 달라지는가?
14. refresh가 불필요한 경우와 명시적 재조회가 필요한 경우를 각각 설명할 수 있는가?
