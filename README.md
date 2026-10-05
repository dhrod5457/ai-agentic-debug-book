# AI Agentic Debugging

Application Runtime Evidence를 AI Coding Agent가 직접 조회하고 해석하여 원인 분석, 수정, 재검증까지 수행할 수 있도록 만드는 방법을 연구하는 책 프로젝트입니다.

## 중심 질문

> 소스코드만 보는 Coding Agent에게 실제 애플리케이션의 실행 상태를 어떻게 보여줄 것인가?

이 저장소의 주제는 일반적인 LLM self-reflection이나 프롬프트 기반 self-debugging이 아닙니다.

대상은 실제 Application Debugging입니다.

- application logs
- metrics
- distributed traces
- exceptions / stack traces
- HTTP request/response
- SQL / datastore evidence
- JVM/runtime state
- continuous profiling
- deploy/config change

이 증거를 OpenTelemetry, Prometheus, Grafana, Loki, Tempo, Pyroscope 같은 오픈소스 observability 도구로 수집하고, Agent가 제한된 Tool/API를 통해 필요한 증거를 조회하도록 만드는 구조를 다룹니다.

## Reference Direction

~~~text
Application
   │
   ├─ Logs
   ├─ Metrics
   ├─ Traces
   ├─ Exceptions
   └─ Profiles
       │
       ▼
Instrumentation / Collection
OpenTelemetry / Micrometer / Actuator
       │
       ▼
Observability Backends
Prometheus / Loki / Tempo / Pyroscope
       │
       ▼
Debug Evidence Interface
PromQL / LogQL / TraceQL / HTTP API
       │
       ▼
AI Coding Agent
       │
       ├─ evidence retrieval
       ├─ hypothesis
       ├─ source inspection
       ├─ patch
       └─ verification
~~~

## 핵심 경계

~~~text
Raw Log ≠ Debug Context
Dashboard ≠ Agent Interface
Telemetry Storage ≠ Agent Memory
Correlation ≠ Root Cause
Anomaly ≠ Root Cause
Evidence Retrieval ≠ Reasoning
Observability Access ≠ Production Write Access
Patch Success ≠ Incident Resolution
~~~

## 현재 단계

~~~text
Phase 1 Research       1차 완료
Phase 2 Concept        완료
Phase 3 Scope          완료
Phase 4 TOC            완료
Phase 5 Chapter Plan   완료
Phase 6 Draft          1차 초고 완료
Phase 7 Review         1차 편집 리뷰 진행 중
~~~

## Research

- `research/meta/methodology.md`
- `research/catalog/source-catalog.md`
- `research/topics/observability-stack.md`
- `research/design/agent-debug-session-contract-v0.1.md`
- `research/design/bounded-debug-tool-surface-v0.1.md`
- `research/topics/papers-and-benchmarks.md`
- `research/topics/lightweight-log-intelligence.md`

기준일: 2026-10-05


## 현재 집필 상태

서문, 1~22장, Epilogue까지 1차 초고가 완료되었습니다.

다음 작업은 새 장 추가보다 기존 원고의 품질을 높이는 데 집중합니다.

1. 가독성 리뷰 — 1차 진행 중
2. 어려운 용어와 영어 표현 축소 — 1차 진행 중
3. 장간 중복 제거
4. 사례 흐름 보강 — 후반부 우선 보강 완료
5. 근거와 출처 연결 점검
6. 출간용 통합 원고 구성

기준일: 2026-10-05


편집 기록:
- `review/editorial-review-v0.1.md`
