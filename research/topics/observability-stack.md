# Topic — Open Source Observability Stack for Agentic Debugging

기준일: 2026-10-05

## 핵심 질문

> Grafana, Prometheus 같은 기존 오픈소스 observability stack을 AI Coding Agent의 Application Debugging 입력 계층으로 사용할 수 있는가?

1차 결론은 가능하다. 다만 Dashboard 화면을 Agent에게 보여주는 것이 아니라 queryable evidence layer로 사용해야 한다.

## Reference Stack

Application
  ↓
OpenTelemetry / Micrometer / Actuator
  ↓
OpenTelemetry Collector / Grafana Alloy
  ↓
Prometheus + Loki + Tempo + Pyroscope
  ↓
Grafana / Backend APIs
  ↓
Grafana MCP 또는 Tempo MCP
  ↓
AI Coding Agent

## 1. Prometheus — 현상의 범위를 좁힌다

Agent의 첫 질문은 로그 전체 검색보다 anomaly window와 affected service를 찾는 것이어야 한다.

대표 질문:
- error rate가 언제 증가했는가
- p95/p99 latency가 어느 endpoint에서 증가했는가
- JVM/DB pool/CPU/GC 신호가 같은 시간대에 변했는가

Prometheus HTTP API와 Grafana MCP의 Prometheus tool을 사용하면 Agent가 PromQL instant/range query를 JSON으로 받을 수 있다.

설계 원칙 후보: broad telemetry dump보다 bounded query를 우선한다.

## 2. Exemplars — 집계 지표에서 실제 요청으로 이동한다

Metric은 집계 신호다. exemplar의 trace ID를 이용하면 latency spike에서 representative request trace로 내려갈 수 있다.

흐름:
metric anomaly → exemplar → trace_id → concrete trace

## 3. Tempo — causal execution path를 찾는다

Tempo는 trace ID 조회, TraceQL search, duration/status/service filter, tag discovery를 API로 제공한다.

2026년 현재 Tempo 공식 문서에는 Agent용 MCP endpoint가 있으며 TraceQL search, trace retrieval, span metrics, attribute discovery를 지원한다. 일부 API는 LLM용 simplified JSON representation도 제공한다.

이 점은 이 책의 핵심 주장을 직접 뒷받침한다. Observability backend 자체가 Agent-native evidence interface로 진화하고 있다.

## 4. Loki — 동일 execution의 로그를 찾는다

Trace를 찾은 뒤 동일 trace_id/request_id의 로그를 조회한다.

저장 원칙:
- service_name, namespace: 낮은 cardinality label 후보
- trace_id, request_id: structured metadata 후보
- user_id, request body, headers: privacy/security 관점에서 수집 자체를 신중히 결정

trace_id를 stream label로 사용하면 cardinality가 폭증할 수 있으므로 correlation 편의 때문에 저장 모델을 망가뜨리면 안 된다.

## 5. Pyroscope — resource hotspot을 source line까지 연결한다

성능 문제는 slow span에서 끝나지 않는다. CPU/heap profile을 span과 연결하면 특정 실행 구간의 resource hotspot을 method/source line 수준까지 좁힐 수 있다.

흐름:
slow span → profile → hot method/source line → source inspection

## 6. Grafana MCP — 현재 가장 직접적인 오픈소스 사례

Grafana는 open source MCP server를 제공하며 AI assistant가 Prometheus, Loki, Tempo, Pyroscope 등 datasource를 조회할 수 있다.

공식 문서에서 확인한 주요 특징:
- PromQL instant/range query
- LogQL query와 log pattern 탐색
- Tempo trace search, metrics, trace diff, attribute exploration
- dashboard summary/panel query
- service account + RBAC
- read-only mode
- Claude Code, Codex CLI, Cursor 등 client setup

따라서 이 책의 실습은 처음부터 custom MCP server를 만드는 것보다 Grafana MCP를 baseline으로 삼고, 그 위에 Debugging-specific policy와 evidence workflow를 추가하는 방향이 현실적이다.

## 7. OpenTelemetry — Agent를 위한 새 로그 포맷보다 먼저 필요한 것

Agentic Debugging의 전제는 AI 전용 로그 문법이 아니라 context propagation과 correlation이다.

핵심 field 후보:
- timestamp
- service.name
- deployment.environment
- trace_id
- span_id
- request_id
- exception.type
- exception.message
- code location
- deployment/build version

같은 execution과 같은 배포 버전을 재구성할 수 있는 것이 중요하다.

## 8. Java/Spring 대표 실습 후보

Application:
- Spring Boot
- Actuator
- Micrometer Observation
- Micrometer Tracing 또는 OTel instrumentation
- structured logging

Observability:
- Prometheus
- Loki
- Tempo
- Pyroscope
- Grafana
- Grafana MCP

Failure scenario:
1. DB connection pool exhaustion
2. slow SQL / N+1
3. external HTTP timeout
4. thread pool starvation
5. Redis latency/cache miss storm
6. JVM GC pause
7. retry amplification
8. trace context propagation loss

각 실습은 symptom → metric → trace → logs/profile → source → hypothesis → fix → before/after verification 순서로 고정한다.

## 9. 중요한 한계

- correlation은 causation이 아니다.
- telemetry가 존재해도 Agent reasoning이 틀릴 수 있다.
- sampling으로 필요한 evidence가 사라질 수 있다.
- trace/log에는 개인정보와 secret이 섞일 수 있다.
- broad Agent query가 observability backend 자체를 과부하시킬 수 있다.
- production read access와 remediation write authority는 분리해야 한다.