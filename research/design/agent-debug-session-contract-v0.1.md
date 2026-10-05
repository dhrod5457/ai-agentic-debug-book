# Agent Debug Session Contract v0.1

기준일: 2026-10-05

상태: Research Synthesis / Draft

이 문서의 Contract는 외부 표준이 아니라 이 책의 연구를 위해 정의한 synthesis다.

## 1. 목적

Agent Debug Session Contract는 Coding Agent에게 무제한 telemetry를 주는 대신, 하나의 incident investigation에서 필요한 범위·증거·권한·추론·검증을 명시적으로 묶는 계약이다.

핵심 목표:
- incident scope를 고정한다.
- runtime version과 source workspace의 일치 여부를 확인한다.
- evidence retrieval budget을 제한한다.
- evidence와 hypothesis를 분리한다.
- diagnostic escalation을 기록한다.
- patch 전후 verification을 같은 session에 연결한다.

## 2. Session Lifecycle

OPEN
→ SCOPED
→ EVIDENCE_COLLECTING
→ HYPOTHESIS_TESTING
→ ROOT_CAUSE_CANDIDATE
→ REPRODUCED
→ PATCHED
→ VERIFIED
→ CLOSED

중간 종료 상태:
- BLOCKED_EVIDENCE_MISSING
- BLOCKED_VERSION_MISMATCH
- BLOCKED_AUTHORITY
- INCONCLUSIVE

## 3. Contract Sections

### A. Incident Scope

필수:
- incident_id
- reported_at
- environment
- affected_service
- symptom
- investigation_time_range

선택:
- endpoint/operation
- tenant scope
- region/cluster
- user-visible impact

규칙:
- Agent는 scope 밖의 production telemetry를 자동 확장하지 않는다.
- time range 확대는 budget change로 기록한다.

### B. Runtime Identity

최소:
- service.name
- service.version
- deployment.environment.name

권장:
- container image digest
- deployment revision/id
- git commit SHA
- config version

판정:
- MATCHED: incident runtime과 workspace source가 일치
- DIFF_KNOWN: 차이는 있으나 diff를 확보
- MISMATCH_UNKNOWN: 대응 source를 찾지 못함

MISMATCH_UNKNOWN 상태에서는 source patch 신뢰도를 낮추거나 작업을 중단한다.

### C. Authority Profile

권한을 다음으로 분리한다.

OBSERVE_L0
- metrics/logs/traces/status read

OBSERVE_L1
- profiles, sanitized DB evidence, detailed trace/log projection

CAPTURE_L2
- JFR dump, thread dump 등 새로운 diagnostic capture

CAPTURE_L3
- heap dump, deep DB diagnostic

MUTATE_DEV
- local/test patch and restart

MUTATE_PROD
- production configuration/deployment mutation

기본 Agent Debug Session은 OBSERVE_L0/L1 + MUTATE_DEV로 시작한다. MUTATE_PROD는 별도 승인 경계다.

### D. Evidence Budget

최소 budget dimension:
- max investigation time range
- max query count
- max telemetry scan bytes where backend supports it
- max rows/log lines
- max trace count
- max returned bytes/tokens
- max diagnostic escalation level

budget이 초과되면 silently truncate하지 않는다. 다음 중 하나를 반환한다.

- truncated=true
- sampled=true
- budget_exceeded=true
- continuation/requery hint

### E. Evidence Record

각 evidence는 최소 다음 provenance를 가진다.

- evidence_id
- evidence_type
- source backend/tool
- query or retrieval intent
- environment/service
- time range
- runtime identity if available
- trace/span/request correlation IDs
- sampled/truncated flag
- collected_at
- summary
- raw reference or replay handle

Evidence Type:
- METRIC
- TRACE
- LOG
- EXCEPTION
- PROFILE
- DB
- JVM
- K8S
- DEPLOYMENT
- SOURCE_DIFF
- TEST_RESULT

