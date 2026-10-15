---
tags: [rag, practice, loader, chunking, langchain]
til: v2 2026-10-15
---

# 2026-10-15 (1)RAG Agent 실습1 설계와 로딩 청킹(Document, TARGET NAMES, FILE CODES, 청크 ID)
> 작성일: 2026-10-15

## 🔗 관련 글

- [PDF 표·이미지 추출(pdfplumber, Gemini, 표 인식 원리, 유형별 청킹)](2026-10-14%20(1)PDF_표_이미지_추출(pdfplumber,_Gemini,_표_인식_원리,_유형별_청킹).md) — 랭체인 폴더의 34번 노트북(`34-pdf-table-image-extraction.ipynb`)을 정리한 글이다. 이 실습의 로딩(`PyPDFLoader`)·청킹(`RecursiveCharacterTextSplitter`)이 그 글의 "PDF 본문 청킹"과 같은 도구다. 이 실습은 pypdf만 쓰고 표·이미지 처리는 비교 실험으로 미룬다.
- [Parent-Child RAG(위키 크롤링, section Parent, Child 청킹, ColBERT)](2026-10-13%20(1)Parent-Child_RAG(위키_크롤링,_section_Parent,_Child_청킹,_ColBERT).md) — 청킹 방식을 바꿔 보는 개선 후보(Parent-Child)가 이 실습의 "미뤄 둔 개선 후보"에 들어 있다.
- [RAG 실험 설계와 스크립트 구축](2026-09-07%20(2)%20RAG%20실험%20설계와%20스크립트%20구축.md) — "한 번에 하나만 바꿔서 비교한다"는 실험 원칙이 이 실습의 진행 규칙과 같다.
- [LangChain 기초 - Runnable, LCEL 파이프라인, 실습 4종](2026-09-01%20LangChain%20기초%20-%20Runnable,%20LCEL%20파이프라인,%20실습%204종.md) — 이 실습에서 쓰는 `prompt | llm | parser` 형태의 바탕이다. (10-16 글에서 이어진다.)

## ❓ 아직 모르겠는 것

대화에서 풀렸지만 복습 때 다시 확인한다. 아래는 실습 노트북(`RAG_Agent_실습_노트북.ipynb`)에 사용자가 직접 남긴 메모와 설명 블록에 근거했다. 실습을 마친 지금은 모두 이해한 상태다.

- `FILE_CODES = {name: f"doc{i}" for i, name in enumerate(TARGET_NAMES, start=1)}` 한 줄이 무슨 일을 하는가? 딕셔너리 컴프리헨션·f-문자열·`enumerate(start=1)`을 풀어 쓰면 어떻게 되는가? (주제 4 참고, 풀림)
- 함수 정의만 있는 셀을 실행하면 왜 아무것도 안 나오는가? (노트북에 "함수니깐없음"이라고 메모해 둔 지점. 주제 5 참고, 풀림)

## 주제 1: 실습의 목적과 정해 둔 것

이 실습은 **RAG 파이프라인과 Agent 파이프라인의 흐름을 직접 구현해 보고, 골든셋을 만들어 평가까지 해 보는 것**이다. 문서(`RAG_Agent_실습_전체흐름.docx`)에는 "금융 도메인 · 자체 실습 · 2026-10-15 기준"이라고 적혀 있다. 참고용 초안(`PRACTICE.md`)이 있었지만 필수 조건은 아니었다.

### 공부 순서: 코드를 먼저 완성하고, 그 코드를 한 줄씩 뜯었다

완성 코드(`RAG_Agent_완성코드.py`)를 먼저 만들어 전체가 돌아가는 것을 확인했다. 그다음 같은 코드를 셀 단위로 나눈 노트북(`RAG_Agent_실습_노트북.ipynb`)에서 한 줄씩 실행하고 설명을 달며 공부했다. 이 글을 포함한 4개 글(10-15 ~ 10-18)은 그 공부를 단계별로 나눠 정리한 것이다.

| 날짜 | 다루는 단계 |
|---|---|
| 10-15 | 설계 결정, 라이브러리·설정, 로딩, 청킹, 청크 ID |
| 10-16 | 임베딩·벡터 DB(Chroma), 검색, 기본 RAG |
| 10-17 | Agent(LangGraph): 근거 판단·검색어 재작성·보류 |
| 10-18 | 골든셋 읽기, 평가 지표, 기본 RAG vs Agent 비교, 청킹 크기 실험 |

### 정한 것(문서의 결정 사항)

