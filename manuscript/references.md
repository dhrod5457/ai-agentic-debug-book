# References

기준일: 2026-10-05

본문의 `[S-...]` 표시는 이 목록의 Source ID를 가리킨다.

제품 문서와 사양은 출간 시점에 변경될 수 있으므로 최종 출간 전 다시 확인한다. Preprint는 peer-reviewed 연구와 구분해 표기한다.

## 공식 문서와 오픈소스

### [S-OTEL-LOGS] OpenTelemetry — Logs

- 유형: 공식 사양
- URL: https://opentelemetry.io/docs/specs/otel/logs/
- 사용 위치: 로그와 trace의 TraceId/SpanId 연결, 공통 Resource context

### [S-OTEL-COLLECTOR] OpenTelemetry — Collector

- 유형: 공식 문서 / 오픈소스
- URL: https://opentelemetry.io/docs/collector/
- 사용 위치: 수집 파이프라인, receiver/processor/exporter 구조

### [S-OTEL-TRANSFORM] OpenTelemetry — Transforming telemetry

- 유형: 공식 문서
- URL: https://opentelemetry.io/docs/collector/transforming-telemetry/
- 사용 위치: filter, transform, redaction, 수집 단계의 데이터 거버넌스
- 주의: 복잡한 transformation은 Collector 성능에 영향을 줄 수 있음

### [S-OTEL-SQL] OpenTelemetry — Semantic conventions for SQL databases client operations

- 유형: 공식 사양
- URL: https://opentelemetry.io/docs/specs/semconv/db/sql/
- 확인일: 2026-10-05
- 사용 위치: db.query.summary, db.query.text, SQL parameter 수집과 민감정보 처리
- 현재 안정성: db.query.summary와 db.query.text는 Stable / Recommended
- 현재 안정성: db.query.parameter.<key>는 Development / Opt-In

### [S-OTEL-SERVICE] OpenTelemetry — Service semantic conventions

- 유형: 공식 사양
- URL: https://opentelemetry.io/docs/specs/semconv/resource/service/
- 확인일: 2026-10-05
- 사용 위치: service.name, service.version, service.instance.id
- 현재 안정성: service.name은 Stable / Required, service.version은 Stable / Recommended

### [S-OTEL-K8S] OpenTelemetry — Specify resource attributes using Kubernetes annotations

- 유형: 공식 가이드
- URL: https://opentelemetry.io/docs/specs/semconv/non-normative/k8s-attributes/
- 사용 위치: Kubernetes 환경에서 service.version 계산, image tag/digest와 서비스 버전 연결

### [S-PROM-API] Prometheus — HTTP API

- 유형: 공식 문서 / 오픈소스
- URL: https://prometheus.io/docs/prometheus/latest/querying/api/
- 사용 위치: instant/range query, metric discovery, machine-readable 결과

### [S-PROM-EXEMPLAR] Prometheus — Exposition formats / Exemplars

- 유형: 공식 문서
- URL: https://prometheus.io/docs/instrumenting/exposition_formats/
- 사용 위치: 집계된 metric에서 실제 trace로 이동

### [S-LOKI-METADATA] Grafana Loki — Structured metadata

- 유형: 공식 문서 / 오픈소스
- URL: https://grafana.com/docs/loki/latest/get-started/labels/structured-metadata/
- 사용 위치: trace_id 같은 high-cardinality 값을 stream label과 분리

### [S-LOKI-QUERY] Grafana Loki — Query best practices

- 유형: 공식 문서 / 오픈소스
- URL: https://grafana.com/docs/loki/latest/query/bp-query/
- 사용 위치: 시간 범위와 label selector를 이용한 로그 검색 범위 축소

### [S-TEMPO-API] Grafana Tempo — HTTP API

- 유형: 공식 문서 / 오픈소스
- URL: https://grafana.com/docs/tempo/latest/api_docs/
- 사용 위치: trace ID 조회, TraceQL search, result/time limit

### [S-TEMPO-METRICS] Grafana Tempo — Metrics from traces

- 유형: 공식 문서 / 오픈소스
- URL: https://grafana.com/docs/tempo/latest/metrics-from-traces/
- 사용 위치: trace에서 RED metric과 service graph 생성, trace 집계

### [S-TEMPO-AI] Grafana Tempo — Tempo and AI

- 유형: 공식 문서 / 오픈소스
- URL: https://grafana.com/docs/tempo/latest/introduction/tempo-and-ai/
- MCP server: https://grafana.com/docs/tempo/latest/api_docs/mcp-server/
- 확인일: 2026-10-05
- 사용 위치: Tempo MCP, Agent용 trace 조회, LLM용 간소화 응답
- 현재 주의: MCP server는 설정에서 별도로 활성화해야 한다.
- 현재 주의: application/vnd.grafana.llm 응답 형식은 변경 가능성이 있어 안정적 프로그램 계약으로 의존하지 않는다.

### [S-PYROSCOPE] Grafana Pyroscope

- 유형: 공식 문서 / 오픈소스
- URL: https://grafana.com/docs/pyroscope/latest/
- 사용 위치: continuous profiling, trace와 profile 연결

