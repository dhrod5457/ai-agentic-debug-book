# Deep Dive — OpenRCA RCA-agent Implementation

기준일: 2026-10-05

## 결론

OpenRCA의 RCA-agent는 'telemetry를 LLM prompt에 넣는 시스템'이 아니라 Controller가 Executor에게 작은 분석 instruction을 반복해서 주고, Executor가 Python/IPython으로 telemetry를 직접 처리한 결과만 Controller에게 돌려주는 구조다.

## 1. Controller / Executor 분리

Controller:
- 직전 실행 결과를 분석
- 다음 investigation instruction을 하나의 atomic step으로 생성
- 최대 step 내 반복
- 충분하면 root cause 후보를 최종 선택

Executor:
- instruction을 Python code로 변환
- IPython kernel에서 실행
- pandas 중심으로 telemetry 처리
- 실행 오류가 나면 code를 수정해 재시도
- 결과를 요약해 Controller로 반환

구조:
Controller reasoning
  ↓ atomic instruction
Executor
  ↓ Python
Telemetry files
  ↓ result
Controller reasoning

## 2. Stateful Analysis Workspace

Executor는 IPython kernel을 유지하고 이전 step의 변수를 재사용하도록 명시한다.

이 패턴은 중요한 의미가 있다.

Agent Debugging에서 매 query마다 대량 telemetry를 context로 다시 넣는 대신:
- telemetry working set은 execution environment에 유지
- model에는 필요한 projection/result만 전달
할 수 있다.

즉:
Model Context ≠ Telemetry Working Set

## 3. Context Budget을 코드 레벨에서 다룬다

Executor는 실행 결과 token 길이를 계산하고 너무 큰 결과를 그대로 반환하지 않는다.

또 pandas 결과가 많은 row를 가진 경우 출력이 일부만 보였음을 명시해 observation bias 가능성을 경고한다.

이 구현에서 얻을 원칙:
- truncated result를 숨기지 않는다.
- Agent에게 '없음'과 '출력되지 않음'을 구분할 metadata를 준다.
- result-size budget은 tool contract에 포함한다.

## 4. Atomic Investigation

Controller prompt는 complex multi-step instruction 대신 한 번에 하나의 atomic request를 Executor에 주도록 요구한다.

이 방식은 debugging trajectory를 audit하고 실패 지점을 분석하기 좋다.

책의 Evidence Retrieval Loop도 다음처럼 유지할 근거가 된다.

scope
→ metric
→ trace
→ logs
→ hypothesis test

## 5. OpenRCA를 그대로 Production에 쓰면 안 되는 이유

RCA-agent Executor는 일반 Python code를 생성/실행한다. benchmark 환경에서는 유용하지만 production observability access gateway로는 지나치게 넓다.

Production에서는:
- arbitrary filesystem access 금지
- bounded PromQL/LogQL/TraceQL tool
- query budget
- read-only credential
- tenant/environment scope
- audit trail
이 필요하다.

따라서 OpenRCA는 reasoning architecture의 근거이지 production security architecture의 완성형은 아니다.

## 6. 책의 실험에 적용

비교군 후보:
A. raw log dump
B. Grafana MCP direct access
C. Debug Workflow Layer + Grafana MCP
D. OpenRCA-style generic Python analysis

측정:
- root cause 정확도
- total telemetry bytes scanned
- context token
- query/tool call 수
- time to root cause
- unsafe/broad query 비율
- evidence provenance completeness

이 비교를 통해 generic code executor와 domain-specific observability tools의 trade-off를 실증할 수 있다.