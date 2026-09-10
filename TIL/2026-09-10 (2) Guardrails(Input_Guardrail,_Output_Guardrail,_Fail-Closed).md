# Guardrails (Input Guardrail, Output Guardrail, Fail-Closed)

## Guardrail이 왜 필요한가

지금까지 만든 것 중 상당수는 "질문 → 답변"만 하는 챗봇이었지만, Agent는 Tool을 호출해서 실제로 시스템에 영향을 주는 행동(API 호출, 데이터 변경 등)을 할 수 있다. 일반 챗봇이 틀린 답을 하면 말 한마디로 끝나지만, Agent가 잘못된 Tool을 잘못된 인자로 실행하면 실제 사고가 난다.

그래서 "시스템 프롬프트에 하지 말라고 적어두는 것"만으로는 부족하다. LLM은 확률적으로 답을 생성하고 프롬프트 인젝션(사용자가 "이전 지시 무시해"라고 속이는 것) 같은 공격에 뚫릴 수 있기 때문이다. Guardrail은 **LLM의 "말"에만 의존하지 않고, 그 바깥에서 코드로 강제로 검사·차단하는 장치**다.

## Guardrail의 4개 계층

Guardrail은 Agent가 동작하는 여러 지점에서 나눠서 검사한다.

```
사용자 입력 → [Input Guardrail] → Agent/LLM → [Tool Guardrail] → Tool 실행
                                                                     ↓
사용자 ← [Output Guardrail] ← 최종 응답 생성 ←──────────────────────┘
```

| 계층 | 검사 시점 | 뭘 보는가 |
|---|---|---|
| Input Guardrail | Agent가 시작하기 전 | 입력이 빈 값인지, 너무 긴지, 유해한지, 프롬프트 인젝션인지 |
| Policy/Tool Guardrail | Tool을 호출하려는 순간 | 이 Tool을 부를 권한이 있는지, 인자가 업무 규칙에 맞는지 |
| Output Guardrail | 답변을 보내기 직전 | 개인정보나 API 키 같은 민감정보가 새어나가는지 |
| Runtime Guardrail | 실행 도중 계속 | 호출 횟수·시간·비용이 한도를 넘는지 |

이 노트북은 이 중 **Input Guardrail**과 **Output Guardrail** 두 개만 LangGraph로 구현한다.

## Input Guardrail — 두 단계 검사

### 1단계: 규칙 기반 (빠르고 확실한 것부터)

입력이 비어있는가, 너무 긴가(예: 1000자 초과) 같은 조건은 LLM 없이 코드로 바로 판단한다. 이건 15번(Router)에서 배운 Deterministic Routing과 같은 방식이다 — 확실한 조건은 LLM 호출 없이 먼저 걸러낸다.

### 2단계: LLM 기반 (맥락 판단이 필요한 것)

규칙으로 못 거르는 건 LLM에게 "안전한지/유해한지/프롬프트 인젝션인지 분류해줘"라고 묻는다. 이때 Structured Output을 써서 `category`가 `safe`/`unsafe`/`prompt_injection` 중 하나로만 나오게 제한한다.

```python
class GuardrailCheck(BaseModel):
    category: Literal["safe", "unsafe", "prompt_injection"] = Field(...)
    reason: str = Field(description="판단 근거 (안전하면 빈 문자열)")

guardrail_checker = llm.with_structured_output(GuardrailCheck)
```

### 핵심: LLM의 "판단"과 코드의 "결정"은 분리된다

- LLM이 하는 일: 입력을 분류해서 `category` 값 하나만 반환 (판단만 함, 실행하지 않음)
- 코드가 하는 일: 그 값을 보고 `if category == "safe": 통과 / else: 차단`이라고 최종 결정 (실제 문을 잠그는 건 코드)

이건 15번 Router 구조(판단 함수 → 조건부 엣지)를 안전성 검사에 그대로 응용한 것이다.

> ♻️ 처음엔 "LLM이 안전성을 판단하는 것도 결국 프롬프트로 규칙을 알려주는 것뿐이라 시스템 프롬프트에 규칙 적어두는 것과 다를 게 없지 않나"라고 생각했으나, 정확히는 **"이 검사 호출 자체가 답변 생성이나 실행 권한을 전혀 갖지 않고, 오직 제한된 라벨 하나만 반환하며, 최종 통과/차단은 코드가 결정한다"**는 구조적 분리가 핵심이다. 프롬프트 인젝션이 이 분류 자체를 속여서(예: unsafe를 safe로) 오분류시키는 것은 여전히 가능하다 — 노트북도 "완벽한 방어는 불가능하다"고 명시한다. Guardrail의 목표는 인젝션을 100% 막는 게 아니라, 하나의 계층이 뚫려도 다른 계층(특히 LLM을 아예 안 쓰는 Output Guardrail)이 최종 사고를 막는 다층 방어(defense in depth)를 만드는 것이다.

### Fail-Closed 정책

검사 자체가 실패(API 에러, 타임아웃 등)했을 때 통과시킬지(fail-open) 차단할지(fail-closed) 정하는 정책이다. Guardrail은 **fail-closed를 기본**으로 한다 — "확인이 안 됐으면 일단 막는다."

```python
try:
    result = guardrail_checker.invoke([...])
    is_safe = result.category == "safe"
    return {"is_safe": is_safe, "safety_reason": result.reason}
except Exception:
    # 검사 실패 시 요청을 통과시키지 않는 fail-closed 정책
    return {"is_safe": False, "safety_reason": "안전성 검사 실패"}
```

## 프롬프트 인젝션 방어 팁 (노트북 정리)

