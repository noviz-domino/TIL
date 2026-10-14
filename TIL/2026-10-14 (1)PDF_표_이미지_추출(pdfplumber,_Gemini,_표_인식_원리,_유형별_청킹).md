---
tags: [pdf, pdfplumber, table-extraction, gemini, chunking]
til: v2 2026-10-14
---

# 2026-10-14 (1)PDF 표·이미지 추출(pdfplumber, Gemini, 표 인식 원리, 유형별 청킹)
> 작성일: 2026-10-14

## 🔗 관련 글

- [Parent-Child RAG(위키 크롤링, section Parent, Child 청킹, ColBERT)](2026-10-13%20(1)Parent-Child_RAG(위키_크롤링,_section_Parent,_Child_청킹,_ColBERT).md) — 같은 RAG 전처리 흐름이다. 그 글은 위키 문서를 section 단위로 나누었고, 이 글은 PDF의 본문·표·이미지를 유형별로 나누어 청킹한다.
- [Gemini 멀티모달 입력과 LLM 이미지 생성 모델 구분](2026-08-27%20(1)%20Gemini%20멀티모달%20입력과%20LLM이미지생성모델%20구분.md) — 이 글의 이미지 읽기에서 Gemini에 이미지를 입력으로 넘기는 방식과 이어진다.

## ❓ 아직 모르겠는 것

수업 중 막히거나 의문을 보였던 부분이다. 대화에서 설명을 듣고 풀렸지만, 헷갈리기 쉬워서 복습 때 다시 확인한다.

- 표를 인식하지 못하는 이유가 무엇인가? PDF 안에는 마크다운 같은 표 형식의 글자가 들어 있는 것이 아닌가? (주제 2 참고)
- PDF 자체에 문제가 있는 것 아닌가? 이렇게 표를 못 읽는 형식이면 쓰면 안 되는 것 아닌가? (주제 2 참고)
- 글자는 텍스트로 있고 위치도 정확한데, 표의 선만 그림일 수 있는가? (주제 2 참고)
- 글자는 왜 그림이 아니라 문자 데이터로 저장되어 있는가? (주제 2 참고)
- 표를 이루는 선의 좌표는 PDF에 저장되어 있는가? (주제 2 참고)
- 디지털 PDF에서 표를 뽑는 것은 문서를 스캔한 것을 뽑는 것과 다를 게 없지 않은가? (주제 2 참고)
- 이미지를 읽는 방식은 OCR인가? 지금 하는 표·이미지 처리는 무엇이라고 부르는가? (주제 5 참고)
- 표 좌표와 글자 좌표를 알면 안에 무엇이 들었는지 알 수 있는데, 왜 이미지 인식을 한 번 더 거치는가? (주제 5 참고)
- 좌표가 있는데도 표 추출 정확도가 100%가 되지 않는 이유가 무엇인가? (주제 2 참고)
- 안 보이는 좌우 변은 PDF에 흰색 선으로 저장되어 있는가, 아예 그려져 있지 않은가? (주제 2 참고)
- 글자는 문자마다 좌표가 있는가, 한 좌표에서 쭉 늘어놓은 것인가? 문맥이 연결되어 있지 않은 글자를 라이브러리는 어떻게 단어·문장으로 묶는가? (주제 2 참고)
- 글꼴 정보도 PDF에 저장되는가? 문자마다 하나씩 있는가? (주제 2 참고)
- 선분 좌표는 표에만 쓰이는가? (주제 2 참고)
- 교차하는 선분과 같은 x의 꼭지점 규칙으로 사람처럼 표를 인식하면 되지 않는가? 변이 빠진 표나 가로선만 있는 표는 어떻게 하는가? (주제 3 참고)
- `load_dotenv()`는 `.env`를 지정하지 않아도 알아서 찾는가? (주제 1 참고)
- (일부 미해결) 노트북 PDF 제품표 영역에는 `lines` 42개, `rects` 7개(헤더 배경)가 있고 좌우 끝 근처에 세로선이 없다는 것까지 확인했다. 좌우 바깥 변이 흰색 선으로 저장된 것인지는 확인하지 못했다. (주제 2 참고)

## 주제 1: PDF 문서 처리의 전체 흐름

### 노트북 개요

노트북 `34-pdf-table-image-extraction.ipynb`의 제목은 "PDF 문서 처리 - 텍스트·표·이미지 추출"이다. PDF에서 추출한 표를 읽기 쉬운 Markdown으로 정리하고, 이미지의 내용을 Gemini로 읽는다. 본문·표·이미지에서 읽은 내용을 구분해 보관한 뒤, 유형에 맞게 RAG용 청크를 만든다.

**PDF 텍스트 추출 → 표·이미지 내용 정리 → 유형별로 구분 → 청킹**

### 환경설정

필요한 패키지를 설치한다.

```bash
pip install pdfplumber==0.11.10
```

> ➕ **더 알아두기** (수업 외 내용: 실습 중 난 오류)
> 노트북의 `load_dotenv(".env", override=True)` 다음 `ChatGoogleGenerativeAI(...)`에서 `ValidationError: API key required for Gemini Developer API`가 났다. 경로를 적으면 **현재 폴더의 `.env`만** 보고, 파일이 없어도 에러 없이 넘어간다. 실제 `.env`는 노트북 폴더(`langchain_`)가 아니라 `aim_home` 폴더에 있었다.
>
> | 호출 | 동작 |
> |---|---|
> | `load_dotenv()` | `.env`라는 이름의 파일을 현재 폴더부터 위로 올라가며 찾고, 처음 나오는 곳에서 멈춘다. |
> | `load_dotenv(".env")` | 현재 폴더의 `.env`만 본다. 없으면 조용히 넘어간다. |
> | `load_dotenv(r"C:\...\.env")` | 그 경로의 파일만 읽는다. |
>
> 하위 폴더에 별도 `.env`가 있으면 거기서 먼저 멈춰서 상위 `.env`의 키가 읽히지 않는다. 사용자가 `load_dotenv(override=True)`로 직접 고쳤다. 이 동작은 python-dotenv의 알려진 동작으로 설명했고, 설치된 버전에서 별도로 실험하지는 않았다.

### PDF 파일 준비

현재 작업 폴더의 `data` 폴더에 있는 `250204_태블릿+PC+품질비교시험_보도자료.pdf`를 사용한다.

### PDF 텍스트 추출

`PyPDFLoader`로 PDF에서 읽은 1페이지 본문과 제품표의 텍스트를 확인한다.

### 텍스트 추출의 한계

표의 글자는 추출되지만 열 이름과 값의 대응, 병합 셀 구조를 파악하기 어렵다.

표의 행·열 구조를 추출할 때는 `pdfplumber` 같은 PDF 처리 라이브러리를 활용할 수 있다.

## 주제 2: PDF에 데이터가 저장되는 방식과 표 인식이 어려운 이유

### PDF에 데이터가 저장되는 방식