| 항목 | 결정 |
|---|---|
| 데이터 범위 | 금융 도메인 PDF, 우선 5개(별첨, 보도자료 3개, 통화신용정책 운영)로 시작 |
| 추출 | pypdf(`PyPDFLoader`)만 사용. 예외 처리는 넣지 않고 단순하게 |
| Agent 구조 | 도메인별 멀티에이전트가 아니라 **하나의 에이전트**. 도구·규칙·규모가 같아서 나눌 이유가 없다. 직접 그린 LangGraph Agentic RAG |
| 골든셋 | 직접 작성하고 PDF 원문과 대조해 검수. 쪽수는 항상 PDF 쪽수(표지=1쪽) |
| 하이브리드 추출 | 기준선(baseline, 비교의 출발점이 되는 점수) 이후의 비교 실험으로 미룸 |

### 단계별 진행표

문서에는 ☑ 완료 / ☐ 예정으로 적혀 있다. (10-15 문서 작성 시점에는 1, 2-1단계만 완료였고, 이후 완성 코드로 전 단계를 마쳤다.)

| 단계 | 할 일과 확인 기준 |
|---|---|
| 1. 데이터 확인 | 문서별 쪽수와 쪽별 글자 수를 확인하고 문제 문서를 파악 |
| 2-1. 로딩 | 5개 PDF를 읽어 쪽별 Document로 모으고 `file_name`, `page_no` 메타데이터를 붙임 (총 48쪽) |
| 2-2. 청킹 | `chunk_size`, `chunk_overlap`을 정해 나누고 청크 개수와 모양을 눈으로 확인 |
| 2-3. 청크 ID | 청크마다 ID를 붙이고 메타데이터가 모두 있는지 확인 |
| 3. 임베딩·벡터 DB | 임베딩 모델과 벡터 DB를 하나로 정해 저장. 저장 개수가 청크 수와 같은지 확인 |
| 4. 검색 단독 확인 | 질문을 넣었을 때 상위 k개 청크와 출처가 나오는지, 정답 쪽이 상위에 있는지 확인 |
| 5. 기본 RAG | 검색 근거로 답변하고 파일명·쪽수를 표시. 근거가 없으면 모른다고 답함 |
| 6. 골든셋과 첫 평가 | 문항 작성·검수(답변 불가 문항 포함), 검색 지표(Hit@k, Recall@k, MRR)로 기준 점수 측정 |
| 7. Agent 확장 | LangGraph로 근거 판단·재검색·답변 보류를 추가 |
| 8. 평가와 비교 | 기본 RAG와 Agent를 같은 골든셋으로 비교하고 실패 사례와 개선점 정리 |

### 진행 규칙

- 한 번에 하나만 바꿔서 비교하고, 단계마다 결과를 눈으로 확인한 뒤 넘어간다.
- 골든셋은 검색 DB에 넣지 않는다.
- 글자 수가 적다고 쪽을 버리지 않는다. 애매하면 표시해 둔다.
- 쪽수는 PDF 쪽수(뷰어 기준)로 통일하고, 문서에 인쇄된 쪽번호와 구분한다.

### 미뤄 둔 개선 후보(기준선 이후 비교 실험)

- ★2019 증시콘서트: 글꼴 정보가 없어 모든 추출기가 실패한다 → OCR 또는 Gemini 이미지 읽기
- 한-호주 퇴직연금 포럼: 20쪽대 이후가 이미지로 된 쪽 → 이미지 읽기
- WP22, KIFVIP: 표지가 이미지(본문은 정상)
- 통화신용정책: 띄어쓰기 소실, 꼬리말 반복 / 별첨: 쪽번호(`- N -`)가 텍스트 앞에 붙음
- 표·그림·순서도 처리(하이브리드 추출), Parent-Child 청킹, Hybrid 검색 등

> ➕ **더 알아두기**
> "기준선(baseline) 먼저, 개선은 나중에"는 실험 설계의 기본이다. 가장 단순한 방식(pypdf + 고정 길이 청킹 + 기본 RAG)의 점수를 먼저 재 두어야, 나중에 표 추출이나 Parent-Child를 넣었을 때 **그 변경 때문에 점수가 올랐는지** 비교할 수 있다.

## 주제 2: 전체 흐름과 코드 구조

완성 코드 맨 위 주석에 적힌 전체 흐름이다.

1. **로딩**: PDF를 쪽별로 읽고 `file_name`, `page_no` 메타데이터를 붙인다.
2. **청킹**: 쪽을 청크로 나누고 청크 ID를 붙인다.
3. **임베딩**: 청크를 임베딩해 Chroma 벡터 DB에 저장한다.
4. **검색**: 질문 → 상위 k개 청크(청크 ID, 파일명, 쪽수 포함)
5. **기본 RAG**: 검색 1번 → 답변(출처 표시)
6. **Agent**: 검색 → 근거 판단 → (부족하면 검색어 재작성, 최대 3회) → 답변 또는 보류
7. **평가**: 골든셋으로 검색 지표(Hit@k, MRR)와 답변 정확도를 측정하고 기본 RAG와 Agent를 비교

