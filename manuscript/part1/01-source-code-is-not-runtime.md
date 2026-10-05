# 1장. 소스코드만 보는 Agent는 애플리케이션을 모른다

월요일 오전 9시 10분, 로그인 API의 p99 응답 시간이 200ms에서 3초로 올라갔다고 해보자.

담당 개발자가 코드를 열었다. 지난주부터 로그인 관련 코드는 바뀌지 않았다. SQL도 단순하다. 단위 테스트와 통합 테스트도 모두 통과한다.

Coding Agent에게 물어본다.

> 로그인 API가 갑자기 느려졌어. 원인을 찾아줘.

Agent는 저장소를 탐색한다. Controller를 찾고, Service를 따라가고, Repository의 SQL을 읽는다. 몇 가지 가능성을 제시한다.

- 인덱스 부족
- N+1
- 외부 인증 서버 지연
- connection pool 부족
- 불필요한 synchronized
- 직렬화 비용

모두 그럴듯하다.

하지만 아직 디버깅은 시작되지 않았다.

## 1. 코드는 가능성을 보여주고 실행은 사실을 보여준다

소스코드는 시스템이 무엇을 할 수 있는지 보여준다. 운영 장애에서 필요한 것은 실제로 무엇이 일어났는지다.

같은 코드는 서로 다른 상태에서 전혀 다른 행동을 보인다.

~~~text
같은 OrderService.java

환경 A
DB connection 충분
Redis 정상
외부 API 50ms
→ 120ms 응답

환경 B
DB pool pending 증가
외부 API 정상
→ 3초 응답

환경 C
DB 정상
Pod memory pressure
GC pause 증가
→ 간헐적 timeout
~~~

Repository를 아무리 오래 읽어도 환경 B인지 C인지는 알 수 없다.

그래서 application debugging에는 두 종류의 상태가 존재한다.

~~~text
Source State
- code
- config template
- tests
- dependencies

Runtime State
- actual request path
- latency
- errors
- pool usage
- JVM state
- pod state
- deployed version
~~~

Coding Agent는 기본적으로 첫 번째 상태에 강하다. 운영 debugging에는 두 번째 상태가 필요하다.

## 2. stack trace가 있으면 충분하지 않은가

예외가 명확한 장애라면 stack trace는 매우 좋은 출발점이다.

~~~text
NullPointerException
  at OrderService.create(OrderService.java:142)
~~~

이런 경우 Agent는 빠르게 코드 위치로 이동할 수 있다.

하지만 production incident의 상당수는 이렇게 친절하지 않다. 성능 저하는 예외가 없을 수 있다.

~~~text
HTTP 200
duration = 3.2s
~~~

connection pool 고갈도 최종적으로는 timeout 하나만 남을 수 있다.

~~~text
SQLTransientConnectionException
Connection is not available
request timed out after 3000ms
~~~

이 메시지만 보면 데이터베이스가 느린 것처럼 보인다. 실제로는 긴 transaction 때문에 connection이 반환되지 않았을 수도 있다.

또 어떤 장애는 애플리케이션 예외가 아니라 platform에서 발생한다.

~~~text
Pod terminated
Reason: OOMKilled
~~~

이 경우 애플리케이션 stack trace를 아무리 검색해도 정답이 없다.

## 3. 사람이 디버깅할 때 이미 하고 있는 일

사람은 이런 문제를 만났을 때 사실상 evidence graph를 만든다.

로그인 지연 사례를 다시 보자. 개발자는 먼저 Grafana에서 지연시간 증가 시점을 찾는다.

~~~text
09:08 정상
09:10 p99 상승
09:14 peak
09:18 회복
~~~

다음으로 같은 시간대에 다른 metric을 본다.

~~~text
DB CPU          정상
Hikari active   max 근접
Hikari pending  급증
JVM CPU         정상
GC pause        정상
~~~

이제 connection pool 쪽이 의심된다.

Tempo에서 느린 trace 하나를 연다.

~~~text
POST /login                  3.1s
 └─ UserService.authenticate 3.0s
     └─ acquireConnection    2.8s
     └─ SELECT user          34ms
~~~

중요한 사실이 보인다. SQL 실행 자체는 34ms다. 느린 것은 connection을 얻는 과정이다.

같은 trace ID를 Loki에서 검색한다.

~~~text
connection acquisition timeout
active=20 idle=0 pending=37
~~~

이제 'SQL이 느리다'라는 첫 가설은 약해진다.

이 과정에서 사람은 사실 다음 순서를 밟았다.

~~~text
Symptom
  ↓
Time Window
  ↓
Metric Correlation
  ↓
Representative Execution
  ↓
Local Evidence
  ↓
Hypothesis
~~~

Agent도 같은 방식으로 움직여야 한다.

