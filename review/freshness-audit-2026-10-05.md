# Freshness Audit — 2026-10-05

출간 직전 변경 가능성이 높은 기술 항목만 공식 문서로 재확인한 기록이다.

## 1. Grafana MCP

상태: 확인 완료

공식 문서:
- https://grafana.com/docs/grafana/latest/developer-resources/mcp/configure/
- https://grafana.com/docs/grafana/latest/developer-resources/mcp/configure/command-line-flags/
- https://grafana.com/docs/grafana/latest/developer-resources/mcp/reference/mcp-tools-table/

확인 내용:
- `--disable-write` read-only mode가 현재도 존재한다.
- `--disable-query`로 datasource query 실행 도구를 제거할 수 있다.
- `--disable-write`는 raw SQL/Influx query 도구도 기본적으로 제거한다.
- RBAC permission과 resource scope를 tool별로 적용할 수 있다.
- Loki 비용 guardrail은 현재 `off | shadow | enforce` 모드를 제공한다.
- Loki guardrail의 현재 기본 모드는 `off`다.
- 운영에서 guardrail을 안전장치로 기대한다면 `enforce`를 명시해야 한다.
- scan byte budget과 effective time range 제한이 존재한다.

본문 반영:
- 9장과 19장에 guardrail 기본값이 off라는 주의를 추가했다.

## 2. Grafana Tempo MCP

상태: 확인 완료

공식 문서:
- https://grafana.com/docs/tempo/latest/introduction/tempo-and-ai/
- https://grafana.com/docs/tempo/latest/api_docs/mcp-server/

확인 내용:
- Tempo는 `/api/mcp` MCP server를 제공한다.
- TraceQL trace search, trace by ID, span-derived metrics, attribute discovery를 제공한다.
- MCP server는 설정에서 별도로 활성화해야 한다.
- Tempo API와 동일한 authentication/multi-tenancy 동작을 사용한다.
- trace/tag API는 `Accept: application/vnd.grafana.llm` 간소화 응답을 지원한다.
- 이 LLM 전용 응답 형식은 변경 가능성이 있으므로 안정적 programmatic contract로 의존하면 안 된다.
- 공식 문서는 tracing data가 외부 LLM provider로 전달될 수 있음을 경고한다.

본문 반영:
- 6장에 MCP 활성화 필요와 LLM 응답 형식 안정성 주의를 추가했다.

## 3. OpenTelemetry Service Semantic Conventions

상태: 확인 완료

공식 문서:
- https://opentelemetry.io/docs/specs/semconv/resource/service/

확인 내용:
- `service.name`: Stable / Required
- `service.version`: Stable / Recommended
- `service.version` 값의 형식은 고정되지 않으며 semantic version이나 Git hash 등을 사용할 수 있다.

본문 판단:
- 실행 버전과 source version을 연결하기 위해 service.version을 사용하는 설명 유지 가능.

## 4. OpenTelemetry SQL Semantic Conventions

상태: 확인 완료

공식 문서:
- https://opentelemetry.io/docs/specs/semconv/db/sql/

확인 내용:
- `db.query.summary`: Stable / Recommended
- `db.query.text`: Stable / Recommended
- non-parameterized query text는 literal sanitization 없이는 기본 수집하지 않을 것을 권고한다.
- `db.query.parameter.<key>`: Development / Opt-In
- parameter 값은 PII/민감정보가 있을 수 있어 기본 수집하지 않을 것을 권고한다.

본문 판단:
- query summary 우선, parameter default deny 설명 유지 가능.

## 5. Spring Boot Observability / Tracing

상태: 확인 완료

공식 문서:
- https://docs.spring.io/spring-boot/reference/actuator/observability.html
- https://docs.spring.io/spring-boot/reference/actuator/tracing.html

확인 내용:
- metrics/traces는 Micrometer Observation을 중심으로 제공된다.
- Micrometer Tracing auto-configuration을 제공한다.
- network trace propagation을 자동 적용하려면 auto-configured `RestTemplateBuilder`, `RestClient.Builder`, `WebClient.Builder`를 사용해야 한다.
- 직접 client를 만들면 자동 trace propagation이 동작하지 않을 수 있다는 경고가 현재 문서에도 유지된다.

본문 판단:
- 4장/13장의 context propagation failure 사례 유지 가능.

## 6. Oracle JDK 25 jcmd / JFR

상태: 확인 완료

공식 문서:
- https://docs.oracle.com/en/java/javase/25/docs/specs/man/jcmd.html

확인 내용:
- `JFR.check`, `JFR.configure`, `JFR.dump`, `JFR.start`의 impact는 Low로 문서화돼 있다.
- `JFR.dump begin/end`로 recording window를 제한할 수 있다.
- `default.jfc`는 continuous recording에 적합한 low-overhead profile이다.
- `profile.jfc`는 더 많은 정보를 수집하지만 overhead도 더 높다.
- heap dump는 High impact로 문서화돼 있다.
- GC root path 수집은 pause를 일으킬 수 있어 memory leak 의심 시에만 켤 것을 권고한다.

본문 판단:
- JVM diagnostic escalation 설명 유지 가능.

## 7. Kubernetes Deployment Revision / Event

상태: 확인 완료

공식 문서:
- https://kubernetes.io/docs/reference/labels-annotations-taints/
- https://kubernetes.io/docs/reference/kubernetes-api/core/event-v1/

확인 내용:
- `deployment.kubernetes.io/revision`은 ReplicaSet에 Deployment controller가 설정하고 Pod template 변경 시 증가한다.
- rollout history와 rollback에 사용된다.
- Event는 원인 판정의 유일한 근거가 아니라 object status/metrics/logs와 함께 보조 자료로 사용하는 현재 본문 방향을 유지한다.

## 8. 출간 시 재확인이 필요한 항목

출간 직전에 다시 볼 항목:
1. Grafana MCP command-line flag 이름과 default
2. Tempo MCP 활성화 설정 및 LLM response media type
3. Spring Boot 최신 stable branch의 tracing/logging 지원 범위
4. OpenTelemetry semantic convention stability badge
5. Kubernetes current stable minor version과 Event/annotation 문서 URL

현재 결론:
2026-10-05 기준 책의 핵심 기술 주장은 유지 가능하며, Grafana Loki guardrail 기본값과 Tempo MCP 활성화/LLM response 안정성만 본문에 보정했다.
