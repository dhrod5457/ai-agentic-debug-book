# Synthesis v0.3 — Full Runtime Evidence Surface

기준일: 2026-10-05

## 1. 현재 Reference Architecture

Application / JVM / DB / Kubernetes
        ↓
Instrumentation & Diagnostics
        ↓
OpenTelemetry / JFR / DB spans / K8s status
        ↓
Collector Governance
        ↓
Prometheus / Loki / Tempo / Pyroscope
        ↓
Agent Observability Interface
Grafana MCP / Tempo MCP / bounded diagnostic tools
        ↓
Debug Workflow & Policy Layer
        ↓
Coding Agent
        ↓
Source / Patch / Reproduction
        ↓
Before-After Verification

## 2. Runtime Evidence Pyramid

Agent는 처음부터 가장 비싼 evidence를 요청하지 않는다.

Level 0 — Low cost
- metrics
- logs
- traces
- service/deployment metadata

Level 1 — Focused observability
- exemplar-selected traces
- correlated logs
- span profiles
- DB query summary
- Kubernetes object status/events

Level 2 — JVM diagnostics
- JFR incident window
- thread dump
- lock/contention evidence

Level 3 — High cost / sensitive
- heap dump
- GC root analysis
- DB execution plan / deep session state
- container exec

원칙:
Evidence Cost와 Diagnostic Value를 함께 고려하여 escalation한다.

## 3. Evidence Dimension

Debug context는 이제 단순 logs/metrics/traces 세 축보다 넓다.

Request Evidence
- trace/span
- HTTP/RPC

Application Evidence
- structured logs
- exceptions

Database Evidence
- query summary
- latency/error
- pool state
- DB-side wait/plan

Runtime Evidence
- JVM/JFR
- thread state
- CPU/heap profile

Platform Evidence
- Pod/Deployment/Node
- Kubernetes events
- rollout revision

Artifact Evidence
- service.version
- image digest
- deployment ID/revision
- source commit/config version

## 4. 가장 중요한 새 원칙

### P11. Runtime Evidence Without Version Is Incomplete
Agent가 보는 telemetry가 어떤 binary/image/source에서 발생했는지 모르면 source patch가 잘못될 수 있다.

### P12. Evidence Escalation
낮은 비용의 telemetry로 먼저 범위를 줄이고 thread/JFR/heap 같은 고비용 evidence는 필요할 때만 수집한다.

### P13. Diagnostic Capture ≠ Diagnostic Read
JFR/heap/thread dump를 새로 생성하는 권한과 기존 observability 데이터를 읽는 권한은 분리한다.

### P14. Query Summary Before Query Text
DB evidence는 low-cardinality summary와 duration/error를 먼저 사용하고 SQL parameter는 기본 차단한다.

### P15. Platform Evidence Is First-Class
Kubernetes 장애를 application code 문제로 오인하지 않도록 platform status/event/deployment evidence를 별도 축으로 둔다.

### P16. Raw Logs Are Not Agent Context
원본 로그 저장과 Agent에게 제공할 debugging context를 분리한다.

### P17. Compress Before Reasoning
Drain3 같은 template mining, aggregation, 경량 anomaly filtering으로 evidence를 줄인 뒤 LLM reasoning을 수행한다.

### P18. Preserve Correlation Keys
로그를 정규화하더라도 trace_id, span_id, request_id, service.version 같은 debugging key는 유지한다.

### P19. Cheap Detection, Expensive Reasoning
상시 탐지는 rule, DeepLog, LogBERT, small Transformer 같은 저비용 계층이 담당하고 LLM은 선택된 evidence에 사용한다.

### P20. Anomaly Is a Retrieval Hint, Not a Root Cause
anomaly score는 Agent가 조사할 evidence를 좁히는 신호이며 root cause 판정 자체가 아니다.

## 5. Agent Debug Session Contract 초안

Incident Scope
- environment
- service
- time range
- symptom

Runtime Identity
- service.version
- image digest
- deployment revision
- source commit if available

Evidence Budget
- max time range
- max scanned bytes
- max result rows
- max traces
- diagnostic escalation level

Evidence
- metrics
- traces
- logs
- DB
- runtime/JVM
- platform/Kubernetes

Reasoning
- hypotheses
- supporting evidence
- contradicting evidence
- unknowns

Action
- reproduce
- patch
- verification

## 6. 대표 Spring Boot 실험 시나리오 후보

### Scenario A — DB pool exhaustion
Prometheus → Tempo → DB span → Hikari metrics → logs

### Scenario B — Lock contention
HTTP latency → trace → wall profile/JFR → thread/lock evidence

### Scenario C — Memory leak
heap/GC metric → JFR allocation → profile → controlled heap escalation

### Scenario D — Bad deployment
error spike → service.version/image digest → Kubernetes revision → source diff

### Scenario E — Trace propagation break
cross-service trace gap → client construction/config inspection → correlation recovery

### Scenario F — Pod instability
HTTP failures → pod restart/status → Kubernetes events → OOM/eviction/config evidence

## 7. 다음 조사 우선순위

1. Drain3 + DeepLog/LogBERT/small Transformer를 이용한 log evidence reduction 실험
2. raw log dump와 narrowed log evidence를 LLM Agent에 각각 연결했을 때 token/cost/RCA 정확도 비교
3. Agent Debug Session Contract를 실제 schema로 설계
4. Grafana MCP + Kubernetes/JFR tool을 하나의 bounded tool surface로 조합하는 방법
5. JFR parser/CLI/OSS 중 Agent-friendly summary 생성 도구 조사
6. DB execution plan/slow-query OSS 및 query sanitization 조사
7. 대표 Spring Boot failure corpus 설계
8. Source-only / raw-log / MCP / workflow-layer 비교 실험 설계