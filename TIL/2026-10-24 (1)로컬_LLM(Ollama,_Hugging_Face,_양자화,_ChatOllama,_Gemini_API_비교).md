---
tags: [ollama, local-llm, huggingface, quantization, langchain]
til: v2 2026-10-24
---

# 2026-10-24 (1)로컬 LLM(Ollama, Hugging Face, 양자화, ChatOllama, Gemini API 비교)
> 작성일: 2026-10-24

## 🔗 관련 글

- [MCP(Host·Client·Server, Tool·Resource·Prompt, JSON-RPC, FastMCP, A2A)](2026-10-23%20(1)MCP(Host_Client_Server,_Tool_Resource_Prompt,_JSON-RPC,_FastMCP,_A2A).md) — 전날 수업이다. 이 글의 `ChatGoogleGenerativeAI`와 `MODEL_NAME`, `.env` 설정이 거기서도 같은 방식으로 쓰였다.
- [RAG Agent 실습2 임베딩, 검색, 기본 RAG](2026-10-16%20(1)RAG_Agent_실습2_임베딩_검색_기본_RAG(Chroma,_build_index,_search,_LCEL).md) — `prompt | llm | parser` 형태의 LCEL 체인을 만든 글이다. 이 글은 그 체인에서 LLM 부분만 로컬 모델로 갈아 끼운다.
- [gemini api 설명](2026-08-25%20(2)gemini%20api%20설명.md) — Gemini API 호출과 `.env`의 API 키 관리를 다뤘다. 이 글 주제 3의 비교 대상이 그 API이다.

## ❓ 아직 모르겠는 것

이 글은 강의 노트북(`37-local-llm.ipynb`)을 읽고 정리했다. 대화 중 사용자가 질문하거나 막힌 부분은 없었다. 복습하면서 막히는 곳을 이 자리에 추가한다.

- (확인 필요) 노트북의 `ChatOllama` 실행 셀(주제 2)은 **실행 결과가 저장되어 있지 않다.** 로컬 모델의 실제 답변은 직접 실행해서 Gemini의 답변과 비교해 봐야 한다.

## 주제 1: 로컬 LLM을 실행하는 도구와 플랫폼

### Ollama

로컬에서 LLM을 쉽게 실행할 수 있게 해 주는 도구이다. `ollama pull`로 모델을 다운로드하고 `ollama run`으로 실행하면 된다.

- 공식 사이트: https://ollama.com
- Llama(Meta), Qwen(Alibaba), Gemma(Google) 등 다양한 오픈소스 모델을 지원한다.
- 로컬 API 서버를 띄워 주기 때문에 LangChain 등 외부 도구에서 쉽게 연동할 수 있다.

**양자화(quantization)** 는 모델 가중치를 낮은 정밀도로 표현해 **모델 크기와 메모리 사용량을 줄이는** 기법이다. Ollama에서는 이미 양자화된 모델을 다운로드해 실행할 수 있다.

### Hugging Face

오픈소스 AI 모델의 **GitHub** 같은 플랫폼이다. LLM뿐만 아니라 이미지, 음성 등 다양한 AI 모델이 공유되어 있다.

- 공식 사이트: https://huggingface.co
- **Models**: 수십만 개의 사전학습 모델을 검색하고 다운로드할 수 있다.
- **Datasets**: 학습용 데이터셋 공유 및 다운로드
- **Spaces**: 모델을 웹에서 바로 체험할 수 있는 데모 공간

Ollama 라이브러리에 없는 모델도 Hugging Face에서 **GGUF 형식**으로 다운로드하면 Ollama로 실행할 수 있다. 이처럼 모델을 모아 놓은 플랫폼을 **Model Hub**라고 부른다.

| 구분 | Ollama | Hugging Face |
|---|---|---|
| 정체 | 로컬에서 LLM을 실행하는 도구 | 오픈소스 AI 모델을 공유하는 플랫폼(Model Hub) |
| 하는 일 | `pull`로 받고 `run`으로 실행, 로컬 API 서버 제공 | 모델·데이터셋 검색과 다운로드, 웹 데모(Spaces) |
| 연결 | Ollama 라이브러리에 없는 모델은 Hugging Face의 GGUF를 받아 실행 | |

