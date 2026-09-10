---
tags: [langgraph, rag, retriever-tool, checkpointer, agent]
til: v2 2026-09-10
---

# RAG Agent (Retriever Tool, create_retriever_tool, Checkpointer)
> 작성일: 2026-09-10

## 🔗 관련 글

- [(2) Memory와 State 관리(Checkpointer, Store, 대화 요약)](2026-09-09_%282%29_Memory와_State_관리%28Checkpointer,_Store,_대화_요약%29.md) — Checkpointer의 뒷부분(오늘 RAG Agent에 그대로 응용)
- [2026-09-02 Tool Agent와 RAG 기초](2026-09-02%20Tool%20Agent와%20RAG%20기초.md) — Tool의 name/description/func 구조, RAG 두 단계(인덱싱/질의응답)의 뒷부분
- [2026-09-03 Vector DB와 RAG 심화](2026-09-03%20Vector%20DB와%20RAG%20심화.md) — Chroma 벡터 저장소의 뒷부분(오늘 컬렉션 개념으로 확장)
- [2026-07-31 n8n 박스오피스 알림 구축](2026-07-31_n8n_박스오피스_알림_구축.md) — n8n AI Agent 노드의 뒷부분(오늘 코드로 직접 구현한 것과 비교)

## RAG란 무엇인가

RAG = Retrieval(검색) + Augmented(보강된) + Generation(생성). LLM이 학습하지 않은 문서(예: 회사 내부 규정)의 내용을, 검색해서 프롬프트에 보강한 다음 답을 생성하게 하는 방식이다.

## 기존 RAG vs RAG Agent

| 구분 | 기본 RAG 파이프라인 | RAG Agent |
|------|----------|-----------|
| 검색 판단 | 항상 검색 | LLM이 필요 여부 판단 |
| 대화 기억 | 별도 구현 필요 | Checkpointer로 유지 |
| 검색 도구 | Retriever를 직접 호출 | Retriever를 Tool로 호출 |

기본 RAG는 질문이 뭐든 무조건 검색부터 하고, 검색된 내용을 LLM이 다듬어서 답한다. RAG Agent는 그 앞에 "이 질문에 검색이 필요한가?"라는 판단 단계가 추가된 것이다 — LLM이 다듬는 능력은 그대로 있고, "애초에 검색할지 말지 정하는 능력"이 새로 생긴다.

판단 기준은 "LLM이 답을 아느냐"가 아니라 **"이 질문이 Retriever Tool의 description이 다루는 문서 범위 안에 들어가느냐"** 다. 예를 들어 사내 규정 검색기는 "안녕하세요" 같은 질문엔 애초에 답할 수 있는 문서가 없으므로 호출을 건너뛴다.

## Tool의 구조 — 이름표, 설명글, 실제 코드

Tool 객체 하나에는 세 부품이 들어있다.

1. **name** (이름표, 문자열) — LLM이 도구를 식별하고 호출할 때 쓰는 이름
2. **description** (설명글, 문자열) — LLM이 "이 도구를 언제 써야 하는지" 판단하는 근거
3. **func** (실제 실행 코드) — 호출됐을 때 진짜로 돌아가는 로직

LLM은 대화 중 name/description **만** 읽고 호출 여부를 판단하고, 호출하기로 하면 그제서야 func가 실행된다.

```python
print(type(retriever_tool))        # Tool 객체 (str 아님)
print(retriever_tool.name)         # "search_company_rules" -> str
print(retriever_tool.description)  # 설명 문자열 -> str
print(retriever_tool.func)         # <function ...> -> 실제 코드
```

## Retriever를 Tool로 포장하기 — create_retriever_tool

retriever는 `vectorstore.as_retriever()`로 얻는 객체로, **이미 실제 검색(유사도 계산) 코드를 갖고 있다.** 다만 이름표와 설명글이 없어서 LLM에게 "이런 도구가 있다"고 알려줄 방법이 없다.

`create_retriever_tool(retriever, name=..., description=...)`은 이 retriever의 실제 코드는 그대로 두고, 그 위에 name/description만 붙여서 Tool 모양으로 포장하는 함수다.

