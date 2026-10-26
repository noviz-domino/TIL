---
tags: [fastapi, layered-architecture, soc, router, service]
til: v2 2026-10-26
---

# 2026-10-26 (1)관심사 분리와 레이어드 아키텍처(SoC, Router·Service·Repository, 계층별 역할)
> 작성일: 2026-10-26

## 🔗 관련 글

- [Validation(빈 문자열 검증, 422, 검증 순서, Field)](2026-10-25%20(3)Validation(빈_문자열_검증,_422,_검증_순서,_Field).md) — 바로 앞 글이다. 그 글에서 한 함수 안에 몰려 있던 검증·저장 코드를 이 글에서 역할별로 나눈다.
- [상태 코드와 응답 모델(HTTPException, 201·204·404·422, response_model, 응답 DTO)](2026-10-25%20(2)상태_코드와_응답_모델(HTTPException,_201_204_404_422,_response_model,_응답_DTO).md) — 이 글의 Router가 지정하는 `response_model`, 성공 상태 코드와 Service가 던지는 `HTTPException`이 거기서 나왔다.
- [데이터베이스 설계(키, 무결성, 정규화, ERD 작성)](2026-10-07%20(5)데이터베이스_설계(키,_무결성,_정규화,_ERD_작성).md) — 데이터를 어떻게 저장할지 설계하는 글이다. 이 글의 Repository 계층이 나중에 그 DB를 다루는 자리가 된다.
- [레이어드 게시판 구현(mysite4, schemas, Repository, Service, Router, 요청 흐름)](2026-10-26%20(2)레이어드_게시판_구현(mysite4,_schemas,_Repository,_Service,_Router,_요청_흐름).md) — 이 글의 개념을 실제 코드로 옮긴 뒤 글이다.

## ❓ 아직 모르겠는 것

이 글은 강의 레슨(관심사 분리 및 레이어드 아키텍쳐)을 읽고 정리했고, 대화 중 사용자가 직접 한 질문은 없다. 아래는 사용자의 발언이 아니라, 함께 학습한 ChatGPT 정리본이 별도 항목으로 짚어 둔 헷갈리기 쉬운 지점이다. 복습 때 다시 확인한다.

- 호출 방향 `Router → Service → Repository` 화살표는 HTTP 요청이 세 번 일어난다는 뜻인가? (주제 3, 아래 ➕ 참고)
- Pydantic이 하는 검증과 Service가 하는 검증은 어떻게 나누는가? (주제 5)
- 계층을 나누면 모든 변경이 한 계층 안에서 끝나는가? (주제 6, 아래 ➕ 참고)

## 주제 1: 관심사 분리(Separation of Concerns, SoC)

**관심사 분리**는 하나의 프로그램을 각각의 독립된 관심사로 분리하는 설계 원칙이다. 프로그램의 각 부분이 **단일한 목적**을 가지도록 구성한다.

`mysite3/post_api.py`의 생성 코드는 요청 처리, 입력 검증, 객체 생성, 저장을 **한 함수에서** 수행한다.

```python
# 요청 경로와 응답 형식
@router.post(
    "",
    response_model=PostDetailResponse,
    status_code=status.HTTP_201_CREATED,
)
def create_post(post_data: PostCreate):
    global post_id

    # 입력 검증
    if post_data.title == "":
        raise HTTPException(
            status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
            detail="title을 입력하세요",
        )

    if post_data.content == "":
        raise HTTPException(
            status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
            detail="content를 입력하세요",
        )

    # ID 부여, 객체 생성, 저장
    post_id += 1
    new_post = Post(post_id, post_data.title, post_data.content)

    posts.append(new_post)

    return new_post
```

| 이 함수가 맡은 일 | 코드 |
|---|---|
| 요청 경로와 응답 형식 | 데코레이터(`@router.post(...)`) |
| 입력 검증 | 빈 문자열 검사와 `HTTPException` |
| ID 부여, 객체 생성, 저장 | `post_id += 1`, `Post(...)`, `posts.append(...)` |

## 주제 2: 레이어드 아키텍처(Layered Architecture)

FastAPI는 **특정 구조를 강제하지 않는** 자유로운 프레임워크이다.

