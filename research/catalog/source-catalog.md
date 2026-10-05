# Source Catalog

기준일: 2026-10-05

이 문서는 링크 목록이 아니라 책의 주장 후보와 일반화 한계를 함께 기록한다.

## A. Agent가 Observability를 직접 조회하는 오픈소스

### [S-GRAFANA-MCP] Open source Grafana MCP server

- Type: OFFICIAL / OSS
- Source: https://grafana.com/docs/grafana/latest/developer-resources/mcp/
- 확인점:
  - MCP-compatible AI assistant가 Grafana instance의 metrics, logs, traces, dashboards, alerting 등에 접근할 수 있다.
  - Prometheus와 Loki datasource query 도구, Tempo trace search/metrics/diff/attribute 탐색, Pyroscope datasource 등을 제공한다.
  - Claude Code, Codex CLI, Cursor 등 MCP client 설정 문서가 제공된다.
  - read-only mode와 Grafana RBAC/service-account 권한을 사용할 수 있다.
- Agentic Debugging 의미:
  - 이 책이 가정한 Observability → Agent Tool Layer가 이미 실제 오픈소스로 존재한다.
  - 별도 proprietary APM 없이도 Prometheus/Loki/Tempo/Grafana를 Coding Agent에 연결할 수 있다는 핵심 구현 사례다.
- 제약:
  - datasource 접근 권한과 telemetry 내 민감정보 문제는 그대로 남는다.
  - Grafana MCP가 RCA correctness를 보장하지 않는다.
- Principle 후보:
  - 사람용 dashboard와 Agent용 machine interface를 분리한다.
  - read-only observability 권한과 remediation write 권한을 분리한다.

### [S-TEMPO-AI] Grafana Tempo and AI

- Type: OFFICIAL / OSS
- Source: https://grafana.com/docs/tempo/latest/introduction/tempo-and-ai/
- 확인점:
  - Tempo 자체 /api/mcp endpoint가 Agent에게 TraceQL search, trace-by-id, span metrics, attribute discovery를 제공한다.
  - trace/tag API는 LLM용 simplified JSON 응답을 지원한다.
  - 공식 문서는 tracing data가 외부 LLM provider로 전달될 수 있음을 명시적으로 경고한다.
- Agentic Debugging 의미:
  - telemetry backend가 Agent-native interface를 직접 제공하는 실제 사례다.
  - token 절감을 위한 LLM-specific representation이 이미 제품 설계 문제로 다뤄지고 있다.
- Principle 후보:
  - Agent interface는 원본 telemetry schema를 그대로 노출할 필요가 없다.
  - LLM-optimized representation과 canonical telemetry storage를 분리한다.

## B. Telemetry Correlation Foundation

### [S-OTEL-LOGS] OpenTelemetry Logging Specification

- Type: OFFICIAL
- Source: https://opentelemetry.io/docs/specs/otel/logs/
- 확인점:
  - LogRecord에 TraceId와 SpanId를 포함해 log와 trace를 직접 상관시킬 수 있다.
  - Resource context를 logs, metrics, traces에 공통으로 사용할 수 있다.
- Principle 후보:
  - Runtime Evidence는 signal별 파일보다 execution context를 중심으로 연결한다.

### [S-OTEL-COLLECTOR] OpenTelemetry Collector

- Type: OFFICIAL / OSS
- Source: https://opentelemetry.io/docs/collector/
- 확인점:
  - receiver, processor, exporter, connector, extension으로 telemetry pipeline을 구성한다.
  - processor는 transform/filter/enrich를 담당한다.
- Agentic Debugging 의미:
  - Agent 앞단에서 redaction, filtering, enrichment, sampling을 수행할 수 있는 중립 수집 계층이다.

## C. Metrics

### [S-PROM-API] Prometheus HTTP API

- Type: OFFICIAL / OSS
- Source: https://prometheus.io/docs/prometheus/latest/querying/api/
- 확인점:
  - instant/range PromQL query가 JSON 결과를 반환한다.
  - series, labels, metadata discovery와 exemplar query API가 존재한다.
- Agentic Debugging 의미:
  - Grafana screenshot 대신 bounded metric query tool을 제공할 수 있다.

### [S-PROM-EXEMPLAR] Prometheus Exemplars

- Type: OFFICIAL
- Source: https://prometheus.io/docs/instrumenting/exposition_formats/
- 확인점:
  - exemplar는 집계 metric과 특정 observation을 연결하며 trace ID를 포함할 수 있다.
- Principle 후보:
  - aggregate anomaly에서 concrete execution evidence로 내려갈 연결 고리를 보존한다.

## D. Logs

### [S-LOKI-METADATA] Grafana Loki Structured Metadata

