---
tags: [rag, evaluation, golden-set, hit-rate, mrr]
til: v2 2026-10-18
---

# 2026-10-18 (1)RAG Agent 실습4 골든셋 평가와 비교(Hit@k, MRR, LLM 채점, 청킹 크기 실험)
> 작성일: 2026-10-18

## 🔗 관련 글

- [RAG Golden Set(질문, 기준 답변, 근거 검수, 답변 불가, 버전 관리)](2026-10-14%20(2)RAG_Golden_Set(질문,_기준_답변,_근거_검수,_답변_불가,_버전_관리).md) — 랭체인 폴더의 35번 노트북(`35-rag-golden-set.ipynb`)을 정리한 글이다. 이 실습의 `load_golden`이 읽는 CSV가 바로 그 글의 방식으로 만든 골든셋이다. 답변 불가 문항, 근거 위치, 버전 관리가 이 글의 평가 코드와 직접 이어진다.
- [RAG 평가(Evaluation)](2026-09-04%20RAG%20평가(Evaluation).md) — Hit@k, MRR 같은 평가 지표의 개념 글이다. 이 글은 그 지표를 직접 코드로 계산한다.
- [RAG Agent 실습3 LangGraph Agent](2026-10-17%20(1)RAG_Agent_실습3_LangGraph_Agent(Grade,_AgentState,_조건부_분기,_재검색,_보류).md) — 이 글에서 기본 RAG와 비교하는 `run_agent`를 만든 글이다.
- [RAG Agent 실습2 임베딩, 검색, 기본 RAG](2026-10-16%20(1)RAG_Agent_실습2_임베딩_검색_기본_RAG(Chroma,_build_index,_search,_LCEL).md) — 비교의 기준선인 `basic_rag`와 청킹 크기 실험에 쓰는 `build_index`, `make_search_fn`이 들어 있다.
- [PDF 표·이미지 추출(pdfplumber, Gemini, 표 인식 원리, 유형별 청킹)](2026-10-14%20(1)PDF_표_이미지_추출(pdfplumber,_Gemini,_표_인식_원리,_유형별_청킹).md) — 34번 노트북 글이다. 이 실습에서 "비교 실험으로 미룬" 표·이미지 추출의 구현이 그 글에 있다.

## ❓ 아직 모르겠는 것

대화에서 사용자가 질문하거나 막힌 부분은 따로 기록되어 있지 않다. 이 단계의 코드는 실습을 마친 뒤 모두 이해한 상태다. 복습하면서 막힌 곳이 생기면 이 자리에 추가한다.

- 

## 주제 1: 골든셋 읽기(`load_golden`)

```python
def load_golden(path):
    """골든셋 CSV를 읽는다. 이번에 쓰는 문서의 질문과, 근거 문서가 없는(답변 불가) 질문만 남긴다."""
    with open(path, encoding="utf-8-sig", newline="") as f:      # utf-8-sig: 엑셀에서 저장한 CSV의 앞머리 표시(BOM)도 처리
        rows = list(csv.DictReader(f))                           # 한 줄을 {열 이름: 값} 딕셔너리로
    golden = []
    for row in rows:
        file_name = row["target_file_name"].strip()
        if file_name and file_name not in TARGET_NAMES:          # 다른 문서의 질문은 제외
            continue
        golden.append({
            "question": row["question"].strip(),
            "reference": row["target_answer"].strip(),           # 기준 답변
            "file_name": file_name,                              # 비어 있으면 답변 불가 문항
            "page_no": int(row["target_page_no"]) if file_name else None,
            "answerable": bool(file_name),
        })
    return golden


golden = load_golden(GOLDEN_SET_PATH)
if QUESTION_LIMIT:
    golden = golden[:QUESTION_LIMIT]
print(f"평가 문항: {len(golden)}개 (답변 가능 {sum(g['answerable'] for g in golden)}, 불가 {sum(not g['answerable'] for g in golden)})")
```