레이어드 아키텍처는 관심사 분리 원칙을 적용하는 한 방법으로, 코드를 **역할별 계층(Layer)** 으로 나누어 구성한다. 게시글 API는 **Router, Service, Repository**로 역할이 나뉜다.

### Presentation Layer (표현 계층) — Router

- HTTP 요청과 응답을 담당한다.
- URL, HTTP 메서드, 응답 모델, 성공 상태 코드를 지정한다.
- Pydantic으로 필수 필드와 타입이 검증된 요청 데이터를 Service에 전달한다.
- 이 실습에서는 `APIRouter`에 등록한 함수들이 해당한다.

### Business Layer (비즈니스 계층) — Service

- 애플리케이션의 규칙을 확인하고, 하나의 작업을 완료하는 **처리 흐름**을 담당한다.
- 필요한 데이터를 Repository에 요청하고, 그 결과에 따라 작업을 계속하거나 중단한다.
- 작업에 따라 여러 Repository의 기능을 조합할 수 있다.
- 이 실습에서는 `PostService`가 해당한다.

### Data Access Layer (데이터 접근 계층) — Repository

- 저장소에서 데이터를 **저장, 조회, 수정, 삭제**한다.
- 이 실습에서는 `PostRepository`가 리스트와 ID 카운터를 관리한다.
- 데이터베이스를 사용하면 **이 계층에서 SQL이나 ORM으로 데이터를 처리**한다.

| 계층 | 이 실습의 이름 | 한 줄 역할 |
|---|---|---|
| Presentation | Router (`APIRouter` 함수) | HTTP 요청·응답 |
| Business | `PostService` | 규칙 확인과 작업 흐름 |
| Data Access | `PostRepository` | 데이터 저장·조회·수정·삭제 |

> 📝 **기사 시험 포인트** (정보처리기사)
> 소프트웨어 아키텍처 패턴 중 **계층(Layers) 패턴**은 시스템을 층으로 나누고 각 층이 바로 아래 층의 서비스를 이용하게 하는 구조이다. 좋은 모듈 설계의 기준으로 **높은 응집도(Cohesion)와 낮은 결합도(Coupling)** 를 묻는 문제가 있으며, 각 계층이 하나의 역할에 집중하는 것이 응집도를 높이고 계층 사이의 의존을 줄이는 것이 결합도를 낮추는 것에 해당한다. (수업에서 "관심사 분리"라고 부른 원칙을 시험 용어로 풀어 본 것이며, 시험에서 이 용어를 그대로 묻는다는 뜻은 아니다.)

## 주제 3: 요청과 응답 흐름

```text
클라이언트 ── HTTP 요청 ──▶ Router ── 작업 호출 ──▶ Service ── 저장·조회 요청 ──▶ Repository
클라이언트 ◀── HTTP 응답 ── Router ◀── 작업 결과 반환 ── Service ◀── 처리 결과 반환 ── Repository
```

- 이 실습의 호출 방향은 **Router → Service → Repository**이다. Router가 Service의 메서드를 호출하고, Service는 데이터 처리가 필요하면 Repository의 메서드를 호출한다.
- 정상 처리 결과는 `return`을 통해 **Repository → Service → Router** 순서로 전달된다.
- 그림에서 결과가 돌아오는 화살표는 **반환**을 나타내며, Repository가 Service나 Router를 호출한다는 뜻은 아니다.

## 주제 4: Service가 담당하는 규칙과 처리 흐름

**비즈니스 로직**은 애플리케이션이 지켜야 할 규칙과 그 규칙을 적용하는 처리 절차이다. Service는 필요한 데이터를 조회하고, 작업의 실행 조건을 판단하며, 여러 기능을 연결하여 하나의 작업을 완성한다.

다음과 같은 규칙을 가진 서비스를 예로 들 수 있다.

| 작업 | 적용할 규칙 | Service의 처리 흐름 |
|---|---|---|
| 게시글 수정 | 게시글이 존재하고, 요청자가 작성자이며, 수정할 내용이 유효해야 한다. | 게시글 조회 → 존재·권한·내용 확인 → 수정 요청 |
| 상품 주문 | 재고가 충분해야 하며, 쿠폰은 사용 조건을 충족할 때만 적용한다. | 상품·쿠폰 조회 → 조건 확인 → 최종 금액 계산 → 주문 저장·재고 반영 |

