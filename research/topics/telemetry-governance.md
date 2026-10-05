# Topic — Telemetry Governance Before Agent Access

기준일: 2026-10-05

## 문제

Agent 접근을 Grafana/Tempo에서 막는 것만으로는 늦을 수 있다. secret과 개인정보가 telemetry backend에 이미 저장됐다면 query layer 제한만으로 data governance 문제가 해결되지 않는다.

따라서 Agentic Debugging은 수집 pipeline과 retrieval gateway 두 군데에 control을 둬야 한다.

## 1. OpenTelemetry Collector를 Policy Point로 사용

OpenTelemetry Collector processor는 telemetry를 backend로 보내기 전에 filter, transform, enrich, redact할 수 있다.

관련 processor:
- filter processor
- attributes/resource processor
- transform processor
- redaction processor
- probabilistic/tail sampling processor

## 2. Filter

OTTL condition으로 불필요한 telemetry 자체를 drop할 수 있다.

Agentic Debugging 적용 예:
- 허용하지 않는 environment 제거
- health-check noise 제거
- 특정 sensitive endpoint span 제거
- debug value가 큰 이벤트 제외

## 3. Redaction

Redaction Processor는 allowed attribute list 밖의 속성을 제거하거나 blocked value를 mask할 수 있다.

공식 use case 자체가 sensitive field leakage와 compliance 방지다.

Agent access 전에 검토할 field:
- authorization
- cookie/session
- API key/token
- request/response body
- DB bind parameter
- email/phone/student number
- tenant/user identifiers

## 4. Sampling

모든 trace를 저장하는 것과 Agent가 모든 trace를 읽는 것은 다른 문제다.

sampling은 storage/cost 문제를 줄이지만 debugging evidence가 사라질 위험이 있다.

따라서 책에서는 다음을 분리한다.

Collection Sampling ≠ Agent Retrieval Sampling

특히 error/slow trace를 tail sampling으로 보존하는 정책과 Agent query 결과 limit은 별도 설계 대상이다.

## 5. Processor Cost

OpenTelemetry 공식 문서도 advanced transformation이 Collector performance에 큰 영향을 줄 수 있다고 경고한다.

즉 privacy filtering을 무제한 복잡한 OTTL로 만들면 observability pipeline 자체가 장애 요인이 될 수 있다.

검증 항목:
- collector CPU
- memory
- dropped telemetry
- queue/backpressure
- transform latency

## 6. Two-stage Governance

권고 구조:

Application
  ↓
Collector Policy
  - redact
  - filter
  - enrich
  - sample
  ↓
Telemetry Backend
  ↓
Agent Gateway Policy
  - identity/RBAC
  - environment/tenant scope
  - query budget
  - output truncation
  - audit
  ↓
Agent

Collector 정책은 저장하면 안 되는 데이터를 막고, Gateway 정책은 저장된 데이터 중 Agent가 볼 수 있는 범위를 제한한다.