- **골든셋 CSV의 열**: `question`(질문), `target_answer`(기준 답변), `target_file_name`(정답 문서), `target_page_no`(정답 쪽수). 10-14 (2) 글에서 정리한 "질문·기준 답변·관련 문서·근거 위치"가 CSV 열로 들어간 모양이다.
- `encoding="utf-8-sig"`: 엑셀에서 CSV를 저장하면 파일 맨 앞에 **BOM**(글자 인코딩을 알리는 보이지 않는 표시)이 붙는 경우가 있다. `utf-8-sig`로 열면 이 표시를 알아서 처리한다. 그냥 `utf-8`로 열면 첫 열 이름 앞에 이상한 글자가 붙어 `row["question"]`을 찾지 못할 수 있다.
- `newline=""`: CSV 모듈이 줄바꿈을 직접 처리하게 하는 표준 방식이다. 값 안에 줄바꿈이 들어 있어도 안전하다.
- `csv.DictReader(f)`: 첫 줄을 열 이름으로 보고, **한 줄을 `{열 이름: 값}` 딕셔너리**로 읽는다.
- 걸러내기: `file_name`이 **있는데** `TARGET_NAMES`에 **없으면** 건너뛴다(이번에 쓰지 않는 문서의 질문). `file_name`이 **비어 있으면**(근거 문서가 없는 질문 = 답변 불가 문항) 조건이 거짓이라 **남는다.**
- 항목 변환: `.strip()`으로 앞뒤 공백을 지운다. `page_no`는 CSV에서 문자열이므로 `int(...)`로 정수로 바꾸되, 답변 불가 문항은 쪽수가 없으므로 `None`이다. `answerable`은 `bool(file_name)`: 문자열이 비어 있으면 `False`, 있으면 `True`.
- `if QUESTION_LIMIT: golden = golden[:QUESTION_LIMIT]`: 처음에는 3개만 쓴다(`None`이면 이 줄을 건너뛰어 전부 사용). **작게 시작하고 늘린다**는 진행 규칙이다.
- `sum(g['answerable'] for g in golden)`: 참/거짓은 1/0으로 더해진다. 그래서 `True`의 개수를 센다. `sum(not g['answerable'] ...)`은 답변 불가의 개수다.
- 확인할 것: 평가 문항 수와 답변 가능·불가 개수가 맞는가.

> ➕ **더 알아두기**
> 골든셋은 **검색 DB에 넣지 않는다**(실습의 진행 규칙, 10-14 (2) 글의 "질문·정답 파일을 검색 대상 원문에 넣지 않는다"와 같다). 이 코드에서도 골든셋은 CSV로 따로 읽을 뿐 `build_index`에는 들어가지 않는다.

## 주제 2: 검색 평가 — Hit@k와 MRR

```python
def match_rank(docs, gold):
    """검색 결과(가까운 순서)에서 정답 문서·쪽이 처음 나온 등수(1부터)를 돌려준다. 없으면 None."""
    for rank, d in enumerate(docs, start=1):
        if d["file_name"] == gold["file_name"] and d["page_no"] == gold["page_no"]:
            return rank
    return None


def eval_retrieval(search_fn, golden, k=TOP_K):
    """검색만 평가한다. 답변 불가 문항은 정답 근거가 없어 계산할 수 없으므로 제외한다."""
    ranks = []
    for gold in golden:
        if gold["answerable"]:
            ranks.append(match_rank(search_fn(gold["question"], k), gold))
    n = len(ranks)
    hit = sum(r is not None for r in ranks) / n                       # Hit@k : 정답 쪽이 상위 k개 안에 있었던 비율
    mrr = sum(1 / r for r in ranks if r) / n                          # MRR   : 정답 등수의 역수 평균 (1등이면 1, 2등이면 0.5)
    print(f"Hit@{k} = {hit:.2f} | MRR = {mrr:.2f} | 평가 문항 {n}개 (답변 불가 {len(golden) - n}개 제외)")
    return {"hit": hit, "mrr": mrr}
```

### `match_rank`

- 검색 결과(가까운 순서의 리스트)를 앞에서부터 보며 **정답 파일 이름과 정답 쪽수가 모두 같은** 첫 결과의 **등수(1부터)**를 돌려준다. 끝까지 없으면 `None`.
- 정답을 **청크 ID가 아니라 (파일, 쪽) 단위**로 판정한다. 그래서 청킹 크기를 바꿔 청크 ID가 달라져도 같은 골든셋으로 비교할 수 있다(10-14 (2) 글의 "근거 위치 vs 청크 ID"와 같은 설계 이유).

### `eval_retrieval`