`get_posts()`처럼 **추가 규칙이 없는 작업**은 Repository의 결과를 그대로 반환할 수 있다.

## 주제 5: Pydantic과 Service의 검증

요청 데이터가 올바른 형식이어도 실제로 작업을 진행할 수 있는지는 추가로 확인해야 한다. 검증의 역할은 다음과 같이 구분할 수 있다.

| 구분 | 주로 확인하는 내용 | 예시 |
|---|---|---|
| Pydantic | 요청 데이터의 **형식과 필드 조건** | 필수 값이 있는가? 제목의 길이가 조건에 맞는가? 주문 수량이 1 이상의 정수인가? |
| Service | 작업을 진행할 수 있는 **조건과 업무 규칙** | 수정할 게시글이 존재하는가? 요청자에게 수정 권한이 있는가? 주문 수량만큼 재고가 있는가? |

- 검증 위치는 **규칙의 성격에 따라** 정한다. 입력값끼리의 비교는 Pydantic에서도 처리할 수 있으며, 할인 조건처럼 작업의 규칙으로 관리할 검증은 Service에 둘 수도 있다.
- 현재 실습의 `PostCreate`는 필수 필드와 타입을 검사하고, Service는 제목과 내용이 빈 문자열인지 검사한다. 이 빈 문자열 조건은 Pydantic의 `Field`로도 설정할 수 있다.
- **게시글 존재 여부**는 Repository의 조회 결과로 판단한다. `PostRepository.find_by_id()`가 `None`을 반환하면 `PostService.get_post()`가 `HTTPException`을 발생시켜 작업을 중단하고, FastAPI가 404 오류 응답을 보낸다.

## 주제 6: 계층 분리의 장점과 변경 범위

### 장점

- **유지보수**: 요청 형식이나 저장 방식을 변경할 때 수정 범위를 줄일 수 있다.
- **테스트**: Router를 거치지 않고 Service나 Repository의 메서드를 직접 호출하여 확인할 수 있다.
- **협업**: 각 계층의 역할과 메서드를 기준으로 작업을 나눌 수 있다.

### 변경 사항과 수정할 계층

| 변경 사항 | 주로 수정할 계층 |
|---|---|
| API 경로나 성공 상태 코드 변경 | Router |
| 할인 조건이나 계산 방식 변경 | Service |
| 리스트 저장을 데이터베이스 저장으로 변경 | Repository |

저장 방식을 변경하더라도 **Repository의 메서드 이름, 매개변수, 반환 형태를 유지하면** 이를 호출하는 다른 계층의 변경을 줄일 수 있다.

## 헷갈리기 쉬운 부분

> ➕ **더 알아두기**
> (ChatGPT 정리본과 함께 학습한 내용이며 수업에 나온 설명은 아니다.)
>
> **1. 화살표는 HTTP 요청이 여러 번 일어난다는 뜻이 아니다**
>
> `Router → Service → Repository`는 서버 **내부의 일반적인 Python 메서드 호출**이다. HTTP 요청은 클라이언트에서 서버로 한 번 들어올 뿐이다. 결과는 `return`으로 호출한 쪽에 되돌아간다. Repository가 거꾸로 Service를 호출하는 것도 아니다.
>
> **2. 계층 분리가 모든 변경을 격리하지는 않는다**
>
> 계층 사이의 **데이터 계약**(주고받는 필드나 반환 형태)이나 트랜잭션 요구가 바뀌면 여러 계층을 함께 고쳐야 할 수 있다. 위 표는 "주로" 고칠 곳이다.
>
> **3. 계층이 늘면 비용도 있다**
>
> 계층 분리는 호출 단계를 늘리는 대신 **변경 이유를 나눈다**. Router는 HTTP 인터페이스, Service는 업무 정책, Repository는 저장 기술이 바뀔 때 고친다. 하지만 아주 작은 기능에 계층을 지나치게 많이 두면 오히려 불필요한 복잡도가 생길 수 있다.
>
> **4. Service가 `HTTPException`에 의존하는 것은 교육용 단순화**
>
> 이 수업의 Service는 FastAPI의 `HTTPException`을 직접 발생시킨다. 규모가 큰 애플리케이션에서는 Service가 프레임워크와 무관한 **도메인 예외**를 던지고, Router나 전역 예외 처리기가 그것을 HTTP 응답으로 바꾸는 설계도 쓴다.
>
> **5. "SoC"와 "레이어드 아키텍처"는 같은 말이 아니다**
>
> 관심사 분리는 **원칙**이고, 레이어드 아키텍처는 그 원칙을 적용하는 **한 가지 방법**이다. FastAPI는 특정 폴더 구조를 강제하지 않는다.

