---
tags: [rag, practice, chroma, retrieval, lcel]
til: v2 2026-10-16
---

# 2026-10-16 (1)RAG Agent 실습2 임베딩 검색 기본 RAG(Chroma, build index, search, LCEL)
> 작성일: 2026-10-16

## 🔗 관련 글

- [RAG Agent 실습1 설계와 로딩, 청킹](2026-10-15%20(1)RAG_Agent_실습1_설계와_로딩_청킹(Document,_TARGET_NAMES,_FILE_CODES,_청크_ID).md) — 이 글의 입력(`pages`, `chunks`, `chunk_id`, 설정 상수)을 만든 앞 단계다.
- [Vector DB와 RAG 심화](2026-09-03%20Vector%20DB와%20RAG%20심화.md) — 임베딩·벡터 DB·유사도 검색의 개념 글이다. 이 글은 그 개념을 Chroma 코드로 구현한다.
- [LangChain 기초 - Runnable, LCEL 파이프라인, 실습 4종](2026-09-01%20LangChain%20기초%20-%20Runnable,%20LCEL%20파이프라인,%20실습%204종.md) — `answer_prompt | llm | StrOutputParser()`의 `|` 파이프라인이 이 글의 기본 RAG에 그대로 쓰인다.
- [RAG Agent(Retriever Tool, create_retriever_tool, Checkpointer)](2026-09-10%20(1)%20RAG_Agent(Retriever_Tool,_create_retriever_tool,_Checkpointer).md) — 이전에는 검색을 도구로 만들어 Agent가 부르게 했다. 이 실습은 검색 함수를 직접 만들어 그래프의 노드로 쓴다(10-17).

## ❓ 아직 모르겠는 것

대화에서 사용자가 질문하거나 막힌 부분은 따로 기록되어 있지 않다. 이 단계의 코드는 실습을 마친 뒤 모두 이해한 상태다. 복습하면서 막힌 곳이 생기면 이 자리에 추가한다.

- 

## 주제 1: 임베딩과 벡터 DB에 저장하기(`build_index`)

```python
def build_index(pages, chunk_size, chunk_overlap):
    """청킹 -> 임베딩 -> Chroma 저장. 같은 설정으로 이미 저장돼 있으면 다시 만들지 않는다.

    컬렉션 이름에 청킹 설정을 넣어서, 청킹을 바꿔 비교할 때 서로 섞이지 않게 한다.
    (재사용 여부는 청크 개수로만 판단한다. PDF나 설정을 바꿨는데 개수가 같다면 outputs/chroma_db 를 지우고 다시 실행한다.)
    """
    chunks = chunk_pages(pages, chunk_size, chunk_overlap)
    name = f"finance_{chunk_size}_{chunk_overlap}"

    def open_collection():
        return Chroma(collection_name=name, embedding_function=embeddings, persist_directory=str(CHROMA_DIR))

    vectorstore = open_collection()
    if len(vectorstore.get()["ids"]) != len(chunks):         # 저장된 개수가 다르면 처음부터 다시 만든다
        vectorstore.delete_collection()
        vectorstore = open_collection()
        ids = [c.metadata["chunk_id"] for c in chunks]
        for start in range(0, len(chunks), 50):               # 50개씩 나눠서 임베딩 (호출 제한 예방)
            vectorstore.add_documents(chunks[start:start + 50], ids=ids[start:start + 50])
    return vectorstore, chunks


vectorstore, chunks = build_index(pages, CHUNK_SIZE, CHUNK_OVERLAP)
print("벡터 DB에 저장된 청크 수:", len(vectorstore.get()["ids"]), "(위의 청크 수와 같아야 한다)")
```

### 임베딩과 벡터 DB가 하는 일

- **임베딩(embedding)**: 글을 숫자의 나열(벡터)로 바꾼다. 의미가 비슷한 글은 벡터도 가까워진다. 이 코드에서는 `GoogleGenerativeAIEmbeddings`(모델 `gemini-embedding-2`)가 한다.
- **벡터 DB(Chroma)**: 벡터와 원본 글, 메타데이터를 저장하고 "질문 벡터와 가까운 것"을 빠르게 찾아 준다.
- `persist_directory=str(CHROMA_DIR)`: 디스크의 폴더(`outputs/chroma_db`)에 저장한다. 그래서 프로그램을 껐다 켜도 다시 임베딩하지 않아도 된다.