- **답변 가능 문항만** 대상이다. 답변 불가 문항은 정답 쪽이 없어서 계산할 수 없다. 제외한 수를 출력에 적는다(`답변 불가 N개 제외`). (10-14 (2) 글: 관련 문서가 비어 있는 문항은 평균에서 제외하고 제외 수를 기록한다.)
- `ranks`: 문항마다 정답 등수(또는 `None`)의 리스트.
- **Hit@k** = 정답 쪽이 상위 k개 안에 있었던 **문항의 비율**. `r is not None`이 참인 개수 ÷ 문항 수.
- **MRR(Mean Reciprocal Rank)** = 정답 등수의 **역수의 평균**. 1등이면 1, 2등이면 0.5, 3등이면 약 0.33이고, 못 찾으면 0으로 친다(`if r`로 `None`을 건너뛰어 합에 더하지 않는다). 정답이 **얼마나 위에** 있는지를 반영한다.
- 예: 문항 3개의 등수가 `[1, None, 2]`이면 Hit = 2/3 ≈ 0.67, MRR = (1 + 0 + 0.5) / 3 = 0.5.
- `{hit:.2f}`: 소수 둘째 자리까지 출력하는 서식이다.
- 이 숫자가 앞으로 청킹·검색 방식을 바꿀 때 비교할 **기준 점수**다.

> ➕ **더 알아두기**
> - **Hit@k는 "있었는가", MRR은 "얼마나 위에 있었는가"**를 본다. 정답이 5등이어도 Hit@5는 1이지만 MRR은 0.2밖에 안 된다. 둘을 함께 보면 "찾긴 찾는데 순서가 나쁜" 경우를 알 수 있다.
> - 설계 문서에는 Recall@k도 적혀 있지만 이 코드는 Hit@k와 MRR만 계산한다. 이 골든셋은 문항마다 정답 쪽이 **하나**이므로 Hit@k가 Recall@k와 같은 값이 된다. (정답 쪽이 여러 개인 문항이 생기면 둘이 달라진다.)
> - 평가 문항 중 답변 가능 문항이 **하나도 없으면** `n`이 0이라 0으로 나누게 되어 에러가 난다. `QUESTION_LIMIT`으로 앞 3개만 쓸 때 우연히 3개가 모두 답변 불가이면 생길 수 있다.

## 주제 3: 답변 평가 — LLM 채점(`judge`)

```python
class Verdict(BaseModel):
    """답변 채점 결과 형식."""
    correct: bool = Field(description="생성 답변이 기준 답변의 핵심 내용과 조건을 모두 담고 있으면 true")
    reason: str = Field(description="판단 이유를 한 문장으로")


judge_chain = ChatPromptTemplate.from_messages([
    ("system",
     "기준 답변과 비교해 생성 답변을 채점합니다. 표현이 달라도 핵심 내용, 수치, 조건이 일치하면 correct=true 입니다. "
     "기준 답변에 없는 내용을 지어내거나 핵심이 빠지면 false 입니다."),
    ("human", "질문: {question}\n\n기준 답변: {reference}\n\n생성 답변: {answer}"),
]) | llm.with_structured_output(Verdict)


def judge(gold, result):
    """답변이 맞았는지 판정한다. 답변 불가 문항은 '보류했는가'로, 가능 문항은 모델 채점으로 본다."""
    if not gold["answerable"]:
        return result["abstained"]
    if result["abstained"]:
        return False
    return judge_chain.invoke({
        "question": gold["question"], "reference": gold["reference"], "answer": result["answer"],
    }).correct
```

- 답변은 문장이 매번 달라서 글자가 같은지로 맞고 틀림을 정할 수 없다. 그래서 **모델이 기준 답변과 비교해 채점**한다(LLM-as-a-judge). `Grade`(10-17)와 같은 방식으로 `Verdict(correct, reason)`을 Structured Output으로 받는다.
- 채점 기준(10-14 (2) 글의 "기준 답변은 문장이 아니라 내용을 정한다"와 같다): **표현이 달라도 핵심 내용·수치·조건이 일치하면 정답**, 기준 답변에 없는 내용을 지어내거나 핵심이 빠지면 오답이다.
- `judge`의 세 갈래:

| 문항 | 답변 | 판정 |
|---|---|---|
| 답변 불가 | (보류했는가) | `result["abstained"]` 그대로. 보류해야 정답, 답을 지어내면 오답 |
| 답변 가능 | 보류함 | `False` (답할 수 있는데 못 답함) |
| 답변 가능 | 답함 | 모델 채점 결과 `.correct` |

- 답변 불가 문항에서는 모델 채점을 부르지 않는다. "기준 답변"이 없고 **"보류했는가"가 곧 정답 기준**이기 때문이다.