- Type: OFFICIAL / OSS
- Source: https://grafana.com/docs/loki/latest/get-started/labels/structured-metadata/
- 확인점:
  - trace_id, process_id, thread_id 같은 high-cardinality metadata를 indexed label이 아닌 structured metadata로 둘 수 있다.
  - OTLP ingestion과 결합된다.
- 제약:
  - trace ID를 stream label로 사용하면 cardinality가 커져 성능 문제가 생길 수 있다.
- Principle 후보:
  - service/namespace 같은 낮은 cardinality 값과 trace/request 같은 correlation key의 저장 방식을 분리한다.

### [S-LOKI-QUERY] Loki Query Best Practices

- Type: OFFICIAL / OSS
- Source: https://grafana.com/docs/loki/latest/query/bp-query/
- 확인점:
  - time range와 precise label selector를 이용해 검색 범위를 줄이는 것이 핵심이다.
- Agentic Debugging 의미:
  - Agent tool에서도 broad log search가 아니라 bounded range/query를 강제해야 한다.

## E. Traces

### [S-TEMPO-API] Tempo HTTP API

- Type: OFFICIAL / OSS
- Source: https://grafana.com/docs/tempo/latest/api_docs/
- 확인점:
  - trace ID 조회, TraceQL search, service/tag discovery, duration/time limit, result limit을 API로 제공한다.
  - trace diff endpoint도 존재하지만 일부 기능은 experimental이다.
- Agentic Debugging 의미:
  - search_traces, get_trace, compare_traces 같은 Tool surface를 실제 API 위에 얇게 구성할 수 있다.

### [S-TEMPO-METRICS] Tempo TraceQL Metrics

- Type: OFFICIAL / OSS
- Source: https://grafana.com/docs/tempo/latest/metrics-from-traces/
- 확인점:
  - trace에서 RED metric과 service graph를 생성한다.
  - stored span을 query-time에 aggregate할 수 있다.
- Principle 후보:
  - raw trace scan보다 aggregate narrowing 후 representative trace로 drill-down한다.

## F. Profiles

### [S-PYROSCOPE] Grafana Pyroscope

- Type: OFFICIAL / OSS
- Source: https://grafana.com/docs/pyroscope/latest/
- 확인점:
  - continuous profiling을 source code line 수준까지 분석하며 metrics/logs/traces와 correlation할 수 있다.
- Agentic Debugging 의미:
  - 느린 span을 CPU/heap hotspot과 source line까지 연결하는 evidence source가 된다.

## G. Java / Spring

### [S-SPRING-OBS] Spring Boot Observability

- Type: OFFICIAL
- Source: https://docs.spring.io/spring-boot/reference/actuator/observability.html
- 확인점:
  - metrics/traces는 Micrometer Observation을 중심으로 제공한다.
  - OpenTelemetry Java Agent/Starter 연계와 OTLP trace export 경로가 있다.
- 주의:
  - Spring Boot 자체의 OTel metrics/log export 지원 범위는 버전별로 다르므로 출간 직전 재검증한다.

## H. Benchmark / Research

### [S-OPENRCA] OpenRCA

- Type: BENCHMARK / OSS / PEER
- Source: https://microsoft.github.io/OpenRCA/
- Repository: https://github.com/microsoft/OpenRCA
- Venue: ICLR 2025
- 확인점:
  - 공개 페이지 기준 335 failures와 68GB 이상의 logs, metrics, traces를 제공한다.
  - RCA-agent baseline은 Python data retrieval/analysis를 사용해 telemetry 전체를 prompt에 넣지 않는다.
  - network fault는 KPI만으로 부족하고 parent/child span latency 관계가 중요할 수 있음을 공식 FAQ가 설명한다.
- Principle 후보:
  - LLM은 telemetry database가 아니다.
  - Evidence Retrieval과 Evidence Reasoning을 별도로 설계하고 평가한다.

### [S-RCA-REALWORLD-2026] How Far Can Root Cause Analysis Go on Real-World Telemetry Data?

- Type: PREPRINT
- Source: https://arxiv.org/abs/2607.13548
- 게시: 2026-07
- 확인점:
  - multimodal telemetry에서 evidence가 있었지만 활용하지 못한 Reasoning Gap과 evidence 자체가 부족한 Data Ambiguity를 구분한다.
- Principle 후보:
  - 더 많은 telemetry를 주는 것만으로 RCA가 해결되지 않는다.

### [S-LLM4LOG] LLM4Log

- Type: PREPRINT / SYSTEMATIC REVIEW
- Source: https://arxiv.org/abs/2604.16359
- 게시: 2026-03
- 확인점:
  - 145개 unique paper를 logging부터 RCA/summarization까지 분류한다.
  - context limit, cost, privacy, hallucination, drift를 deployment risk로 다룬다.
  - retrieval grounding, tool/agent augmentation, verification을 주요 패턴으로 정리한다.