### [S-GRAFANA-MCP] Grafana — MCP server

- 유형: 공식 문서 / 오픈소스
- URL: https://grafana.com/docs/grafana/latest/developer-resources/mcp/
- 설정: https://grafana.com/docs/grafana/latest/developer-resources/mcp/configure/
- CLI flags: https://grafana.com/docs/grafana/latest/developer-resources/mcp/configure/command-line-flags/
- Tool/RBAC reference: https://grafana.com/docs/grafana/latest/developer-resources/mcp/reference/mcp-tools-table/
- 저장소: https://github.com/grafana/mcp-grafana
- 확인일: 2026-10-05
- 사용 위치: Prometheus/Loki/Tempo/Pyroscope를 Agent가 직접 조회, read-only와 query guardrail
- 현재 주의: Loki guardrail mode의 기본값은 off이며 운영 보호 장치로 사용하려면 enforce를 명시해야 한다.
- 현재 주의: --disable-write는 raw SQL/Influx query 도구도 제거한다.

### [S-SPRING-OBS] Spring Boot — Observability

- 유형: 공식 문서
- URL: https://docs.spring.io/spring-boot/reference/actuator/observability.html
- Tracing: https://docs.spring.io/spring-boot/reference/actuator/tracing.html
- 확인일: 2026-10-05
- 사용 위치: Micrometer Observation, tracing, Spring Boot 관측 구성
- 현재 주의: 자동 network trace propagation에는 auto-configured RestTemplateBuilder, RestClient.Builder, WebClient.Builder 사용이 필요하다.
- 주의: Spring Boot 버전별 지원 범위를 출간 전 재확인

### [S-ORACLE-JCMD] Oracle JDK 25 — The jcmd Command

- 유형: 공식 문서
- URL: https://docs.oracle.com/en/java/javase/25/docs/specs/man/jcmd.html
- 확인일: 2026-10-05
- 사용 위치: JFR.start, JFR.check, JFR.dump, JVM 진단 명령과 영향도
- 현재 상태: JFR.start/check/dump는 Low impact로 문서화되어 있고 heap dump는 High impact로 문서화되어 있다.
- 주의: JFR.dump의 GC root 경로 수집은 애플리케이션 pause를 유발할 수 있음

### [S-K8S-EVENT] Kubernetes — Event API

- 유형: 공식 문서
- URL: https://kubernetes.io/docs/reference/kubernetes-api/core/event-v1/
- 사용 위치: Kubernetes Event의 제한된 retention과 best-effort 성격

### [S-DRAIN3] Drain3

- 유형: 오픈소스
- URL: https://github.com/logpai/Drain3
- 사용 위치: 반복 로그의 template mining과 패턴 축약

## 연구와 벤치마크

### [S-OPENRCA] OpenRCA

- 유형: peer-reviewed benchmark / 오픈소스
- Venue: ICLR 2025
- 프로젝트: https://microsoft.github.io/OpenRCA/
- 저장소: https://github.com/microsoft/OpenRCA
- 사용 위치: logs/metrics/traces를 함께 사용하는 RCA, Python 기반 telemetry retrieval, model context와 telemetry working set 분리

### [S-NEXT] NExT: Teaching Large Language Models to Reason about Code Execution

- 유형: peer-reviewed paper
- Venue: ICML 2024
- URL: https://proceedings.mlr.press/v235/ni24a.html
- 사용 위치: execution trace를 이용한 runtime-aware reasoning과 program repair
- 한계: production distributed observability를 직접 다루는 연구는 아님

### [S-LLM4LOG] LLM4Log

- 유형: systematic review / preprint
- 게시: 2026-03
- URL: https://arxiv.org/abs/2604.16359
- 사용 위치: LLM 기반 logging, parsing, anomaly detection, RCA 연구 지형
- 주의: preprint이며 세부 주장에는 가능한 한 원 논문을 우선함

### [S-RCA-REALWORLD-2026] How Far Can Root Cause Analysis Go on Real-World Telemetry Data?

- 유형: preprint
- 게시: 2026-07
- URL: https://arxiv.org/abs/2607.13548
- 사용 위치: Reasoning Gap과 Data Ambiguity 구분
- 주의: preprint

### [S-BTS-AGENTBENCH] BTS-AgentBench

- 유형: benchmark / 오픈소스 / preprint
- 게시: 2026-08
- URL: https://arxiv.org/abs/2608.27334
- 저장소: https://github.com/kjy7567/BTS-AgentBench
- 사용 위치: read-only telemetry를 typed/bounded agent episode로 구성, gold evidence와 replay
- 한계: building telemetry 대상이며 software observability benchmark는 아님

## 본문에서 직접 사용하지 않는 보조 연구

다음 자료는 research 디렉터리에 보존하지만 현재 본문의 핵심 References에는 포함하지 않는다.

- MicroRCA
- MicroRCA-Agent
- SoK: LLM-based Log Parsing
- DeepLog
- LogBERT
- LogGPT
- LogLLM
- LogRAIL
- AgentDebugX

필요한 장을 확장할 때 원 논문을 다시 확인한 뒤 References에 승격한다.
