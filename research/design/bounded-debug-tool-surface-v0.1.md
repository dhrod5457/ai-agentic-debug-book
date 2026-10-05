# Bounded Debug Tool Surface v0.1

기준일: 2026-10-05

상태: Research Synthesis / Draft

## 1. 목적

이 문서는 Agent Debug Session Contract를 실제 tool capability에 연결하기 위한 최소 surface를 정의한다.

특정 MCP 구현을 표준으로 선언하지 않는다. Grafana MCP와 Tempo MCP를 baseline으로 사용하되 Kubernetes/JVM diagnostic tool은 별도 adapter가 필요할 수 있다.

## 2. Tool Group

### Discovery

- list_services
- describe_service
- get_runtime_identity

목적:
Agent가 먼저 datasource와 service/version scope를 확인하게 한다.

### Metrics

- query_metric
- query_metric_range
- get_exemplars

Guardrail:
- allowed datasource
- max range
- max series
- max returned samples

### Traces

- search_traces
- get_trace
- get_trace_summary
- compare_traces
- query_trace_metrics

Guardrail:
- max time range
- max trace count
- trace response projection 기본값
- full trace는 explicit expansion

### Logs

- query_logs
- get_logs_by_trace
- get_exception_summary
- get_log_patterns

Guardrail:
- max scan bytes
- max effective range
- enforced environment/service matcher
- max lines/bytes
- sensitive-field redaction

### Profiles

- query_profile
- get_profile_for_span
- compare_profiles

Guardrail:
- allowed profile type
- max range
- top-N projection

### Database

- get_db_query_summary
- get_slow_db_spans
- get_pool_metrics
- get_db_wait_summary

기본적으로 제공하지 않음:
- arbitrary SQL execution
- raw parameter values

### Kubernetes

- get_workload_status
- get_pod_status
- get_k8s_events
- get_rollout_history
- get_image_digest
- get_node_pressure

기본 read-only.

별도 mutation capability:
- exec
- restart
- scale
- patch
- delete

### JVM

OBSERVE_L1:
- get_jvm_metric_summary
- query_existing_profile

CAPTURE_L2:
- capture_thread_dump
- capture_jfr_window

CAPTURE_L3:
- capture_heap_dump
- request_gc_root_analysis

capture tool은 session authority와 escalation reason이 없으면 실행하지 않는다.

### Source / Deployment

- get_source_version
- compare_runtime_to_workspace
- get_deployment_diff
- get_config_diff

## 3. Tool Result Envelope

모든 tool response에 가능한 한 공통 metadata를 붙인다.

- evidence_id
- tool
- source
- environment
- service
- time_range
- runtime_version
- sampled
- truncated
- budget_consumed
- collected_at
- continuation_available

목적은 backend별 response 차이를 없애는 것이 아니라 Agent가 결과의 한계를 놓치지 않게 하는 것이다.

## 4. Tool 호출 정책

기본 정책:
1. discovery/version
2. aggregate
3. representative execution
4. local evidence
5. diagnostic escalation

예외:
- 명시적 stacktrace incident처럼 이미 concrete evidence가 있으면 중간 단계를 건너뛸 수 있다.
- 기존 evidence가 충분하면 추가 query를 하지 않는다.

## 5. Grafana MCP Mapping

Grafana MCP가 직접 담당할 수 있는 영역:
- Prometheus
- Loki
- Tempo
- Pyroscope
- dashboard/alert metadata

추가 adapter가 필요한 영역:
- Kubernetes object/event API
- JFR/thread/heap capture
- Git/source/deployment metadata
- 일부 DB-side diagnostic

따라서 architecture는 단일 MCP server 강제가 아니라 Policy Gateway 아래 여러 provider를 조합하는 형태가 적절하다.

## 6. Policy Gateway 책임

Policy Gateway는 telemetry를 해석하지 않아도 된다.

책임:
- session scope injection
- identity/RBAC
- query budget
- result envelope
- redaction
- evidence ID/provenance
- authority/elevation
- audit

비책임:
- root cause 추론
- patch 생성
- 모델 reasoning

## 7. 왜 범용 shell/python만 주지 않는가

OpenRCA처럼 generic Python executor는 연구 환경에서 유연하지만 production에서는:
- filesystem/network 범위가 넓다.
- arbitrary query를 만들 수 있다.
- 비용 예측이 어렵다.
- audit semantic이 약하다.

따라서 production baseline은 domain-specific bounded tools로 두고, generic executor는 sandboxed analysis workspace나 offline benchmark에서 별도 비교한다.

## 8. 최소 Production Profile

권고 baseline:
- read-only Grafana service account
- disable-write
- datasource UID scope
- Loki enforced matcher
- Loki scan/range guardrail
- bounded Tempo search
- Kubernetes get/list/watch 최소 RBAC
- JVM capture disabled by default
- source/deploy metadata read only
- full audit of tool request/result metadata

## 9. 실험에서 검증할 질문

- bounded tool surface가 raw MCP보다 broad query를 줄이는가
- tool result envelope이 truncation/sampling 오판을 줄이는가
- version-first tool이 wrong-version patch를 줄이는가
- diagnostic escalation이 불필요한 JFR/heap capture를 줄이는가
- domain-specific tool이 generic Python executor보다 accuracy를 잃지 않으면서 safety/cost를 개선하는가