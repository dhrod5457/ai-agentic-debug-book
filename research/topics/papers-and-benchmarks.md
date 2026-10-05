# Topic — Papers and Benchmarks

기준일: 2026-10-05

Application Runtime Evidence를 Agent Debugging에 연결하는 자료만 우선 정리한다.

## 1. OpenRCA

- ICLR 2025
- Project: https://microsoft.github.io/OpenRCA/
- Repository: https://github.com/microsoft/OpenRCA

OpenRCA는 software failure RCA를 위해 KPI time series, dependency trace graph, semi-structured logs를 함께 다룬다. 공개 페이지 기준 335 failures와 68GB 이상의 telemetry를 제공한다.

가장 중요한 점은 RCA-agent baseline이 Python을 사용해 telemetry를 검색/분석함으로써 전체 데이터를 LLM context에 넣지 않는다는 것이다.

책에서 사용할 질문:
- 어떤 retrieval tool이 필요한가
- 어떤 query가 evidence를 가장 잘 좁히는가
- context reduction이 root-cause accuracy에 어떤 영향을 주는가
- evidence retrieval 실패와 reasoning 실패를 어떻게 구분할 것인가

## 2. How Far Can Root Cause Analysis Go on Real-World Telemetry Data?

- arXiv:2607.13548
- 2026-07
- Preprint

이 연구는 multimodal telemetry에서 RCA failure를 Reasoning Gap과 Data Ambiguity로 분리한다.

책에서 사용할 핵심 경계:
Evidence Availability ≠ Reasoning Quality

더 많은 telemetry를 Agent에 전달하는 것만으로 정확도가 자동으로 올라가지 않는다는 근거로 사용한다.

## 3. LLM4Log

- arXiv:2604.16359
- 2026-03
- Systematic Review / Preprint
- literature cutoff: 2025-11
- 145 unique papers

logging, parsing, representation, anomaly detection, failure prediction, root cause analysis, summarization을 하나의 pipeline으로 정리한다.

deployment 문제로 context limit, latency/cost, privacy, hallucination, drift를 다루며 retrieval grounding, tool/agent augmentation, verification을 주요 패턴으로 정리한다.

이 자료는 umbrella source로 사용하고 세부 주장은 가능한 한 원 논문을 다시 확인한다.

## 4. SoK: LLM-based Log Parsing

- arXiv:2504.04877
- 2025-04
- Preprint

29개 LLM 기반 log parsing 접근을 검토하고 7개 open-source parser를 public dataset에서 비교한다.

Agent에게 raw log text를 그대로 주는 방식과 structured event pipeline을 비교하는 근거로 사용한다.

## 5. NExT

- ICML 2024
- https://proceedings.mlr.press/v235/ni24a.html

line execution과 variable state trace를 이용해 LLM이 runtime behavior를 reasoning하도록 학습하고 program repair에서 평가했다.

이 연구는 source text만 보는 것보다 execution evidence가 debugging에 유용하다는 직접 근거다.

일반화 한계: distributed production observability 연구는 아니다.

## 6. MicroRCA

- NOMS 2020
- DOI: 10.1109/NOMS47738.2020.9110353
- OSS: https://github.com/elastisys/MicroRCA

application response-time symptom과 system resource utilization을 graph로 연결해 root cause를 찾는다.

LLM 이전부터 RCA의 핵심이 multi-signal correlation과 dependency relation이었다는 기술적 기준점으로 사용한다.

## 7. MicroRCA-Agent

- arXiv:2509.15635
- 2025-09
- Preprint / OSS

Drain 기반 log parsing, filtering, trace anomaly detection, statistical processing 뒤에 LLM multimodal analysis를 배치한다.

raw telemetry → LLM 직결이 아니라 preprocessing → narrowing → reasoning 구조라는 점을 참고한다.

## 8. 후속 연구 공백

### Gap A — Observability와 Coding Agent 사이 Debug Contract
Grafana/Tempo MCP가 등장했지만 Application Debugging workflow 자체의 표준 evidence contract는 아직 별도 연구 가치가 있다.

### Gap B — Evidence Budget
context token, query latency, backend scan cost를 함께 제한하는 기준이 필요하다.

### Gap C — Deploy/Source Version Correlation
runtime evidence가 어떤 commit, image, configuration에서 나온 것인지 Agent가 확인할 수 있어야 한다.

### Gap D — Verification Loop
RCA에서 끝나지 않고 patch 후 동일 workload와 telemetry를 비교해 incident resolution을 검증하는 loop가 필요하다.