### 한 줄씩 보기

- 함수 안에서 `chunk_pages`를 다시 호출한다. `build_index`가 청킹까지 맡고, `(vectorstore, chunks)` 두 가지를 돌려준다.
- `name = f"finance_{chunk_size}_{chunk_overlap}"`: **컬렉션 이름에 청킹 설정을 넣는다.** 컬렉션은 Chroma 안의 저장 공간(표의 이름 같은 것)이다. 청크 크기를 바꿔 비교할 때 서로 다른 컬렉션(`finance_500_50`, `finance_1000_100`)에 저장되므로 **결과가 섞이지 않는다.**
- `open_collection()`은 함수 안에 정의한 **안쪽 함수**다. 같은 설정으로 컬렉션을 여는 코드를 두 번 쓰지 않으려고 묶었다. 바깥 함수의 `name`을 가져다 쓴다.
- `vectorstore.get()["ids"]`: 저장된 모든 청크의 ID 목록이다. 길이를 세면 저장된 개수다.
- **재사용 판단**: 저장된 개수와 방금 만든 청크 수가 **같으면** 이미 저장한 것으로 보고 임베딩을 다시 하지 않는다. 다르면 `delete_collection()`으로 지우고 **처음부터 다시 만든다.**
- `ids = [c.metadata["chunk_id"] for c in chunks]`: 10-15에서 만든 청크 ID를 Chroma의 고유 ID로 쓴다.
- `for start in range(0, len(chunks), 50)`: 0, 50, 100, … 으로 건너뛰며 `chunks[start:start + 50]`처럼 **50개씩** 잘라 넣는다. 임베딩은 API 호출이므로 한꺼번에 많이 보내면 호출 제한에 걸릴 수 있어서 나눈다. `add_documents(문서들, ids=ID들)`가 임베딩하고 저장한다.

> ➕ **더 알아두기**
> - 슬라이싱(`chunks[start:start + 50]`)은 끝 번호가 리스트 길이를 넘어도 **에러가 나지 않고** 있는 데까지만 잘라 준다. 그래서 마지막 묶음이 50개보다 적어도 별도 처리가 필요 없다.
> - 재사용 판단이 **개수만** 비교한다는 점이 한계다. docstring에도 적혀 있듯이 PDF나 설정을 바꿨는데 개수가 우연히 같으면 옛 벡터를 그대로 쓰게 된다. 그럴 때는 `outputs/chroma_db` 폴더를 지우고 다시 실행한다.
> - 같은 셀을 다시 실행하면 임베딩을 다시 하지 않아 훨씬 빨리 끝난다.

### 확인할 것

- 벡터 DB에 저장된 청크 수가 위의 청크 수와 같은가
- 같은 셀을 다시 실행했을 때 빨리 끝나는가(저장된 것을 재사용)

## 주제 2: 검색 — 질문에서 상위 k개 청크

```python
def make_search_fn(vectorstore):
    """벡터 DB 하나를 감싸서 search(query, k) 함수를 만든다. 결과는 평가하기 쉽게 딕셔너리 리스트로 바꾼다."""
    def search(query, k=TOP_K):
        docs = vectorstore.similarity_search(query, k=k)    # 질문과 의미가 가까운 청크 k개 (가까운 순서)
        return [
            {
                "chunk_id": d.metadata["chunk_id"],
                "file_name": d.metadata["file_name"],
                "page_no": d.metadata["page_no"],
                "text": d.page_content,
            }
            for d in docs
        ]
    return search


search = make_search_fn(vectorstore)

# 검색만 단독으로 확인한다. 답변을 붙이기 전에 이 결과가 정답 근처인지 눈으로 본다.
for item in search("지방은행이 시중은행으로 전환할 때 예비인가를 받아야 하나요?", k=3):
    print(item["chunk_id"], "|", item["file_name"][:20], "| p.", item["page_no"])
    print("   ", item["text"][:80].replace("\n", " "))
```

