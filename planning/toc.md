# Table of Contents v0.1

기준일: 2026-10-05

## 책의 질문

> 소스코드만 보는 Coding Agent에게 실제 애플리케이션의 실행 상태를 어떻게 보여줄 것인가?

이 책은 "AI가 알아서 디버깅하게 만드는 프롬프트"를 다루지 않는다.

대신 다음 문제를 다룬다.

- 어떤 runtime evidence를 수집해야 하는가
- logs, metrics, traces, profiles를 어떻게 하나의 사건으로 연결하는가
- Agent에게 무엇을 그대로 주고 무엇을 query하게 할 것인가
- Grafana, Prometheus, Loki, Tempo 같은 기존 오픈소스를 어떻게 활용할 것인가
- production telemetry를 Agent에게 열 때 어떤 권한과 비용 경계를 두어야 하는가
- root cause를 찾은 뒤 patch와 verification까지 어떻게 닫힌 loop로 연결할 것인가

---

# Part I. 로그를 주는 것과 디버깅을 가능하게 하는 것은 다르다

## 1장. 소스코드만 보는 Agent는 애플리케이션을 모른다

- 코드가 맞는데 운영에서는 왜 실패하는가
- source state와 runtime state의 차이
- stack trace 하나로 충분하지 않은 이유
- Application Runtime Evidence라는 관점
- 첫 번째 원칙: Raw Log ≠ Debug Context

## 2장. 로그 파일을 통째로 넣으면 왜 실패하는가

- context window는 telemetry storage가 아니다
- interleaved logs와 causal relation 손실
- 로그가 많을수록 추론이 좋아진다는 착각
- token, cost, privacy, stale evidence
- Model Context ≠ Telemetry Working Set
- Push Context에서 Pull Evidence로

## 3장. 디버깅 증거는 한 종류가 아니다

- logs
- metrics
- traces
- exceptions
- profiles
- DB evidence
- JVM diagnostics
- Kubernetes/platform evidence
- deployment/source identity
- Evidence Pyramid와 escalation

---

# Part II. Observability를 Agent의 눈과 귀로 바꾼다

## 4장. OpenTelemetry를 상관관계의 뼈대로 사용한다

- traceId/spanId/resource
- log-trace correlation
- context propagation
- Collector의 역할
- filtering, redaction, sampling
- correlation이 causation은 아닌 이유

## 5장. Prometheus로 문제 공간을 먼저 줄인다

- metrics가 답이 아니라 범위 축소 도구인 이유
- RED signals
- PromQL과 machine-readable API
- exemplars
- anomaly에서 representative execution으로
- bounded query

## 6장. Tempo로 실제 실행 경로를 따라간다

- trace와 span
- TraceQL
- trace-derived metrics
- representative trace
- trace diff
- distributed failure에서 causal path 찾기
- Tempo MCP와 LLM-oriented representation

## 7장. Loki에서 같은 execution의 로그를 찾는다

- label과 structured metadata
- trace_id를 label로 쓰면 안 되는 이유
- LogQL
- query range와 scan budget
- stack trace projection
- raw log보다 event evidence

## 8장. Pyroscope와 JVM diagnostics로 더 깊이 내려간다

- continuous profiling
- trace-to-profile
- JFR
- thread dump
- heap dump
- Evidence Escalation
- Capture Authority ≠ Read Authority

---

# Part III. Observability API를 Agent Debug Interface로 만든다

## 9장. Grafana MCP는 어디까지 해결해 주는가

- Grafana MCP의 실제 tool surface
- Prometheus/Loki/Tempo/Pyroscope 연결
- read-only
- RBAC
- Loki query guardrail
- Capability ≠ Query ≠ Mutation
- 범용 MCP가 해결하지 않는 것

## 10장. Agent Debug Session Contract

- incident scope
- runtime identity
- evidence budget
- evidence provenance
- hypothesis
- contradicting evidence
- reproduction
- patch
- verification
- No Evidence, No Claim

## 11장. Agent에게 주는 Tool은 작고 제한적이어야 한다

- query_metric
- search_traces
- get_logs_by_trace
- get_profile_for_span
- get_k8s_events
- capture_jfr_window
- arbitrary shell/python의 장단점
- bounded tool surface
- query budget과 audit

## 12장. Evidence와 추론을 분리한다

- correlation과 root cause
- supporting evidence
- contradicting evidence
- unknowns
- retrieval miss와 reasoning error
- OpenRCA가 보여주는 구조
- lucky diagnosis를 구분하는 방법

---

# Part IV. Spring Boot 애플리케이션을 실제로 디버깅한다

## 13장. Spring Boot에 Agent가 읽을 수 있는 흔적을 남긴다

- Micrometer Observation
- tracing
- MDC
- structured JSON logging
- service.version
- deployment identity
- HTTP client context propagation
- DB spans
- 최소 instrumentation

## 14장. DB Connection Pool Exhaustion을 찾는다

- symptom
- Prometheus
- exemplar
- Tempo
- Hikari metrics
- Loki
- 잘못된 slow SQL 가설 제거
- patch와 before/after verification

## 15장. Lock Contention은 로그만으로 보이지 않는다

- low CPU, high latency
- slow trace
- wall profile
- JFR/thread evidence
- expensive evidence escalation
- source location으로 연결

## 16장. N+1은 느린 SQL 한 개가 아니다

- db.query.summary
- span multiplicity
- sanitized SQL
- parameters default deny
- query count와 request size correlation
- patch verification

## 17장. 장애가 코드가 아닐 때

- Pod restart
- OOMKilled
- eviction
- failed scheduling
- Kubernetes Events의 한계
- rollout revision
- image digest
- platform evidence를 first-class로 다루기

## 18장. 이미 다른 버전이 운영 중이라면

- service.version
- image digest
- deployment revision
- commit SHA
- current workspace와 incident runtime 비교
- stale incident와 wrong-version patch

---

# Part V. Production Agentic Debugging

## 19장. Production Telemetry를 Agent에게 열어도 되는가

- read-only baseline
- tenant/environment scope
- service account
- RBAC
- redaction
- secret/PII
- query cost
- audit

## 20장. Root Cause에서 Patch로 넘어갈 때 경계가 바뀐다

- observation authority
- diagnostic capture authority
- development mutation
- production mutation
- approval
- sandbox
- rollback

## 21장. 수정했다고 끝난 것이 아니다

- reproduction
- same-signal verification
- before/after telemetry
- regression
- residual anomaly
- incident resolution과 test pass의 차이

## 22장. Agentic Debugging을 평가하는 방법

- Source only
- Raw logs
- Observability MCP
- Debug Workflow Layer
- outcome score
- process score
- evidence precision/recall
- cost/safety metrics
- replayability
- benchmark limitations

---

# Epilogue. Agent에게 더 많은 로그가 아니라 더 좋은 관측 인터페이스를 준다

최종 메시지:

> Agent가 디버깅을 잘하도록 만드는 핵심은 모든 정보를 prompt에 넣는 것이 아니다.
> 실행 중인 시스템에서 필요한 증거를 안전하게 찾고, 연결하고, 검증할 수 있는 인터페이스를 만드는 것이다.