### F. Hypothesis Record

각 hypothesis는 evidence와 별도 artifact다.

필수:
- hypothesis_id
- statement
- status
- supporting_evidence_ids
- contradicting_evidence_ids
- missing_evidence
- next_discriminating_action

status:
- OPEN
- WEAKENED
- SUPPORTED
- REJECTED
- ROOT_CAUSE_CANDIDATE

Agent는 같은 evidence를 사실과 원인으로 동시에 기록하지 않는다.

### G. Reproduction

root cause 후보를 patch하기 전에 가능한 경우 reproduction artifact를 만든다.

필드:
- reproduction_id
- environment
- workload/input
- preconditions
- observed failure
- evidence_ids
- deterministic 여부

production incident가 직접 재현 불가능하면 가장 가까운 controlled reproduction과 차이를 명시한다.

### H. Patch

필드:
- patch_id
- target source version
- changed components
- hypothesis_id
- expected observable change
- risk

중요:
Patch는 hypothesis와 연결되어야 하며 단순히 test가 통과했다는 이유로 root cause evidence를 대체하지 않는다.

### I. Verification

verification은 patch 전후 동일한 관찰 항목을 비교하는 것을 우선한다.

필드:
- verification_id
- reproduction_id
- before evidence
- after evidence
- functional result
- regression result
- telemetry result
- residual anomaly
- verdict

verdict:
- VERIFIED
- PARTIAL
- FAILED
- INCONCLUSIVE

## 4. Evidence Retrieval Order

기본 순서는 강제가 아니라 default policy다.

1. scope/runtime identity
2. aggregate metric
3. representative trace
4. correlated logs/exceptions
5. profile/DB/platform evidence
6. JFR/thread diagnostic
7. heap/deep diagnostic

Agent는 앞 단계에서 root cause를 구분할 수 없을 때만 다음 단계로 escalation한다.

## 5. 최소 Session Output

Session 종료 시 최소 다음이 남아야 한다.

- 어떤 runtime version을 조사했는가
- 어떤 evidence를 실제로 읽었는가
- 어떤 hypothesis를 만들고 버렸는가
- 최종 root cause 후보의 supporting/contradicting evidence는 무엇인가
- 어떤 patch를 왜 만들었는가
- patch 후 동일 failure가 사라졌는가
- 아직 확인하지 못한 unknown은 무엇인가

## 6. Contract Invariants

### I1. No Evidence, No Claim
root cause claim에는 최소 하나 이상의 evidence reference가 필요하다.

### I2. Version Before Patch
runtime identity 확인 없이 production-derived patch를 확정하지 않는다.

### I3. Truncation Must Be Visible
잘린 결과를 완전한 결과처럼 취급하지 않는다.

### I4. Escalation Is Explicit
JFR/thread/heap 같은 capture는 별도 escalation event로 남긴다.

### I5. Observation Is Read-only by Default
관측과 production mutation을 같은 capability로 묶지 않는다.

### I6. Verification Reuses the Symptom Signal
가능하면 incident를 처음 감지한 signal을 patch 후에도 다시 확인한다.

## 7. 기존 자료와 연결

- Grafana MCP: bounded query, write/query/tool exposure 분리의 구현 근거
- OpenRCA: telemetry working set과 model context 분리, atomic investigation의 연구 근거
- OpenTelemetry: signal correlation/provenance의 표준 기반
- BTS-AgentBench: read-only telemetry를 typed, bounded episode와 evidence attribution으로 변환하는 최신 benchmark 사례
- NExT: execution evidence가 program reasoning/repair에 유용하다는 근거

## 8. 아직 미확정인 부분

- canonical serialization format
- MCP resource/tool과 Contract field의 직접 mapping
- evidence confidence score 사용 여부
- automatic hypothesis generation rule
- cross-session memory 승격 기준

이 항목은 실험 전 고정하지 않는다.