### 왜 함수를 돌려주는 함수인가(`make_search_fn`)

`make_search_fn(vectorstore)`는 벡터 DB 하나를 받아 **`search(query, k)`라는 함수를 만들어 돌려준다.** 안쪽 `search`는 바깥의 `vectorstore`를 기억하고 있다(클로저). 이렇게 만들면 `search(질문)`처럼 벡터 DB를 신경 쓰지 않고 호출할 수 있고, **다른 벡터 DB(청킹 크기가 다른 것)로도 같은 모양의 검색 함수를 만들 수 있다.** 10-18의 청킹 크기 비교 실험에서 `make_search_fn(store)`로 바로 쓰인다.

### 한 줄씩 보기

- `vectorstore.similarity_search(query, k=k)`: 질문을 임베딩해 **의미가 가까운 청크 k개**를 가까운 순서대로 돌려준다. 질문 임베딩에만 API를 쓰므로 가볍다.
- 결과를 `Document` 그대로 쓰지 않고 **딕셔너리 리스트**로 바꾼다: `chunk_id`, `file_name`, `page_no`, `text`. 평가(정답 쪽과 비교)하기 쉽고, 프롬프트에 출처를 붙이기 쉽다.
- `k=TOP_K`: 기본값은 상수 `TOP_K`(5)다. 호출할 때 `k=3`처럼 바꿀 수 있다.
- 확인 출력: `item["file_name"][:20]`은 파일 이름의 앞 20글자만, `item["text"][:80].replace("\n", " ")`은 본문 앞 80글자를 줄바꿈 없이 보여 준다.

> ➕ **더 알아두기**
> `def search(query, k=TOP_K)`의 기본값은 **함수를 정의하는 시점**에 한 번 정해진다. 나중에 전역 변수 `TOP_K`를 바꿔도 이미 만들어진 `search`의 기본값은 그대로다. 값을 바꿔 쓰려면 `search(질문, k=3)`처럼 호출할 때 직접 넘긴다.

### 확인할 것

- 검색 결과에 **정답 문서·쪽이 상위에 있는가.** 이 단계의 목적은 답변을 붙이기 전에 **검색만 단독으로** 점검하는 것이다. 검색이 틀리면 뒤의 답변은 틀릴 수밖에 없다.
- 질문을 바꿔 가며 `search("...", k=3)` 결과의 `file_name`, `page_no`를 직접 확인해 본다.

## 주제 3: 기본 RAG — 검색 한 번 → 답변

### 근거 문구와 컨텍스트 만들기

```python
NO_ANSWER = "제공된 문서로는 확인할 수 없습니다."   # 근거가 없을 때 쓸 고정 문구. 보류 여부를 이 문구로 판단한다.


def format_context(docs):
    """검색된 청크를 프롬프트에 넣을 한 덩어리 문자열로 만든다. 출처(파일명, 쪽수)를 청크마다 붙인다."""
    return "\n\n".join(
        f"[{i}] (출처: {d['file_name']}, {d['page_no']}쪽)\n{d['text']}"
        for i, d in enumerate(docs, start=1)
    )
```

- `NO_ANSWER`: 근거가 없을 때 쓰는 **고정 문구**다. 모델에게 "근거가 없으면 정확히 이 문장만 답하라"고 시키고, 코드에서는 답변에 이 문구가 들어 있는지로 **보류 여부를 판단**한다. 문구를 한 곳에 상수로 두었기 때문에 프롬프트와 판단 코드가 어긋나지 않는다.
- `format_context`: 검색된 청크들을 프롬프트에 넣을 하나의 긴 글로 만든다. 각 청크 앞에 `[1] (출처: 파일명, N쪽)`을 붙이고, 청크 사이는 빈 줄(`"\n\n".join`)로 구분한다. `enumerate(docs, start=1)`로 번호를 1부터 매긴다.
- **출처를 청크마다 붙이는 이유**: 모델이 답변 끝에 `(출처: 파일명, N쪽)`을 쓰려면 근거에 출처 정보가 있어야 한다.

### 프롬프트와 체인

