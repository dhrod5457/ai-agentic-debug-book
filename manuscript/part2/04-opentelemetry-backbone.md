# 4장. OpenTelemetry를 상관관계의 뼈대로 사용한다

Agent에게 logs, metrics, traces를 모두 연결한다고 말하면 가장 먼저 기술 스택을 떠올리기 쉽다. Prometheus를 쓸 것인가, Loki를 쓸 것인가, Tempo를 쓸 것인가.

하지만 그보다 먼저 해결해야 할 문제가 있다.

> 서로 다른 signal이 같은 실행에서 나온 것임을 어떻게 알 것인가?

이 문제를 해결하지 않으면 backend를 몇 개 붙여도 Agent는 여전히 조각난 정보를 본다.

## 1. 먼저 같은 요청의 흔적을 서로 연결한다

하나의 로그인 요청을 생각해보자.

Metric에는 요청 duration이 기록되고, trace에는 trace_id와 span_id가 남고, log에는 authentication failed가 기록되며, DB span에는 SELECT user가 남을 수 있다.

이 각각이 따로 저장되면 나중에 시간과 서비스 이름을 추측해 연결해야 한다.

반면 같은 맥락가 이어지면 trace_id를 중심으로 execution을 재구성할 수 있다.

## 2. OpenTelemetry가 제공하는 공통 언어

OpenTelemetry의 강점은 특정 backend가 아니다. Instrumentation과 관측 데이터 model을 vendor-neutral하게 정의한다는 데 있다.

Logs에서는 TraceId와 SpanId를 LogRecord에 담아 trace와 연결할 수 있다. Resource는 서비스와 runtime에 대한 공통 맥락를 표현한다.

예를 들면 서비스.name, 서비스.version, 배포.environment.name 같은 값이다.

Agent 입장에서 이 값들은 단순한 부가 정보가 아니다. 같은 장애의 자료를 묶어 찾는 기준이 된다.

## 3. trace_id만 있으면 충분한가

아니다. trace_id는 요청 correlation에는 강하지만 모든 debugging scope를 표현하지 못한다.

예를 들어 배포 전체에서 발생한 GC regression, 특정 Pod만 반복 재시작하는 문제, 특정 version만 error가 증가하는 문제는 하나의 trace ID로 설명되지 않는다.

그래서 증거에는 Environment, 서비스, Version, 배포, Pod, Trace, Span, 요청 같은 여러 scope가 필요하다.

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

이때 중요한 실패 유형이 있다. 개발자가 auto-configured HTTP client builder를 쓰지 않고 직접 client를 만들면 맥락 propagation이 끊길 수 있다.

그러면 서비스 A trace와 서비스 B trace가 분리된다. 서비스 B가 장애를 냈는데 Agent가 A의 trace만 보면 downstream 호출이 사라진 것처럼 보인다.

이는 instrumentation이 존재하는 것과 debuggable correlation이 완성된 것이 다르다는 좋은 사례다.

## 5. Collector는 단순 전달기가 아니다

OpenTelemetry Collector를 단순히 관측 데이터를 옮기는 proxy로 보면 역할을 과소평가하게 된다.

Collector는 Receiver → Processor → Exporter pipeline을 제공한다.

Agentic Debugging에서는 Processor가 특히 중요하다.

- Filter는 수집하지 않아도 되는 관측 데이터를 제거한다.
- Transform은 attribute를 정규화하거나 변환한다.
- Redaction은 민감 값을 삭제하거나 mask한다.
- Sampling은 보존할 trace를 선택한다.
- Enrichment는 서비스/배포 metadata를 추가한다.

즉 Agent에게 관측 데이터가 도달하기 전에 이미 quality와 security가 결정된다.

## 6. 저장하기 전에 민감한 정보를 걸러낸다

Agent 조회 layer에서 authorization header를 모델에게 보여주지 말라고 막을 수 있다.

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

trace sampling은 비용을 줄인다. 하지만 드문 오류 요청를 버리면 debugging 증거가 사라진다.

이 때문에 다음을 구분해야 한다.

> 무엇을 저장할지 정하는 일과, 저장된 것 중 Agent가 무엇을 볼지 정하는 일은 다르다.

Collection sampling은 무엇을 저장할지 결정한다. Retrieval limit은 저장된 것 중 Agent에게 무엇을 보여줄지 결정한다.

예를 들어 error trace와 매우 느린 trace는 tail sampling으로 보존하고, Agent 조회에서는 그 중 5개만 representative trace로 반환할 수 있다.

## 8. 실행 버전도 같은 흐름에 넣는다

trace와 log가 어떤 서비스에서 나왔는지만 알아서는 부족할 때가 있다.

같은 서비스라도 배포 버전이 다르면 코드가 다를 수 있다.

그래서 서비스.version 같은 값을 관측 데이터에 함께 남기면 나중에 운영 실행과 소스코드를 연결하기 쉬워진다.

여기서는 OpenTelemetry가 이런 정보를 함께 실어 나를 수 있다는 점만 기억하자.

실제 장애에서 운영 버전과 현재 workspace를 비교하는 방법은 18장에서 다룬다.

## 9. Agent-friendly schema를 만들 때 주의할 점

여기서 새로운 거대한 AI 로그 표준을 만들 필요는 없다. 가능하면 기존 semantic convention을 사용한다.

책에서 별도 schema가 필요한 부분은 storage format이 아니라 session/증거 envelope이다.

~~~text
OpenTelemetry
= canonical telemetry context

Agent Debug Session Contract
= investigation workflow context
~~~

로 책임을 분리하는 편이 낫다.

## 10. 정보를 연결하는 것은 디버깅의 시작일 뿐이다

trace와 log가 연결됐다고 원인가 나온 것은 아니다.

trace_id가 같다는 것은 사실이고, 이 warning이 latency의 원인이라는 것은 추론이다.

Agent가 이 둘을 섞지 않도록 이후 장에서 증거 Record와 원인 후보 Record를 분리할 것이다.

## 11. 네 번째 원칙

> Agent용 관측 시스템를 설계할 때 새로운 로그 문법보다 먼저 execution correlation을 완성한다.

그리고:

> Collection Governance와 Retrieval Governance를 분리한다.

OpenTelemetry는 이 두 작업의 중심에서 logs, metrics, traces, resource identity를 연결하는 뼈대 역할을 한다.

다음 장부터는 이 correlated 관측 데이터를 실제로 Agent가 어떻게 단계적으로 좁혀가는지 본다. 첫 번째 도구는 Prometheus다.

### 참고 자료

출처 상세: [References](../references.md)

- [S-OTEL-LOGS] OpenTelemetry Logging Specification
- [S-OTEL-COLLECTOR] OpenTelemetry Collector
- [S-SPRING-OBS] Spring Boot 관측 시스템