## 주제 2: 같은 체인에 Gemini와 로컬 모델 갈아 끼우기

### 먼저 Gemini API 모델로 체인 만들기

프로젝트의 `.env` 파일에 `GOOGLE_API_KEY`를 설정한다.

```dotenv
GOOGLE_API_KEY=발급받은_API_키
```

`load_dotenv()`로 환경 변수를 불러오면 `ChatGoogleGenerativeAI`가 API 키를 사용한다.

```python
from dotenv import load_dotenv
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

load_dotenv()

MODEL_NAME = "gemini-3.6-flash"

prompt = ChatPromptTemplate.from_messages([
    ("system", "너는 {role} 전문가야. 모든 답변은 한국어로 해줘."),
    ("human", "{question}"),
])

llm = ChatGoogleGenerativeAI(model=MODEL_NAME)
parser = StrOutputParser()

chain = prompt | llm | parser

result = chain.invoke({
    "role": "Python",
    "question": "for 문을 설명해줘",
})

print(result)
```

실행하면 먼저 자동 함수 호출(AFC)을 `generate_content`에서 직접 쓰는 것은 권장하지 않는다는 SDK 경고 메시지가 출력되고, 이어서 `for` 문의 기본 구조, `range()`, 문자열 반복, `break`와 `continue`를 설명하는 한국어 답변이 출력된다.

### Ollama 설치