```python
retriever_tool = create_retriever_tool(
    retriever,
    name="search_company_rules",
    description=(
        "회사 내부 규정에서 근무 시간, 휴가, 복리후생, 출장비, "
        "보안 및 인사 정책을 검색한다. "
        "회사 규정이나 사내 정책에 관한 질문을 받았을 때 사용한다."
    ),
)
```

**중요한 시점 구분**: `create_retriever_tool(...)`을 호출하는 순간엔 검색이 전혀 일어나지 않는다. 이건 "포장"만 하는 시점이다. 실제 검색(유사도 계산)은 나중에 이 Tool이 진짜로 호출될 때(`retriever_tool.invoke(...)` 또는 Agent가 판단해서 부를 때)만 실행된다.

내부적으로는 이런 식으로 동작한다고 이해하면 된다(단순화):

```python
def create_retriever_tool(retriever, name, description):
    def retrieval_function(query):           # 라이브러리가 대신 만들어주는 함수 (클로저)
        docs = retriever.invoke(query)
        return "\n\n".join(d.page_content for d in docs)

    return Tool(name=name, description=description, func=retrieval_function)
```

> ➕ **더 알아두기 — @tool 데코레이터와의 차이**
> `@tool` 방식은 **내가 직접 쓴 함수** 위에 데코레이터를 붙여서 Tool로 변환한다. docstring이 자동으로 description이 된다.
> ```python
> @tool(parse_docstring=True)
> def search_weather(city: str) -> str:
>     """주어진 도시의 현재 날씨를 검색한다."""
>     ...
> ```
> `create_retriever_tool`은 **라이브러리가 미리 만들어둔 함수**를 호출만 하는 것이다. retriever는 우리가 만든 함수가 아니라 Chroma가 만든 객체라서 docstring을 적을 자리가 없기 때문에, description을 인자로 직접 넘긴다. 둘 다 최종 결과물(Tool 객체)은 동일하다 — "이름표 붙이는 개념"은 같고, 그 작업을 내가 손으로 하느냐(흔치 않은 로직) 이미 만들어진 함수가 대신 하느냐(흔한 반복 로직)의 차이일 뿐이다. `sorted()`를 쓸 때 정렬 알고리즘을 직접 안 짜는 것과 같은 원리다.

## 검색 결과 형식 정하기 — document_prompt, document_separator

Retriever는 질문 하나에 여러 개(k=3 등)의 문서 조각을 돌려주고, 이 조각들은 **하나의 문자열로 합쳐져서** LLM에게 전달된다.

- **document_prompt**: 문서 조각 하나를 어떤 형식으로 보여줄지 정하는 템플릿
- **document_separator**: 여러 조각을 합칠 때 조각 사이에 넣을 구분자

```python
document_prompt = PromptTemplate.from_template(
    "[규정명: {policy_name}]\n"
    "[출처: {source}]\n"
    "{page_content}"
)

retriever_tool = create_retriever_tool(
    retriever,
    name="search_company_rules",
    description=(...),
    document_prompt=document_prompt,
    document_separator="\n\n---\n\n",
)
```

구분자가 필요한 이유: 구분자 없이 이어붙이면 "...25일로 한다.[규정명: 복리후생규정]..."처럼 한 문서의 끝과 다음 문서의 시작이 아무 여백 없이 붙어버려서, LLM이 문서 경계를 명확히 못 잡고 서로 다른 규정 내용을 섞어서 착각할 위험이 있다. `\n\n---\n\n` 같은 눈에 띄는 구분자를 넣으면 "여기서 한 문서가 끝나고 새 문서가 시작된다"가 명확해진다.

## 벡터 저장소와 컬렉션의 관계

- **벡터 저장소(vector store, 예: `PERSIST_DIR = "./chroma_db"`)**: 문서를 벡터로 저장하는 시스템/위치 전체 — 파일 캐비닛
- **컬렉션(collection, 예: `COLLECTION_NAME`)**: 그 저장소 안에서 데이터를 나눠 담는 서랍 하나

