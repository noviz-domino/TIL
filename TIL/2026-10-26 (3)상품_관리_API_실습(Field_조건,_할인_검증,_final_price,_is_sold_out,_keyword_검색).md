---
tags: [fastapi, layered-architecture, pydantic, field, practice]
til: v2 2026-10-26
---

# 2026-10-26 (3)상품 관리 API 실습(Field 조건, 할인 검증, final_price, is_sold_out, keyword 검색)
> 작성일: 2026-10-26

## 🔗 관련 글

- [레이어드 게시판 구현(mysite4, schemas, Repository, Service, Router, 요청 흐름)](2026-10-26%20(2)레이어드_게시판_구현(mysite4,_schemas,_Repository,_Service,_Router,_요청_흐름).md) — 이 실습의 틀이 되는 구조이다. 게시글 대신 상품으로 같은 Router → Service → Repository를 만든다.
- [관심사 분리와 레이어드 아키텍처(SoC, Router·Service·Repository, 계층별 역할)](2026-10-26%20(1)관심사_분리와_레이어드_아키텍처(SoC,_Router_Service_Repository,_계층별_역할).md) — 이 실습의 계층별 역할 표와 "할인 조건은 Service" 구분이 그 글의 상품 주문 예시와 이어진다.
- [Path·Query·Body 파라미터(경로 매개변수, 쿼리 파라미터, Query, 검색, 라우팅 순서)](2026-10-25%20(1)Path_Query_Body_파라미터(경로_매개변수,_쿼리_파라미터,_Query,_검색,_라우팅_순서).md) — 목록 조회의 `keyword`가 그 글의 쿼리 파라미터이다.
- [Validation(빈 문자열 검증, 422, 검증 순서, Field)](2026-10-25%20(3)Validation(빈_문자열_검증,_422,_검증_순서,_Field).md) — 이 실습의 `Field`와 422, "검증에 실패하면 데이터가 바뀌지 않는다"가 그 글에서 나왔다.

## ❓ 아직 모르겠는 것

이 글은 강의 레슨(레이어드 아키텍쳐를 활용한 RESTful API 제작 실습)의 문제 설명을 읽고 정리한 것이며, **풀이 코드는 이 글에 없다.** 실습 코드를 저장소에서 찾지 못했다. 대화 중 사용자가 직접 한 질문도 없었다. 아래는 사용자의 발언이 아니라 설계할 때 헷갈리기 쉬운 지점이다. 복습 때 직접 풀어 보며 확인한다.

- 어느 검증을 `Field`에 두고 어느 검증을 Service에 두는가? (주제 3, 아래 ➕ 참고)
- `final_price`와 `is_sold_out`은 저장하는 값인가, 응답할 때 계산하는 값인가? (주제 3, 아래 ➕ 참고)
- 수정(PUT)에서 `discount_price`와 `stock`을 생략하면 어떻게 되는가? (주제 2)

## 주제 1: 이 실습의 목표와 계층별 역할

상품 등록·목록 조회·상세 조회·수정 API를 **Router → Service → Repository** 구조로 구성한다. 데이터는 메모리의 리스트에 저장한다.

| 구성 요소 | 역할 |
|---|---|
| Pydantic 모델 | 요청·응답의 필드와 타입을 정의하고, `Field`로 입력값의 **길이·범위·기본값**을 설정한다. |
| Router | 요청을 받아 Service를 호출하고, `response_model`과 상태 코드를 지정한다. |
| Service | **할인 조건을 검사**하고, 상품의 **존재 여부를 확인**하며, 응답에 필요한 값을 **계산**한다. |
| Repository | 상품의 ID를 부여하고, 리스트에서 저장·조회·수정을 수행한다. |

## 주제 2: 상품 데이터와 요청 모델

### 상품 데이터

저장소에서 관리할 `Product` 객체는 다음 값을 가진다. ID는 저장할 때 **서버가 부여**한다.

| 필드 | 타입 | 의미 |
|---|---|---|
| `id` | int | 상품의 고유 식별자 |
| `name` | str | 상품명 |
| `price` | int | 할인 전 가격 |
| `discount_price` | int | 가격에서 차감할 할인 금액 |
| `stock` | int | 재고 수량 |
| `category` | str | 카테고리 |

### 요청 모델

`ProductCreate`와 `ProductUpdate`에 적용된 조건은 다음과 같다. 요청 JSON의 필드 이름도 표와 같다.

| 필드 | 타입 | 조건 | 기본값 |
|---|---|---|---|
| `name` | str | 2~50자 | 필수 |
| `price` | int | 100 이상 | 필수 |
| `discount_price` | int | 0 이상 | 0 |
| `stock` | int | 0 이상 | 10 |
| `category` | str | 2자 이상 | 필수 |

수정은 **상품 정보 전체를 교체**한다. `name`, `price`, `category`는 수정 요청에도 포함해야 하며, `discount_price`와 `stock`을 생략하면 **기본값을 적용**한다.

## 주제 3: 응답 모델과 Service의 처리 규칙

### 응답 모델