처음에는 문서 5개, 질문 몇 개로 돌려 보고, 잘 되면 `QUESTION_LIMIT`을 늘린다.

### `# %%` 셀 구분

`.py` 파일이지만 `# %%`로 셀이 나뉘어 있다. VS Code에서 열면 셀마다 "Run Cell"이 보이고, 위에서부터 하나씩 실행하며 각 단계의 결과를 확인할 수 있다. 통째로 `python 파일.py`로 실행해도 된다. 노트북(`.ipynb`)은 같은 코드를 셀로 나누고 셀마다 "확인할 것"을 달아 둔 버전이다.

## 주제 3: 라이브러리 가져오기

```python
import csv                      # 골든셋 CSV를 읽는 표준 라이브러리
import json                     # 실행 결과를 JSON 줄로 저장
import logging                  # 라이브러리가 내보내는 경고 메시지 조절
import time                     # 실행 시간 측정, 호출 사이 대기
from datetime import datetime   # 실행 기록에 시각 남기기
from pathlib import Path        # 파일 경로를 객체로 다룬다
from typing import TypedDict    # LangGraph State의 구조를 적는 데 사용

from dotenv import load_dotenv                                   # .env 파일의 API 키를 환경변수로 읽어 온다
from langchain_chroma import Chroma                              # 벡터 DB
from langchain_community.document_loaders import PyPDFLoader     # PDF를 쪽별 Document로 읽는 로더
from langchain_core.output_parsers import StrOutputParser        # 모델 응답을 문자열로 꺼낸다
from langchain_core.prompts import ChatPromptTemplate           # 프롬프트 틀
from langchain_google_genai import ChatGoogleGenerativeAI, GoogleGenerativeAIEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langgraph.graph import END, START, StateGraph              # Agent 그래프를 만드는 도구
from pydantic import BaseModel, Field                            # 모델이 돌려줄 구조화된 응답의 형식
```

- 위 5줄(`csv`~`typing`)은 **파이썬 표준 라이브러리**라 따로 설치하지 않는다. 아래는 외부 패키지다.
- 노트북 커널에는 `langchain`, `langchain-google-genai`, `langchain-chroma`, `langchain-community`, `langgraph`, `pypdf`, `python-dotenv`가 설치되어 있어야 한다.
- 각 import가 쓰이는 단계: `csv`·`json`·`datetime`은 평가(10-18), `Chroma`·`GoogleGenerativeAIEmbeddings`는 임베딩(10-16), `StateGraph`·`TypedDict`·`BaseModel`은 Agent(10-17), `PyPDFLoader`·`RecursiveCharacterTextSplitter`는 이 글이다.

### 경고 메시지 숨기기

```python
# pypdf가 한글 글꼴 인코딩 때문에 내보내는 에러 메시지를 화면에서 숨긴다. (추출 결과에는 영향 없음)
logging.getLogger("pypdf._cmap").setLevel(logging.CRITICAL)
```

`logging.getLogger("이름")`은 그 이름의 로그 출력 담당자를 가져오고, `.setLevel(logging.CRITICAL)`은 **가장 심각한 수준(CRITICAL)이 아닌 메시지는 내보내지 않게** 한다. pypdf가 한글 PDF의 글꼴 인코딩(`_cmap`)을 읽다가 내는 에러 메시지가 화면을 가득 채우는 것을 막는 줄이며, 추출된 글자에는 영향이 없다.

## 주제 4: 설정

### 경로 설정

```python
# 이 파일은 프로젝트 폴더 밖(바탕화면)에 있어서 .env 경로를 직접 지정한다.
load_dotenv(dotenv_path=r"C:\Users\USER\Desktop\aim_home\.env", override=True)

BASE_DIR = Path(r"C:\Users\USER\Desktop\aim_home\260811\aim-ai-agent-practice\langchain_\rag-agent-practice")
DATA_DIR = BASE_DIR / "data" / "finance"                  # 금융 PDF가 있는 폴더
GOLDEN_SET_PATH = BASE_DIR / "data" / "golden_set.csv"    # 골든셋 CSV (직접 만든 파일 경로로 바꿔도 된다)
OUTPUT_DIR = BASE_DIR / "outputs"                         # 벡터 DB와 실행 기록을 저장할 폴더
CHROMA_DIR = OUTPUT_DIR / "chroma_db"
RESULT_PATH = OUTPUT_DIR / "runs.jsonl"
OUTPUT_DIR.mkdir(parents=True, exist_ok=True)             # 폴더가 없으면 만든다
```