하나의 벡터 저장소(캐비닛) 안에 여러 컬렉션(서랍)이 같이 들어있을 수 있다. 실제로 실습 중 `chroma_db` 안에 `spri_ai_brief`(90개 청크, 09번 노트북에서 생성)와 `company_rules_search_advanced`(14번에서 생성)가 함께 존재하는 걸 확인했다.

컬렉션은 문서 1개당 1개씩 생기는 게 아니라, **여러 문서(1개든 100개든)를 모아서 벡터화한 결과 전체가 컬렉션 하나**에 담긴다.

```python
documents = []
for path in sorted(DATA_DIR.glob("*.md")):        # 파일 여러 개를 순회
    ...
    documents.extend(splitter.split_documents(loaded_documents))  # 하나의 리스트에 계속 누적

vectorstore = Chroma.from_documents(
    documents=documents,             # 누적된 전체를 한 번에
    embedding=embeddings,
    collection_name=COLLECTION_NAME, # 컬렉션 이름은 하나만 지정
    client=client,
)
```

임베딩(embedding)과 컬렉션은 다른 개념이다. 임베딩은 텍스트를 벡터로 바꾸는 **동작**(문서 저장 시에도, 질문 검색 시에도 반복적으로 일어남)이고, 컬렉션은 그 결과를 모아두는 **그릇**(한 번 만들고 계속 재사용)이다.

## 문서 기반 답변만 허용하기 — System Prompt로 제한

기본 Agent는 검색 결과가 부족해도 LLM이 자기 지식으로 채워 답할 수 있다. 정확성이 중요한 경우(사내 문서 QA 등)엔 이게 위험할 수 있어서, System Prompt로 "문서에 없으면 모른다고 답해라"를 강제할 수 있다. 새로운 도구나 구조가 필요한 게 아니라 **프롬프트 문구 하나로 제어하는 것**이다.

```python
strict_rag_agent = create_agent(
    model=llm,
    tools=[retriever_tool],
    system_prompt="""
사내 규정과 관련된 질문에는 search_company_rules 도구를 반드시 사용한다.
답변은 검색된 사내 규정만 근거로 작성하고, 참고한 규정명과 출처를 포함한다.
검색 결과에 질문에 답할 충분한 근거가 없으면
'해당 정보를 사내 규정에서 찾을 수 없습니다.'라고 답한다.
""",
)
```

- **개방형**: System Prompt 없이, 문서 + 일반 지식으로 폭넓게 답변
- **제한형**: System Prompt로 문서 기반 답변만 강제, 정확성 중시할 때

## 대화를 기억하는 RAG Agent — Checkpointer

이건 새로운 개념이 아니라, 이전 노트북(Memory/State 관리)에서 배운 Checkpointer의 응용이다. `checkpointer=MemorySaver()`를 추가하면 "아까 그거 좀 더 알려줘" 같은 후속 질문에서 이전 대화 맥락을 이어갈 수 있다.

```python
rag_agent = create_agent(
    model=llm,
    tools=[retriever_tool],
    checkpointer=MemorySaver(),
)

config = {"configurable": {"thread_id": "rag-session-1"}}
result = rag_agent.invoke({"messages": [("user", "재택근무는 일주일에 몇 번 가능해?")]}, config=config)
```

대화 흐름 예시:
- "재택근무 가능 횟수?" → 검색 필요 → Tool 호출 → 규정 기반 답변
- "그 중에서 가장 중요한 것만" → 이전 답변 맥락에서 선별 → 새로 검색할 필요 없음
- "영어로 번역해줘" → Tool 호출 없이 대화 맥락으로 처리

> ➕ **더 알아두기 — n8n의 AI Agent 노드와 같은 구조**
> n8n에서 "AI Agent" 노드 아래 Tool 노드를 연결해본 것과 오늘 배운 구조는 개념적으로 동일하다.
> - n8n의 AI Agent 노드 = `create_agent(model=llm, tools=[...])`
> - n8n의 Tool 노드들 = `retriever_tool`
> - n8n이 "어떤 도구를 쓸지 LLM이 알아서 고른다"고 안내하는 그 동작 = 오늘 배운 판단 로직
>
> n8n은 이 과정을 미리 만들어진 UI 부품으로 조립하게 해주고, 코드로 짜면 `document_prompt`/`document_separator`처럼 n8n 노드 안에 감춰져 있던 세부 설정까지 직접 손으로 만질 수 있다는 차이가 있다.