PDF는 페이지를 화면에 표시하거나 인쇄할 수 있도록, 어떤 내용을 어디에 어떤 모습으로 그릴지 저장한다.

| 구성 요소 | 저장되는 정보 |
| --- | --- |
| 글자 | 표시할 문자, 글꼴과 크기, 배치 위치 |
| 선·도형 | 선의 시작점과 끝점, 도형의 위치와 크기, 색상 등 |
| 이미지 | 이미지 데이터와 페이지에 배치할 위치·크기 |

PDF 처리 라이브러리는 이 정보를 해석해 글자·선·이미지의 좌표와 영역을 구한다.

예를 들어 표의 제품명과 가격은 서로 다른 위치에 놓인 글자이고, 테두리는 별도의 선일 수 있다. 화면에서는 하나의 표로 보이지만, 행·열과 셀의 관계가 항상 함께 저장되는 것은 아니다. 따라서 표를 추출할 때는 글자의 배치와 선의 위치를 이용해 구조를 파악해야 한다.

스캔한 PDF는 페이지 전체가 이미지로 저장되기도 한다. 별도의 텍스트 정보가 없다면 OCR이나 비전 모델로 이미지 속 글자를 읽어야 한다.

> ♻️ 처음엔 PDF 안에 마크다운처럼 "표 형식으로 된 글자"가 들어 있어서 그것을 읽으면 된다고 알았으나, PDF에는 표라는 객체가 없고 글자·선·이미지를 어디에 그릴지만 좌표로 저장되어 있다. 행·열의 관계는 라이브러리가 좌표를 보고 추정해야 하는 것이 맞다.

> ♻️ 처음엔 표를 못 읽는 것이 PDF 쪽의 문제라서 쓰면 안 되는 형식이라고 보았으나, PDF는 어디서 열어도 똑같이 보이도록 화면·인쇄 모양을 고정하는 형식이다. 구조 정보가 없는 것은 결함이 아니라 설계이고, 구조를 추정하는 일은 추출하는 쪽의 몫이다.

> ➕ **더 알아두기** (수업 외 내용: 대화 중 보충)
>
> **글자와 선은 서로 연결되어 있지 않다.** 글자 하나하나의 위치는 정확히 알지만, 그 글자가 "같은 행의 두 번째 칸"이라는 정보는 PDF에 적혀 있지 않다. 칸을 나누는 선도 그냥 선분일 뿐이고 "이 선들이 모여 표를 이룬다"는 정보가 없다. 그래서 글자 좌표가 정확해도 표 구조는 추측해야 한다.
>
> **선의 좌표는 PDF에 있다.** 벡터로 저장된 선은 "(100,330)으로 이동 → (400,330)까지 선을 긋는다 → 그려라" 같은 그리기 명령이고, `pdfplumber`는 이를 `page.lines`, `page.rects`, `page.images`로 꺼내 준다. 표를 찾는 과정은 ① 선분 좌표를 모은다 → ② 가로선과 세로선의 교차점을 구한다 → ③ 교차점으로 만들어지는 칸을 표로 묶는다 → ④ 각 칸 안에 들어가는 글자를 좌표로 찾아 채우는 순서이다.
>
> **선의 좌표가 있어도 표를 못 찾을 수 있는 경우**는 다음과 같다.
>
> | 상황 | 이유 |
> |---|---|
> | 선이 끊겨 있거나 서로 어긋남 | 교차점이 만들어지지 않는다. |
> | 선이 `line`이 아니라 얇은 `rect`로 저장됨 | 설정에 따라 선으로 취급되지 않을 수 있다. |
> | 가로선만 있고 세로선이 없음 | 칸을 닫을 수 없다. |
> | 표 전체가 이미지로 들어 있음 | 선과 글자의 좌표 자체가 없다. |
>
> 선을 얇게 채운 사각형이나 이미지로 저장하는 이유는 문서를 만든 프로그램이 표 테두리를 어떻게 구현했느냐에 따라 다르다. 화면에서는 모두 똑같이 선으로 보인다.
>
> **글자가 문자 데이터로 저장되는 이유.** 글자를 그림이 아니라 문자 코드로 저장하면 ① 용량이 훨씬 작고, ② 폰트가 윤곽선(벡터)이라 확대해도 깨지지 않고, ③ 검색(Ctrl+F)·복사·스크린 리더가 가능하며, ④ 뷰어가 화면 해상도에 맞게 부드럽게 그려 준다. 그래서 PDF를 만드는 프로그램은 기본적으로 글자를 문자 데이터로 내보낸다. 글자가 그림이 되는 경우는 종이를 스캔한 PDF, 캡처 이미지를 붙인 경우, 일부러 글자를 이미지로 변환한 경우이다.
>
> **디지털 PDF와 스캔 PDF의 차이.** 둘 다 표 구조를 모른다는 점은 같지만 글자가 있느냐가 다르다.
>
> | | 디지털 PDF | 스캔 PDF |
> |---|---|---|
> | 글자 | 텍스트로 존재함(복사 가능) | 이미지뿐(복사 불가) |
> | 글자 정확도 | 원본 그대로 | OCR 품질에 따라 오인식 |
> | 표 구조 | 선과 좌표로 추측 | 선과 칸 모두 픽셀로 추측 |
> | 추출 방법 | `pdfplumber`, `PyMuPDF` | OCR 또는 멀티모달 LLM |
>
> 디지털 PDF도 표가 복잡하면 이미지로 렌더링해서 LLM에 넘기게 되는데, 그 단계부터는 스캔본 처리와 방식이 같아진다.
>
> **표 데이터를 쓰려면 원본이 가장 좋다.** 엑셀·CSV·DB·HTML 같은 원본을 구할 수 있으면 PDF에서 추출하지 말고 원본을 쓰는 것이 낫다. PDF밖에 없을 때 이 노트북 같은 추출 과정이 필요하다.

> ♻️ 처음엔 프로그램이 만든 표의 선은 끝이 어긋날 수 없다고 보았으나, 같은 프로그램이 만든 PDF에서도 선(`line`)과 사각형(`rect`)처럼 **서로 다른 객체**가 만나는 곳에서는 차이가 있다. 노트북 PDF에서 헤더 배경 사각형의 왼쪽 끝은 x=58.0이고 같은 위치의 선은 x=57.6에서 시작했다. 그래서 1px 정도의 오차 허용이 필요하다.

> ♻️ 처음엔 선을 투명하게 해도 셀이 남아 있어서 글자가 바깥으로 나오지 않으니 표 구조를 알 수 있다고 보았으나, "셀이 남아 있다"는 것은 표를 **만든 프로그램(Word, 한글 등) 안**의 이야기이다. PDF로 내보내면 셀 구조는 사라지고 글자·선의 좌표만 남는다. 다만 정상적으로 만든 표에서 글자가 가로선 범위를 넘지 않는 것은 대체로 맞아서 범위를 잡는 단서로는 쓸 수 있다.