| 모델 | 포함할 필드 |
|---|---|
| `ProductListResponse` | `id: int`, `name: str`, `final_price: int`, `category: str` |
| `ProductDetailResponse` | 목록 응답의 필드와 `stock: int`, `is_sold_out: bool` |

`final_price`와 `is_sold_out`은 **Service가 계산하여** 응답 데이터에 포함한다.

### Service의 처리 규칙

- 등록·수정 시 `discount_price`는 `price`보다 **작아야** 한다. 조건을 만족하지 않으면 `HTTPException`으로 **422**를 반환한다.
- 상세 조회·수정 시 대상 상품이 없으면 **404**를 반환한다.
- `final_price`는 `price - discount_price`로 계산한다.
- `is_sold_out`은 `stock == 0`이면 `True`, 그 외에는 `False`로 계산한다.
- **검증을 통과한 경우에만** 저장하거나 수정한다. 검증에 실패하면 기존 데이터는 변경되지 않아야 한다.

등록·수정·조회 응답에 계산된 값을 반영한다. 가격, 할인 금액, 재고가 바뀌면 응답 값에도 **변경된 데이터**가 사용되어야 한다.

## 주제 4: API 엔드포인트

| 메서드 | 경로 | 요청 | 성공 상태 코드 | 응답 모델 |
|---|---|---|---|---|
| POST | `/products` | `ProductCreate` | 201 | `ProductDetailResponse` |
| GET | `/products` | 선택 쿼리 파라미터 `keyword` | 200 | `list[ProductListResponse]` |
| GET | `/products/{product_id}` | `product_id: int` | 200 | `ProductDetailResponse` |
| PUT | `/products/{product_id}` | `product_id: int`, `ProductUpdate` | 200 | `ProductDetailResponse` |

### 목록 조회와 검색

목록 조회에 사용하는 쿼리 파라미터는 다음과 같다.

| 매개변수 | 선언 | 동작 |
|---|---|---|
| `keyword` | `keyword: str \| None = None` | 상품명에 검색어가 포함된 상품 조회 |

- `keyword`를 생략하거나 빈 문자열로 보내면 **전체 목록**을 반환한다. 검색 결과가 없으면 **빈 리스트**를 반환한다.
- 예를 들어 `GET /products?keyword=키보드`는 상품명에 `키보드`가 포함된 상품을 조회한다.

## 주제 5: 확인용 데이터

### 정상 등록 데이터

```json
{
  "name": "기계식 키보드",
  "price": 120000,
  "discount_price": 20000,
  "stock": 50,
  "category": "전자제품"
}
```

201 응답에서 `final_price`는 100000, `is_sold_out`은 `false`인지 확인한다.

### 기본값 확인 데이터

```json
{
  "name": "심플 에코백",
  "price": 15000,
  "category": "패션잡화"
}
```

할인 금액은 0, 재고는 10이 적용된다. 응답의 `final_price`는 15000, `stock`은 10, `is_sold_out`은 `false`여야 한다.

### 품절 상품 데이터

```json
{
  "name": "텀블러",
  "price": 25000,
  "discount_price": 2500,
  "stock": 0,
  "category": "생활용품"
}
```

재고가 0인 상품도 등록할 수 있다. 응답의 `final_price`는 22500, `is_sold_out`은 `true`여야 한다.

## 주제 6: 검증 및 수정 확인

정상 등록 데이터를 기준으로 아래 조건을 각각 적용한다. 수정 요청에는 생성 응답에서 받은 ID를 사용한다.

| 확인할 조건 | 기대 결과 |
|---|---|
| `name`을 "A"로 변경하여 등록 | 필드 검증 실패, 422 |
| `price`를 99로 변경하여 등록 | 필드 검증 실패, 422 |
| `stock`을 -1로 변경하여 등록 | 필드 검증 실패, 422 |
| `discount_price`를 -1로 변경하여 등록 | 필드 검증 실패, 422 |
| `discount_price`를 120000으로 변경하여 등록·수정 | Service의 할인 조건 검사 실패, 422 |
| 수정 요청에서 `stock`을 0으로 변경 | 200, `is_sold_out`이 `true`로 변경 |
| 수정 요청에서 `discount_price`를 10000으로 변경 | 200, `final_price`가 110000으로 변경 |
| 목록 조회 | `id`, `name`, `final_price`, `category`만 반환 |
| 없는 ID로 상세 조회·정상 데이터로 수정 | 404 |
| 검증에 실패한 등록·수정 후 다시 조회 | 잘못된 데이터가 저장되거나 반영되지 않음 |

## 헷갈리기 쉬운 부분