공식 사이트(https://ollama.com)에서 설치 파일을 다운로드한다.

- **macOS**: `brew install ollama` 또는 공식 사이트에서 다운로드
- **Windows**: 공식 사이트에서 설치 파일 다운로드

설치 후 Ollama 서버를 실행한다. Ollama 앱이 이미 실행 중이면 이 단계는 생략한다.

```bash
ollama serve
```

서버를 실행한 터미널은 열어 둔 상태에서 **새 터미널**을 열어 모델을 다운로드한다.

```bash
ollama pull qwen3.5:0.8b
```

모델명의 `0.8b`는 언어 모델의 파라미터가 약 **8억 개**라는 뜻이다. `B`는 billion(10억)을 의미한다. 파라미터는 모델이 학습 과정에서 조정하는 내부 수치이다.

노트북 실습 중에도 Ollama 서버를 실행한 상태로 유지한다.

```bash
# LangChain Ollama 연동 패키지 설치
pip install langchain-ollama
```

### ChatOllama로 모델만 교체하기

```python
from langchain_ollama import ChatOllama

# 모델만 교체하면 된다
local_llm = ChatOllama(model="qwen3.5:0.8b", reasoning=False)

# 나머지는 동일
chain = prompt | local_llm | parser

result = chain.invoke({
    "role": "Python",
    "question": "for 문을 설명해줘",
})

print(result)
```

- 프롬프트와 파서는 그대로 두고 **LLM 객체만 `ChatGoogleGenerativeAI`에서 `ChatOllama`로 바꾸면** 같은 체인이 로컬 모델로 동작한다. 그래서 주석에 "모델만 교체하면 된다", "나머지는 동일"이라고 적혀 있다.
- Gemini API 쪽은 `.env`의 `GOOGLE_API_KEY`가 필요하지만, `ChatOllama` 쪽은 코드에 API 키가 나오지 않는다. 대신 **Ollama 서버가 실행 중이어야** 한다.

> ➕ **더 알아두기**
> 수업 코드의 `reasoning=False`는 설명 없이 쓰였다. LangChain의 `ChatOllama`에서 이 옵션은 모델이 답을 내기 전에 거치는 "생각(thinking)" 과정을 켜고 끄는 설정으로, `False`면 생각 과정 없이 바로 답한다. 설치된 모델을 확인할 때는 `ollama list`를 쓴다. (둘 다 수업에서 설명하지 않은 내용이다.)

## 주제 3: 로컬 모델과 Gemini API 비교

| 비교 항목 | 로컬 LLM (qwen3.5:0.8b) | Gemini API (gemini-3.6-flash) |
|---|---|---|
| 비용 | API 호출 비용 없음, 장비·전력 필요 | API 요금제와 사용량에 따라 달라짐 |
| 인터넷 | 모델 다운로드 후 로컬 추론에는 불필요 | 필요 |
| 사용 한계 | PC 메모리와 연산 성능의 제약 | 요청 수·토큰 수 등의 API 호출 한도 |

- 로컬 LLM은 **돈이 아니라 내 컴퓨터의 자원**을 쓴다. 인터넷 없이도 동작하지만 PC 성능이 한계가 된다.
- Gemini API는 컴퓨터 성능과 상관없이 쓸 수 있지만 인터넷과 요금제, 호출 한도에 매인다.

> ➕ **더 알아두기**
> 파라미터 수가 작을수록 필요한 메모리와 연산량이 줄어 가벼운 PC에서도 돌아가지만, 대체로 답변 품질은 낮아지는 경향이 있다. 양자화는 같은 모델을 더 작게 만들어 메모리 부담을 줄이는 방법이라, 로컬 환경에서는 모델 크기(파라미터 수)와 양자화를 함께 고려한다. (수업에서 설명하지 않은 내용이다.)

## 정리

- Ollama는 로컬에서 LLM을 쉽게 실행하는 도구다. `ollama pull`로 받고 `ollama run`으로 실행하며, 로컬 API 서버를 띄워 LangChain 같은 도구와 연동된다.
- 양자화는 모델 가중치를 낮은 정밀도로 표현해 모델 크기와 메모리 사용량을 줄이는 기법이다. Ollama에서는 이미 양자화된 모델을 받아 쓴다.
- Hugging Face는 오픈소스 AI 모델의 GitHub 같은 Model Hub다. Models, Datasets, Spaces로 이루어지고, Ollama에 없는 모델은 GGUF 형식으로 받아 Ollama로 실행할 수 있다.
- 순서는 `ollama serve`(서버 유지) → 새 터미널에서 `ollama pull qwen3.5:0.8b` → `pip install langchain-ollama` → `ChatOllama`다. 모델명의 `0.8b`는 파라미터 약 8억 개(billion)다.
- `prompt | llm | parser` 체인에서 LLM만 `ChatGoogleGenerativeAI`에서 `ChatOllama`로 바꾸면 나머지는 그대로다.
- 로컬 LLM은 비용이 없고 인터넷이 필요 없지만 PC 성능이 한계다. Gemini API는 인터넷과 호출 한도가 한계다.

## ✅ 확인 질문

1. Ollama는 무엇이고, `ollama pull`과 `ollama run`은 각각 무엇을 하나요? Ollama가 로컬 API 서버를 띄워 주면 LangChain에서 어떤 점이 편해지나요?
2. 양자화(quantization)란 무엇이고, 어떤 이점이 있나요? Ollama에서 모델을 받을 때 양자화는 어떻게 다뤄지나요?
3. Hugging Face의 Models, Datasets, Spaces는 각각 무엇이고, Ollama 라이브러리에 없는 모델을 Ollama로 실행하려면 어떤 형식으로 받아야 하나요?
4. Ollama와 Hugging Face는 역할이 어떻게 다른가요? "Model Hub"라는 말은 무엇을 가리키나요?
5. `ollama serve`를 실행한 터미널을 닫지 않고 새 터미널에서 `ollama pull`을 하는 이유는 무엇이고, 노트북 실습 중에도 서버를 켜 둬야 하는 이유는 무엇인가요?
6. 모델명 `qwen3.5:0.8b`에서 `0.8b`는 무엇을 뜻하나요? 파라미터는 무엇이고 `B`는 무엇의 약자인가요?
7. Gemini 체인을 로컬 모델 체인으로 바꿀 때 코드의 어떤 부분만 바뀌고 어떤 부분은 그대로인가요? `prompt | llm | parser` 구조로 설명해 보세요.
8. Gemini API 쪽에는 `GOOGLE_API_KEY`가 필요하지만 `ChatOllama` 쪽에는 필요하지 않았습니다. 대신 `ChatOllama`가 동작하려면 무엇이 준비되어 있어야 하나요?
9. 로컬 LLM과 Gemini API를 비용, 인터넷, 사용 한계 세 가지 면에서 비교해 보세요. 어떤 상황이면 로컬 LLM을, 어떤 상황이면 API를 고르겠나요?