### [S-SOK-LOG-PARSING] SoK: LLM-based Log Parsing

- Type: PREPRINT / SYSTEMATIC REVIEW
- Source: https://arxiv.org/abs/2504.04877
- 게시: 2025-04
- 확인점:
  - 29개 LLM log parsing 접근을 검토하고 7개 open-source parser를 비교한다.
- Agentic Debugging 의미:
  - raw log를 structured representation으로 바꾸는 전처리 계층의 근거다.

### [S-NEXT] NExT: Teaching Large Language Models to Reason about Code Execution

- Type: PEER
- Source: https://proceedings.mlr.press/v235/ni24a.html
- Venue: ICML 2024
- 확인점:
  - line execution과 variable state trace를 이용한 runtime-aware reasoning을 program repair에서 평가했다.
- 일반화 한계:
  - distributed production observability가 아니라 program execution trace 중심이다.

### [S-MICRORCA] MicroRCA

- Type: PEER / OSS
- Source: https://doi.org/10.1109/NOMS47738.2020.9110353
- Repository: https://github.com/elastisys/MicroRCA
- Venue: NOMS 2020
- 확인점:
  - response-time symptom과 resource utilization을 dependency graph로 연결해 root cause를 찾는다.
- 역할:
  - LLM 이전부터 RCA의 핵심이 multi-signal correlation과 topology였다는 기준점이다.

### [S-MICRORCA-AGENT] MicroRCA-Agent

- Type: PREPRINT / OSS
- Source: https://arxiv.org/abs/2509.15635
- 게시: 2025-09
- 확인점:
  - Drain log parsing, filtering, trace anomaly detection, 통계 처리 뒤에 LLM 분석을 둔다.
- Principle 후보:
  - deterministic/statistical narrowing과 LLM reasoning을 조합한다.


## I. Lightweight Log Intelligence / Evidence Reduction

### [S-DRAIN3] Drain3

- Type: OSS
- Source: https://github.com/logpai/Drain3
- 확인점:
  - streaming log template mining을 수행한다.
  - variable token을 wildcard/template parameter로 축약한다.
  - online learning과 inference-only matching을 지원한다.
  - masking, persistence, parameter extraction 기능이 있다.
- Agentic Debugging 의미:
  - raw log를 그대로 LLM에 주기 전에 반복 패턴과 변수 값을 분리하는 evidence reduction 계층으로 사용할 수 있다.
  - template ID sequence를 경량 anomaly detector 입력으로 사용할 수 있다.
- 제약:
  - parsing/template mining 자체는 anomaly detection이나 RCA가 아니다.
  - trace_id/span_id/service.version 같은 correlation key를 제거하지 않도록 별도 structured field로 보존해야 한다.
- Principle 후보:
  - Raw Log ≠ Agent Context
  - Compress Before Reasoning

### [S-DEEPLOG] DeepLog

- Type: PEER
- Source: https://doi.org/10.1145/3133956.3134015
- Venue: CCS 2017
- 확인점:
  - LSTM을 사용해 정상 log sequence를 모델링하고 예상 패턴에서 벗어난 sequence를 anomaly로 탐지한다.
- Agentic Debugging 의미:
  - 생성형 LLM보다 훨씬 작은 sequence model을 상시 1차 탐지기로 사용할 수 있는 대표 baseline이다.
- 제약:
  - anomaly는 root cause가 아니며 최신 Transformer 계열과 직접 성능 비교 시 dataset/전처리 조건을 맞춰야 한다.

### [S-LOGBERT] LogBERT

- Type: PREPRINT / OSS
- Source: https://arxiv.org/abs/2103.04475
- Repository: https://github.com/HelenGuohx/logbert
- 확인점:
  - self-supervised BERT framework로 정상 log sequence pattern을 학습한다.
  - 공개 구현은 HDFS, BGL, Thunderbird dataset을 포함한다.
- Agentic Debugging 의미:
  - raw log 전체를 LLM에 전달하기 전 sequence anomaly window를 좁히는 경량 Transformer baseline으로 적합하다.
- 제약:
  - production OpenTelemetry/Grafana pipeline과 직접 통합된 시스템은 아니다.
  - anomaly 결과는 retrieval hint이며 RCA 결과가 아니다.

### [S-LOGGPT] LogGPT

- Type: PREPRINT / OSS
- Source: https://arxiv.org/abs/2309.14482
- Repository: https://github.com/nokia-steward/LogGPT
- 확인점:
  - 이전 log sequence를 기반으로 next log entry를 예측하고 RL fine-tuning으로 anomaly detection을 보강한다.
- Agentic Debugging 의미:
  - generative sequence modeling을 이용한 log anomaly detection 비교군으로 사용할 수 있다.
- 제약:
  - 상시 lightweight filter 목적에는 DeepLog/LogBERT/small Transformer보다 계산 비용이 클 수 있다.