## 주제 4: 실행 기록 저장과 기본 RAG vs Agent 비교

```python
def save_run(kind, question, result):
    """질문, 답변, 검색한 청크 ID와 근거 내용, 실행 시간을 JSON 한 줄로 저장한다."""
    record = {
        "time": datetime.now().isoformat(timespec="seconds"),
        "kind": kind,
        "question": question,
        "answer": result["answer"],
        "chunk_ids": [d["chunk_id"] for d in result["docs"]],
        "contexts": [d["text"] for d in result["docs"]],
        "search_count": result["search_count"],
        "seconds": result["seconds"],
    }
    with open(RESULT_PATH, "a", encoding="utf-8") as f:             # "a": 기존 내용 뒤에 이어 쓴다
        f.write(json.dumps(record, ensure_ascii=False) + "\n")
```

- **JSONL(JSON Lines)**: 한 줄에 JSON 하나씩 쌓는 형식이다. 실행할 때마다 **끝에 한 줄씩 덧붙이면** 되므로 기록용으로 편하다.
- `open(..., "a")`: 추가(append) 모드다. 기존 내용을 지우지 않고 뒤에 이어 쓴다(`"w"`는 덮어쓴다).
- `json.dumps(record, ensure_ascii=False)`: 딕셔너리를 JSON 문자열로 바꾼다. `ensure_ascii=False`를 주어야 한글이 `지...` 같은 코드가 아니라 **글자 그대로** 저장된다.
- `datetime.now().isoformat(timespec="seconds")`: 현재 시각을 `2026-10-18T14:03:21`처럼 초 단위까지 적는다.
- 기록 내용: 시각, 방식(`kind`), 질문, 답변, **검색된 청크 ID와 근거 본문(`contexts`)**, 검색 횟수, 시간. 나중에 틀린 질문의 원인을 볼 때 "어떤 근거가 들어갔는지"가 필요해서다. 10-14 (2) 글의 "실행 후 기록하는 결과(실제 검색한 청크, 출처 ID, 생성 답변)"와 같다.

```python
def compare(golden):
    """같은 골든셋으로 기본 RAG와 Agent를 돌려 질문별 결과를 모은다."""
    rows = []
    for gold in golden:
        for kind, run in (("rag", basic_rag), ("agent", run_agent)):
            result = run(gold["question"])
            save_run(kind, gold["question"], result)
            rows.append({
                "kind": kind,
                "answerable": gold["answerable"],
                # 정답 쪽이 (Agent는 모든 검색을 통틀어) 근거에 들어 있었는가
                "hit": bool(gold["answerable"] and match_rank(result["docs"], gold)),
                "correct": judge(gold, result),
                "abstained": result["abstained"],
                "search_count": result["search_count"],
                "seconds": result["seconds"],
            })
            time.sleep(SLEEP_SEC)
    return rows
```

- `for kind, run in (("rag", basic_rag), ("agent", run_agent))`: **함수 자체를 값으로** 묶어 두고 반복한다. 같은 질문에 두 방식을 차례로 돌리면서 같은 코드를 쓴다. `run(gold["question"])`은 `basic_rag(...)` 또는 `run_agent(...)`가 된다. 10-16·10-17에서 두 함수의 결과 모양을 맞춰 둔 덕분에 가능하다.
- `hit`: `bool(답변 가능 and match_rank(...))`. 정답 쪽이 **최종 근거(`docs`)에 들어 있었는가**다. Agent의 `docs`는 모든 검색을 합친 근거(`all_docs`)다. 답변 불가 문항은 `False`. `match_rank`는 등수(1 이상) 또는 `None`이라 `bool`로 바꿔도 안전하다.
- `time.sleep(SLEEP_SEC)`: 질문마다 잠깐 쉰다. **호출 제한(429 에러)**을 예방한다. 완성 코드는 1초, 노트북은 4초로 설정했다(10-15 글 주제 7).

```python
def summarize(rows):
    """방식(rag/agent)별 평균 지표를 출력한다."""
    for kind in ("rag", "agent"):
        mine = [r for r in rows if r["kind"] == kind]
        answerable = [r for r in mine if r["answerable"]]
        hit = sum(r["hit"] for r in answerable) / len(answerable) if answerable else float("nan")
        print(
            f"{kind:<6} | 근거 포함률 {hit:.2f} | 정답률 {sum(r['correct'] for r in mine) / len(mine):.2f} | "
            f"보류 {sum(r['abstained'] for r in mine)}개 | 평균 검색 {sum(r['search_count'] for r in mine) / len(mine):.1f}회 | "
            f"평균 {sum(r['seconds'] for r in mine) / len(mine):.1f}초"
        )


search_scores = eval_retrieval(search, golden)    # 검색 지표만 먼저 (API 호출은 질문 임베딩뿐이라 가볍다)
rows = compare(golden)                            # 기본 RAG vs Agent (질문 수 x 여러 번의 모델 호출)
summarize(rows)
print("실행 기록:", RESULT_PATH)
```