## 실습: SPRi AI Brief 대화형 RAG Agent 설계

과제: SPRi AI Brief 9월호/10월호/11월호(PDF)로 AI 산업 동향을 검색하는 대화형 RAG Agent 구성.

오늘 배운 구조(imports, retry_client, 벡터 저장소 확인/생성, Retriever Tool, Agent+Checkpointer)를 그대로 재사용하되, 바뀌는 부분만 결정하며 설계했다.

- **로더 교체**: `.md` 파일용 `TextLoader` → PDF용 `PyPDFLoader`. PyPDFLoader는 파일 하나당 **페이지 단위로 여러 Document**를 만든다(TextLoader는 파일 하나 = Document 하나였음).
- **새 컬렉션 이름 필요**: 기존 `chroma_db` 안에 이미 `spri_ai_brief`, `company_rules_search_advanced` 컬렉션이 있어서, 겹치지 않는 새 이름(`spri_ai_brief_rag_agent`)을 지어야 했다. 컬렉션 이름 자체는 임의로 지어도 되지만, PERSIST_DIR/DATA_DIR 값은 실제 파일 위치와 정확히 일치해야 한다.
- **모델 기본값 규칙 적용**: 원본 노트북은 `MODEL_NAME = "gemini-3.6-flash"`(하루 20회 한도)였지만, 반복 실습에 여유가 있는 `gemini-3.5-flash-lite`(하루 500회)로 교체했다.
- Tool의 name/description은 AI 산업 동향 도메인에 맞게 새로 작성(`search_ai_industry_trends`).

디버깅 중 겪은 것: `load_dotenv()` **호출**을 빠뜨려서(import만 하고 실행을 안 함) `genai.Client()`에서 "No API key was provided" 에러가 났다 — `from dotenv import load_dotenv`(가져오기)와 `load_dotenv()`(실행)는 별개라는 걸 다시 확인했다.

## 오늘 배운 것 요약

| 개념 | 역할 |
|---|---|
| Tool | 이름표(name) + 설명글(description) + 실제 코드(func)로 이뤄진 상자 |
| create_retriever_tool | retriever의 실제 검색 코드는 그대로 두고 name/description만 붙여 포장 |
| document_prompt/separator | 여러 검색 결과를 하나의 문자열로 보기 좋게 정리 |
| 컬렉션 vs 벡터 저장소 | 컬렉션 = 서랍(용도별 분리), 벡터 저장소 = 캐비닛(공유 가능) |
| System Prompt 제한 | 새 도구 없이 프롬프트만으로 "문서 기반 답변만" 강제 |
| Checkpointer | 기존 개념 재사용, RAG와 결합하면 검색 내용을 기억하며 대화 |

## ✅ 확인 질문

1. RAG Agent에서 LLM이 판단하는 기준은 "LLM이 답을 아느냐"가 아니라 무엇인가?
2. `create_retriever_tool(...)`을 호출하는 시점과 실제 검색이 실행되는 시점은 왜 다른가?
3. `@tool` 데코레이터 방식과 `create_retriever_tool` 방식의 공통점과 차이점은 무엇인가?
4. `document_separator`가 없으면 어떤 문제가 생길 수 있는가?
5. 벡터 저장소(PERSIST_DIR)와 컬렉션(COLLECTION_NAME)의 관계를 비유로 설명하라.
6. 문서 100개를 임베딩하면 컬렉션이 100개 생기는가, 1개 생기는가? 왜인가?
7. TextLoader와 PyPDFLoader의 결과물 구조(Document 개수)는 어떻게 다른가?
8. `from dotenv import load_dotenv`와 `load_dotenv()`는 각각 무엇을 하는가?
