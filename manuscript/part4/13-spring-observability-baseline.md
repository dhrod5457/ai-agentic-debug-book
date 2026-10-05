# 13장. Spring Boot에 Agent가 읽을 수 있는 흔적을 남긴다

Agent가 운영 문제를 잘 찾게 하려면 먼저 애플리케이션이 충분한 흔적을 남겨야 한다.

이 장의 목표는 거창한 관측 시스템을 만드는 것이 아니다.

Spring Boot 애플리케이션에서 최소한 무엇을 남겨야 다음 장의 디버깅이 가능한지 정리한다.

## 1. 로그만 남겨서는 부족하다

다음 로그가 있다고 하자.

~~~text
2026-10-05 09:10:14 WARN connection timeout
~~~

이 한 줄만으로는 알기 어렵다.

- 어떤 요청이었는가
- 어느 서버에서 발생했는가
- 어느 버전이었는가
- 같은 요청의 trace는 무엇인가

그래서 로그에는 최소한 요청과 실행 버전을 연결할 수 있는 정보가 필요하다.

## 2. traceId와 spanId를 로그에 같이 남긴다

Spring Boot와 Micrometer Tracing을 사용하면 traceId와 spanId를 MDC에 넣어 로그와 trace를 연결할 수 있다.

예:
~~~text
traceId=4f91... spanId=7ab2...
service=login-service
version=a81c92f
message=connection timeout
~~~

이제 Agent는 trace ID를 기준으로 로그와 실행 경로를 묶어 볼 수 있다.

## 3. service.version을 반드시 남긴다

운영 장애에서 가장 위험한 실수 중 하나는 다른 버전의 코드를 고치는 것이다.

그래서 최소한 다음은 남겨야 한다.

~~~text
service.name
service.version
deployment.environment.name
~~~

가능하면 image digest와 deployment revision도 함께 둔다.

## 4. HTTP 호출의 trace가 끊기지 않게 한다

서비스 A가 서비스 B를 호출한다.

trace가 잘 이어지면 한 요청으로 보인다.

~~~text
A /login
  ↓
B /user
~~~

하지만 직접 만든 HTTP client 때문에 context propagation이 빠지면 두 trace가 분리된다.

Agent는 downstream 호출 자체가 없었던 것처럼 오해할 수 있다.

그래서 auto-configured client를 사용하거나 trace propagation을 명시적으로 확인해야 한다.

## 5. DB 호출도 trace에 남긴다

DB 문제를 찾으려면 적어도 다음은 보여야 한다.

- query summary
- duration
- error type
- connection acquire와 query execute의 구분

특히 connection을 얻는 데 오래 걸린 것과 SQL 실행이 느린 것은 전혀 다른 문제다.

## 6. 로그는 가능하면 구조화한다

JSON 로그는 Agent에게도 유리하다.

예:
~~~json
{
  "service": "login-service",
  "traceId": "4f91...",
  "event": "connection_acquire_timeout",
  "active": 20,
  "idle": 0,
  "pending": 37
}
~~~

자연어 로그보다 숫자와 key가 분리되어 있어 검색과 비교가 쉽다.

## 7. 그래도 모든 값을 로그에 넣지 않는다

request body, Authorization header, cookie, SQL parameter 같은 값은 민감할 수 있다.

Agent가 보기 편하다는 이유로 수집 범위를 늘리면 안 된다.

필요한 correlation 정보와 디버깅 정보만 남긴다.

## 8. 최소 구성

다음 정도면 이후 실습을 진행하기에 충분하다.

~~~text
Spring Boot
  ├─ Micrometer Observation
  ├─ Tracing
  ├─ traceId/spanId log correlation
  ├─ structured log
  ├─ DB span
  ├─ JVM metrics
  └─ service.version

Backend
  ├─ Prometheus
  ├─ Loki
  ├─ Tempo
  └─ Pyroscope
~~~

## 9. 이제 실제 장애를 따라가 보자

여기까지는 준비 단계였다.

다음 장부터는 이 구성을 실제 장애에 적용한다.

첫 번째 사례는 가장 흔하면서도 오판하기 쉬운 DB connection pool 문제다.

로그에 SQL timeout이 보이지만 실제 SQL은 빠른 상황을 따라가 보자.

## 10. 이 장에서 기억할 것

> Agent가 볼 수 있는 흔적을 남기되, 같은 요청과 같은 실행 버전을 서로 연결할 수 있어야 한다.

### 주요 근거

- [S-SPRING-OBS] Spring Boot Observability
- [S-OTEL-LOGS] OpenTelemetry Logging Specification