> ➕ **더 알아두기** (수업 외 내용: 대화 중 직접 만든 테스트 PDF와 노트북 PDF로 확인한 사실)
>
> **글꼴·크기는 파일에 한 번 지정되고, 라이브러리가 글자마다 풀어 준다.** 직접 만든 PDF 안에는 `/F2 12 Tf`(글꼴과 크기 지정)가 한 번 있고, 그 뒤 `(Brand) Tj`, `(Name) Tj` 두 문자열이 같은 설정으로 그려졌다. `/F1 10 Tf`로 바꾼 뒤부터는 다른 글꼴이다. `pdfplumber`의 `page.chars`는 모든 글자에 `fontname`, `size`를 붙여 준다. Brand·Name은 `Helvetica-Bold 12.0`, Samsung·Galaxy S10은 `Helvetica 10.0`이었다. 노트북 PDF 헤더 글자는 `unGothicBold 12.0`이었다.
>
> **글자별 좌표는 파일에 없고 계산된 값이다.** 파일에는 문자열 하나와 시작 위치(`95 700 Td`)뿐이다. `pdfplumber`는 `B` 95.0~103.7, `r` 103.7~108.3처럼 글자마다 좌표를 만들어 준다. 폭은 글꼴의 글자 폭 정보에서 나온 것으로 보이며 확인하지는 못했다. 문자열 안의 공백(`Galaxy S10`)은 공백 문자로 존재했다.
>
> **단어와 줄은 의미가 아니라 거리 규칙으로 묶는다.** 글자 사이 간격이 허용값보다 크면 단어를 끊고, 세로 위치(`top`)가 비슷하면 같은 줄로 묶는다. 위에서 아래, 같은 줄이면 왼쪽에서 오른쪽 순서로 읽는다. 노트북 PDF에서 '구'→'분'은 간격 0(같은 단어), '분'→'브'는 간격 17.1(다른 단어)이었다. 허용값의 기본값(`x_tolerance` 등)은 확인하지 못했다.
>
> **이 규칙은 표에서 틀린다.** 노트북 PDF의 헤더 칸 "제품명(모델명)"은 세로 가운데 정렬 때문에 '제품명'(top 571.2)과 '(모델명)'(top 583.2)이 두 줄로 쪼개지고, 그 사이 높이(577.2)에 '구분 브랜드 화면크기 …'가 한 줄로 이어졌다. 그래서 표는 줄 묶기에 맡기지 않고 선 좌표로 칸을 먼저 만든 뒤 칸 안의 글자를 모은다. 칸 경계가 "같은 칸이면 한 덩어리"라는 기준을 준다. LLM은 이 좌표를 직접 받지 않고, 조립된 텍스트나 Markdown 표를 받는다.
>
> **선은 표 전용이 아니다.** 노트북 PDF 1페이지의 선+사각형은 1,782개인데 표 영역 안은 51개, 밖은 1,731개였다. 표 밖은 대부분(1,712개)이 너비·높이 1.5 미만의 작은 조각이었고 세로 위치 280~310 근처에 몰려 있었다. 용도(그래프 등)는 확인하지 못했다. 가로선 16개(길이 약 480인 것 포함)와 사각형 3개도 있었다. 이 PDF에서 `curves`는 0개, `images`는 2개였다. 선 좌표만으로는 표 테두리인지 밑줄인지 그림의 일부인지 구분되지 않으므로, `pdfplumber`는 선들이 **격자를 이루는 모양**으로 표를 추정한다.
>
> **노트북 PDF의 제품표.** 표 영역 안에 `lines` 42개, `rects` 7개가 있었다. 사각형 7개는 헤더 배경색(회색 0.898)이고, 세로선 x좌표는 104, 152, 256, 318, 377, 462였으며 표 좌우 끝(57.2, 538.9) 근처에는 세로선이 없었다. 노트북이 `explicit_vertical_lines`로 좌우를 보완하는 이유와 일치한다.
>
> **보이지 않는(흰색) 선과 아예 없는 선은 다르다.** 직접 만든 테스트 PDF에서 오른쪽 변(x=300)을 흰색(`1 1 1 RG`)으로 그렸더니 `page.lines`에 `stroking_color (1.0, 1.0, 1.0)`로 잡혔고, `find_tables`는 표 `(90, 80, 300, 140)`을 찾아 `[['Brand','Name'], ['Samsung','Galaxy S10']]`을 반환했다. 흰색 선을 빼고 선 5개만 그리면 표 영역이 `(90, 80, 190, 140)`이 되어 `[['Brand'], ['Samsung']]`만 나왔고 `Name` 열이 통째로 빠졌다. 노트북 PDF의 좌우 변이 흰색 선인지 아예 없는 것인지는 확인하지 못했다.
>
> **좌표가 있어도 정확도가 100%가 아닌 이유.** 좌표는 재료이고 표 구조는 그것으로 추측한 결과이기 때문이다. 병합 셀(칸이 여러 행에 걸쳐도 그 정보가 없다), 칸 경계에 걸친 글자, 한 칸 안에서 줄바꿈된 글자, 선이 없는 표, 표처럼 보이는 비표 영역에서 틀린다. "70%" 같은 수치는 출처를 알 수 없어 고정값으로 받아들이지 않는다. 정확도는 문서마다 크게 다르다.

## 주제 3: pdfplumber가 표를 인식하는 원리와 표 탐색

### pdfplumber가 표를 인식하는 원리

`pdfplumber`는 PDF에 저장된 선과 글자의 위치를 이용해 표의 행·열 구조를 추정한다.

**경계선 찾기 → 교차점 찾기 → 셀 구성 → 표로 묶기 → 글자 추출**

- 가로·세로선과 사각형 테두리를 찾아 셀의 경계를 추정한다.
- 선이 만나는 위치를 이용해 사각형 셀을 만들고, 연결된 셀들을 하나의 표로 묶는다.
- 각 셀 안에 있는 글자를 모아 행과 열의 순서에 맞게 정리한다.

선이 없는 표는 글자가 정렬된 위치를 기준으로 행·열을 추정할 수 있다. 이미지로만 된 표는 OCR이나 비전 모델을 이용한 처리가 필요하다.

> ➕ **더 알아두기** (수업 외 내용: 대화 중 보충)
> 기본 설정(`vertical_strategy`, `horizontal_strategy`가 `"lines"`)은 실제로 그려진 선을 찾아 격자를 만든다. 선 대신 글자 정렬로 열을 추정하게 하려면 `"text"` 전략을 쓰고, 선이 살짝 어긋난 경우에는 `snap_tolerance` 같은 허용 오차를 키우는 방법이 있다. 다만 이 노트북은 이 방법이 아니라 아래 주제 4의 `explicit_vertical_lines`로 경계를 보완한다.