```python
answer_prompt = ChatPromptTemplate.from_messages([
    ("system",
     "당신은 제공된 근거만 사용해 한국어로 답하는 도우미입니다.\n"
     "- 근거에 있는 내용만 답하고, 근거에 없는 내용은 추측하지 마세요.\n"
     "- 답변 끝에 사용한 근거의 출처를 '(출처: 파일명, N쪽)' 형식으로 표시하세요.\n"
     f"- 근거로 답할 수 없으면 정확히 '{NO_ANSWER}' 라고만 답하세요."),
    ("human", "질문: {question}\n\n근거:\n{context}"),
])
answer_chain = answer_prompt | llm | StrOutputParser()   # 프롬프트 -> 모델 -> 문자열 (LCEL 파이프라인)
```

- `ChatPromptTemplate.from_messages([...])`: 역할(`system`, `human`)과 내용의 쌍으로 프롬프트 틀을 만든다. `system`은 모델의 행동 규칙, `human`은 사용자 입력이다.
- system 메시지의 규칙 세 가지: ① 근거에 있는 내용만 답한다(추측 금지) ② 답변 끝에 출처를 `(출처: 파일명, N쪽)` 형식으로 쓴다 ③ 근거로 답할 수 없으면 정확히 `NO_ANSWER`만 답한다.
- **`f` 접두사가 마지막 줄에만 붙어 있다.** 문자열 여러 개를 이어서 쓰면 파이썬이 하나로 합친다. 마지막 줄만 `f"..."`라서 `{NO_ANSWER}`가 값으로 채워진다. 나머지 줄에는 중괄호가 없다.
- human 메시지의 `{question}`, `{context}`는 f-문자열이 아니라 **프롬프트 템플릿의 빈칸**이다. `invoke`할 때 딕셔너리로 값을 채운다.
- `answer_prompt | llm | StrOutputParser()`: **LCEL(LangChain Expression Language) 파이프라인**이다. 앞의 출력이 뒤의 입력이 된다. 프롬프트가 빈칸을 채워 메시지를 만들고 → 모델이 답하고 → `StrOutputParser`가 모델 응답 객체에서 **글자(문자열)만** 꺼낸다.

### `basic_rag`

```python
def basic_rag(question, search_fn=None, k=TOP_K):
    """검색을 딱 한 번 하고 답한다. 비교 실험의 기준선이다."""
    search_fn = search_fn or search
    start = time.perf_counter()
    docs = search_fn(question, k)                                        # 1) 검색
    answer = answer_chain.invoke({"question": question, "context": format_context(docs)}).strip()   # 2) 답변
    return {
        "answer": answer,
        "docs": docs,
        "abstained": NO_ANSWER in answer,
        "search_count": 1,
        "seconds": round(time.perf_counter() - start, 2),
    }


result = basic_rag("지방은행이 시중은행으로 전환할 때 예비인가를 받아야 하나요?")
print(result["answer"])
print("검색된 청크:", [d["chunk_id"] for d in result["docs"]], "| 시간:", result["seconds"], "초")
```

- 검색을 **딱 한 번** 하고 답한다. 이것이 비교 실험의 **기준선**이다. Agent(10-17)는 이 방식에 "근거 판단·재검색·보류"를 더한 것이다.
- `search_fn = search_fn or search`: 검색 함수를 인자로 넘기지 않으면(`None`) 전역 `search`를 쓴다. `or`는 앞 값이 비어 있거나 `None`이면 뒤 값을 고른다. 청킹 크기 비교 실험(10-18)에서 다른 검색 함수를 넘길 수 있게 하려는 설계다.
- `time.perf_counter()`: 시간을 정밀하게 재는 시계다. 시작과 끝의 차이를 `round(..., 2)`로 소수 둘째 자리까지 기록한다.
- `answer_chain.invoke({"question": ..., "context": ...})`: 딕셔너리로 프롬프트의 빈칸을 채워 체인을 실행한다. `.strip()`은 앞뒤 공백·줄바꿈을 지운다.
- 돌려주는 값의 모양: `answer`, `docs`, `abstained`, `search_count`(항상 1), `seconds`. **Agent의 `run_agent`도 같은 모양으로 돌려주도록** 맞췄다. 그래야 같은 평가 코드로 둘을 비교할 수 있다.
- `abstained`: `NO_ANSWER in answer`. 답변 안에 보류 문구가 **들어 있으면** 보류로 본다(문자열 포함 검사).