## 정리

- 관심사 분리(SoC)는 프로그램을 독립된 관심사로 나누어 각 부분이 단일한 목적을 갖게 하는 원칙이다. 한 함수가 요청 처리·검증·객체 생성·저장을 모두 하던 코드가 그 예(분리 전)이다.
- 레이어드 아키텍처는 SoC를 적용하는 한 방법으로 코드를 계층으로 나눈다. FastAPI는 구조를 강제하지 않는다.
- **Router(Presentation)**: HTTP 요청·응답, URL·메서드·응답 모델·성공 상태 코드, 검증된 요청 데이터를 Service에 전달한다.
- **Service(Business)**: 규칙 확인과 작업 흐름. 필요한 데이터를 Repository에 요청하고 결과에 따라 작업을 계속하거나 중단한다. 규칙이 없는 작업은 Repository 결과를 그대로 반환할 수 있다.
- **Repository(Data Access)**: 저장·조회·수정·삭제. DB를 쓰면 이 계층에서 SQL이나 ORM으로 처리한다.
- 호출은 Router → Service → Repository, 결과는 `return`으로 Repository → Service → Router → HTTP 응답이다.
- 검증은 Pydantic(형식·필드 조건)과 Service(작업 진행 조건·업무 규칙)로 성격에 따라 나눈다. 게시글 존재 여부는 Repository 결과로 Service가 판단하고, 없으면 `HTTPException`(404)이다.
- 변경 범위: 경로·성공 코드는 Router, 할인 조건은 Service, 리스트→DB는 Repository. Repository의 메서드 이름·매개변수·반환 형태를 유지하면 다른 계층 변경을 줄인다.

## ✅ 확인 질문

1. 관심사 분리(SoC)란 무엇이고, `create_post` 함수 하나가 하던 일 네 가지는 무엇이었나요? 이것이 왜 문제가 될 수 있나요?
2. 레이어드 아키텍처는 SoC와 어떤 관계인가요? FastAPI가 구조를 강제하지 않는다는 말은 무슨 뜻인가요?
3. Router, Service, Repository가 각각 맡는 일을 이 실습의 클래스·함수 이름과 함께 설명해 보세요.
4. 요청이 들어와서 응답이 나갈 때까지의 호출 방향과 결과가 돌아오는 방향을 설명해 보세요. 결과가 돌아오는 화살표는 Repository가 Service를 호출한다는 뜻인가요?
5. 게시글 수정과 상품 주문의 예에서 Service는 어떤 규칙을 어떤 순서로 확인하나요? `get_posts()`는 왜 Service가 추가로 하는 일이 거의 없나요?
6. Pydantic의 검증과 Service의 검증은 각각 무엇을 확인하나요? 같은 "빈 문자열 검사"를 어느 쪽에 둘 수 있고, 위치는 어떤 기준으로 정하나요?
7. "게시글이 존재하는가"는 어떤 계층의 결과로, 어떤 계층이 판단하며, 없으면 클라이언트는 무엇을 받나요?
8. 계층을 나누면 유지보수·테스트·협업에서 각각 무엇이 좋아지나요?
9. API 경로 변경, 할인 계산 방식 변경, 리스트 저장을 DB 저장으로 바꾸는 변경은 각각 어느 계층을 주로 고치나요? Repository 메서드의 이름·매개변수·반환 형태를 유지하는 것이 왜 중요한가요?
