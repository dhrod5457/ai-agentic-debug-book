# Research Map

기준일: 2026-10-05

## 중심 연구 문제

소스코드만 보는 Coding Agent에게 실제 Application Runtime Evidence를 어떻게 제공할 것인가?

## R1. Instrumentation / Collection

질문:
- 어떤 telemetry를 반드시 수집해야 하는가
- trace_id/span_id/request_id는 어디에서 주입하고 어떻게 전파하는가
- logs/metrics/traces/profiles를 어떻게 correlation하는가
- sampling과 redaction은 어디서 적용하는가

1차 자료:
- OpenTelemetry
- OpenTelemetry Collector
- Micrometer / Spring Boot Observability
- Grafana Alloy

## R2. Open-source Storage / Query

질문:
- Agent가 metric anomaly를 어떻게 찾는가
- metric에서 trace로 어떻게 내려가는가
- trace에서 log/profile로 어떻게 연결하는가
- high-cardinality correlation key를 어떻게 저장하는가

1차 자료:
- Prometheus
- Loki
- Tempo
- Pyroscope
- Grafana

## R3. Agent Access Layer

질문:
- Agent에게 raw HTTP API를 줄 것인가 MCP/tool로 감쌀 것인가
- tool response를 얼마나 줄여야 하는가
- query time range/result limit/cost를 어떻게 제한하는가
- read-only 권한을 어떻게 강제하는가

현재 핵심 오픈소스:
- Grafana MCP server
- Tempo MCP endpoint

특히 Grafana MCP는 Prometheus/Loki/Tempo/Pyroscope를 MCP-compatible client에 연결하고 read-only mode와 RBAC를 지원하므로 baseline implementation으로 우선 분석한다.

## R4. Evidence Selection / Context Construction

질문:
- raw telemetry를 어떤 순서로 좁혀야 하는가
- raw log를 template/sequence 단위로 어떻게 압축하는가
- deterministic rule과 경량 anomaly model을 어디까지 사용할 것인가
- 대표 trace와 anomalous log window를 어떻게 선정하는가
- supporting/contradicting evidence를 어떻게 분리하는가
- sampled/truncated 결과를 Agent에게 어떻게 알리는가

현재 가설:
metric anomaly → exemplar/trace → span → correlated logs → template/sequence anomaly → profile/source/deploy version

핵심 자료:
- Drain3
- DeepLog
- LogBERT
- LogGPT / LogLLM
- LogRAIL

설계 원칙:
- Raw Log ≠ Agent Context
- anomaly model은 RCA 엔진이 아니라 evidence narrowing 계층이다.
- 상시 탐지는 저비용 계층이 담당하고 LLM은 좁혀진 evidence를 reasoning한다.

## R5. Root Cause Reasoning

질문:
- correlation과 causation을 어떻게 구분하는가
- evidence가 없어서 틀린 것과 evidence를 잘못 해석해서 틀린 것을 어떻게 나누는가
- topology/dependency graph를 어떻게 제공하는가

핵심 자료:
- OpenRCA
- How Far Can Root Cause Analysis Go on Real-World Telemetry Data?
- MicroRCA
- MicroRCA-Agent

## R6. Coding Agent Repair Loop

질문:
- runtime evidence를 source location과 어떻게 연결하는가
- patch 전에 reproduction을 어떻게 확보하는가
- patch 후 동일 telemetry에서 개선을 어떻게 검증하는가
- code change와 deploy/runtime version을 어떻게 연결하는가

핵심 자료:
- NExT
- Automated Program Repair 계열 후속 targeted research

## R7. Security / Privacy

질문:
- production logs에 포함된 secret/PII를 어떻게 제거하는가
- tenant별 telemetry 접근을 어떻게 격리하는가
- Agent가 arbitrary expensive query를 실행하지 못하게 어떻게 제한하는가
- observability read와 production mutation 권한을 어떻게 분리하는가

확인할 구현:
- Grafana MCP RBAC / read-only mode
- OpenTelemetry Collector filtering/redaction
- Grafana/Loki multi-tenancy

## R8. Evaluation

비교할 시스템:
A. 소스코드만 제공
B. 소스 + raw log dump
C. 소스 + bounded observability tools
D. 소스 + correlated multi-signal evidence tools

측정 후보:
- root cause accuracy
- evidence retrieval precision/recall
- time to first correct hypothesis
- telemetry bytes read
- context tokens consumed
- query count
- irrelevant evidence ratio
- correct patch rate
- regression pass rate
- production query cost

## 다음 Targeted Research

1. Grafana MCP 실제 tool inventory와 read-only/security 경계
2. Tempo MCP와 LLM-optimized response schema
3. Loki/Prometheus query guardrail
4. OTel Collector redaction/sampling/filter pipeline
5. Spring Boot에서 trace-log correlation 최소 구성
6. OpenRCA RCA-agent의 retrieval/tool 구현 분석
7. deploy version / commit SHA를 OpenTelemetry Resource로 연결하는 사례
8. patch 전후 telemetry diff를 자동 검증하는 연구와 OSS
9. Drain3 + DeepLog/LogBERT/small Transformer 기반 경량 log narrowing 실험
10. raw log → lightweight detector → MCP/tool → LLM Agent 연결 시 token/cost/RCA 정확도 비교