> ➕ **더 알아두기** (수업 외 내용: 대화 중 나온 사용자 아이디어와 실험)
>
> **아이디어:** 교차하는 선분 덩어리를 한 표로 보고, 가로선들의 끝 x가 같으면(오차 1px 정도) 빠진 바깥 변이 있다고 추정하고, 가운데 세로선이 없으면 글자 간격으로 열을 나눈다. 바탕화면 `아이디어` 폴더의 `PDF_표추출_하이브리드_알고리즘.md`에 정리해 두었다.
>
> - 이 규칙은 `pdfplumber`의 원리(경계선 → 교차점 → 셀 → 표)와 거의 같다. 빠진 변 추정은 이 노트북이 `explicit_vertical_lines`로 사람이 하는 일을 자동화하는 것에 해당한다.
> - 가로선만 있는 표에는 교차가 없으므로 "교차하는 선분만 본다"는 규칙이 적용되지 않는다. 별도 기준이 필요하다.
> - 글자 간격만으로 열을 나누기는 애매할 수 있다. 노트북 PDF에서 같은 칸 안의 단어 간격은 6.0('펜' x1=480.6 → '가격' x0=486.6), 다른 칸 사이 간격은 12.4('본체가격*[원]' x1=456.2 → '펜' x0=468.6)였다.
> - 실험: 제품표에 `horizontal_strategy: lines` + `vertical_strategy: text`를 적용하니 3행 × 7열이었고 헤더 행이 사라졌으며 '1,180,000** 본' / '체 포함'처럼 "본체"가 잘렸다. 노트북 방식(`lines` + 좌우 경계 보완)은 5행 × 7열(헤더 포함, 정상 분리)이었다. 이유는 확인하지 못했다. 이는 `pdfplumber`의 `text` 구현의 결과이고, 아이디어 알고리즘을 직접 구현해 돌려 본 것이 아니다. 최소 테스트 PDF에서는 같은 설정으로 표가 잡히지 않았다.

### 표 탐색

PDF 전체에서 표 후보를 찾는다. 테두리로 둘러싸인 안내 영역도 표 후보로 잡힐 수 있으므로, 탐색 후 내용을 확인해 처리할 표를 선택한다.

영역 좌표(bounding box)는 표나 이미지가 차지하는 사각형의 범위다. 왼쪽·위쪽·오른쪽·아래쪽 경계를 이용해 필요한 부분만 잘라내거나, 추출한 내용의 원래 위치를 찾을 수 있다.

```python
# 표 후보마다 페이지 번호, 페이지 안의 표 번호, 영역 좌표를 저장한다.
table_candidates = []

with pdfplumber.open(pdf_path) as pdf:
    for page_no, page in enumerate(pdf.pages, start=1):
        # 각 페이지에서 기본 설정으로 표 후보를 탐지한다. 번호는 1부터 센다.
        tables = page.find_tables()
        for table_no, table in enumerate(tables, start=1):
            table_candidates.append({
                "page_no": page_no,
                "table_no": table_no,
                "bbox": table.bbox,  # (왼쪽, 위쪽, 오른쪽, 아래쪽) 좌표다.
            })
        if tables:
            print(f"{page_no}페이지: 표 후보 {len(tables)}개")

print(f"전체 표 후보: {len(table_candidates)}개")
```

### 표 인식 결과 시각화

1페이지 위에 감지한 선(빨강), 교차점(원), 표로 인식한 영역(파랑)을 표시한다. 코드는 `page.to_image(resolution=120).debug_tablefinder()`이다. 표가 왜 인식되지 않는지 눈으로 확인하는 디버깅용이다.

## 주제 4: 표 선택·내용 추출·Markdown 변환

### 처리할 표 선택

탐색 결과에서 1페이지의 두 번째 표인 하단 제품표를 선택한다(`TABLE_PAGE_NO = 1`, `TABLE_NO = 2`). 목록에 저장한 좌표로 표 영역을 잘라(`page.crop(table_bbox)`) 내용을 확인한다.

### 표 내용 추출

예제 표는 양쪽 바깥 세로선이 없어 표 영역의 좌우 좌표로 경계를 보완한다. 표 아래의 각주도 함께 가져온다.

> ➕ **더 알아두기** (수업 외 내용: 대화 중 보충)
> 이 노트북의 PDF가 주제 2에서 말한 "선이 불완전해서 표 인식이 어려운 표"의 실제 예이다. 양쪽 바깥 세로선이 없어서 `explicit_vertical_lines`에 표 영역의 왼쪽·오른쪽 좌표를 직접 넣어 보완한다.

```python
# 선택한 표에서 빠진 경계를 보완해 행과 열을 다시 추출한다.
table_settings = {
    "vertical_strategy": "lines",  # 세로선으로 열 경계를 찾는다.
    "horizontal_strategy": "lines",  # 가로선으로 행 경계를 찾는다.
    "explicit_vertical_lines": [table_bbox[0], table_bbox[2]],  # 빠진 좌우 경계를 보완한다.
}

with pdfplumber.open(pdf_path) as pdf:
    page = pdf.pages[TABLE_PAGE_NO - 1]
    region = page.crop(table_bbox)  # 선택한 표 영역만 잘라낸다.
    corrected_tables = region.find_tables(table_settings)  # 보완한 설정으로 표를 다시 찾는다.
    if not corrected_tables:
        raise ValueError("보완한 설정에서 표를 찾지 못했습니다.")
    corrected_rows = corrected_tables[0].extract()  # 셀 내용을 행별 리스트로 추출한다.

    # 표 아래에서 별표로 시작하는 각주 줄을 찾는다.
    footnote_lines = [
        line for line in page.extract_text_lines(return_chars=False)
        if line["top"] >= table_bbox[3] and line["text"].startswith("*")
    ]
    # 각주 텍스트는 표와 함께 보관하고 이후 표 청크에도 붙인다.
    footnotes = "\n".join(line["text"] for line in footnote_lines)
    # 마지막 각주까지 영역을 확장한다. 각주가 없으면 기존 하단 경계를 유지한다.
    table_with_footnotes_bbox = (
        table_bbox[0], table_bbox[1], table_bbox[2],
        max((line["bottom"] for line in footnote_lines), default=table_bbox[3]),
    )

pprint(corrected_rows)
print("\n각주:", footnotes)
```

### 셀 내용 정리

병합된 셀에서는 일부 행의 구분 값이 비어 있을 수 있다. 각 제품 행을 독립적으로 읽을 수 있도록 같은 범주의 값을 채우고, 셀 안의 불필요한 줄바꿈을 정리한다.

```python
# 빈 셀은 빈 문자열로 바꾸고, 줄바꿈과 연속 공백은 한 칸으로 정리한다.
def normalize_cell(value):
    return " ".join((value or "").split())

# 첫 행은 열 이름으로, 나머지 행은 제품 데이터로 사용한다.
headers = [normalize_cell(value) for value in corrected_rows[0]]
rows = []
current_category = ""
for extracted_row in corrected_rows[1:]:
    row = [normalize_cell(value) for value in extracted_row]
    # 병합된 구분 셀의 빈칸에는 직전에 읽은 구분 값을 채운다.
    if row[0]:
        current_category = row[0]
    else:
        row[0] = current_category
    rows.append(row)

pprint(rows)
```