- `load_dotenv(dotenv_path=..., override=True)`: `.env` 파일에 적힌 `KEY=값` 줄을 환경변수로 읽어 들인다. API 키(Gemini)가 여기서 들어온다. 보통 현재 폴더의 `.env`를 찾지만, 이 파일은 프로젝트 폴더 밖에 있어서 경로를 직접 적었다. `override=True`는 같은 이름의 환경변수가 이미 있어도 `.env` 값으로 덮어쓴다는 뜻이다.
- `r"C:\Users\..."`: 앞의 `r`은 raw 문자열이다. 윈도우 경로의 역슬래시(`\`)를 이스케이프 문자로 해석하지 않고 글자 그대로 쓰게 한다.
- `Path(...)`와 `/` 연산자: `BASE_DIR / "data" / "finance"`처럼 `/`로 경로를 이어 붙인다. 운영체제마다 다른 구분자(`\` vs `/`)를 신경 쓰지 않아도 된다.
- `OUTPUT_DIR.mkdir(parents=True, exist_ok=True)`: 폴더를 만든다. `parents=True`는 중간 폴더도 함께 만든다는 뜻이고, `exist_ok=True`는 이미 있어도 에러를 내지 않는다는 뜻이다. 그래서 몇 번을 다시 실행해도 안전하다.
- 출력물 두 가지: `chroma_db`(벡터 DB, 10-16)와 `runs.jsonl`(실행 기록, 10-18)이 `outputs` 폴더에 쌓인다.

### 이번에 사용할 문서 5개

```python
# 이번에 사용할 문서 5개. 문서를 늘리고 싶으면 이름을 추가한다. (골든셋의 target_file_name 과 같은 이름)
TARGET_NAMES = [
    "[별첨] 지방은행의 시중은행 전환시 인가방식 및 절차.pdf",
    "240130(보도자료) 지방은행의 시중은행 전환시 인가방식 및 절차.pdf",
    "240320(보도자료) 금융권의 상생금융 추진현황.pdf",
    "240409(보도자료) 금융위 핀테크 투자 생태계 활성화 나선다.pdf",
    "2024년 3월_2. 통화신용정책 운영.pdf",
]
```

- 폴더에는 PDF가 더 많이 있어도 **이 목록에 있는 5개만** 읽는다(주제 5).
- 이 이름은 골든셋 CSV의 `target_file_name` 열과 **글자 하나까지 같아야** 한다. 골든셋의 질문이 어느 문서에 속하는지 이 이름으로 맞춰 보기 때문이다(10-18).
- 문서 5개의 쪽수는 15 + 7 + 7 + 6 + 13 = 48쪽이다(위 목록 순서대로).

### FILE_CODES: 파일 이름 → 짧은 코드 대응표

```python
# 청크 ID에 쓸 짧은 문서 코드. 예: 첫 번째 문서의 4쪽 첫 청크 -> "doc1-p4-c0"
FILE_CODES = {name: f"doc{i}" for i, name in enumerate(TARGET_NAMES, start=1)}
```

이 한 줄은 **딕셔너리 컴프리헨션**이다. 반복문으로 딕셔너리를 한 줄에 만드는 문법이다.

| 부분 | 설명 |
|---|---|
| `{ ... }` | 딕셔너리를 만든다는 표시 |
| `name: f"doc{i}"` | 한 항목의 모양. 앞이 키(파일 이름), 뒤가 값(`"doc1"` 같은 짧은 코드) |
| `f"doc{i}"` | f-문자열. 중괄호 안의 `i` 값을 글자에 끼워 넣는다. `i`가 1이면 `"doc1"` |
| `for i, name in ...` | 아래 `enumerate`가 주는 값을 하나씩 꺼내 번호는 `i`에, 파일 이름은 `name`에 넣으며 반복 |
| `enumerate(TARGET_NAMES, start=1)` | 리스트를 돌면서 (번호, 원소) 쌍을 만든다. `start=1`은 번호를 1부터 센다는 뜻(기본값은 0) |

같은 일을 풀어서 쓰면 이렇다.

```python
FILE_CODES = {}
for i, name in enumerate(TARGET_NAMES, start=1):
    FILE_CODES[name] = f"doc{i}"
```

결과는 이런 모양이다. 타입은 딕셔너리(`dict`)이고, 키는 파일 이름, 값은 짧은 코드(문자열)다.

```python
{
  "[별첨] 지방은행의 시중은행 전환시 인가방식 및 절차.pdf": "doc1",
  "240130(보도자료) 지방은행의 ... .pdf": "doc2",
  "240320(보도자료) 금융권의 상생금융 추진현황.pdf": "doc3",
  "240409(보도자료) 금융위 핀테크 ... .pdf": "doc4",
  "2024년 3월_2. 통화신용정책 운영.pdf": "doc5",
}
```

**왜 필요한가**: 청크 ID에 파일 이름 전체를 넣으면 너무 길다. `FILE_CODES[파일 이름]`으로 `"doc1"`을 꺼내 쓰면 청크 ID가 `doc1-p4-c0`처럼 짧고 일정한 모양이 된다. 쓰는 곳은 `chunk_pages` 함수의 `code = FILE_CODES[chunk.metadata["file_name"]]` 줄이다(주제 6).

> ➕ **더 알아두기**
> 코드 번호(`doc1`…)는 **`TARGET_NAMES` 목록의 순서**로 매겨진다. 파일을 읽는 순서(이름순 정렬, 주제 5)와는 무관하다. 그래서 목록 중간에 문서를 끼워 넣으면 뒤 문서의 코드가 바뀌고, 골든셋에서 청크 ID로 정답을 지정했다면 그 ID도 바뀐다. 문서를 늘릴 때는 목록 **맨 뒤에 추가**하는 것이 안전하다.

### 모델 이름과 상수

```python
EMBEDDING_MODEL_NAME = "gemini-embedding-2"
MODEL_NAME = "gemini-3.6-flash"

CHUNK_SIZE = 1000      # 청크 하나의 최대 글자 수
CHUNK_OVERLAP = 100    # 청크끼리 겹치게 하는 글자 수 (경계에서 문장이 잘려도 앞뒤 문맥이 이어지게)
TOP_K = 5              # 한 번 검색할 때 가져올 청크 수
MAX_SEARCH = 3         # Agent의 최대 검색 횟수 (최초 검색 포함)

QUESTION_LIMIT = 3     # 평가에 사용할 질문 수. 처음에는 작게 시작하고 늘린다. (None이면 전부)
SLEEP_SEC = 1          # 질문 사이 대기 시간(초). 호출 제한(429 에러) 예방용

# 모델 객체. max_retries 는 일시적인 오류가 났을 때 다시 시도하는 횟수다.
embeddings = GoogleGenerativeAIEmbeddings(model=EMBEDDING_MODEL_NAME)
llm = ChatGoogleGenerativeAI(model=MODEL_NAME, timeout=90, max_retries=3)
```

| 상수 | 뜻과 쓰이는 곳 |
|---|---|
| `EMBEDDING_MODEL_NAME` | 글을 숫자 벡터로 바꾸는 임베딩 모델 이름 (10-16) |
| `MODEL_NAME` | 답변·판단·채점에 쓰는 생성 모델 이름 (10-16~18) |
| `CHUNK_SIZE` / `CHUNK_OVERLAP` | 청킹 설정 (이 글 주제 6). 10-18에서 이 값을 바꿔 비교한다 |
| `TOP_K` | 검색할 때 가져올 청크 수 (10-16) |
| `MAX_SEARCH` | Agent가 검색하는 최대 횟수. 최초 검색을 포함한다 (10-17) |
| `QUESTION_LIMIT` | 평가 문항 수 제한. `None`이면 전부 (10-18) |
| `SLEEP_SEC` | 질문 사이 대기 시간. 호출 제한(429 에러)을 예방한다 (10-18) |

- 상수 이름을 **대문자로** 쓰는 것은 "바뀌지 않는 설정값"이라는 파이썬 관례다.
- `embeddings`와 `llm`은 **모델 객체**다. `timeout=90`은 응답을 90초까지 기다린다는 뜻이고, `max_retries=3`은 일시적인 오류가 나면 최대 3번 다시 시도한다는 뜻이다.
- `SLEEP_SEC`는 완성 코드(`.py`)에서는 1초, 노트북에서는 4초로 되어 있다(주제 7 참고).

## 주제 5: 로딩 — PDF를 쪽별 Document로

```python
def load_pages():
    """TARGET_NAMES 의 PDF를 쪽별 Document 리스트로 읽는다. 리스트의 한 칸 = PDF의 한 쪽."""
    pages = []
    for pdf_path in sorted(DATA_DIR.glob("*.pdf")):        # 폴더의 PDF를 이름순으로 하나씩
        if pdf_path.name not in TARGET_NAMES:               # 목록에 없는 파일은 건너뛴다
            continue
        for page in PyPDFLoader(str(pdf_path)).load():      # 쪽마다 Document 하나
            # 로더가 자동으로 넣은 메타데이터(깨진 title 등)는 버리고, 우리가 쓸 두 가지만 남긴다.
            page.metadata = {
                "file_name": pdf_path.name,
                "page_no": page.metadata["page"] + 1,        # 로더의 page 는 0부터 -> PDF 쪽수(1부터)로 맞춘다
            }
            pages.append(page)
    return pages


pages = load_pages()
print(f"전체 쪽 수: {len(pages)}")   # 5개 문서면 48이 나와야 한다 (15 + 7 + 7 + 6 + 13)
```

### 한 줄씩 보기

- `DATA_DIR.glob("*.pdf")`: 폴더 안에서 `.pdf`로 끝나는 파일을 모두 찾는다(`*`는 아무 글자). `sorted(...)`로 이름순 정렬한다.
- `pdf_path.name not in TARGET_NAMES` → `continue`: 파일 이름만 뽑아 목록에 없으면 이번 반복을 건너뛴다. 폴더에 PDF가 수십 개 있어도 5개만 읽는 이유다.
- `PyPDFLoader(str(pdf_path)).load()`: 로더에 PDF 경로를 주고 `load()`하면 **쪽마다 `Document` 하나**가 담긴 리스트가 나온다. 로더는 경로를 문자열로 받으므로 `str(...)`로 바꿔 준다.
- `Document`는 `page_content`(그 쪽의 글자)와 `metadata`(출처 정보 딕셔너리)를 가진 LangChain의 기본 글 단위다.
- 메타데이터 교체: 로더가 자동으로 넣은 메타데이터(`source`, 깨진 `title` 등)를 **버리고** 우리가 쓸 두 가지(`file_name`, `page_no`)만 새 딕셔너리로 넣는다.
- `page.metadata["page"] + 1`: 로더가 주는 `page`는 **0부터** 시작한다. PDF 뷰어의 쪽수(1부터, 표지=1쪽)에 맞추려고 1을 더한다. 문서의 진행 규칙("쪽수는 PDF 쪽수로 통일")을 코드로 구현한 줄이다.
- 전체 쪽 수가 **48**이어야 한다. 5개 문서의 쪽수 합이므로 자기 점검용 숫자다.

### 노트북 확인 셀

- 노트북의 `# print (pages) 함수니깐없음` 셀: `load_pages`는 **함수 정의**일 뿐이라 정의만 해서는 출력이 없다. 함수를 호출하는 `pages = load_pages()` 셀에서 결과가 나온다. (`print(pages)`처럼 함수 이름만 출력하려 해도 함수가 실행되는 것이 아니다.)
- `print(type(pages))` → `list`. 리스트의 한 칸이 PDF 한 쪽이다.
- `print(pages[0])`로 첫 쪽의 `page_content`와 `metadata`를 직접 본다.
- 확인할 것: `전체 쪽 수: 48`이 나오는가, `pages[0].metadata`에 `file_name`, `page_no`가 들어 있고 `page_no`가 1부터 시작하는가.

> ➕ **더 알아두기**
> `sorted(DATA_DIR.glob("*.pdf"))`의 정렬은 파일 이름의 문자 순서(유니코드 코드 순서)다. 숫자(`0`~`9`)가 `[`보다 앞이므로 `2024년 3월_...` → `240130...` → `240320...` → `240409...` → `[별첨]...` 순서로 읽힌다고 추론할 수 있다. 이 순서는 `FILE_CODES`의 번호(`doc1`~`doc5`)와 다르다. 전체 쪽 수(48)에는 영향이 없다.

## 주제 6: 청킹 — 쪽을 청크로 나누고 청크 ID 붙이기

```python
def chunk_pages(pages, chunk_size, chunk_overlap):
    """쪽을 청크로 나누고 청크마다 ID를 붙인다. 청크의 메타데이터는 원래 쪽의 것을 이어받는다."""
    splitter = RecursiveCharacterTextSplitter(chunk_size=chunk_size, chunk_overlap=chunk_overlap)
    chunks = splitter.split_documents(pages)

    counters = {}   # (파일명, 쪽수)별로 지금까지 만든 청크 수. 청크 번호(c0, c1...)를 매기는 데 쓴다.
    for chunk in chunks:
        key = (chunk.metadata["file_name"], chunk.metadata["page_no"])
        number = counters.get(key, 0)
        counters[key] = number + 1
        code = FILE_CODES[chunk.metadata["file_name"]]
        chunk.metadata["chunk_id"] = f"{code}-p{chunk.metadata['page_no']}-c{number}"
    return chunks


chunks = chunk_pages(pages, CHUNK_SIZE, CHUNK_OVERLAP)
lengths = [len(c.page_content) for c in chunks]
print(f"청크 수: {len(chunks)}")
print(f"길이 최소/평균/최대: {min(lengths)} / {sum(lengths) // len(lengths)} / {max(lengths)}")
print("첫 청크 메타데이터:", chunks[0].metadata)
```

### 청킹이 왜 필요한가

한 쪽(page)은 통째로 임베딩하기엔 길고, 한 쪽에 여러 주제가 섞여 있을 수 있다. 그래서 **적당한 길이의 조각(청크)**으로 나눠 각 조각을 임베딩·검색한다.

### `RecursiveCharacterTextSplitter(chunk_size, chunk_overlap)`

- `chunk_size=1000`: 청크 하나의 **최대 글자 수**다.
- `chunk_overlap=100`: 청크끼리 **겹치게 하는 글자 수**다. 경계에서 문장이 잘려도 앞뒤 문맥이 이어지게 하려는 장치다.
- `split_documents(pages)`: `Document` 리스트를 받아 쪼갠 `Document` 리스트를 돌려준다. **쪼개진 청크는 원래 쪽의 메타데이터(`file_name`, `page_no`)를 그대로 이어받는다.** 그래서 어떤 청크든 "어느 파일 몇 쪽에서 나왔는지" 알 수 있다.

> ➕ **더 알아두기**
> "Recursive"는 **큰 구분자부터 차례로 시도**한다는 뜻이다. 기본 구분자는 문단(`\n\n`) → 줄(`\n`) → 공백(` `) → 글자 단위(`""`) 순서다. 문단 경계로 나눠 보고 그래도 크기를 넘으면 줄, 그래도 넘으면 공백으로 더 잘게 나눈다. 그래서 문장 중간이 잘리는 일을 가능한 한 줄인다. 길이는 기본적으로 **문자 수(`len`)**로 센다. 토큰 수가 아니다. 실제 겹침 길이도 구분자 위치에 따라 100자와 달라질 수 있다(34번 노트북에서 확인한 내용과 같다).

### 청크 ID 붙이기

청크 ID는 `doc1-p4-c0` 모양이다. **문서 코드 - 쪽수 - 그 쪽 안에서의 청크 번호**다.

| 줄 | 하는 일 |
|---|---|
| `counters = {}` | (파일명, 쪽수)별로 지금까지 만든 청크 수를 기록할 빈 딕셔너리 |
| `key = (file_name, page_no)` | 튜플을 키로 쓴다. 같은 파일의 같은 쪽이면 같은 키 |
| `number = counters.get(key, 0)` | 이 키의 지금까지 청크 수. 아직 없으면 0 |
| `counters[key] = number + 1` | 다음 청크를 위해 1 올려 둔다 |
| `code = FILE_CODES[file_name]` | 파일 이름을 짧은 코드(`doc1`)로 바꾼다 |
| `chunk.metadata["chunk_id"] = f"..."` | 메타데이터에 ID를 추가한다 |

예: 첫 번째 문서의 4쪽에서 청크가 둘 나오면 `doc1-p4-c0`, `doc1-p4-c1`이다. 청크 번호는 **쪽이 바뀌면 다시 0부터** 시작한다. (`counters`의 키에 쪽수가 들어 있기 때문이다.)

> ➕ **더 알아두기**
> `f"{code}-p{chunk.metadata['page_no']}-c{number}"`에서 바깥 따옴표가 `"`이므로 안쪽에서는 `'page_no'`처럼 **작은따옴표**를 쓴다. 같은 종류의 따옴표를 쓰면 문자열이 거기서 끝난 것으로 해석된다. (파이썬 3.12부터는 f-문자열 안에서 같은 따옴표도 쓸 수 있다.)

### 청크 ID가 왜 중요한가

- 검색 결과에서 청크를 **식별**한다(10-16의 `search`가 `chunk_id`를 돌려준다).
- Agent가 여러 번 검색한 결과를 합칠 때 **중복 제거의 기준**이 된다(10-17의 `retrieve_node`).
- 실행 기록(`runs.jsonl`)에 어떤 청크를 근거로 썼는지 남긴다(10-18).
- 벡터 DB에 저장할 때 청크의 고유 ID로 쓴다(10-16의 `build_index`).

### 확인 셀

- 청크 수와 길이 분포(최소·평균·최대)가 나온다. **최대가 `CHUNK_SIZE`(1000)를 넘지 않아야** 한다.
- `sum(lengths) // len(lengths)`: `//`는 몫(정수 나눗셈)이다. 평균 길이를 정수로 보여 준다.
- 직접 `chunks[10].page_content`를 출력해서 문장이 어색하게 잘렸는지, 쪽번호(`- N -`) 같은 잡음이 붙었는지 눈으로 본다. 이 잡음(별첨 문서의 쪽번호, 통화신용정책의 꼬리말 반복)이 "미뤄 둔 개선 후보"에 적혀 있다.
- 청킹을 마치면 청크 ID와 메타데이터가 모두 들어 있는지 확인한다(2-3단계).

> ➕ **더 알아두기**
> 이 실습의 청킹은 **표 구조를 따로 판단하지 않는다.** 34번 노트북(10-14 (1) 글)에서 본 것처럼 길이만으로 자르면 표가 중간에서 갈릴 수 있다. 이 실습은 기준선을 만드는 단계라서 일부러 단순하게 두었고, 표·이미지 처리는 비교 실험으로 미뤘다.

## 주제 7: 완성 코드와 노트북의 차이

| 항목 | 완성 코드(`.py`) | 노트북(`.ipynb`) |
|---|---|---|
| 형태 | `# %%`로 셀이 나뉜 파이썬 파일 | 셀마다 "확인할 것"(마크다운) 포함 |
| `SLEEP_SEC` | 1초 | 4초 (호출 제한을 더 여유 있게 피하려는 값) |
| 설정 셀 | 주석 한 줄 | `FILE_CODES` 한 줄을 풀어 쓴 설명 주석 추가 |
| 평가 셀 | 검색 지표와 비교를 한 셀에서 실행 | 검색 지표(`eval_retrieval`)와 비교(`compare`)를 셀을 나눠 실행 |

> 📝 **기사 시험 포인트** (정보처리기사)
> 정보처리기사 실기에는 프로그래밍 언어(C·Java·Python) 코드를 읽고 출력 결과를 쓰는 문제가 있다. 이 글의 코드에 나온 문법(`enumerate`, 딕셔너리, 슬라이싱, 정수 나눗셈 `//`)은 Python 코드 읽기 연습에 그대로 쓰인다. 코드를 보고 **변수 값이 바뀌는 순서를 손으로 추적**하는 연습이 도움이 된다. (출제 언어와 범위는 최신 출제기준을 확인한다.)

## 정리

- 이 실습은 로딩 → 청킹 → 임베딩 → 검색 → 기본 RAG → Agent → 평가를 직접 구현하고 골든셋으로 비교하는 것이다. 기준선(pypdf + 고정 길이 청킹 + 기본 RAG)을 먼저 만들고, 개선은 비교 실험으로 미룬다.
- Agent는 도메인별 멀티에이전트가 아니라 **하나의 에이전트**다. 도구·규칙·규모가 같아서 나눌 이유가 없기 때문이다.
- 설정의 핵심은 `TARGET_NAMES`(읽을 문서 5개), `FILE_CODES`(파일 이름 → `doc1`~`doc5`), 그리고 `CHUNK_SIZE`·`CHUNK_OVERLAP`·`TOP_K`·`MAX_SEARCH` 같은 상수다.
- 로딩은 `PyPDFLoader`로 쪽마다 `Document`를 만들고, 메타데이터를 `file_name`과 `page_no`(1부터) 두 가지로 **바꿔 넣는다.** 5개 문서는 총 48쪽이다.
- 청킹은 `RecursiveCharacterTextSplitter`로 나누고, 청크는 쪽의 메타데이터를 이어받는다. 청크 ID는 `문서코드-p쪽수-c번호`이며, 쪽이 바뀌면 번호가 0부터 다시 시작한다.
- 한 번에 하나만 바꿔 비교하고, 단계마다 결과를 눈으로 확인한 뒤 넘어간다.

## ✅ 확인 질문

1. 이 실습에서 완성 코드를 먼저 만들고 노트북에서 한 줄씩 뜯어 공부하는 방식의 장점은 무엇인가요?
2. Agent를 도메인별 멀티에이전트로 나누지 않고 하나로 만든 이유는 무엇인가요?
3. "기준선 이후의 비교 실험으로 미룬다"는 것은 무슨 뜻이고, 왜 한 번에 하나만 바꿔서 비교해야 하나요?
4. `FILE_CODES = {name: f"doc{i}" for i, name in enumerate(TARGET_NAMES, start=1)}`가 만드는 딕셔너리의 키와 값은 무엇이며, 같은 일을 `for` 반복문으로 풀어 쓰면 어떻게 되나요?
5. `FILE_CODES`로 청크 ID를 짧게 만드는 이유는 무엇이고, 목록 중간에 문서를 끼워 넣으면 어떤 문제가 생기나요?
6. `load_pages`에서 `page.metadata["page"] + 1`을 하는 이유는 무엇이고, 로더가 넣은 메타데이터를 버리고 두 가지만 남기는 이유는 무엇인가요?
7. 함수를 정의하는 셀을 실행하면 출력이 없는데, 결과를 보려면 어떻게 해야 하나요?
8. `chunk_size=1000`, `chunk_overlap=100`의 의미를 설명하고, 겹침(overlap)을 두는 이유를 말해 보세요.
9. `chunk_pages`에서 `counters` 딕셔너리와 `counters.get(key, 0)`이 하는 일은 무엇이며, 청크 번호가 쪽이 바뀔 때 0부터 다시 시작하는 이유는 무엇인가요?
10. 청크 ID(`doc1-p4-c0`)를 이후 단계에서 어디에 쓰나요?
11. 청킹 결과를 확인할 때 눈으로 봐야 하는 것(최대 길이, 문장 잘림, 쪽번호 잡음)은 각각 왜 확인하나요?
