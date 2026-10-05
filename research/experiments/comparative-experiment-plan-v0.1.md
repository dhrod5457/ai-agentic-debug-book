# Comparative Experiment Plan v0.1

기준일: 2026-10-05

상태: Research Experiment Draft

## 1. 연구 질문

RQ1. Source code만 보는 Agent보다 runtime evidence를 사용할 수 있는 Agent가 root cause를 더 정확히 찾는가?

RQ2. raw log dump보다 bounded observability tools가 더 적은 context로 더 정확한 diagnosis를 만드는가?

RQ3. 범용 Grafana MCP 위에 Debug Session Contract/workflow를 추가하면 evidence quality와 patch verification이 개선되는가?

RQ4. 추가 observability access가 불필요한/broad production query를 늘리지는 않는가?

## 2. 비교군

### A. Source Only
- repository
- tests/build tools
- incident symptom
- runtime telemetry 접근 없음

### B. Source + Raw Log Bundle
- A 조건
- incident time window의 preselected raw application log bundle

### C. Source + Observability MCP
- A 조건
- Grafana MCP/Tempo MCP read-only
- Prometheus/Loki/Tempo/Pyroscope query 가능
- 기본 backend guardrail 적용

### D. Source + Debug Workflow Layer
- C 조건
- Agent Debug Session Contract 적용
- incident scope/runtime version/evidence budget 강제
- evidence/hypothesis 분리
- escalation policy
- verification 요구

## 3. 통제 변수

가능하면 동일하게 고정한다.

- model/version
- reasoning effort
- repository snapshot
- failure snapshot
- tool timeout
- wall-clock budget
- turn/tool-call budget
- system instruction의 일반 coding 능력 부분
- test environment

## 4. Primary Metrics

### Diagnosis
- Root Cause Exact Match
- Root Cause Component Match
- Root Cause Reason Match

### Repair
- Correct Patch Rate
- Reproduction Success
- Regression Test Pass
- Incident Verification Pass

### Evidence
- Minimum Evidence Recall
- Evidence Precision
- Unsupported Claim Count
- Contradicting Evidence Handling

## 5. Efficiency Metrics

- total input/output tokens
- observability result bytes returned to model
- backend scan bytes where measurable
- tool/query count
- time to first correct hypothesis
- time to verified patch
- diagnostic escalation count

## 6. Safety / Operational Metrics

- out-of-scope environment query count
- over-wide time range attempts
- blocked query count
- sensitive-field exposure count
- mutation attempt count
- unnecessary L2/L3 diagnostic capture

## 7. Trajectory Scoring

최종 정답만 보지 않는다.

trajectory에서 확인:
1. runtime version을 확인했는가
2. symptom signal로 범위를 좁혔는가
3. representative execution을 찾았는가
4. evidence와 hypothesis를 구분했는가
5. 반대 evidence를 다뤘는가
6. patch 전에 reproduction 또는 equivalent evidence를 확보했는가
7. patch 후 동일 symptom signal을 재검증했는가

## 8. Gold Evidence 평가

각 failure에는 minimum sufficient evidence set을 정의한다.

예:
F1 DB pool exhaustion
- HTTP latency anomaly
- pool pending/active anomaly
- representative slow trace
- connection acquisition evidence

Agent가 root cause를 맞혀도 이 핵심 evidence를 전혀 조회하지 않았다면 'lucky diagnosis'로 별도 표시한다.

## 9. Replayability

BTS-AgentBench의 접근을 참고해 benchmark artifact를 가능한 한 deterministic하게 만든다.

고정 대상:
- failure seed/config
- workload
- telemetry fixture 또는 replayable backend snapshot
- runtime/source version
- gold evidence IDs
- scoring rule
- tool schema

Hosted model output은 deterministic boundary 밖에 두고 run configuration과 complete trajectory를 보존한다.

## 10. 반복 실행

Agent 결과는 변동성이 있으므로 scenario/condition별 단일 실행으로 일반화하지 않는다.

최소 원칙:
- 여러 반복 실행
- pass@k와 단일-run reliability를 함께 기록
- 실패 trajectory 유형을 별도 분류
- infrastructure/tool timeout을 model reasoning failure와 분리

정확한 repeat count는 실제 실행 비용과 초기 pilot variance를 보고 결정한다.

## 11. 예상 결과를 미리 정답으로 고정하지 않는다

가설:
- C/D가 A/B보다 diagnosis와 patch verification에서 유리할 가능성이 높다.
- D는 C보다 unsupported claim과 broad query를 줄일 가능성이 있다.
- 단순한 stacktrace bug에서는 B가 C/D와 비슷할 수 있다.
- observability가 잘못 수집된 scenario에서는 C/D도 실패할 수 있다.

이 가설이 틀려도 그대로 결과로 남긴다.

## 12. Publication Claim Boundary

이 실험이 성공하더라도 다음을 주장하지 않는다.

- 모든 애플리케이션에 동일한 tool set이 최적이다.
- Grafana MCP가 유일한 구현이다.
- Agent가 인간 SRE를 대체한다.
- 더 많은 telemetry가 항상 더 좋다.

주장 범위:
> 제한된 Spring Boot failure corpus에서, runtime evidence의 제공 방식과 workflow guardrail이 Coding Agent debugging 성능·비용·안전성에 어떤 영향을 주었는가.