### Markdown 표로 표현

정리한 행과 열을 Markdown 표로 만들고 각주를 붙인다. 원본과 비교하여 값이 맞는지 확인한다.

```python
# 셀 목록을 Markdown 표의 한 행으로 변환한다.
def markdown_row(values):
    # 셀 값 안의 |가 열 구분자로 해석되지 않도록 이스케이프한다.
    escaped = [value.replace("|", r"\|") for value in values]
    return "| " + " | ".join(escaped) + " |"

# 열 이름, 헤더 구분선, 데이터 행 순서로 표를 구성한다.
table_markdown = "\n".join([
    markdown_row(headers),
    markdown_row(["---"] * len(headers)),
    *[markdown_row(row) for row in rows],
])
# 각주의 별표가 Markdown 목록이나 강조 표시로 해석되지 않도록 처리한다.
footnotes_markdown = "\n\n".join(
    line.replace("*", r"\*") for line in footnotes.splitlines()
)
# 본문에 넣을 교체 텍스트에는 표와 각주를 함께 포함한다.
table_content = f"{table_markdown}\n\n{footnotes_markdown}"

# 원본 문자열과 렌더링한 표를 비교해 형식을 확인한다.
print(table_content)
display(Markdown(table_content))

# 표마다 텍스트와 행·열 구조를 함께 보관한다. 여러 표는 이 목록에 추가한다.
table_results = [{
    "page_no": TABLE_PAGE_NO,
    "bbox": table_with_footnotes_bbox,
    "content_type": "table",
    "table_id": f"page-{TABLE_PAGE_NO}-table-{TABLE_NO}",
    "text": table_content,
    "headers": headers,
    "rows": rows,
    "footnotes": footnotes_markdown,
}]
```

## 주제 5: 이미지의 내용 읽기(Gemini)

### 이미지 탐색과 선택

PDF에 저장된 이미지에는 로고나 아이콘도 포함된다. 이미지가 페이지에서 차지하는 면적을 기준으로 읽을 대상을 고르고, Gemini로 내용을 추출한다.

**이미지 탐색 → 면적 기준으로 선택 → 이미지 확인 → Gemini로 내용 추출**

```python
# 이미지 후보의 위치와 페이지 대비 면적 비율을 저장한다.
image_candidates = []

with pdfplumber.open(pdf_path) as pdf:
    for page_no, page in enumerate(pdf.pages, start=1):
        page_area = page.width * page.height
        # PDF 내부에 이미지로 저장된 객체를 찾는다. 로고나 아이콘도 포함된다.
        for image_index, img in enumerate(page.images, start=1):
            # 표와 같은 좌표 순서를 사용해 나중에 동일한 교체 함수로 처리한다.
            bbox = (img["x0"], img["top"], img["x1"], img["bottom"])
            image_area = (bbox[2] - bbox[0]) * (bbox[3] - bbox[1])
            image_candidates.append({
                "page_no": page_no,
                "image_index": image_index,
                "bbox": bbox,
                "area_ratio": image_area / page_area,  # 작은 이미지 후보를 걸러내는 기준이다.
            })
        if page.images:
            print(f"{page_no}페이지: 이미지 {len(page.images)}개")

print(f"전체 이미지: {len(image_candidates)}개")
```

이 예제에서는 페이지 면적의 10%를 초과하는 이미지를 선택한다. 선택한 이미지가 읽으려는 내용을 포함하는지 미리보기로 확인한다.

```python
MIN_IMAGE_AREA_RATIO = 0.10  # 페이지 면적의 10%를 초과하는 이미지를 선택한다.
# 면적 기준을 만족한 후보의 좌표와 페이지 정보를 그대로 유지한다.
selected_images = [
    image_info for image_info in image_candidates
    if image_info["area_ratio"] > MIN_IMAGE_AREA_RATIO
]

print(f"전체 {len(image_candidates)}개 중 선택한 이미지: {len(selected_images)}개")
# 모델에 전달할 후보의 페이지와 크기를 확인한다.
for image_info in selected_images:
    print(
        f"{image_info['page_no']}페이지 이미지 {image_info['image_index']} | "
        f"페이지 면적의 {image_info['area_ratio']:.1%}"
    )
```

### 선택한 이미지 확인

선택한 영역을 이미지로 확인한다. 읽어야 할 글자나 그림이 잘리지 않았는지 확인한 뒤 Gemini에 전달한다. 코드에서는 `page.crop(bbox).to_image(resolution=180)`로 PNG를 만들고, 메모리 버퍼(`BytesIO`)에 저장해 `image_bytes`로 보관한다.

### Gemini 준비와 추출 지시문

선택한 이미지의 내용을 Gemini로 읽어 Markdown으로 변환한다. 검색에 필요한 내용이 빠지지 않도록 요약 대신 이미지 속 본문과 표를 보존하도록 요청한다.

```python
# 이미지의 표·수치·각주를 보존하고 읽을 수 없는 내용은 구분하도록 지시한다.
extraction_prompt = """첨부한 이미지의 내용을 RAG용 Markdown으로 변환하세요.
- 제목과 본문을 읽기 순서대로 작성하고, 내용을 요약하거나 생략하지 마세요.
- 표는 열 이름과 값의 관계가 드러나는 Markdown 표로 작성하세요.
- 수치, 단위, 제품명과 각주를 보존하고, 각주 번호는 수치와 구분하세요.
- 그림 설명은 [그림 설명]으로 구분하세요.
- 읽을 수 없는 부분은 [확인 불가]로 표시하고 내용을 추측하지 마세요.
- 결과 전체를 코드 블록으로 감싸지 마세요.
"""
```

### Gemini로 이미지 내용 추출

선택한 이미지를 하나씩 Gemini에 전달하고 결과를 Markdown으로 표시한다. 각 이미지와 비교하여 내용과 숫자가 맞는지 확인한다.

```python
# 이미지 해석 결과를 원래 페이지와 영역 좌표에 연결해 저장한다.
image_results = []

for image_info in selected_images:
    # 선택한 이미지마다 텍스트 지시와 이미지 데이터를 하나의 메시지에 담는다.
    message = HumanMessage(content=[
        {"type": "text", "text": extraction_prompt},
        {
            "type": "image",
            # PNG 바이트를 Base64로 인코딩하고 JSON에 넣을 문자열로 변환한다.
            "base64": base64.b64encode(image_info["image_bytes"]).decode("ascii"),
            "mime_type": "image/png",
        },
    ])
    # 모델을 호출하고 응답 텍스트 양끝의 공백을 제거한다.
    response = llm.invoke([message])
    image_text = parser.invoke(response).strip()
    # 빈 응답으로 기존 영역의 내용을 교체하지 않도록 중단한다.
    if not image_text:
        raise ValueError(
            f"{image_info['page_no']}페이지 이미지 {image_info['image_index']}: 추출 결과가 비어 있습니다."
        )

    # bbox를 보관해야 해석한 내용을 원래 페이지의 해당 위치에 넣을 수 있다.
    image_results.append({
        "page_no": image_info["page_no"],
        "image_index": image_info["image_index"],
        "bbox": image_info["bbox"],
        "content_type": "image",
        "image_id": f"page-{image_info['page_no']}-image-{image_info['image_index']}",
        "text": image_text,
    })
    print(f"{image_info['page_no']}페이지 이미지 {image_info['image_index']}")
    # 원본 이미지와 대조할 수 있도록 해석 결과를 렌더링한다.
    display(Markdown(image_text))
```