## 4. 필요한 것은 로그가 아니라 Runtime Evidence다

이 책에서는 debugging을 위해 Agent가 사용하는 실행 중 증거를 Application Runtime Evidence라고 부른다. 외부 표준 용어는 아니다.

Runtime Evidence에는 로그만 들어가지 않는다.

- Logs는 무슨 사건이 기록됐는지 보여준다.
- Metrics는 언제, 어디서, 얼마나 문제가 커졌는지 보여준다.
- Traces는 request가 어떤 service와 operation을 지나갔는지 보여준다.
- Profiles는 CPU, allocation, lock time이 어디서 쓰였는지 보여준다.
- JVM diagnostics는 GC, thread, lock 같은 내부 실행 상태를 보여준다.
- Database evidence는 query, pool, lock/wait 같은 DB 경계의 사실을 보여준다.
- Platform evidence는 Pod, Node, deployment 상태를 보여준다.
- Artifact identity는 이 모든 evidence가 어떤 실행 버전에서 발생했는지 알려준다.

이 중 어느 하나가 항상 정답을 주지는 않는다. 핵심은 서로 연결할 수 있다는 데 있다.

## 5. Correlation이 먼저다

다음 로그와 trace가 있다고 하자.

~~~text
09:10:13 WARN connection timeout

POST /login 3.1s
~~~

시간이 비슷하다는 이유만으로 같은 사건이라고 볼 수는 없다.

반면 둘 다 다음 값을 가지고 있다면 상황이 달라진다.

~~~text
trace_id = 4f91...
~~~

이제 하나의 execution으로 연결할 수 있다.

OpenTelemetry가 중요한 이유도 여기에 있다. TraceId와 SpanId, Resource context를 logs와 traces 사이에 연결할 수 있기 때문이다.

이 책에서 첫 번째로 강조할 것은 AI 전용 로그 포맷이 아니다. 먼저 제대로 된 correlation이다.

## 6. correlation도 root cause는 아니다

같은 시간에 두 값이 올라갔다고 하나가 다른 하나의 원인이라는 뜻은 아니다.

~~~text
API latency ↑
CPU ↑
~~~

CPU가 원인일 수도 있고 긴 retry loop 때문에 결과적으로 CPU가 올라간 것일 수도 있다.

그래서 Agent가 'CPU가 상승했으므로 CPU 부족이 root cause입니다'라고 말하면 위험하다. 관측과 추론이 섞였기 때문이다.

더 나은 구조는 다음과 같다.

~~~text
Evidence E1
API p99가 09:10~09:18 상승

Evidence E2
같은 시간 CPU가 45%→82% 상승

Hypothesis H1
CPU saturation이 latency 원인일 수 있음

Missing Evidence
run queue, GC, profile, throttling

Next Test
CPU profile과 container throttling 확인
~~~

Evidence는 사실이다. Hypothesis는 해석이다. 이 둘은 시스템 차원에서 분리해야 한다.

## 7. Agent에게 애플리케이션을 보여준다는 말의 의미

'Agent에게 운영 로그를 보여준다'는 표현은 너무 좁다.

우리가 원하는 것은 다음에 가깝다.

~~~text
Source Code
       ↕
Coding Agent
       ↕
Runtime Evidence Interface
       ↕
Running Application
~~~

Agent는 필요할 때 물어야 한다.

- 어느 시간대가 문제인가?
- 어느 service인가?
- 정상 request와 느린 request의 차이는 무엇인가?
- 같은 trace의 로그는 무엇인가?
- 해당 span의 profile은 어떤가?
- incident 당시 어떤 image가 배포돼 있었는가?
- 현재 내가 보고 있는 source와 같은 버전인가?

이 질문에 시스템이 machine-readable evidence로 답할 수 있어야 한다.

그때부터 Coding Agent는 비로소 애플리케이션을 '본다'고 말할 수 있다.

## 8. 첫 번째 원칙

> Raw Log ≠ Debug Context

로그는 Debug Context의 일부일 뿐이다.

그리고:

> Source State ≠ Runtime State

코드만 잘 읽는 Agent는 프로그램 구조를 이해할 수 있다. 하지만 장애가 일어난 실행을 이해하려면 실제 runtime evidence가 필요하다.

다음 장에서는 가장 흔한 해결책 하나를 검토한다.

'그렇다면 로그를 전부 Agent에게 넣으면 되지 않을까?'

그 방식이 왜 작은 데모에서는 동작하고 실제 시스템에서는 무너지는지 살펴보자.

### 주요 근거

- [S-OTEL-LOGS] OpenTelemetry Logging Specification
- [S-OPENRCA] OpenRCA, ICLR 2025
- [S-NEXT] NExT, ICML 2024