### 비교 표의 지표 읽는 법

| 지표 | 의미와 계산 범위 |
|---|---|
| 근거 포함률 | 정답 쪽이 근거에 들어 있었던 비율. **답변 가능 문항만** 대상 |
| 정답률 | `judge`가 정답으로 본 비율. **답변 불가 문항을 포함한 전체**가 대상(보류해야 정답) |
| 보류 | 답변을 보류한 문항 수 |
| 평균 검색 | 질문당 평균 검색 횟수(기본 RAG는 항상 1.0) |
| 평균 시간 | 질문당 평균 걸린 초 |

- `f"{kind:<6}"`: 글자를 **왼쪽 정렬, 폭 6칸**으로 맞춰 표처럼 줄이 맞게 한다. `float("nan")`은 "숫자가 아님" 값이라, 답변 가능 문항이 없을 때 0으로 나누는 에러를 피하고 `nan`으로 표시한다.
- **검색 지표를 먼저 단독으로** 본다(`eval_retrieval`). 질문 임베딩만 호출하므로 가볍다. 그다음 모델 호출이 많은 `compare`를 돌린다.
- 확인할 것: 질문마다 모델 호출이 여러 번이라 시간이 걸린다. **숫자만 보지 말고** `rows`에서 `correct`가 `False`인 질문을 골라 실패 원인을 직접 본다. **검색 실패**(정답 쪽이 근거에 없음)인지, **근거는 찾았는데 답이 틀린 것**인지 구분하는 것이 개선 방향을 정한다. 실행 기록은 `outputs/runs.jsonl`에 쌓인다.

> ➕ **더 알아두기**
> 비교할 때는 **같은 골든셋(같은 버전)으로** 돌려야 한다(10-14 (2) 글 주제 9). 문항 수를 `QUESTION_LIMIT=3`으로 제한하면 표본이 너무 작아 방식 간 차이가 우연일 수 있다. 처음에는 흐름을 확인하는 용도로 작게 돌리고, 잘 되면 늘려서 해석한다.

## 주제 5: (선택) 청킹 크기 비교 실험

```python
for size in (500, 1000):
    store, chunk_list = build_index(pages, size, size // 10)       # 크기에 맞춰 겹침도 10%로
    print(f"[chunk_size={size}] 청크 {len(chunk_list)}개")
    eval_retrieval(make_search_fn(store), golden)
```

- **한 번에 하나만 바꿔야 원인을 알 수 있다.** 여기서는 청크 크기만 바꾸고 검색 방식은 그대로 둔다. 겹침은 크기의 10%로 같이 맞춘다(`size // 10` → 50, 100).
- `build_index(pages, size, size // 10)`는 컬렉션 이름이 `finance_500_50`, `finance_1000_100`으로 달라 **서로 섞이지 않는다**(10-16). 1000은 이미 만들어 둔 것을 재사용하므로 임베딩 비용이 안 든다.
- `make_search_fn(store)`로 그 벡터 DB 전용 검색 함수를 만들어 `eval_retrieval`에 넣는다. 같은 골든셋으로 Hit@5와 MRR을 비교한다.
- 이 실험이 가능한 이유: 정답을 청크 ID가 아니라 **(파일, 쪽)**으로 판정하기 때문에 청킹을 바꿔도 같은 골든셋을 쓸 수 있다(주제 2).

## 주제 6: 실습 전체 흐름 한눈에