> ♻️ 처음엔 표를 처리할 때도 이미지 인식 단계를 한 번 더 거친다고 보았으나(좌표를 알면 이미지 인식이 필요 없지 않느냐고 의문), 이 노트북에서 **표는 좌표 기반(`pdfplumber`)으로만 처리**하고 이미지 인식(Gemini)은 `page.images`에 잡히는 **PDF 안에 이미지로 저장된 객체**에만 쓴다. 표 좌표와 글자 좌표가 있으면 이미지 인식은 필요 없는 것이 맞다.

> ➕ **더 알아두기** (수업 외 내용: 대화 중 보충)
>
> **이 노트북의 두 처리 경로.**
>
> | 대상 | 좌표·글자 데이터 | 처리 방법 |
> |---|---|---|
> | 본문, 표 | 있음 | 좌표로 계산. 이미지 인식 안 함 |
> | `page.images`에 잡히는 이미지 | 없음(픽셀뿐) | Gemini가 보고 읽음 |
>
> 이미지 객체는 PDF 입장에서 "이 위치에 이 그림을 붙여라"일 뿐이라 그 안의 내용을 좌표로 알 수 없다. 막대그래프 이미지라면 막대 높이나 글자가 문자로 적혀 있지 않으므로 비전 모델이 한 번 봐야 한다. 이미지 인식이 표에 쓰이는 때는 **표가 이미지로 들어 있을 때** 하나뿐이다.
>
> **이 방식은 OCR이 아니다.** OCR(Optical Character Recognition, 광학 문자 인식)은 이미지에서 글자만 읽어내는 도구이고 표의 행·열 구조는 따로 재구성해야 한다. Gemini 같은 멀티모달 LLM(비전 모델)은 이미지를 보고 이해해서 "3열짜리 표이고 내용은 이렇다"처럼 구조까지 함께 출력한다.
>
> | | 전통 OCR(Tesseract 등) | 멀티모달 LLM |
> |---|---|---|
> | 하는 일 | 이미지 → 글자와 위치 | 이미지 → 이해 후 원하는 형식으로 출력 |
> | 표 구조 | 직접 재구성 | 마크다운이나 JSON 표로 바로 출력 가능 |
> | 정확도 | 글자는 안정적, 구조는 약함 | 구조는 강하지만 숫자를 잘못 읽거나 지어낼 수 있음 |
> | 비용 | 거의 무료 | 호출마다 토큰 비용 |
>
> **용어 정리.** 표 쪽은 "표 추출(Table Extraction)" 또는 "레이아웃 기반(규칙 기반) 파싱"이라고 부른다. 이미지 쪽은 "비전 모델 기반 이미지 이해"(멀티모달 LLM 활용)라고 부른다. 코드로 되는 것은 코드로, 안 되는 것만 이미지로 넘기는 방식이 경제적이다.

## 주제 6: 본문·표·이미지를 구분해 청킹하기

### 청킹을 위해 본문·표·이미지 내용을 구분하기

PDF에서 추출한 내용을 한꺼번에 길이 기준으로 청킹하면 표 중간이 잘릴 수 있다. 그러면 열 이름과 데이터 행, 각주가 서로 다른 청크로 나뉘어 검색된 내용만으로 표를 이해하기 어려워진다.

이를 방지하기 위해 청킹 전에 본문 텍스트, 표, 이미지에서 읽은 내용을 구분해 보관한다. 본문과 이미지 해석 텍스트에는 일반 청킹을 적용하고, 표는 크기에 따라 통째로 유지하거나 헤더·각주를 붙여 행 단위로 나눈다.

### 본문과 추출 결과의 순서 유지

표·이미지 영역에서 읽힌 기존 글자는 제외하고, 그 위치에 정리한 표와 이미지 해석 결과를 배치한다. 같은 내용이 중복되지 않도록 하면서 주변 본문과 원래 위치 관계를 유지한다.

```text
PDF 페이지 → [본문 텍스트] [표] [본문 텍스트] [이미지 해석 결과]
```

표나 이미지가 여러 개 있어도 각각 구분하고, 사이에 있는 본문은 이어서 읽을 수 있도록 묶는다.

```python
# 선택한 영역의 텍스트와 구조를 보존한 채 페이지를 블록 목록으로 만든다.
# 원본 PDF는 변경하지 않고, 유형·내용·좌표가 있는 딕셔너리 목록을 반환한다.
def build_page_blocks(page, regions):
    def outside_regions(obj):
        # 글자의 중심점이 표·이미지 영역 안에 있으면 기존 글자를 제외한다.
        if obj["object_type"] != "char":
            return True
        center_x = (obj["x0"] + obj["x1"]) / 2
        center_y = (obj["top"] + obj["bottom"]) / 2
        return not any(
            region["bbox"][0] <= center_x <= region["bbox"][2]
            and region["bbox"][1] <= center_y <= region["bbox"][3]
            for region in regions
        )

    # 남은 본문 줄을 텍스트와 영역 좌표가 있는 블록으로 만든다.
    body_page = page.filter(outside_regions)
    positioned_blocks = [
        {
            "content_type": "text",
            "text": line["text"],
            "bbox": (line["x0"], line["top"], line["x1"], line["bottom"]),
        }
        for line in body_page.extract_text_lines(return_chars=False)
    ]
    # 표의 헤더·행·각주와 이미지 ID를 포함해 추출 결과를 복사한다.
    positioned_blocks.extend(dict(region) for region in regions)
    # bbox의 위쪽 좌표를 먼저, 높이가 같으면 왼쪽 좌표를 기준으로 정렬한다.
    positioned_blocks.sort(key=lambda block: (block["bbox"][1], block["bbox"][0]))

    blocks = []
    for block in positioned_blocks:
        # 표나 이미지 사이에 연속해서 나온 본문 줄만 하나로 합친다.
        if blocks and block["content_type"] == "text" and blocks[-1]["content_type"] == "text":
            previous = blocks[-1]
            previous["text"] += "\n" + block["text"]
            x0, top, x1, bottom = previous["bbox"]
            bx0, btop, bx1, bbottom = block["bbox"]
            previous["bbox"] = (min(x0, bx0), min(top, btop), max(x1, bx1), max(bottom, bbottom))
        else:
            blocks.append(block)
    return blocks
```

