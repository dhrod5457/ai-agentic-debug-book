# Synthesis v0.2 — Agent Debug Evidence Architecture

기준일: 2026-10-05

## 현재까지의 가장 강한 결론

AI Coding Agent에게 Application Debugging 정보를 제공하는 기본 단위는 raw log가 아니라 queryable, correlated, bounded runtime evidence다.

## Reference Architecture

Application
  ↓
Instrumentation
  OpenTelemetry / Micrometer / structured logging
  ↓
Collector Policy
  filter / redact / enrich / sample
  ↓
Observability Backends
  Prometheus / Loki / Tempo / Pyroscope
  ↓
Observability Agent Interface
  Grafana MCP / Tempo MCP
  ↓
Debug Workflow & Policy Layer
  scope / budget / provenance / hypothesis
  ↓
Coding Agent
  source inspection / patch / test
  ↓
Verification
  reproduction + telemetry before/after

## 왜 Debug Workflow Layer가 별도로 필요한가

Grafana MCP와 Tempo MCP는 이미 강력한 Agent-native observability interface를 제공한다. 그러나 범용 observability assistant이므로 다음을 자동으로 보장하지 않는다.

- incident의 시간/환경/service scope 고정
- evidence provenance 기록
- sampled/truncated 여부의 일관된 전달
- supporting/contradicting evidence 구분
- runtime version과 source version 확인
- patch 전 reproduction
- patch 후 before/after verification

따라서 이 책의 독자적 설계 대상은 새로운 telemetry backend가 아니라 이 중간 workflow layer다.

## 핵심 설계 원칙 v0.2

### P1. Raw Log ≠ Debug Context
로그 전체가 아니라 사건에 필요한 evidence projection을 만든다.

### P2. Model Context ≠ Telemetry Working Set
대량 telemetry는 backend/query runtime에 두고 모델에는 좁혀진 결과만 넣는다.

### P3. Correlation First
trace/span/resource/deployment identity를 이용해 signal을 연결한다.

### P4. Progressive Narrowing
metric → representative trace → span → logs/profile → source 순서로 범위를 줄인다.

### P5. Bounded Query
Agent query에는 time range, scan bytes, result count, datasource/service/environment scope를 둔다.

### P6. Capability ≠ Query ≠ Mutation
tool discovery, telemetry query, production mutation 권한을 분리한다.

### P7. Collection Governance ≠ Retrieval Governance
Collector에서 저장 불가 데이터를 제거하고, Agent gateway에서 접근 가능 데이터를 다시 제한한다.

### P8. Evidence ≠ Hypothesis
관측 사실과 Agent 추론을 다른 artifact로 기록한다.

### P9. Retrieval Failure ≠ Reasoning Failure
증거를 못 찾은 경우와 찾았지만 잘못 판단한 경우를 별도 평가한다.

### P10. RCA ≠ Debugging Complete
root cause를 맞혔다고 끝내지 않고 patch/reproduction/before-after verification까지 연결한다.

## 실험 방향

대표 Spring Boot failure corpus를 만든다.

비교:
A. Source only
B. Source + raw logs
C. Source + Grafana MCP
D. Source + Debug Workflow Layer + Grafana MCP

측정:
- root cause accuracy
- correct patch rate
- regression pass rate
- evidence retrieval precision
- telemetry scanned bytes
- context tokens
- tool calls
- time to first correct hypothesis
- unsafe/broad query count
- provenance completeness

## 다음 연구

1. Grafana MCP Loki guardrail 구현 코드 상세 추적
2. Tempo MCP response schema와 token reduction 정량 분석
3. OpenTelemetry tail sampling이 debugging evidence 보존에 미치는 영향
4. git commit/image digest/deployment ID를 telemetry Resource에 연결하는 표준/실전 사례
5. JVM thread dump, heap dump, JFR을 Agent evidence source로 넣는 방법
6. SQL/DB evidence를 Agent에게 안전하게 제공하는 방법
7. Kubernetes events/config/deploy diff와 application telemetry correlation
8. patch 후 automated verification loop 관련 OSS/논문