| 단계 | 핵심 함수·객체 | 입력 → 출력 |
|---|---|---|
| 로딩 | `load_pages` | PDF 5개 → 쪽별 `Document` 48개 |
| 청킹 | `chunk_pages` | 쪽 → 청크 + 청크 ID |
| 임베딩·저장 | `build_index` | 청크 → Chroma 컬렉션 |
| 검색 | `make_search_fn` / `search` | 질문 → 상위 k개 청크(딕셔너리) |
| 기본 RAG | `basic_rag` | 질문 → 검색 1번 → 답변 |
| Agent | `agent` / `run_agent` | 질문 → 검색·판단·재검색 → 답변 또는 보류 |
| 골든셋 | `load_golden` | CSV → 문항 리스트 |
| 평가 | `eval_retrieval`, `judge` | Hit@k·MRR, 답변 정답 여부 |
| 비교 | `compare`, `summarize` | 같은 문항으로 rag vs agent 표 |
| 실험 | 청킹 크기 루프 | 500 vs 1000 검색 지표 비교 |

## 정리

- `load_golden`은 CSV를 `{열 이름: 값}` 딕셔너리로 읽고(`utf-8-sig`로 BOM 처리), 이번 문서의 질문과 근거 문서가 없는(답변 불가) 질문만 남긴다. 답변 불가 문항은 `file_name`이 비어 있고 `page_no`는 `None`이다.
- 검색 평가: `match_rank`가 정답 (파일, 쪽)의 첫 등수를 찾고, Hit@k는 정답이 상위 k개에 있었던 문항의 비율, MRR은 등수 역수의 평균이다. 답변 불가 문항은 검색 지표에서 제외하고 제외 수를 출력한다.
- 답변 평가: 답변 가능 문항은 모델이 기준 답변과 비교해 채점하고(`Verdict`), 답변 불가 문항은 보류했는지로 판정한다. 가능한데 보류하면 오답이다.
- `save_run`은 JSONL로 한 줄씩 덧붙여(append) 질문·답변·청크 ID·근거·시간을 남기고, `compare`는 같은 질문으로 `basic_rag`와 `run_agent`를 차례로 돌려 지표를 모으며, `summarize`가 방식별 평균(근거 포함률, 정답률, 보류 수, 평균 검색 횟수·시간)을 출력한다.
- 숫자만 보지 말고 `correct`가 `False`인 질문을 열어 **검색 실패인지, 답변 생성 실패인지** 구분한다.
- 청킹 크기 실험은 한 번에 하나(크기)만 바꾸고 컬렉션을 분리해 같은 골든셋으로 비교한다. 정답을 (파일, 쪽)으로 지정해서 가능하다.

## ✅ 확인 질문

1. `load_golden`에서 `utf-8-sig`와 `newline=""`을 쓰는 이유는 무엇인가요? `csv.DictReader`는 한 줄을 어떤 모양으로 읽나요?
2. `file_name and file_name not in TARGET_NAMES` 조건은 어떤 문항을 걸러내고 어떤 문항을 남기나요? 답변 불가 문항이 어떻게 구분되는지도 설명해 보세요.
3. `match_rank`가 정답을 청크 ID가 아니라 (파일 이름, 쪽수)로 판정하는 이유는 무엇인가요?
4. Hit@k와 MRR은 각각 무엇을 측정하나요? 문항 3개의 정답 등수가 `[1, None, 2]`일 때 두 값을 계산해 보세요.
5. 정답이 5등인 경우 Hit@5와 MRR은 어떻게 다르게 나오며, 이것이 두 지표를 함께 보는 이유와 어떻게 연결되나요?
6. `eval_retrieval`이 답변 불가 문항을 제외하는 이유는 무엇이고, 제외한 수를 출력하는 이유는 무엇인가요?
7. 답변을 글자 비교가 아니라 모델로 채점하는 이유는 무엇이며, 채점 기준은 무엇인가요?
8. `judge`는 답변 불가 문항, 답변 가능한데 보류한 문항, 답변 가능하고 답한 문항을 각각 어떻게 판정하나요?
9. `save_run`에서 JSONL 형식, `"a"` 모드, `ensure_ascii=False`를 쓰는 이유는 각각 무엇인가요? 근거 본문(`contexts`)까지 기록하는 이유는 무엇인가요?
10. `compare`가 `(("rag", basic_rag), ("agent", run_agent))`처럼 함수를 값으로 묶어 반복할 수 있는 이유는 무엇인가요?
11. `summarize`의 "근거 포함률"과 "정답률"은 계산 대상 문항이 어떻게 다른가요?
12. `correct`가 `False`인 질문을 볼 때 검색 실패와 답변 생성 실패를 구분해야 하는 이유는 무엇인가요?
13. 청킹 크기 비교 실험에서 한 번에 하나만 바꿔야 하는 이유와, 컬렉션 이름에 청킹 설정을 넣어 둔 이유는 무엇인가요?