### PDF 본문 청킹

먼저 유형 구분 없이 페이지 본문에 일반 청킹을 적용한다. 최대 1,000자, 겹침 목표 100자로 나누어 결과를 살펴본다.

실제 겹침은 문단·줄바꿈에 따라 달라지고, 긴 표는 청크 경계에서 나뉠 수 있다. 이후 본문과 표를 구분해 청킹한 결과와 비교한다.

### 청킹 결과 확인

1페이지에서 표와 각주가 같은 청크에 포함되는지, 원문 출처를 확인할 수 있는지 살펴본다.

이 예제에서는 표와 각주 511자가 1,000자 기준의 청크 하나에 함께 들어간다. 이는 현재 텍스트의 길이와 줄바꿈에 따른 결과다. 일반 청킹은 표 구조를 따로 판단하지 않으므로 기준 길이를 줄이거나 표가 길어지면 표가 나뉠 수 있다.

### 길이 기준 청킹에서 표가 나뉘는 경우

이번에는 같은 제품표와 각주를 300자 기준으로 나누어 본다. 1,000자 기준에서 함께 유지되던 내용이 여러 청크로 나뉘고, 열 이름은 첫 번째 청크에만 남는다. 뒤쪽 청크에는 데이터 행이나 각주만 남을 수 있다.

청크 하나만 검색되었을 때 제품명·가격·단위의 관계를 읽을 수 있는지 확인한다.

### 구분한 본문과 표 확인

제품표가 있는 페이지에서 표 앞뒤의 본문과 표가 각각 구분되어 있는지 확인한다. 이때 표와 각주는 함께 유지되어야 한다.

### 표를 통째로 유지하기

본문과 이미지 해석 텍스트만 길이 기준으로 나누고, 표와 각주는 하나의 청크로 유지한다.

이 방법은 표가 중간에 잘리는 것을 막지만, 청크 길이가 커질 수 있다. 이 예제의 표와 각주는 511자로, 비교 기준인 300자를 초과한다. 표의 내용을 보존하면서 청크 길이도 줄이려면 행 구조를 이용해 나누어야 한다.

### 헤더와 각주를 반복하며 행 단위로 나누기

표가 기준 길이 안에 들어오면 통째로 유지하고, 기준을 넘으면 데이터 행 사이에서만 나눈다. 각 청크에 동일한 열 이름과 각주를 붙여 행의 의미를 유지한다. 반복되는 헤더·각주와 줄바꿈도 길이 계산에 포함한다.

제품명과 가격이 분리되지 않도록 한 행은 자르지 않는다. 헤더·각주·한 행만으로 기준 길이를 넘는 경우에는 해당 청크의 길이 초과를 허용한다.

```python
# 한 행을 자르지 않고, 헤더와 각주가 포함된 Markdown 표 문자열 목록을 만든다.
def split_table_rows(headers, rows, footnotes="", chunk_size=1000):
    # 분할 기준은 양수여야 한다.
    if chunk_size <= 0:
        raise ValueError("chunk_size는 양수여야 합니다.")

    # 모든 표 청크에 반복해서 넣을 열 이름과 Markdown 구분선을 준비한다.
    header_lines = [
        markdown_row(headers),
        markdown_row(["---"] * len(headers)),
    ]
    # 각주가 있을 때만 앞에 빈 줄을 붙인다.
    footnote_suffix = f"\n\n{footnotes.strip()}" if footnotes.strip() else ""

    # 길이를 계산할 때도 최종 출력과 같은 헤더·행·각주 문자열을 사용한다.
    def render_table(row_group):
        lines = header_lines + [markdown_row(row) for row in row_group]
        return "\n".join(lines) + footnote_suffix

    parts = []  # 완성된 표 청크를 저장한다.
    current_rows = []  # 현재 청크에 넣을 데이터 행을 모은다.
    for row in rows:
        # 다음 행까지 넣었을 때의 길이를 헤더·각주·줄바꿈을 포함해 확인한다.
        candidate = render_table(current_rows + [row])
        # 다음 행을 넣으면 기준 길이를 넘는 경우, 지금까지 모은 행을 저장한다.
        if current_rows and len(candidate) > chunk_size:
            parts.append(render_table(current_rows))
            current_rows = []
        # 현재 청크가 비어 있으면 한 행은 반드시 넣는다. 한 행만으로 길이가 초과돼도 자르지 않는다.
        current_rows.append(row)

    # 마지막에 남은 행을 저장한다. 데이터 행이 없어도 헤더와 각주를 한 번 반환한다.
    if current_rows or not parts:
        parts.append(render_table(current_rows))
    return parts
```

### 내용 유형에 맞게 청킹하기

본문과 이미지 해석 텍스트에는 일반 청킹을, 표에는 행 단위 분할을 적용한다. 제품표가 있는 페이지를 300자 기준으로 나누어 각 표 청크에 열 이름과 각주가 함께 남는지 확인한다.

```python
# 각 블록에 보관한 유형과 데이터를 사용하므로 여러 표에도 같은 함수를 적용할 수 있다.
def split_document_blocks(blocks, chunk_size=1000, chunk_overlap=0):
    text_splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap,
    )
    chunks = []
    for block in blocks:
        # 본문과 이미지 해석 결과는 각각의 출처를 유지하며 일반 청킹한다.
        if block["content_type"] != "table":
            block_doc = Document(page_content=block["text"], metadata=block["metadata"])
            chunks.extend(text_splitter.split_documents([block_doc]))
            continue

        # 표마다 자신의 헤더·행·각주를 사용해 행 사이에서만 나눈다.
        table_parts = split_table_rows(
            block["headers"], block["rows"],
            footnotes=block["footnotes"],
            chunk_size=chunk_size,
        )
        # 원래 table_id는 유지하고, 같은 표 안의 청크 순서를 table_part로 기록한다.
        for part_no, text in enumerate(table_parts, start=1):
            chunks.append(Document(
                page_content=text,
                metadata={**block["metadata"], "table_part": part_no},
            ))
    return chunks

table_page_chunks = split_document_blocks(page_blocks, chunk_size=TABLE_DEMO_CHUNK_SIZE)
for chunk in table_page_chunks:
    if chunk.metadata["content_type"] == "table":
        print(
            f"{chunk.metadata['table_id']} | 부분 {chunk.metadata['table_part']} | "
            f"{len(chunk.page_content)}자"
        )
        display(Markdown(chunk.page_content))
```

### PDF 전체에 청킹 적용

PDF 전체에 유형별 청킹을 적용하고, 나뉜 내용과 원문 출처를 확인한다.

이 예제에서 행·열 구조를 추출한 표는 1페이지 제품표다. 다른 표에도 같은 방법을 적용하려면 먼저 행·열 구조를 추출해야 한다. 선택하지 않은 표와 이미지 해석 결과 안의 Markdown 표는 일반 청킹 대상이다.