- 다층 방어: 키워드 필터와 LLM 판단을 함께 사용한다
- 메시지 역할 분리(SystemMessage/HumanMessage 구분)는 기본 조치이며 방어를 보장하지 않는다
- LLM 기반 감지는 새로운 공격 패턴에도 어느 정도 대응 가능하다
- 최소 권한: 허용된 작업과 도구 권한을 제한해 감지 실패의 영향을 줄인다
- 장애 처리: 검사 모델의 오류·시간 초과 시 재시도 횟수와 차단 정책을 정한다
- 완벽한 방어는 불가능하므로 입력 검증, 권한 통제, Output Guardrails를 함께 쓴다

## Output Guardrail — 정규식 기반 민감정보 마스킹

Input Guardrail이 사용자의 **질문**을 검사한다면, Output Guardrail은 LLM이 만든 **답변**에 민감정보가 새어나갔는지 검사한다. 여기서는 **LLM을 아예 쓰지 않는다.**

전화번호, 주민번호, 카드번호, API 키는 딱 정해진 문자열 패턴이 있어서, 정규식(regex)으로 바로 잡아낼 수 있다.

```python
PII_PATTERNS = [
    r"(?<!\d)\d{6}[- ]?\d{7}(?!\d)",                 # 주민등록번호 형식
    r"(?<!\d)(?:\d{4}[- ]?){3}\d{4}(?!\d)",        # 카드번호 형식
    r"(?<!\d)01[016789][- ]?\d{3,4}[- ]?\d{4}(?!\d)",  # 휴대전화번호 형식
]
```

패턴이 발견되면 해당 부분을 `[전화번호 마스킹]` 등으로 치환한다. 이 계층은 문자열 매칭일 뿐이라 프롬프트 인젝션이라는 개념 자체가 적용될 여지가 없다 — 그래서 Input Guardrail이 뚫려도 이 계층이 마지막 안전망 역할을 한다.

### 마스킹 대신 재생성하기

마스킹("[전화번호 마스킹]")은 답변 중간에 부자연스럽게 끼어든다. 더 나은 방법은 민감정보가 발견되면 마스킹하지 않고 **LLM에게 답변을 다시 생성**시키는 것이다.

```
generate(답변 생성) → validate(민감정보 검사) → 있으면 → generate로 다시 (재생성)
                                              → 없으면 → 끝
```

이건 17번(Reflection)에서 배우는 "생성 → 검토 → 다시 생성" 루프를 미리 응용한 것이다. 재생성 루프에는 반드시 `retry_count`(최대 재시도 횟수)를 둬서, 계속 민감정보가 나오는 극단적 상황에서도 무한 반복되지 않고 안전한 고정 응답으로 종료하게 만들어야 한다. 같은 구조로 할루시네이션 필터링, 응답 포맷 검증, 톤/스타일 검증도 구현할 수 있다.

## Input/Output Schema — 내부 State와 외부 인터페이스 분리

Output Guardrail 그래프의 State에는 `query`, `raw_response`(마스킹 전 원본), `has_sensitive_info`, `final_response` 같은 필드가 있는데, `graph.invoke(...)`를 그냥 실행하면 이 내부 필드가 전부 결과에 노출된다. 사용자에게는 `final_response`만 필요하고, `raw_response`(마스킹 전 원본!)까지 노출되면 오히려 위험할 수 있다.

`StateGraph`에 `input_schema`/`output_schema`를 지정하면 내부 State와 외부에 노출할 형태를 분리할 수 있다.

```python
class GuardrailInput(TypedDict):
    query: str

class GuardrailOutput(TypedDict):
    final_response: str

schema_builder = StateGraph(OutputState, input_schema=GuardrailInput, output_schema=GuardrailOutput)
```

- 첫 번째 인자: 내부 State (노드 간 공유, 전체 필드)
- `input_schema`: 그래프를 호출할 때 넣어야 하는 필드만 제한
- `output_schema`: 그래프가 끝났을 때 돌려줄 필드만 제한

이건 완전히 새로운 개념이라기보다 지금까지 배운 TypedDict 기반 State 정의의 응용이다 — "내부용 State"와 "바깥에서 보이는 모양"을 별도의 TypedDict 두 개로 나눠서 지정할 수 있다는 점이 새롭다.

## 오늘 배운 것 요약

| 개념 | 역할 |
|---|---|
| Guardrail | LLM의 판단이 아니라 코드가 최종 결정을 내리는 안전장치 |
| Input Guardrail | 규칙 기반(빈 값·길이) + LLM 기반(Structured Output으로 안전성 분류)의 2단계 검사 |
| Fail-Closed | 검사 자체가 실패하면 기본값을 "차단"으로 둔다 |
| Output Guardrail | LLM 없이 정규식으로 민감정보 탐지·마스킹(또는 재생성 유도) |
| Input/Output Schema | 내부 State와 외부에 노출할 필드를 분리 |

## ✅ 확인 질문

1. Guardrail이 "시스템 프롬프트에 규칙을 적어두는 것"만으로는 부족한 이유는 무엇인가?
2. Input Guardrail에서 규칙 기반 검사를 LLM 기반 검사보다 먼저 실행하는 이유는 무엇인가?
3. Structured Output으로 category를 제한하는 것이 실제로 막아주는 것과, 여전히 막지 못하는 것은 각각 무엇인가?
4. LLM의 "판단"과 코드의 "결정"이 분리되어야 하는 이유를 예를 들어 설명하라.
5. Fail-Closed와 Fail-Open의 차이는 무엇이며, Guardrail은 왜 Fail-Closed를 기본으로 하는가?
6. Output Guardrail이 LLM을 전혀 쓰지 않는 이유는 무엇인가?
7. 민감정보 발견 시 마스킹 대신 재생성을 택할 때, 왜 retry_count가 반드시 필요한가?
8. input_schema/output_schema를 지정하지 않으면 어떤 문제가 생기는가?
