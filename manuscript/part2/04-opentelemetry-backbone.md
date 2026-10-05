# 4장. OpenTelemetry를 상관관계의 뼈대로 사용한다

Agent에게 logs, metrics, traces를 모두 연결한다고 말하면 가장 먼저 기술 스택을 떠올리기 쉽다. Prometheus를 쓸 것인가, Loki를 쓸 것인가, Tempo를 쓸 것인가.

하지만 그보다 먼저 해결해야 할 문제가 있다.

> 서로 다른 signal이 같은 실행에서 나온 것임을 어떻게 알 것인가?

이 문제를 해결하지 않으면 backend를 몇 개 붙여도 Agent는 여전히 조각난 정보를 본다.

## 1. 먼저 correlation을 설계한다

하나의 로그인 요청을 생각해보자.

Metric에는 request duration이 기록되고, trace에는 trace_id와 span_id가 남고, log에는 authentication failed가 기록되며, DB span에는 SELECT user가 남을 수 있다.

이 각각이 따로 저장되면 나중에 시간과 service 이름을 추측해 연결해야 한다.

반면 같은 context가 이어지면 trace_id를 중심으로 execution을 재구성할 수 있다.

## 2. OpenTelemetry가 제공하는 공통 언어

OpenTelemetry의 강점은 특정 backend가 아니다. Instrumentation과 telemetry model을 vendor-neutral하게 정의한다는 데 있다.

Logs에서는 TraceId와 SpanId를 LogRecord에 담아 trace와 연결할 수 있다. Resource는 service와 runtime에 대한 공통 context를 표현한다.

예를 들면 service.name, service.version, deployment.environment.name 같은 값이다.

Agent 입장에서 이 값들은 단순 metadata가 아니라 Debug Context를 묶는 key다.

## 3. trace_id만 있으면 충분한가

아니다. trace_id는 request correlation에는 강하지만 모든 debugging scope를 표현하지 못한다.

예를 들어 deployment 전체에서 발생한 GC regression, 특정 Pod만 반복 재시작하는 문제, 특정 version만 error가 증가하는 문제는 하나의 trace ID로 설명되지 않는다.

그래서 evidence에는 Environment, Service, Version, Deployment, Pod, Trace, Span, Request 같은 여러 scope가 필요하다.

tenant와 user 정보는 privacy 때문에 더 조심해야 한다.

## 4. Spring Boot에서 correlation이 실제로 어떻게 이어지는가

Spring Boot와 Micrometer Tracing을 사용하면 traceId/spanId를 MDC에 넣어 로그 correlation에 사용할 수 있다.

~~~text
HTTP Request
   ↓
Trace Context
   ↓
Controller
   ↓
Service
   ↓
HTTP Client / DB Client
   ↓
MDC Log
~~~

이때 중요한 실패 유형이 있다. 개발자가 auto-configured HTTP client builder를 쓰지 않고 직접 client를 만들면 context propagation이 끊길 수 있다.

그러면 service A trace와 service B trace가 분리된다. service B가 장애를 냈는데 Agent가 A의 trace만 보면 downstream 호출이 사라진 것처럼 보인다.

이는 instrumentation이 존재하는 것과 debuggable correlation이 완성된 것이 다르다는 좋은 사례다.

## 5. Collector는 단순 전달기가 아니다

OpenTelemetry Collector를 단순히 telemetry를 옮기는 proxy로 보면 역할을 과소평가하게 된다.

Collector는 Receiver → Processor → Exporter pipeline을 제공한다.

Agentic Debugging에서는 Processor가 특히 중요하다.

- Filter는 수집하지 않아도 되는 telemetry를 제거한다.
- Transform은 attribute를 정규화하거나 변환한다.
- Redaction은 민감 값을 삭제하거나 mask한다.
- Sampling은 보존할 trace를 선택한다.
- Enrichment는 service/deployment metadata를 추가한다.

즉 Agent에게 telemetry가 도달하기 전에 이미 quality와 security가 결정된다.

## 6. Collection Governance를 먼저 생각한다

Agent query layer에서 authorization header를 모델에게 보여주지 말라고 막을 수 있다.

하지만 그 header가 이미 Loki에 평문 저장돼 있다면 운영 보안 관점에서는 늦었다.

그래서 두 단계가 필요하다.

~~~text
Application
  ↓
Collector Governance
  ↓
Telemetry Backend
  ↓
Agent Retrieval Governance
~~~

Collector는 저장 자체를 통제한다. Agent Gateway는 접근을 통제한다. 이 둘은 다른 책임이다.

## 7. Sampling은 Debugging과 충돌할 수 있다

trace sampling은 비용을 줄인다. 하지만 드문 오류 request를 버리면 debugging evidence가 사라진다.

이 때문에 다음을 구분해야 한다.

> Collection Sampling ≠ Retrieval Sampling

Collection sampling은 무엇을 저장할지 결정한다. Retrieval limit은 저장된 것 중 Agent에게 무엇을 보여줄지 결정한다.

예를 들어 error trace와 매우 느린 trace는 tail sampling으로 보존하고, Agent query에서는 그 중 5개만 representative trace로 반환할 수 있다.

## 8. service.version을 잊지 않는다

Agentic Debugging에서 OpenTelemetry Resource가 특별히 중요한 이유가 하나 더 있다. runtime과 source를 연결하기 위해서다.

~~~text
service.name = login-service
service.version = a81c92f
deployment.environment.name = production
~~~

이 값이 trace와 log에 연결돼 있으면 Agent는 현재 workspace와 비교할 수 있다.

~~~text
runtime a81c92f
workspace b115e91
~~~

다르다면 바로 patch하지 않는다. 먼저 해당 version의 source를 찾거나 diff를 확인한다.

이 과정은 debugging에서 매우 중요하지만 전통적인 dashboard 중심 observability에서는 쉽게 빠진다.

## 9. Agent-friendly schema를 만들 때 주의할 점

여기서 새로운 거대한 AI 로그 표준을 만들 필요는 없다. 가능하면 기존 semantic convention을 사용한다.

책에서 별도 schema가 필요한 부분은 storage format이 아니라 session/evidence envelope이다.

~~~text
OpenTelemetry
= canonical telemetry context

Agent Debug Session Contract
= investigation workflow context
~~~

로 책임을 분리하는 편이 낫다.

## 10. correlation은 디버깅의 시작일 뿐이다

trace와 log가 연결됐다고 root cause가 나온 것은 아니다.

trace_id가 같다는 것은 사실이고, 이 warning이 latency의 원인이라는 것은 추론이다.

Agent가 이 둘을 섞지 않도록 이후 장에서 Evidence Record와 Hypothesis Record를 분리할 것이다.

## 11. 네 번째 원칙

> Agent용 observability를 설계할 때 새로운 로그 문법보다 먼저 execution correlation을 완성한다.

그리고:

> Collection Governance와 Retrieval Governance를 분리한다.

OpenTelemetry는 이 두 작업의 중심에서 logs, metrics, traces, resource identity를 연결하는 뼈대 역할을 한다.

다음 장부터는 이 correlated telemetry를 실제로 Agent가 어떻게 단계적으로 좁혀가는지 본다. 첫 번째 도구는 Prometheus다.

### 주요 근거

- [S-OTEL-LOGS] OpenTelemetry Logging Specification
- [S-OTEL-COLLECTOR] OpenTelemetry Collector
- [S-SPRING-OBS] Spring Boot Observability