300자는 분할 결과를 살펴보기 위한 값이다. 기준을 1,000자로 늘려 짧은 제품표가 하나의 청크로 유지되는지도 비교한다.

## 주제 7: 임베딩 연결·추가 방법론·다른 도구

### 임베딩과 검색으로 연결

```text
PDF 텍스트 추출
  → 표·이미지 내용 정리
  → 본문·표·이미지 내용을 구분
  → 유형에 맞게 청킹
  → 임베딩 → 벡터 저장소 → 검색
```

생성한 청크를 임베딩해 벡터 저장소에 넣으면 질문과 관련된 내용을 검색할 수 있다. 각 청크의 출처 정보를 유지하면 검색 결과가 원문 어느 페이지에서 나온 것인지 확인할 수 있다.

### 추가 방법론

- **Contextual Retrieval**: LLM이 문서 전체와 해당 청크를 참고해 생성한 짧은 문맥 설명을 청크 앞에 붙여 임베딩과 검색에 활용한다.
- **표 요약 기반 검색**: 표의 요약을 임베딩해 검색하고, 검색된 요약과 연결된 원본 표를 답변의 근거로 사용한다.

### 다른 PDF·OCR 도구 활용

- PDF의 텍스트·이미지 추출에는 PyMuPDF, 표 추출에는 Camelot 같은 라이브러리도 활용할 수 있다.
- 이미지 속 글자를 추출할 때는 LLM 대신 Tesseract나 PaddleOCR 같은 OCR(광학 문자 인식) 도구를 사용할 수 있다.

## 정리

- PDF에는 표 객체가 없다. 글자(문자·글꼴·위치), 선·도형(시작점·끝점·위치·색), 이미지(데이터·위치·크기)를 그리는 정보만 있고, 행·열 관계는 저장되지 않는다.
- `pdfplumber`는 선과 글자의 좌표로 경계선 → 교차점 → 셀 → 표 → 글자 추출 순서로 표 구조를 추정한다. 선이 불완전하면 `explicit_vertical_lines`로 경계를 보완한다.
- 표는 좌표 기반으로, PDF에 이미지로 저장된 객체는 Gemini(비전 모델)로 읽는다. 이 방식은 OCR이 아니다.
- 이미지는 페이지 면적의 10%를 넘는 것만 골라(로고·아이콘 제외) PNG로 렌더링해 Gemini에 보낸다. 지시문에는 요약 대신 본문과 표를 보존하라고 요청한다.
- 청킹 전에 본문·표·이미지 해석을 구분한다. 일반 청킹은 표를 중간에서 자를 수 있으므로, 표는 통째로 유지하거나 헤더·각주를 반복 부착해 행 단위로만 나눈다.
- 구분한 청크는 임베딩해 벡터 저장소에 넣고, 출처(페이지) 메타데이터를 유지한다.
- 글꼴·크기는 PDF에 한 번 지정되고 `pdfplumber`가 글자마다 붙여 준다. 글자별 좌표는 계산된 값이다. 단어·줄은 의미가 아니라 거리·높이 규칙으로 묶기 때문에 표에서는 틀린다.
- 선분 좌표는 표 전용이 아니다. 흰색으로 그린 선은 좌표와 색이 저장되어 인식되고, 아예 그리지 않은 변은 인식되지 않는다.
- `load_dotenv(".env")`는 현재 폴더만 보고, 인자 없는 `load_dotenv()`는 위로 올라가며 `.env`를 찾는다.

## ✅ 확인 질문

1. PDF에는 글자·선·이미지가 각각 어떤 정보로 저장되는지, 그리고 왜 "표"라는 구조는 저장되지 않는지 설명해 보시오.
2. 글자 좌표가 정확한데도 표의 행·열을 알 수 없는 이유와, 선의 좌표가 있어도 표를 못 찾는 경우 두 가지를 설명해 보시오.
3. `pdfplumber`가 표를 인식하는 다섯 단계를 순서대로 말하고, 선이 없는 표와 이미지로 된 표는 각각 어떻게 처리하는지 설명해 보시오.
4. 디지털 PDF와 스캔 PDF는 글자 측면에서 어떻게 다르며, 각각 어떤 추출 방법을 쓰는지 설명해 보시오.
5. 이 노트북의 제품표에서 `explicit_vertical_lines`를 쓴 이유는 무엇이며, 병합 셀의 빈 값은 어떻게 처리했는지 설명해 보시오.
6. Markdown 표를 만들 때 셀 안의 `|`와 각주의 `*`를 이스케이프하는 이유를 설명해 보시오.
7. 이 노트북에서 표와 이미지는 각각 어떤 방식으로 읽는가? 표 좌표와 글자 좌표를 알면 이미지 인식이 필요 없는데도 Gemini를 쓰는 대상은 무엇인지 설명해 보시오.
8. 전통 OCR과 멀티모달 LLM의 차이를 표 구조 처리와 정확도 측면에서 비교해 보시오.
9. 면적 10% 기준으로 이미지를 고르는 이유는 무엇인가?
10. PDF 전체를 길이 기준으로 한 번에 청킹하면 표에서 어떤 문제가 생기는지, 300자 기준 예시로 설명해 보시오.
11. 청킹 전에 본문·표·이미지 내용을 구분해 보관하고 원래 위치 순서를 유지하는 이유를 설명해 보시오.
12. 표를 통째로 유지하는 방법과 헤더·각주를 반복하며 행 단위로 나누는 방법의 장단점을 비교해 보시오.
13. 임베딩·검색 단계에서 청크의 출처 메타데이터(페이지 등)를 유지하면 어떤 점이 좋은지 설명해 보시오.
14. Contextual Retrieval과 표 요약 기반 검색은 각각 어떤 방식으로 검색 품질을 높이는지 설명해 보시오.
15. PDF 안에서 글꼴과 크기는 어떻게 지정되고, `pdfplumber`는 이를 글자마다 어떻게 보여 주는가?
16. 글자별 좌표는 파일에 저장된 값인지 계산된 값인지, 단어와 줄은 어떤 기준으로 묶는지 설명해 보시오.
17. 노트북 PDF의 헤더 "제품명(모델명)"이 줄 단위 추출에서 어떻게 잘못 묶였고, 표 추출은 이를 어떻게 피하는지 설명해 보시오.
18. 흰색으로 그린 선과 아예 그리지 않은 선은 `find_tables` 결과에서 어떻게 달랐는지 설명해 보시오.
19. 선 좌표가 있는데도 표 추출 정확도가 100%가 되지 않는 이유를 세 가지 이상 들어 보시오.
20. `load_dotenv(".env")`와 `load_dotenv()`의 차이를 설명하고, `API key required` 오류가 났던 원인을 말해 보시오.
21. 가로선 범위와 글자 간격으로 열을 나누는 아이디어의 장점과, 노트북 PDF에서 확인된 문제를 설명해 보시오.
