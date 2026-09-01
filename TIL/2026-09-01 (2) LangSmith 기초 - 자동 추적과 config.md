# 2026-09-01 (2) LangSmith 기초 — 자동 추적과 config

## LangSmith가 뭔가

**한 줄**: LangChain으로 만든 체인이 내부적으로 어떻게 실행됐는지 웹 대시보드에서 눈으로 보여주는 관측(observability) 도구.

확인 가능한 것: 프롬프트→모델→파서 실행 순서, 변수가 채워진 최종 프롬프트, 토큰 사용량, 각 단계 소요 시간, 에러 발생 지점.

## 준비

`.env`에 추가:
```
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=lsv2_...
LANGSMITH_PROJECT=프로젝트이름
```
smith.langchain.com 가입 → Settings → API Keys → Personal Access Token 발급.

## ⭐ 핵심: 추적 코드를 한 줄도 안 썼는데 왜 자동으로 기록되나

```python
chain.invoke({"question": "..."})   # 이게 다. LangSmith 관련 코드 없음
```

**이유**: `Runnable.invoke()` 내부에 이미 "환경변수에 `LANGSMITH_TRACING=true`가 있나? 있으면 이번 실행을 LangSmith 서버로도 보내라"는 확인 로직이 심어져 있음. `import langsmith`조차 우리가 직접 안 해도 됨.

**비유**: 건물에 CCTV가 미리 설치되어 있고, 전원 스위치(`LANGSMITH_TRACING`)만 켜면 자동 녹화 시작. "저 좀 찍어주세요" 요청할 필요 없음.

**설계 이유**: 환경변수 기반이라 코드를 하나도 안 건드리고 기능을 켜고 끌 수 있음. 학습용 코드와 실제 서비스 코드가 완전히 동일하게 유지됨 (`.env`만 바꾸면 켜짐/꺼짐).

## config — 실행에 "이름표" 붙이기 (선택사항)

```python
chain.invoke(
    {"question": "..."},              # 무엇을 처리할지 (데이터)
    config={                            # 어떻게 실행/기록할지 (지침)
        "run_name": "python-basic-question",
        "tags": ["gemini", "basic", "class-demo"],
        "metadata": {"lesson": "01-1", "model_family": "gemini"},
    },
)
```

- `run_name`, `tags`, `metadata` — **고정된 키 이름**. `("system", ...)`처럼 정확한 소문자 철자로 써야 인식됨.
- `run_name`: 목록에서 이 실행의 이름 (안 쓰면 자동생성 이름 `RunnableSequence`가 대신 뜸)
- `tags`: 문자열 리스트. 필터링용 짧은 라벨 여러 개
- `metadata`: 딕셔너리. 세부 정보 (정확한 값으로 필터링 가능)

### `tags` vs `metadata`
비유: `tags`는 옷에 붙이는 여러 개의 **스티커**(세일, 신상품), `metadata`는 옷의 **스펙표**(사이즈: L, 소재: 면100%).

### ⚠️ 여러 번 헷갈렸던 오해들, 정정

**1. config 없으면 자동으로 뭔가 채워지는 게 아니다.**
config를 안 쓰면 `run_name`/`tags`/`metadata`는 그냥 **없는 채로** 기록됨. "자동" = 입력/출력/토큰/시간처럼 실행 과정을 관찰해서 알 수 있는 것만 해당. "이게 무슨 의미의 실행인지"는 사람만 아는 정보라 자동화 불가능.

**2. AI가 값을 넣어주는 게 아니다.**
`config`에 들어가는 값은 전부 개발자가 준비한 **평범한 파이썬 변수/문자열**. 고정값이면 문자열 직접 쓰고, 매번 달라져야 하면 변수/함수로 만들어서 넣어야 함. 실무에서 `user_id` 같은 값이 "자동으로" 들어간다는 건, 서버가 로그인 처리하며 이미 알고 있는 값을 변수로 재사용한다는 뜻이지, config 코드 자체를 안 써도 된다는 뜻이 아님.

**3. tags는 "내용 요약"이 아니라 "코드 경로 표시"라 자동화(하드코딩) 가능.**
`tags: ["translate"]`처럼 어떤 기능 코드에 한 번 박아두면, 그 기능을 통과하는 모든 요청에 자동으로 붙음. **답변 내용을 읽을 필요가 전혀 없음** — 태그는 "이 요청이 어떤 코드/환경을 통과했는가"를 나타내는 것이지 "무슨 내용인가"를 요약하는 게 아님.

**4. LangSmith 라벨링은 학습데이터(fine-tuning용)가 아니다.**
목적이 완전히 다름. 학습데이터는 모델 성능 개선용, LangSmith 라벨은 이미 완성된 모델을 실행한 기록을 **나중에 찾기 쉽게** 해두는 관측/디버깅용.

## 왜 config가 필요한가 (실전 시나리오)

서비스가 하루 10만 번 호출되면, config 없이는 목록이 전부 `RunnableSequence`로 똑같이 보임. 고객이 "어제 오후에 이상한 답 받았어요"라고 하면, `user_id`로 필터링해서 그 사람 것 몇 개만 찾아야 하는데 라벨이 없으면 불가능. → config = "나중에 다시 찾기 위한 이름표".

## 선택적 추적 (`tracing_context`)

```python
import langsmith as ls

with ls.tracing_context(enabled=True, project_name="langchain-gemini-selective"):
    result = chain.invoke({"question": "..."})
```

`.env`는 전체 설정이라 코드 중간에 못 바꿈. 특정 구간만 다른 규칙 적용하려면 `with` 블록 사용 (이전에 배운 `async with httpx.AsyncClient`와 같은 "시작할 때 준비, 끝나면 원복" 패턴).

### `enabled`과 `project_name`은 완전히 독립적인 옵션 (실험으로 증명함)

| 옵션 | 목적 | 데이터가 기록되나? |
|---|---|---|
| `enabled=True` + `project_name="다른이름"` | 다른 프로젝트로 **라우팅** | ✅ 기록됨 (위치만 다름) |
| `enabled=False` | 추적 **자체를 끔** | ❌ 아무 데도 기록 안 됨 |

**실험**: `enabled=False`에 `project_name="should-not-exist-test"`를 **일부러 같이** 지정하고 실행 → LangSmith 프로젝트 목록에 그 이름이 전혀 생기지 않음. `enabled=False`가 `project_name`보다 우선해서 아예 전송 자체를 막아버린다는 게 실제로 확인됨.

**용도 예시**:
- 다른 프로젝트로 라우팅: 실험적인 프롬프트 테스트를 본 수업 기록과 분리해서 모아두고 싶을 때
- `enabled=False`: 민감한 사용자 데이터를 처리하는 구간은 아예 서버로 전송 자체를 막고 싶을 때 (모델 응답은 정상 작동, 기록만 안 남김)

## 핵심 한 줄 정리

- 추적 자체는 **완전 자동** (`.env` 키만 있으면 코드 불필요) — `Runnable.invoke()`에 이미 내장된 로직
- `config`는 **선택사항**, "나중에 찾기 위한 이름표"이지 자동화도 학습데이터도 아님
- `tags`/`metadata`는 사람이 직접 준비한 값만 들어감 (AI가 알아서 채워주지 않음)
- `enabled`(끄고 켜기)와 `project_name`(어디로 보낼지)은 독립된 별개 옵션
