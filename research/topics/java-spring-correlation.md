# Topic — Java/Spring Runtime Evidence Correlation

기준일: 2026-10-05

## 목표

Spring Boot 애플리케이션에서 Agent가 log 한 줄이 아니라 request execution을 재구성할 수 있는 최소 instrumentation을 정의한다.

## 1. Trace/Span correlation

Spring Boot는 Micrometer Tracing 사용 시 traceId와 spanId를 MDC에 넣고 correlation ID를 로그에 기본 포함한다.

OpenTelemetry LogRecord도 TraceId/SpanId를 correlation dimension으로 정의한다.

따라서 Java 실습의 최소 경로는 다음과 같다.

HTTP Request
  ↓
traceId/spanId
  ↓
Spring/Micrometer span
  ↓
MDC log
  ↓
OTel/Loki
  ↓
Tempo trace와 동일 execution 연결

## 2. Context Propagation Failure도 실습 대상이다

Spring Boot 공식 문서는 auto-configured RestTemplateBuilder, RestClient.Builder, WebClient.Builder를 사용해야 자동 trace propagation이 된다고 명시한다.

즉 다음은 좋은 failure case다.

개발자가 직접 HTTP client를 생성
→ trace propagation 누락
→ downstream log/trace가 다른 execution처럼 보임
→ Agent가 causal chain을 찾지 못함

이 사례는 'observability가 켜져 있다'와 'debuggable correlation이 완전하다'가 다르다는 것을 보여준다.

## 3. Structured Logging

현재 Spring Boot는 ECS, GELF, Logstash JSON structured logging을 기본 지원한다.

MDC key-value도 structured JSON에 포함할 수 있다.

Agent 관점에서 유용한 field 후보:
- service name
- service version
- environment
- traceId
- spanId
- requestId
- event/action
- exception type
- error code

Spring Boot의 structured JSON include/exclude/rename/add 기능은 Agent-friendly canonical schema를 만들 때 유용하다.

## 4. Stack Trace Cost

Spring Boot structured logging은 exception의 complete stack trace를 JSON에 포함할 수 있고, 공식 문서도 ingestion cost 때문에 조정할 수 있음을 설명한다.

Agentic Debugging에서는 stacktrace 전체를 항상 prompt에 전달하지 않는다.

권고:
- backend에는 필요한 수준으로 보존
- retrieval 시 root cause / top frames / application frames 중심 projection
- full stacktrace는 Agent가 추가 요청할 때 제공

## 5. Service Version이 중요하다

ECS structured logging에서 service.version을 설정할 수 있다.

책에서는 이를 더 확장해 telemetry Resource에 다음을 연결하는 방향을 조사한다.

- application version
- container image digest
- git commit SHA
- deployment ID

Runtime evidence와 현재 workspace source가 같은 version인지 Agent가 확인하지 않으면 이미 배포가 바뀐 incident를 잘못 수정할 수 있다.

## 6. 최소 실습 기준

Spring Boot application에 다음을 확보한다.

1. service.name
2. service.version
3. environment
4. traceId/spanId
5. structured JSON log
6. HTTP server/client trace propagation
7. DB span
8. exception event
9. Prometheus-compatible metric
10. deployment/commit identifier

이후 Grafana MCP 또는 Tempo MCP를 통해 Agent가 runtime evidence를 조회한다.