> ➕ **더 알아두기**
> (이 실습의 요구사항을 이전 수업의 내용에 대응시켜 본 설명이며 수업에 나온 설명은 아니다.)
>
> **1. 어느 검증이 어디로 가는가**
>
> 레이어드 아키텍처 수업의 구분(Pydantic은 입력 형식·필드 조건, Service는 업무 규칙)을 이 실습에 적용하면 다음과 같다.
>
> | 확인할 조건 | 어디서 검증하나 | 이유 |
> |---|---|---|
> | `name` 2~50자, `category` 2자 이상 | Pydantic `Field` | 한 필드의 길이 조건이다. |
> | `price` 100 이상, `discount_price` 0 이상, `stock` 0 이상 | Pydantic `Field` | 한 필드의 범위 조건이다. |
> | `discount_price`가 `price`보다 작은가 | Service (레슨이 지정) | 두 필드를 비교하는 업무 규칙이다. |
> | 상품이 존재하는가 | Service | Repository 조회 결과를 보고 판단한다. |
>
> 두 필드를 비교하는 검증은 Pydantic에서도 처리할 수 있지만, 이 레슨은 할인 조건을 **Service에서 검사**한다고 정했다.
>
> **2. `Field`로 쓰는 모양 (Pydantic v2)**
>
> ```python
> from pydantic import BaseModel, Field
>
> class ProductCreate(BaseModel):
>     name: str = Field(min_length=2, max_length=50)
>     price: int = Field(ge=100)
>     discount_price: int = Field(default=0, ge=0)
>     stock: int = Field(default=10, ge=0)
>     category: str = Field(min_length=2)
> ```
>
> 기본값이 없는 `Field`는 필수이고 `default=`를 주면 생략할 수 있다. `ge`는 "이상"(greater than or equal)이다. 위는 표의 조건을 옮긴 예시이며, 실습 정답이 아니다.
>
> **3. 계산한 값은 저장하는가**
>
> 레슨은 `final_price`와 `is_sold_out`을 "Service가 계산하여 응답에 포함"한다고 했고, 저장되는 `Product` 필드 목록(`id`, `name`, `price`, `discount_price`, `stock`, `category`)에는 없다. 그래서 가격이나 재고가 바뀌면 다음 조회 때 **그 시점의 값으로 다시 계산**되는 구조이다.
>
> **4. 404와 422의 순서**
>
> 수정에서 없는 `product_id`와 검증 실패 데이터가 함께 오면, 필드 조건(`Field`)은 함수에 들어오기 전에 검사되어 422가 먼저 나온다. 할인 조건(Service)과 존재 확인 중 무엇을 먼저 할지는 직접 정해야 한다. 앞 수업의 게시글 코드는 검증을 먼저 하고 존재 확인을 했다.
>
> **5. 경로 이름이 `/{id}`가 아니라 `/{product_id}`**
>
> 경로의 `{product_id}`와 함수 매개변수 `product_id: int`의 **이름이 같아야** 경로 매개변수로 연결된다. 게시글 실습의 `{id}`와 같은 규칙이다.

## 정리

- 상품 관리 API는 게시글 레이어드 구조를 그대로 쓰되, 필드 조건은 `Field`로, 할인 조건·존재 확인·계산은 Service로, ID 부여·저장은 Repository로 나눈다.
- 요청 모델의 조건: `name` 2~50자, `price` 100 이상, `discount_price` 0 이상(기본 0), `stock` 0 이상(기본 10), `category` 2자 이상. 수정은 상품 정보 전체를 교체하며 생략한 `discount_price`·`stock`은 기본값이다.
- 응답 모델은 목록(`id`, `name`, `final_price`, `category`)과 상세(목록 필드에 `stock`, `is_sold_out` 추가)로 나뉜다. `final_price = price - discount_price`, `is_sold_out = (stock == 0)`이며 Service가 계산한다.
- `discount_price`가 `price` 이상이면 Service가 422, 상품이 없으면 404이다. 검증에 실패하면 데이터를 바꾸지 않는다.
- 엔드포인트는 POST·GET(목록, `keyword` 검색)·GET(단건)·PUT 네 개이고 성공 코드는 201·200·200·200이다.

## ✅ 확인 질문

1. 이 실습에서 Router, Service, Repository, Pydantic 모델은 각각 무엇을 맡나요?
2. `ProductCreate`의 다섯 필드에 걸린 조건과 기본값을 말해 보세요. 수정 요청에서 `discount_price`와 `stock`을 생략하면 어떻게 되나요?
3. 같은 "검증"인데 `price` 100 이상은 `Field`에서, `discount_price < price`는 Service에서 검사하는 이유를 설명해 보세요.
4. `final_price`와 `is_sold_out`은 어떻게 계산하고, 왜 Service에서 계산하나요? `stock`을 0으로 수정하면 응답은 어떻게 바뀌나요?
5. 목록 응답과 상세 응답은 필드가 어떻게 다르고, 왜 `ProductListResponse`와 `ProductDetailResponse`를 따로 두나요?
6. `GET /products?keyword=키보드`에서 `keyword`는 Path·Query·Body 중 무엇이고, 생략했을 때와 일치하는 상품이 없을 때 각각 무엇이 반환되나요?
7. 검증에 실패한 등록·수정 뒤에 다시 조회했을 때 어떤 상태여야 하나요? 그것을 지키려면 코드의 순서를 어떻게 해야 하나요?
8. 없는 `product_id`로 상세 조회나 수정을 하면 어떤 상태 코드가 나오고, 그 판단은 어느 계층이 하나요?
