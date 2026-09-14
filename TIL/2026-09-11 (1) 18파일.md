
[TIL] LangGraph 병렬 처리(Parallel Processing) & Voting 패턴 정리
💡 Today I Learned Summary
LangGraph에서 여러 노드를 동시에 실행하는 병렬 처리(Parallel Processing)의 기본 개념과 구현 방법, 그리고 병렬 처리를 응용한 Voting(투표/추론) 패턴에 대해 학습했다.

1. 병렬 처리(Parallel Processing)의 핵심 개념
전제 조건: 병렬로 실행하려는 작업 간에 독립성이 보장되어야 함 (작업 B가 작업 A의 실행 결과에 의존하지 않아야 함).

실행 시간: 전체 처리 시간은 병렬 실행 작업 중 가장 오래 걸리는 작업(병목)의 실행 시간에 맞춰짐.

2. LangGraph 실행 구조: Fan-out & Fan-in
Fan-out (분기): START 노드에서 여러 작업 노드로 Edge를 다중 연결하면 LangGraph가 자동으로 동시 실행함.

Fan-in (합류): 분기된 작업 노드들을 하나의 종합 노드(예: 요약/결과 출력)로 모아 연결함. LangGraph는 연결된 모든 병렬 노드가 종료될 때까지 기다렸다가 다음 노드를 실행함.

3. 병렬 처리 시 주요 문제와 해결책
State 업데이트 충돌 (Write Conflict)

문제: 여러 노드가 공유 State의 동일한 필드에 동시에 결과를 쓰려고 하면 마지막 노드의 결과만 남고 나머지는 덮어씌워짐.

해결책: State 정의 시 Annotated[list, operator.add]와 같은 Reducer를 지정하여 결과들을 리스트 형태로 이어 붙이도록(Concat) 설정함.

결과 순서의 비결정성 (Non-deterministic Order)

문제: 네트워크/LLM 응답 속도 차이로 인해 결과가 State에 들어오는 순서가 매번 달라짐.

해결책: 각 노드가 반환할 때 (순서_번호, 내용) 형태의 식별자(태그) 튜플을 반환하고, Fan-in 노드에서 번호 순으로 정렬함.

4. Voting 패턴과 추론(Reasoning)
Voting 패턴이란?
하나의 질문/입력에 대해 여러 스타일(예: 핵심 중심, 예시 중심, 비유 중심)의 답변 후보를 병렬로 동시 생성한 뒤, 심판 LLM이 이를 평가하여 최선의 답변을 고르는 패턴.

추론(Reasoning) 관점에서의 의의:
단순히 첫 번째 생성 결과를 출력하는 것이 아니라, 내부적으로 다양한 생각/접근 방식을 뻗어본 뒤 Self-Consistency 검증 및 LLM-as-a-Judge 기법을 활용해 스스로 최적의 답을 추론해내는 구조임.

Structured Output 활용:
심판 LLM이 평가 결과를 반환할 때 Pydantic/JSON 형식({"best_id": 2, "reason": "..."})으로 출력하도록 강제하여 프로그래밍적으로 손쉽게 최종 결과를 픽업함.