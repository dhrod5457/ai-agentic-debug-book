# Research Methodology

기준일: 2026-10-05

## 1. 연구 질문

이 책은 다음 질문에 답하기 위한 자료만 수집한다.

> 실제 애플리케이션의 runtime evidence를 AI Coding Agent가 안전하고 효율적으로 조회하여 debugging에 사용할 수 있게 하려면 어떤 수집, 상관관계, 조회, 검증 구조가 필요한가?

일반적인 LLM self-reflection, prompt self-critique, autonomous agent 철학은 직접적인 관련성이 있을 때만 보조 근거로 사용한다.

## 2. 출처 우선순위

1. 공식 specification / documentation
2. 오픈소스 공식 repository와 구현 문서
3. peer-reviewed paper
4. benchmark 공식 repository와 dataset
5. preprint
6. 신뢰할 수 있는 engineering case

블로그 요약만 남기지 않고 가능한 경우 원문까지 확인한다.

## 3. Source Note 기록 항목

- Source ID
- Source / URL
- Date / Version
- Source Type
- 무엇을 관찰하는가
- Agentic Debugging과 직접 연결되는 점
- 실패 또는 제약
- 책에서 사용할 수 있는 Engineering Principle
- 일반화하면 안 되는 범위
- publication-time freshness recheck 여부

## 4. Runtime Evidence 분류

- Logs
- Metrics
- Traces
- Exceptions / Stack traces
- Profiles
- Runtime/JVM state
- Request/Response
- DB/SQL
- Deployment/Configuration change
- Infrastructure state

한 소스가 여러 signal을 다루면 signal 간 correlation 방법을 우선 기록한다.

## 5. Agent Interface 관점

관측 도구를 사람용 dashboard 기능만으로 평가하지 않는다. machine-readable API, query language, time/service/trace filtering, response-size 제한, structured output, read-only authorization, audit, sampling을 확인한다.

## 6. 핵심 경계

Raw Log ≠ Debug Context
Dashboard ≠ Agent Interface
Telemetry Storage ≠ Agent Memory
Correlation ≠ Root Cause
Anomaly ≠ Root Cause
Evidence Retrieval ≠ Reasoning
Observability Access ≠ Production Write Access

## 7. 숫자와 benchmark

정량 수치는 dataset/version, task, environment, model, tool/scaffold, grader, sample size, limitation을 함께 기록한다.

## 8. Preprint

Preprint는 본문에서 명시하며 증명했다보다 제안했다, 보고했다, 관찰했다, 평가했다를 사용한다.

## 9. 비연구 범위

현재 1차 조사에서는 일반적인 Agent 자기성찰, prompt engineering 일반론, fine-tuning 일반론, multi-agent 자체를 중심 주제로 삼지 않는다. Application Runtime Evidence와 직접 연결될 때만 다룬다.