### [S-LOGLLM] LogLLM

- Type: OSS / RESEARCH
- Source: https://github.com/guanwei49/LogLLM
- 확인점:
  - BERT semantic encoder와 LLM을 결합한 log anomaly detection 구조를 제공한다.
  - HDFS, BGL, Liberty, Thunderbird dataset 실험 구성을 공개한다.
- Agentic Debugging 의미:
  - 모든 로그를 비교적 큰 모델에 직접 처리시키는 접근의 비교 기준으로 사용할 수 있다.
- 제약:
  - 1차 상시 필터로는 작은 sequence model보다 자원 비용이 커질 수 있다.

### [S-LOGRAIL] LogRAIL

- Type: PEER / OSS
- Source: https://github.com/Choiwongwang/LogRAIL
- Venue: IEEE Access 2026
- DOI: 10.1109/ACCESS.2026.3688834
- 확인점:
  - Stage 1 LogFormer로 전체 window를 scoring한 뒤 near-threshold case만 Vector DB + LLM으로 재검증한다.
  - 공개 AOSP Android log 결과에서 Stage 1 F1 0.9086, precision-oriented LogRAIL F1 0.9273, recall-oriented F1 0.9248을 보고한다.
  - Stage 2 LLM은 Llama 3 8B Instruct를 사용한다.
- Agentic Debugging 의미:
  - cheap detection → selective expensive reasoning 구조의 직접적인 구현 근거다.
  - 모든 로그를 LLM에 넣지 않고 애매한 사례만 LLM에 전달하는 설계가 실험적으로 평가되어 있다.
- 일반화 한계:
  - 특정 AOSP dataset과 설정의 결과이며 일반 production 환경 성능으로 일반화하면 안 된다.
- Principle 후보:
  - Cheap Detection, Expensive Reasoning
  - Anomaly Is a Retrieval Hint, Not a Root Cause

## 1차 Synthesis

현재 가장 강한 공통 패턴은 다음과 같다.

1. signal 양보다 correlation key 보존이 중요하다.
2. raw telemetry 전체를 LLM context에 넣지 않는다.
3. Agent는 query/aggregation/filtering 도구로 evidence를 단계적으로 좁힌다.
4. metric → trace → log/profile → source로 drill-down할 연결이 필요하다.
5. evidence availability와 reasoning correctness는 별개다.
6. production observability access는 기본 read-only로 시작한다.
7. 2026년 현재 Grafana와 Tempo는 Agent용 MCP 인터페이스를 공식적으로 제공하므로 이 책은 가상의 구조가 아니라 실제 OSS 기반으로 실습할 수 있다.

## J. Telemetry-to-Agent Benchmark Construction

### [S-BTS-AGENTBENCH] BTS-AgentBench

- Type: PREPRINT / OSS / BENCHMARK
- Source: https://arxiv.org/abs/2608.27334
- Repository: https://github.com/kjy7567/BTS-AgentBench
- 게시: 2026-08
- 확인점:
  - read-only telemetry를 normalized tool store로 변환하고 typed, bounded multi-turn agent episode를 구성한다.
  - telemetry value, tool output, evidence ID, verifier를 deterministic code path로 생성한다.
  - 532개 BTS episode와 별도 XAI4HEAT portability corpus를 공개한다.
  - construction/scoring의 deterministic boundary와 hosted model generation의 nondeterministic boundary를 분리한다.
- Agentic Debugging 의미:
  - Agent Debug Session Contract를 benchmark episode로 고정할 수 있다는 직접적 설계 근거다.
  - gold answer만 아니라 tool-derived gold evidence와 provenance를 함께 보존하는 방식이 debugging experiment에 적합하다.
- 일반화 한계:
  - building telemetry를 대상으로 하며 software observability/log/trace benchmark는 아니다.
- Principle 후보:
  - Read-only Telemetry → Typed Bounded Episode
  - Gold Answer ≠ Gold Evidence
  - Construction Determinism ≠ Model Determinism

### [S-AGENTDEBUGX] AgentDebugX

- Type: PREPRINT / OSS
- Source: https://arxiv.org/abs/2607.18754
- 게시: 2026-07
- 확인점:
  - Agent 자체 실패를 Detect → Attribute → Recover → Rerun의 closed loop로 다룬다.
  - trajectory-level attribution과 recovery를 분리한다.
- Agentic Debugging 의미:
  - 대상은 application runtime이 아니라 Agent trajectory이므로 이 책의 중심 자료는 아니다.
  - 다만 root-cause diagnosis가 repair/rerun까지 이어져야 한다는 closed-loop 구조의 보조 근거로 사용한다.
- 일반화 한계:
  - application logs/metrics/traces를 통한 production debugging 연구로 해석하지 않는다.