### 확인할 것

- 답변이 검색된 근거와 맞는가, 출처(파일명·쪽수)가 실제 근거 청크의 것과 같은가
- 문서에 없는 내용(예: 인가 수수료)을 물으면 `제공된 문서로는 확인할 수 없습니다.`로 답하는가

> ➕ **더 알아두기**
> 기본 RAG의 `abstained`는 "모델이 보류 문구를 말했는가"로 판단하므로 **모델이 지시를 지켰는지에 달려 있다.** 반면 Agent의 보류(10-17)는 코드의 분기(`abstain_node`)로 결정되어 항상 같은 문구를 돌려준다. 이 차이가 10-18의 비교 결과를 해석할 때 중요하다.

## 정리

- 임베딩은 글을 의미가 반영된 벡터로 바꾸고, 벡터 DB(Chroma)는 그 벡터를 저장해 질문과 가까운 청크를 찾는다. 저장 폴더(`persist_directory`)가 있어서 다시 임베딩하지 않아도 된다.
- `build_index`는 컬렉션 이름에 청킹 설정을 넣어 비교 실험이 섞이지 않게 하고, 저장된 개수가 청크 수와 같으면 재사용한다. 50개씩 나눠 넣어 호출 제한을 예방한다. 개수만 비교하므로 PDF나 설정을 바꾸고 개수가 같다면 `chroma_db`를 지워야 한다.
- `make_search_fn`은 벡터 DB를 감싼 `search(query, k)` 함수를 만든다. 결과는 `chunk_id`, `file_name`, `page_no`, `text` 딕셔너리 리스트다. 답변을 붙이기 전에 검색만 단독으로 눈으로 확인한다.
- 기본 RAG는 `format_context`(출처 붙인 근거 덩어리) + `answer_prompt` + LCEL(`prompt | llm | StrOutputParser()`)로 검색 한 번에 답한다. 근거가 없으면 고정 문구 `NO_ANSWER`로 답하게 하고, 이 문구의 포함 여부로 보류를 판단한다.
- `basic_rag`와 `run_agent`는 같은 모양의 결과를 돌려줘 같은 평가 코드로 비교할 수 있다.

## ✅ 확인 질문

1. 임베딩과 벡터 DB는 각각 무엇을 하고, `persist_directory`를 지정하면 무엇이 달라지나요?
2. `build_index`에서 컬렉션 이름에 `chunk_size`와 `chunk_overlap`을 넣는 이유는 무엇인가요?
3. 이미 저장된 벡터 DB를 재사용할지 어떻게 판단하며, 이 방식의 한계와 해결 방법은 무엇인가요?
4. 청크를 50개씩 나눠 `add_documents`에 넣는 이유는 무엇이고, `range(0, len(chunks), 50)`과 `chunks[start:start + 50]`은 어떻게 동작하나요?
5. `make_search_fn`처럼 함수를 돌려주는 함수로 검색 함수를 만들면 어떤 점이 편한가요?
6. 검색 결과를 `Document` 그대로 쓰지 않고 `chunk_id`, `file_name`, `page_no`, `text` 딕셔너리로 바꾸는 이유는 무엇인가요?
7. 답변을 붙이기 전에 검색만 단독으로 점검하는 이유는 무엇인가요?
8. `NO_ANSWER`를 상수로 두고 프롬프트와 보류 판단에 함께 쓰는 이유는 무엇인가요?
9. `format_context`가 청크마다 `[번호] (출처: 파일명, N쪽)`을 붙이는 이유는 무엇인가요?
10. `answer_prompt | llm | StrOutputParser()`에서 각 부분이 하는 일을 설명해 보세요. system 메시지의 마지막 줄에만 `f`가 붙은 이유는 무엇인가요?
11. `basic_rag`가 `search_fn = search_fn or search`로 검색 함수를 받는 이유와, 돌려주는 값의 모양을 Agent와 맞춘 이유는 무엇인가요?
