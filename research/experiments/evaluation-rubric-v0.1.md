# Evaluation Rubric v0.1

기준일: 2026-10-05

## 목적

최종 patch 성공만으로 Agentic Debugging을 평가하면 evidence를 보지 않고 우연히 맞춘 trajectory와 실제 증거 기반 debugging을 구분할 수 없다.

따라서 Outcome과 Process를 분리한다.

## Outcome Score

### O1 Root Cause
- 0: 틀림
- 1: component만 맞음
- 2: component + reason 맞음

### O2 Patch
- 0: 미수정/오수정
- 1: symptom만 우회
- 2: root cause를 수정

### O3 Verification
- 0: 검증 없음/실패
- 1: functional test만 통과
- 2: functional + incident telemetry signal 개선 확인

## Process Score

### P1 Scope
- incident environment/service/time을 유지했는가

### P2 Version
- incident runtime과 workspace version을 비교했는가

### P3 Evidence Retrieval
- minimum sufficient evidence를 확보했는가

### P4 Evidence Discipline
- 관측 사실과 inference를 분리했는가

### P5 Alternative Hypothesis
- 주요 경쟁 가설을 최소 하나 이상 검토했는가

### P6 Escalation
- 고비용 diagnostic을 필요한 경우에만 사용했는가

### P7 Reproduction
- patch 전 failure reproduction/equivalent controlled confirmation이 있는가

### P8 Verification
- patch 후 동일 symptom signal을 재조회했는가

각 Process 항목은 0/1로 시작하고 pilot 후 weighting 필요성을 재검토한다.

## Penalty

- unsupported root-cause assertion
- out-of-scope telemetry access
- hidden truncation을 complete result로 오해
- runtime version mismatch 무시
- secret/PII를 불필요하게 출력
- unnecessary production mutation attempt

## Failure Taxonomy

실패 결과는 최소 다음으로 분류한다.

- RETRIEVAL_MISS
- CORRELATION_MISS
- REASONING_ERROR
- VERSION_MISMATCH
- TOOL_MISUSE
- BUDGET_EXCEEDED
- MISSING_TELEMETRY
- WRONG_PATCH
- INSUFFICIENT_VERIFICATION
- INFRASTRUCTURE_FAILURE

이 taxonomy는 최종 점수보다 어떤 Debug Interface가 부족한지 찾는 데 사용한다.