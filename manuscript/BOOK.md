# AI Agentic Debugging

> 소스코드만 보는 Coding Agent에게 실제 애플리케이션의 실행 상태를 어떻게 보여줄 것인가?

# 들어가며 — Agent에게 애플리케이션을 보여주는 법

코딩 Agent는 소스코드를 잘 읽는다. 파일을 찾고, 함수를 따라가고, 테스트를 실행하고, 오류 메시지를 보고 수정안을 만든다. 로컬에서 재현 가능한 버그라면 이것만으로도 꽤 많은 문제를 해결할 수 있다.

문제는 애플리케이션이 실행 중일 때 시작된다.

운영에서 특정 API의 응답 시간이 갑자기 3초를 넘는다. 간헐적으로 500 응답이 발생한다. 특정 배포 이후부터 오류율이 올라간다. 어떤 요청은 정상인데 어떤 요청은 같은 코드에서 실패한다. 로그에는 SQL timeout이 보이지만 데이터베이스 CPU는 정상이다. Pod는 재시작됐지만 애플리케이션 stack trace에는 특별한 예외가 없다.

사람은 이런 문제를 소스코드만 보고 풀지 않는다.

Grafana를 열어 지표를 본다. Prometheus에서 오류율과 지연시간을 확인한다. Tempo에서 느린 trace를 찾는다. Loki에서 같은 trace ID를 가진 로그를 검색한다. 필요하면 profiler를 열고, thread dump를 뜨고, Kubernetes Event와 배포 revision을 확인한다. 데이터베이스 connection pool과 slow 조회를 비교한다. 마지막에는 장애가 발생한 버전과 현재 checkout된 코드가 같은지도 확인한다.

그런데 Coding Agent에게는 종종 이 중 아무것도 주어지지 않는다.

대신 "운영에서 느리대. 코드 봐줘."라는 요청이나 application.log 전체가 전달된다.

이 책은 이 간극을 다룬다.

> 소스코드만 보는 Coding Agent에게 실제 애플리케이션의 실행 상태를 어떻게 보여줄 것인가?

이 질문을 풀기 위해 새로운 APM을 만들 필요는 없다. 이미 소프트웨어 산업은 오랫동안 관측 시스템 문제를 해결해 왔다. OpenTelemetry는 logs, metrics, traces를 공통 맥락으로 연결한다. Prometheus는 metric을 조회한다. Loki는 log를 저장하고 검색한다. Tempo는 distributed trace를 찾는다. Pyroscope는 continuous profile을 제공한다. Grafana는 이 신호들을 사람이 탐색할 수 있게 묶는다.

2026년에는 이 경계가 한 단계 더 움직였다. Grafana와 Tempo는 Agent가 관측 데이터를 직접 질의할 수 있는 MCP 인터페이스까지 제공하기 시작했다.

따라서 질문은 더 이상 "AI가 로그를 읽을 수 있는가?"가 아니다.

> Agent에게 어떤 증거를, 어떤 범위로, 어떤 순서로, 어떤 권한 아래에서 조회하게 해야 하는가?

이 책에서는 이를 실행 증거(Runtime Evidence)라고 부른다. 이 용어는 외부 표준이 아니라 이 책의 설명을 위해 정리한 표현다.

실행 증거에는 로그만 들어가지 않는다.

~~~text
Request
  ├─ trace
  ├─ span
  └─ HTTP/RPC

Application
  ├─ structured log
  ├─ exception
  └─ state transition

Database
  ├─ query summary
  ├─ latency/error
  └─ connection pool

Runtime
  ├─ JVM
  ├─ JFR
  ├─ thread
  └─ profile

Platform
  ├─ Pod
  ├─ Deployment
  ├─ Event
  └─ Node

Artifact
  ├─ service.version
  ├─ image digest
  ├─ deployment revision
  └─ source commit
~~~

이 정보는 모두 같은 가치와 비용을 가지지 않는다. Prometheus metric 조회는 저렴하지만 heap dump는 비싸다. trace는 요청 경로를 보여주지만 원인를 자동으로 알려주지는 않는다. Kubernetes Event는 힌트지만 canonical truth가 아니다. SQL text는 유용하지만 parameter에는 개인정보가 들어갈 수 있다.

그래서 이 책은 "더 많은 정보"보다 "더 좋은 관측 인터페이스"를 설계하는 데 집중한다.

책 전체에서 반복해서 지킬 경계는 다음과 같다.

~~~text
Raw Log ≠ Debug Context
Dashboard ≠ Agent Interface
Telemetry Storage ≠ Model Context
Correlation ≠ Root Cause
Anomaly ≠ Root Cause
Evidence Retrieval ≠ Reasoning
Observation Authority ≠ Mutation Authority
Test Pass ≠ Incident Resolution
~~~

1부에서는 왜 소스코드와 로그만으로는 부족한지 설명한다. 2부에서는 OpenTelemetry, Prometheus, Tempo, Loki, Pyroscope를 Agent의 눈과 귀로 바꾸는 방법을 본다. 3부에서는 Grafana MCP를 포함한 실제 Agent interface와 그 위의 Debug Workflow를 설계한다. 4부에서는 Spring Boot 장애를 실제 사례로 따라간다. 5부에서는 운영 권한, 비용, 보안, 검증, 평가를 다룬다.

소스코드는 프로그램이 무엇을 하도록 작성되었는지를 보여준다.

실행 증거는 프로그램이 실제로 무엇을 했는지를 보여준다.

Agentic Debugging은 이 두 세계를 연결하는 일에서 시작한다.

---

# Part I. 로그를 주는 것과 디버깅을 가능하게 하는 것은 다르다

# 1장. 소스코드만 보는 Agent는 애플리케이션을 모른다

월요일 오전 9시 10분, 로그인 API의 p99 응답 시간이 200ms에서 3초로 올라갔다고 해보자.

담당 개발자가 코드를 열었다. 지난주부터 로그인 관련 코드는 바뀌지 않았다. SQL도 단순하다. 단위 테스트와 통합 테스트도 모두 통과한다.

Coding Agent에게 물어본다.

> 로그인 API가 갑자기 느려졌어. 원인을 찾아줘.

Agent는 저장소를 탐색한다. Controller를 찾고, 서비스를 따라가고, Repository의 SQL을 읽는다. 몇 가지 가능성을 제시한다.

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

하지만 운영 장애의 상당수는 이렇게 친절하지 않다. 성능 저하는 예외가 없을 수 있다.

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

사람은 이런 문제를 만났을 때 여러 증거를 서로 연결해서 본다.

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

## 4. 필요한 것은 로그가 아니라 실행 중에 남은 증거다

이 책에서는 Agent가 장애를 조사할 때 사용하는 이런 자료를 편의상 '실행 증거(실행 증거)'라고 부르겠다. 외워야 할 표준 용어는 아니다.

실행 증거에는 로그만 들어가지 않는다.

- Logs는 무슨 사건이 기록됐는지 보여준다.
- Metrics는 언제, 어디서, 얼마나 문제가 커졌는지 보여준다.
- Traces는 요청가 어떤 서비스와 operation을 지나갔는지 보여준다.
- Profiles는 CPU, allocation, lock time이 어디서 쓰였는지 보여준다.
- JVM diagnostics는 GC, thread, lock 같은 내부 실행 상태를 보여준다.
- Database 증거는 조회, pool, lock/wait 같은 DB 경계의 사실을 보여준다.
- Platform 증거는 Pod, Node, 배포 상태를 보여준다.
- Artifact identity는 이 모든 증거가 어떤 실행 버전에서 발생했는지 알려준다.

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

OpenTelemetry가 중요한 이유도 여기에 있다. 같은 요청에서 나온 로그와 trace를 TraceId와 SpanId로 묶어 볼 수 있기 때문이다.

이 책에서 첫 번째로 강조할 것은 AI 전용 로그 포맷이 아니다. 먼저 같은 실행에서 나온 정보를 제대로 연결하는 것이 중요하다.

## 6. 같이 보인다고 원인인 것은 아니다

같은 시간에 두 값이 올라갔다고 하나가 다른 하나의 원인이라는 뜻은 아니다.

~~~text
API latency ↑
CPU ↑
~~~

CPU가 원인일 수도 있고 긴 retry loop 때문에 결과적으로 CPU가 올라간 것일 수도 있다.

그래서 Agent가 'CPU가 상승했으므로 CPU 부족이 원인입니다'라고 말하면 위험하다. 관측과 추론이 섞였기 때문이다.

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

확인한 사실과 원인에 대한 추측은 다르다. 이 둘은 시스템 차원에서 분리해야 한다.

## 7. Agent에게 애플리케이션을 보여준다는 말의 의미

'Agent에게 운영 로그를 보여준다'는 표현은 너무 좁다.

우리가 원하는 것은 다음에 가깝다.

~~~text
Source Code
       ↕
Coding Agent
       ↕
실행 정보 조회 도구
       ↕
Running Application
~~~

Agent는 필요할 때 물어야 한다.

- 어느 시간대가 문제인가?
- 어느 서비스인가?
- 정상 요청와 느린 요청의 차이는 무엇인가?
- 같은 trace의 로그는 무엇인가?
- 해당 span의 profile은 어떤가?
- 장애 당시 어떤 image가 배포돼 있었는가?
- 현재 내가 보고 있는 source와 같은 버전인가?

이 질문에 시스템이 machine-readable 증거로 답할 수 있어야 한다.

그때부터 Coding Agent는 비로소 애플리케이션을 '본다'고 말할 수 있다.

## 8. 첫 번째 원칙

> Raw Log ≠ Debug 맥락

로그는 Debug 맥락의 일부일 뿐이다.

그리고:

> Source State ≠ Runtime State

코드만 잘 읽는 Agent는 프로그램 구조를 이해할 수 있다. 하지만 장애가 일어난 실행을 이해하려면 실제로 실행 중에 남은 정보가 필요하다.

다음 장에서는 가장 흔한 해결책 하나를 검토한다.

'그렇다면 로그를 전부 Agent에게 넣으면 되지 않을까?'

그 방식이 왜 작은 데모에서는 동작하고 실제 시스템에서는 무너지는지 살펴보자.

### 참고 자료

출처 상세: [References](../references.md)

- [S-OTEL-LOGS] OpenTelemetry Logging Specification
- [S-OPENRCA] OpenRCA, ICLR 2025
- [S-NEXT] NExT, ICML 2024

# 2장. 로그 파일을 통째로 넣으면 왜 실패하는가

처음 Agent에게 운영 정보를 제공하려 할 때 가장 자연스러운 방법은 로그를 붙여 넣는 것이다.

작은 프로그램에서는 잘 동작한다.

~~~text
ERROR NullPointerException ...
stacktrace...
~~~

몇십 줄짜리 로그를 주면 모델은 중요한 문장을 찾고 코드와 연결한다. 이 경험 때문에 다음 단계도 자연스럽게 보인다.

~~~text
application.log 10MB
→ LLM
~~~

그런데 운영 규모에서는 이 접근이 빠르게 무너진다. 문제는 단순히 컨텍스트 창 크기만이 아니다.

## 1. 로그는 실행 단위로 정렬되어 있지 않다

다음과 같은 로그를 생각해보자.

~~~text
09:10:13 req-A start /login
09:10:13 req-B start /login
09:10:13 req-C start /search
09:10:14 req-B db timeout
09:10:14 req-A auth success
09:10:14 req-D start /login
09:10:15 req-B failed
09:10:15 req-A completed
~~~

사람이 보더라도 correlation ID가 없다면 읽기 어렵다. 멀티스레드 애플리케이션, 여러 Pod, 여러 서비스로 가면 로그는 더 뒤섞인다.

시간순으로 나열된 텍스트는 causal order가 아니다. 로그를 많이 제공한다고 요청 boundary가 복원되는 것은 아니다.

## 2. irrelevant 증거가 추론을 방해한다

운영 로그 대부분은 현재 장애와 무관하다.

~~~text
health check
cache hit
scheduler heartbeat
normal request
background batch
debug message
startup log
unrelated tenant traffic
~~~

장애 요청 20줄을 찾기 위해 20만 줄을 넣으면 모델에게는 두 문제가 생긴다. 첫째는 비용이고 둘째는 attention이다.

맥락는 storage가 아니다. 모델이 읽을 수 있다고 해서 모든 정보가 같은 중요도로 처리되는 것은 아니다.

OpenRCA가 흥미로운 이유도 여기에 있다. 이 benchmark의 RCA-agent baseline은 방대한 관측 데이터를 model 맥락에 넣지 않고 Python으로 필요한 부분을 검색하고 분석한다.

데이터는 실행 환경에 남긴다. 모델에는 결과를 가져온다.

## 3. 모델이 읽는 내용과 실제 관측 데이터 전체는 다르다

이 차이를 명확히 해두자.

~~~text
Telemetry Working Set

수 GB logs
수백만 metric samples
수만 traces
profiles
events
~~~

와

~~~text
Model Context

현재 incident에 필요한
몇 개의 metric 결과
대표 trace
관련 로그
source diff
~~~

는 같은 것이 아니다.

이 책에서는 다음 원칙을 사용한다.

> 모델이 한 번에 읽는 내용과 관측 데이터 전체는 같은 것이 아니다.

관측 데이터는 조회 가능한 외부 상태로 남겨두는 편이 낫다. Agent는 필요한 증거만 단계적으로 가져온다.

## 4. 로그 전체를 넣으면 시간 범위도 흐려진다

장애가 09:10~09:15에 발생했다고 하자. 그런데 하루치 로그를 모두 넣으면 00:00부터 23:59까지의 사건이 함께 들어간다.

Agent가 우연히 다른 시간대의 같은 exception을 발견하면 잘못된 가설을 만들 수 있다.

실제 debugging에서는 time range가 매우 중요하다. 그래서 첫 질문은 보통 '언제부터 언제까지 문제가 있었는가?'다.

Agent 조회에도 같은 제약이 들어가야 한다. Grafana MCP의 Loki guardrail이 최대 effective time range를 두는 이유도 기술적으로 같은 문제와 맞닿아 있다.

## 5. 로그 전체는 보안 경계도 무너뜨린다

로그에는 생각보다 많은 정보가 들어간다.

- Authorization header
- cookie
- session ID
- user ID
- email
- DB bind value
- 요청 body
- internal URL
- tenant identifier

사람이 Grafana에서 필요한 조회만 보는 것과 로그 파일 전체를 외부 모델 맥락로 보내는 것은 보안적으로 전혀 다른 행위다.

그래서 privacy 문제는 prompt 직전에만 해결할 수 없다. 수집 단계에서부터 redaction과 filtering이 필요하다.

~~~text
Application
  ↓
Collector
  ├─ filter
  ├─ redact
  └─ transform
  ↓
Telemetry Backend
  ↓
Agent Query Gateway
~~~

저장하면 안 되는 값은 Collector에서 제거한다. 저장된 데이터 중 Agent가 볼 수 있는 범위는 조회 Gateway에서 다시 제한한다.

이를 다음처럼 구분할 수 있다.

> Collection Governance ≠ Retrieval Governance

## 6. 오래된 로그는 현재 코드와 맞지 않을 수 있다

운영 장애가 난 버전과 Agent가 보고 있는 현재 코드가 다를 수 있다.

이 사실을 확인하지 않고 stack trace의 line number만 따라가면 엉뚱한 코드를 수정할 수 있다.

그래서 로그와 trace에는 가능하면 실행 버전을 함께 남긴다.

버전을 실제로 어떻게 비교하고 어떤 코드를 열어야 하는지는 18장에서 자세히 다룬다.

## 7. Push 맥락에서 Pull 증거로

기존 방식은 관측 데이터를 모아서 prompt에 넣는 것이다.

더 나은 방식은 Agent가 관측 데이터 backend에 질문하는 것이다.

~~~text
Telemetry Backend
      ↑
    query
      ↑
Agent
~~~

Agent는 먼저 문제 시간대의 login-서비스 p99 latency를 묻는다. 그 결과를 보고 Hikari pending connection을 묻는다. 다음에는 3초 이상 걸린 trace 몇 개를 찾고, 그중 하나의 trace ID로 로그를 검색한다.

이렇게 넓은 현상에서 시작해 필요한 자료만 단계적으로 좁혀간다.

## 8. 필요한 자료를 찾는 과정도 무제한이면 안 된다

필요한 자료를 그때그때 찾는 방식도 범위를 정하지 않으면 지나치게 넓어질 수 있다.

예를 들어 30일 전체 로그를 한 번에 검색하게 두는 것은 좋은 기본값이 아니다.

여기서는 한 가지 원칙만 기억하면 된다.

> Agent가 조회할 수 있다고 해서 무제한으로 조회하게 두지는 않는다.

시간 범위, 결과 수, scan 크기 같은 제한은 11장에서 도구 설계와 함께 자세히 다룬다.

## 9. 결과가 잘렸다면 알려야 한다

Agent에게 'No errors found'라는 결과가 돌아왔다고 하자.

실제로는 100000 lines 중 앞 1000줄만 반환된 결과라면 결론은 완전히 달라진다.

그래서 tool response에는 조회 결과뿐 아니라 한계도 들어가야 한다.

~~~text
sampled=true
truncated=true
result_count=1000
continuation_available=true
~~~

OpenRCA의 Executor도 큰 DataFrame 결과가 잘린 경우 observation bias 가능성을 명시적으로 경고한다.

결과의 불완전성을 숨기지 않는 것이 중요하다.

## 10. 두 번째 원칙

> Model 맥락 ≠ 관측 데이터 Working Set

그리고:

> 모든 로그를 밀어 넣기보다 필요한 자료를 그때그때 찾아보는 방식이 확장하기 쉽다.

Agent에게 모든 로그를 읽게 하지 않는다. 대신 scope → aggregate → representative execution → local 증거 → source 순서로 필요한 증거를 가져오게 한다.

다음 장에서는 이 증거가 로그만으로 구성되지 않는 이유를 살펴본다.

### 참고 자료

출처 상세: [References](../references.md)

- [S-OPENRCA] OpenRCA
- [S-GRAFANA-MCP] Grafana MCP
- [S-LLM4LOG] LLM4Log
- [S-OTEL-COLLECTOR] OpenTelemetry Collector

# 3장. 디버깅 증거는 한 종류가 아니다

애플리케이션 장애가 발생하면 가장 먼저 로그를 찾는 습관이 있다. 로그는 중요하다. 하지만 로그가 모든 장애를 설명하지는 않는다.

성능 저하는 로그에 아무것도 남기지 않을 수 있다. CPU hotspot은 stack trace로 나타나지 않는다. Pod eviction은 애플리케이션 로그보다 Kubernetes 쪽이 더 정확하다. 잘못된 배포는 exception보다 version metadata가 더 중요한 증거일 수 있다.

Agent에게 중요한 것은 로그 하나가 아니라 장애를 설명할 수 있는 여러 종류의 자료다.

## 1. Metrics — 어디가 이상한지 알려준다

Metric은 aggregate signal이다.

~~~text
error rate
latency
throughput
CPU
memory
connection pool
GC
queue depth
~~~

Metric은 원인을 직접 말해주지 않는다. 대신 문제 공간을 줄인다.

예를 들어 login p99는 상승했는데 DB CPU는 정상이고 Hikari pending이 급증했다면 DB 자체보다 connection acquisition 쪽을 먼저 볼 이유가 생긴다.

그래서 metric의 역할은 무엇이 문제인지 확정하는 것보다 어디를 볼지 결정하는 것에 가깝다.

## 2. Traces — 실제 execution path를 보여준다

Metric이 집계라면 trace는 구체적인 요청다.

~~~text
POST /checkout 2.8s
 ├─ inventory 40ms
 ├─ payment 2.6s
 │   └─ external API 2.5s
 └─ save order 20ms
~~~

이제 slow 요청가 어디서 시간을 썼는지 볼 수 있다.

하지만 trace 역시 원인 그 자체는 아니다. external API span이 느린 이유가 network, upstream saturation, retry, DNS, connection pool, timeout configuration 중 무엇인지는 추가 증거가 필요하다.

## 3. Logs — execution에서 무슨 사건이 있었는지 보여준다

trace를 찾은 뒤 같은 trace ID로 로그를 좁히면 로그의 가치가 올라간다.

~~~text
trace_id=abc
WARN retry attempt=3
ERROR upstream timeout
~~~

이제 수백만 줄 중 몇 줄만 읽는다.

이것이 로그가 필요 없다는 뜻은 아니다. 로그를 나중에, 더 정확하게 읽는다는 뜻이다.

## 4. Profiles — 실제로 자원이 어디서 쓰였는지 보여준다

Trace에서 한 span이 2초 걸렸다고 하자. 그 2초 동안 CPU를 썼는지, lock에서 기다렸는지, I/O를 기다렸는지는 trace만으로 부족할 수 있다.

Profile은 이 문제를 해결한다.

~~~text
Slow Span
  ↓
Wall Profile
  ↓
LockSupport.park
  ↓
application method
~~~

Pyroscope처럼 trace와 profile을 연결할 수 있으면 Agent는 특정 요청의 느린 구간에서 어떤 코드가 시간을 사용했는지 더 직접적으로 좁힐 수 있다.

## 5. JVM diagnostics — 더 깊이 들어갈 때 사용한다

Java 애플리케이션에서는 JFR과 thread dump가 강력하다. 하지만 이 증거는 일반 metric 조회보다 비싸다.

Thread dump는 deadlock, blocked thread, thread pool starvation을 보여준다. JFR은 allocation, GC, lock, socket I/O, thread park 같은 runtime event를 보여준다. Heap dump는 retained object와 memory leak를 추적하는 데 강하다.

여기서 중요한 것은 '가능하면 다 수집'이 아니다. 필요한 경우 단계적으로 내려간다.

## 6. Database 증거 — SQL만 보면 부족하다

다음 SQL이 2.8초 걸렸다고 보인다고 하자.

실제로 DB execution은 30ms였고 connection을 얻는 데 2.7초 걸렸다면 SQL 튜닝은 잘못된 수정이다.

DB debugging에는 connection acquire, 조회 execute, lock wait, network, pool state, DB resource를 구분해야 한다.

OpenTelemetry의 DB semantic convention도 조회 summary와 조회 text를 구분한다. Agent에게는 low-cardinality 조회 summary와 duration/error를 먼저 주는 편이 안전하다. parameter 값은 기본적으로 숨기는 것이 낫다.

## 7. Platform 증거 — 코드가 문제가 아닐 수 있다

Kubernetes 환경에서는 애플리케이션 장애가 platform에서 시작될 수 있다.

~~~text
Pod restart
OOMKilled
Evicted
FailedScheduling
ImagePullBackOff
~~~

Kubernetes Event는 중요한 힌트다. 하지만 Event 하나를 canonical truth로 보면 위험하다. Pod status, container state, resource metric, 배포 revision을 함께 봐야 한다.

## 8. Artifact identity — 어떤 코드가 실제로 실행됐는가

이 증거는 자주 빠진다.

~~~text
service.version
container image digest
deployment revision
commit
config version
~~~

장애를 분석하는 Agent가 반드시 알아야 하는 이유가 있다.

~~~text
운영 trace = commit A
현재 workspace = commit B
~~~

일 수 있기 때문이다. Runtime version을 확인하지 않은 수정는 논리적으로 불완전하다.

## 9. 증거를 단계별로 본다

모든 증거를 같은 비용으로 취급하지 말자.

~~~text
Level 0
Metrics / Logs / Traces / Version Metadata

        ↓ 필요 시

Level 1
Representative Trace
Correlated Logs
Profile
DB Evidence
Kubernetes Status

        ↓ 필요 시

Level 2
JFR
Thread Dump
Lock Diagnostics

        ↓ 필요 시

Level 3
Heap Dump
GC Root
Deep DB Diagnostics
Container Exec
~~~

이 책에서는 이렇게 필요할 때 더 깊은 진단으로 내려가는 방식을 '증거 단계 올리기'라고 설명하겠다. 외워야 할 표준 용어는 아니다.

## 10. 왜 escalation이 필요한가

첫째는 운영 비용 때문이다. heap dump는 metric 조회와 같은 행위가 아니다.

둘째는 보안 때문이다. heap에는 사용자 데이터와 credential fragment가 있을 수 있다.

셋째는 권한 때문이다. 기존 JFR 파일을 읽는 것과 운영 JVM에서 새 recording을 시작하는 것은 다른 권한이다.

그래서 다음을 분리해야 한다.

> Diagnostic Capture ≠ Diagnostic Read

## 11. 좋은 Agent는 무엇을 더 볼지가 아니라 무엇을 아직 안 봐도 되는지 안다

Agent가 자율적이라고 해서 모든 tool을 사용해야 하는 것은 아니다.

좋은 debugging trajectory는 보통 metric에서 서비스를 좁히고, trace에서 operation을 좁히고, log/profile에서 원인 후보를 좁힌 뒤 그래도 구분이 안 될 때 JFR/thread로 내려간다.

반대로 첫 단계에서 heap dump부터 요청한다면 시스템 설계가 잘못된 것이다.

## 12. 세 번째 원칙

> 증거는 종류마다 역할과 비용이 다르다.

그리고:

> 낮은 비용의 증거로 먼저 좁히고, 필요한 경우에만 더 비싼 증거로 escalation한다.

다음 장에서는 이 여러 signal을 하나의 execution으로 연결하는 기반인 OpenTelemetry를 살펴본다.

### 참고 자료

출처 상세: [References](../references.md)

- [S-PROM-API] Prometheus HTTP API
- [S-TEMPO-API] Tempo HTTP API
- [S-PYROSCOPE] Grafana Pyroscope
- [S-SPRING-OBS] Spring Boot 관측 시스템

---

# Part II. 관측 시스템을 Agent의 눈과 귀로 바꾼다

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

# 5장. Prometheus로 문제 공간을 먼저 줄인다

운영 장애를 만났을 때 Agent가 가장 먼저 해야 할 일은 로그를 검색하는 것이 아닐 수 있다.

먼저 범위를 줄여야 한다.

어느 서비스가 문제인지, 언제 시작됐는지, error인지 latency인지, 특정 endpoint인지 전체 시스템인지부터 알아야 한다.

Prometheus는 이 단계에 잘 맞는다.

## 1. Metric은 원인보다 먼저 문제 범위를 알려준다

예를 들어 로그인 지연이 보고됐다.

Agent가 처음부터 login-서비스 로그 10만 줄을 읽는 대신 다음을 확인한다.

~~~text
request rate
error rate
p95/p99 latency
CPU
memory
GC
DB pool
~~~

그 결과 특정 8분 동안 p99만 급증하고 CPU와 GC는 정상이며 Hikari pending이 늘었다고 하자.

이제 어디를 더 봐야 할지가 훨씬 분명해졌다.

Metric은 답을 주지 않았지만 무엇을 다음에 볼지 정해줬다.

## 2. Agent에게 Dashboard를 보여줄 필요는 없다

사람은 Grafana graph를 보는 것이 편하다. Agent는 숫자와 구조화된 조회 result가 더 직접적이다.

Prometheus HTTP API는 instant 조회와 range 조회를 JSON으로 반환한다.

따라서 Agent tool은 다음 정도면 충분할 수 있다.

~~~text
query_metric(query, time)
query_metric_range(query, start, end, step)
get_exemplars(query, start, end)
~~~

중요한 것은 PromQL 문법보다 tool boundary다.

## 3. Broad 조회를 허용하면 안 된다

Agent가 임의 PromQL을 사용할 수 있다고 해서 무제한 range와 series를 읽게 할 필요는 없다.

Tool gateway는 다음을 제한할 수 있다.

- datasource
- environment
- 서비스
- time range
- returned series
- sample count
- 조회 timeout

Agent에게 조회 capability를 주는 것과 관측 시스템 backend 전체를 맡기는 것은 다르다.

## 4. Exemplars가 중요한 이유

Metric은 aggregate다.

~~~text
p99 = 3.1s
~~~

이 값만으로는 어떤 요청가 3초 걸렸는지 알 수 없다.

Exemplar가 trace ID를 가지고 있으면 aggregate anomaly에서 concrete execution으로 이동할 수 있다.

~~~text
Latency Spike
   ↓
Exemplar
   ↓
trace_id
   ↓
Representative Trace
~~~

이 연결이 Agentic Debugging에서 중요하다.

## 5. anomaly를 원인로 부르지 않는다

Prometheus에서 connection pending이 증가했다고 connection pool이 반드시 원인인 것은 아니다.

upstream timeout 때문에 transaction이 길어져 결과적으로 pool이 고갈됐을 수도 있다.

지표 결과는 원인 후보를 세우기 위한 근거다.

~~~text
Evidence
Hikari pending ↑

Hypothesis
connection acquisition bottleneck

Next
slow trace에서 connection acquire span 확인
~~~

이 구분을 지켜야 한다.

## 6. Prometheus의 역할은 다음 조회를 더 좋게 만드는 것이다

좋은 investigation은 metric에서 끝나지 않는다.

~~~text
Prometheus
  ↓
affected service
  ↓
affected operation
  ↓
incident window
  ↓
representative trace
~~~

즉 Prometheus는 Agent가 다음 증거를 더 정확하게 찾게 하는 첫 번째 범위 축소 도구다.

## 7. 사람이라면 그래프를 보고, Agent라면 질문을 남긴다

사람은 Grafana 화면을 보면서 자연스럽게 비교한다.

~~~text
평소보다 늦어졌나?
모든 API가 그런가?
특정 Pod만 그런가?
배포 직후부터인가?
~~~

Agent에게도 비슷한 질문 순서가 필요하다.

예를 들어 첫 질문부터 복잡한 PromQL을 만들게 하기보다:

~~~text
/login p99를 09:00~09:30 범위로 보여줘
~~~

결과가 이상하면:

~~~text
같은 시간 Hikari pending을 보여줘
~~~

그다음:

~~~text
이 시간대에 배포가 있었는지 확인해줘
~~~

처럼 진행한다.

여기서 중요한 것은 PromQL을 얼마나 잘 쓰느냐가 아니다.

**다음 질문의 범위를 얼마나 잘 좁히느냐**가 더 중요하다.

## 8. 기준선이 없으면 숫자를 해석하기 어렵다

CPU 70%가 높을까?

서비스에 따라 다르다.

p99 800ms가 장애일까?

평소 700ms인 서비스라면 아닐 수 있다.

그래서 Agent에게 단일 숫자만 주기보다 평소 값과 비교할 수 있게 하는 편이 낫다.

~~~text
현재 p99 3.1s
지난 7일 같은 시간대 중앙값 210ms
~~~

이렇게 보면 이상 정도를 훨씬 쉽게 판단할 수 있다.

반대로 기준선 없이 threshold만 적용하면 정상적인 트래픽 증가를 장애로 오인할 수 있다.

## 9. Metric 이름을 모두 Agent에게 외우게 할 필요는 없다

실제 시스템에는 metric이 수백, 수천 개 있을 수 있다.

Agent가 처음부터 모든 이름을 알 필요는 없다.

서비스별로 자주 쓰는 지표를 작은 묶음으로 제공할 수 있다.

예를 들어 Spring Boot API라면:

~~~text
HTTP latency / error
JVM heap / GC
Hikari pool
CPU / memory
~~~

정도에서 시작한다.

필요할 때 metric discovery를 확장한다.

이렇게 해야 Agent가 이름 찾기에 시간을 쓰지 않고 문제 판단에 집중할 수 있다.

## 10. 다섯 번째 원칙

> 먼저 지표로 문제 범위를 줄이고, 실제 느린 요청 하나로 내려갈 길을 남긴다.

다음 장에서는 이 concrete execution을 Tempo와 TraceQL로 따라간다.

### 참고 자료

출처 상세: [References](../references.md)

- [S-PROM-API] Prometheus HTTP API
- [S-PROM-EXEMPLAR] Prometheus Exemplars
- [S-GRAFANA-MCP] Grafana MCP

# 6장. Tempo로 실제 실행 경로를 따라간다

Metric에서 login-서비스의 특정 시간대가 문제라는 사실까지 좁혔다.

이제 질문이 바뀐다.

> 실제로 느렸던 요청 하나에서는 어디에서 시간이 쓰였는가?

이 질문에 trace가 답한다.

## 1. 대표적인 느린 요청 하나를 찾는다

모든 trace를 읽을 필요는 없다.

예를 들어 장애 window에서 duration이 2초를 넘고 status가 error이거나 latency가 높은 trace 몇 개를 찾는다.

~~~text
search_traces
service=login-service
operation=POST /login
duration>2s
range=09:10~09:18
limit=5
~~~

이제 대량 관측 데이터가 몇 개의 execution으로 줄었다.

## 2. Trace는 시간의 구조를 보여준다

대표 trace 하나가 다음과 같다고 하자.

~~~text
POST /login              3.1s
 └ authenticate          3.0s
    ├ acquireConnection  2.8s
    └ SELECT user        34ms
~~~

코드만 보거나 SQL timeout 로그만 봤다면 SELECT가 문제라고 생각할 수 있다.

Trace는 시간이 실제로 어디에서 소비됐는지를 보여준다.

## 3. TraceQL은 Agent의 탐색 언어가 될 수 있다

Tempo는 TraceQL로 trace를 검색하고 조건을 좁힐 수 있다.

Agent 입장에서 중요한 것은 TraceQL 전체 문법을 아는 것이 아니라 다음 질문을 표현할 수 있다는 점이다.

- 느린 trace
- error trace
- 특정 서비스를 거친 trace
- 특정 span attribute를 가진 trace
- 특정 duration 이상인 span

## 4. Trace-derived metric으로 다시 넓게 볼 수 있다

한 trace에서 이상을 발견했다고 전체 장애의 원인이라고 단정하면 안 된다.

예를 들어 한 trace에서 payment-서비스가 느렸다면 다음 질문이 필요하다.

> 장애 window의 payment span이 전반적으로 느렸는가?

TraceQL metrics나 trace-derived metrics를 사용하면 span 집합을 다시 aggregate할 수 있다.

즉 debugging은 항상 한 방향 drill-down만 하는 것이 아니다.

~~~text
aggregate
→ trace
→ suspicious span
→ 다시 전체 경향 확인
~~~

이 왕복이 correlation을 causation으로 오해하는 것을 줄인다.

## 5. Trace Diff

정상 trace와 비정상 trace를 비교하는 것도 강력하다.

~~~text
Normal
POST /login 180ms

Slow
POST /login 3100ms
~~~

둘의 span tree를 비교하면 새로 생긴 retry, 누락된 cache hit, 길어진 DB acquire 같은 차이를 찾기 쉽다.

Agent tool에 compare_traces가 유용한 이유다.

## 6. Tempo가 Agent-native interface를 제공하기 시작했다

2026년 Tempo 공식 문서는 Agent용 MCP endpoint와 LLM-oriented 응답을 제공한다.

이 변화는 중요하다.

관측 시스템 backend가 더 이상 사람의 UI만을 위한 저장소가 아니라 machine reasoning client를 직접 고려하기 시작했다는 뜻이다.

하지만 Tempo MCP가 원인을 보장하는 것은 아니다.

Tempo는 증거를 제공한다. 원인 후보와 수정는 다른 책임이다.

## 7. Full trace를 항상 모델에 넣지 않는다

큰 distributed trace는 span이 수백~수천 개일 수 있다.

Agent가 먼저 필요한 것은 다음과 같은 summary일 수 있다.

- critical path
- slowest spans
- error spans
- 서비스 transitions
- repeated spans
- relevant attributes

필요할 때 full trace를 확장한다.

원본 trace는 그대로 보관하되, Agent에게는 먼저 필요한 부분만 보여주는 편이 낫다.

## 8. 정상 요청과 비교하면 더 빨리 보인다

느린 trace 하나만 보면 무엇이 이상한지 감이 안 올 때가 있다.

이럴 때는 같은 API의 정상 trace 하나를 옆에 놓는다.

~~~text
Normal
POST /login 180ms
 └ authenticate 120ms
    ├ acquireConnection 8ms
    └ SELECT user 34ms

Slow
POST /login 3100ms
 └ authenticate 3000ms
    ├ acquireConnection 2800ms
    └ SELECT user 34ms
~~~

두 trace를 비교하면 차이가 거의 설명 자체가 된다.

SQL 시간은 같다.

connection을 얻는 시간만 달라졌다.

Agent에게 trace 비교 기능이 유용한 이유가 여기 있다.

## 9. trace가 끊겨 있으면 그 자체가 단서다

분산 시스템에서는 trace가 완벽하게 이어진다고 가정하면 안 된다.

A 서비스에서는 B를 호출한 흔적이 있는데 B 쪽 span이 보이지 않을 수 있다.

가능성은 여러 가지다.

- 맥락 propagation이 빠졌다.
- sampling 정책이 다르다.
- instrumentation이 누락됐다.
- 다른 trace로 분리됐다.

이때 Agent가 'B 호출은 없었다'고 결론 내리면 안 된다.

trace의 빈 곳도 조사 대상이다.

## 10. trace에서 너무 많은 attribute를 꺼내지 않는다

Span에는 많은 attribute가 들어갈 수 있다.

Agent에게 처음부터 전부 보여주면 오히려 중요한 정보가 묻힌다.

처음에는 다음 정도로 충분할 수 있다.

~~~text
service
operation
duration
status
error
critical path
version
~~~

필요할 때 DB, HTTP, messaging attribute를 더 펼친다.

사람이 trace UI에서 span을 하나씩 눌러보는 것과 비슷하다.

## 11. 여섯 번째 원칙

> 실제로 느렸던 요청 하나를 찾아 어디에서 시간이 쓰였는지 본다.

그리고:

> 한 요청에서 발견한 이상이 전체 장애에서도 반복되는지 다시 확인한다.

다음 장에서는 같은 trace ID를 사용해 Loki에서 실제 사건 로그를 찾아간다.

### 참고 자료

출처 상세: [References](../references.md)

- [S-TEMPO-API] Tempo HTTP API
- [S-TEMPO-METRICS] Tempo TraceQL Metrics
- [S-TEMPO-AI] Grafana Tempo and AI

# 7장. Loki에서 같은 execution의 로그를 찾는다

Trace를 통해 느린 요청와 suspicious span을 찾았다.

이제 로그를 본다.

중요한 점은 로그를 처음 보는 것이 아니라는 것이다.

이미 서비스, time range, trace ID가 있다.

## 1. 로그는 범위를 좁힌 뒤 검색한다

나쁜 조회는 전체 운영 로그에서 timeout을 검색한다.

좋은 조회는 특정 서비스, 특정 시간대, 특정 trace ID를 기준으로 검색한다.

~~~text
service=login-service
range=09:12:10~09:12:14
trace_id=abc
~~~

범위가 좁아질수록 관계없는 로그가 크게 줄어든다.

## 2. Label과 Structured Metadata를 구분한다

Loki에서는 모든 correlation key를 label로 만들면 안 된다.

서비스 이름이나 namespace처럼 cardinality가 낮은 값은 label에 잘 맞는다.

반면 trace_id, request_id처럼 요청마다 바뀌는 값은 cardinality가 매우 높다.

이런 값을 stream label로 만들면 저장과 조회 성능이 나빠질 수 있다.

Grafana Loki는 structured metadata를 통해 이런 high-cardinality field를 다룰 수 있다.

Agent 편의를 위해 backend 구조를 망가뜨리지 않는 것이 중요하다.

## 3. 로그는 event 증거로 본다

Agent에게 원본 log line만 주는 것보다 event 의미를 보존하는 구조가 낫다.

예:
~~~text
event=connection_acquire_timeout
trace_id=abc
pool=primary
active=20
idle=0
pending=37
~~~

이런 structured event는 자연어 parsing 부담을 줄인다.

물론 모든 로그를 구조화할 수는 없다. 그래서 raw message와 structured fields를 함께 유지하는 방식이 현실적이다.

## 4. Stack trace도 처음부터 전부 보여줄 필요는 없다

긴 Java stack trace를 매번 전부 Agent에 줄 필요는 없다.

첫 응답에는 다음이 더 유용할 수 있다.

- exception type
- message
- 원인
- top application frames
- caused-by chain
- trace/span ID

Agent가 더 필요하면 full stack을 요청한다.

필요한 만큼만 먼저 보여주고, 더 필요할 때 펼쳐보는 방식이다.

## 5. Loki 조회에도 예산이 필요하다

Grafana MCP는 Loki 조회에 대해 최대 scan bytes와 최대 effective time range를 제한할 수 있다.

또 모든 조회에 label matcher를 강제로 추가해 운영/staging 또는 특정 서비스 범위를 벗어나지 못하게 할 수 있다.

이 패턴은 매우 중요하다.

~~~text
Agent 조회 제한
= scope
+ time range
+ scan bytes
+ result size
~~~

## 6. 로그가 없다는 사실을 조심해서 해석한다

Agent가 관련 로그를 못 찾았다고 해서 사건이 없었다는 뜻은 아니다.

가능성은 여러 가지다.

- 실제로 없음
- logging level 때문에 없음
- sampling/filtering
- 조회 scope 오류
- trace propagation 깨짐
- retention 만료
- result truncation

따라서 tool result에는 sampled, truncated, retention, 조회 scope 같은 metadata가 필요하다.

## 7. 같은 오류 메시지가 항상 같은 원인은 아니다

운영 로그에서 자주 보는 실수가 있다.

같은 exception message가 보이면 이전 장애와 같은 원인이라고 생각하는 것이다.

예를 들어 connection timeout이라는 문구는 다음 상황에서 모두 나타날 수 있다.

- DB connection pool 고갈
- 네트워크 지연
- DB 장애
- timeout 설정이 너무 짧음

그래서 로그 문구만 보지 않고 같은 trace의 시간 흐름과 지표를 함께 본다.

로그는 강한 단서지만 단독 판결문은 아니다.

## 8. 로그 패턴을 먼저 묶는 것도 도움이 된다

장애 시간대에 같은 warning이 수천 번 반복되면 원문을 모두 볼 필요는 없다.

먼저 패턴별로 묶어서 볼 수 있다.

~~~text
connection timeout       1,240
retry attempt exceeded     380
cache miss                 96
~~~

그다음 비정상적으로 늘어난 패턴의 실제 로그 몇 개를 펼친다.

이 방식은 Agent가 반복 로그에 맥락를 낭비하는 것을 줄인다.

Drain3 같은 log template parser가 이런 전처리의 대표적인 예다.

## 9. 구조화 로그도 사람이 읽을 문장은 남긴다

모든 로그를 key-value만으로 만들면 사람이 보기 불편해질 수 있다.

예를 들어 다음처럼 둘 다 남길 수 있다.

~~~text
event=connection_acquire_timeout
pending=37
message="DB connection을 얻지 못해 요청이 지연되었습니다."
~~~

Agent에게도 숫자와 event key는 유용하고, 개발자에게는 설명 문장이 유용하다.

관측 시스템을 AI만을 위해 다시 설계할 필요는 없다.

## 10. 일곱 번째 원칙

> 로그는 처음부터 전부 읽기보다, 이미 좁혀진 요청에서 무슨 일이 있었는지 확인하는 데 사용한다.

그리고:

> trace ID처럼 매번 달라지는 값은 검색에 필요하지만, 무조건 Loki label로 만들지는 않는다.

다음 장에서는 로그와 trace만으로 부족한 CPU, lock, allocation 문제를 profile과 JVM diagnostics로 내려가 본다.

### 참고 자료

출처 상세: [References](../references.md)

- [S-LOKI-METADATA] Loki Structured Metadata
- [S-LOKI-조회] Loki 조회 Best Practices
- [S-GRAFANA-MCP] Grafana MCP

# 8장. Pyroscope와 JVM diagnostics로 더 깊이 내려간다

Trace는 한 span이 2초 걸렸다는 사실을 알려준다.

하지만 그 2초 동안 CPU를 썼는지, lock에서 기다렸는지, allocation과 GC가 문제였는지는 다른 자료가 더 필요하다.

이때 profile과 JVM diagnostic이 등장한다.

## 1. Profile은 시간의 내부를 보여준다

다음 trace를 보자.

~~~text
POST /report 2.4s
 └ generateReport 2.2s
~~~

generateReport가 느리다는 것은 알았다.

CPU profile에서 특정 JSON serialization 함수가 대부분의 CPU를 사용한다면 수정 방향이 달라진다.

wall profile에서 LockSupport.park가 대부분이라면 CPU optimization은 의미가 없다.

## 2. Trace-to-Profile

Pyroscope는 profile과 trace를 연결할 수 있다.

특정 slow span에서 profile을 조회하면 전체 서비스 profile보다 훨씬 좁은 증거를 얻을 수 있다.

~~~text
slow trace
  ↓
slow span
  ↓
span profile
  ↓
hot method
  ↓
source line
~~~

Agent가 소스코드와 runtime cost를 연결하기 좋은 구조다.

## 3. 그래도 부족하면 JFR로 내려간다

Java Flight Recorder는 JVM runtime event를 기록한다.

예를 들어 다음을 볼 수 있다.

- allocation
- GC
- thread park
- monitor/lock
- socket I/O
- file I/O

하지만 JFR도 원본 파일 전체를 model 맥락에 넣는 대상은 아니다.

장애 window의 event summary나 top contention 같은 projection을 제공하는 편이 낫다.

## 4. Thread dump가 유효한 장애

Thread dump는 특히 다음 문제에서 강하다.

- deadlock
- blocked thread
- executor starvation
- stuck 요청
- synchronized contention

Agent가 보기 좋은 형태는 raw dump 수천 줄보다 다음 요약일 수 있다.

~~~text
RUNNABLE  18
WAITING   74
BLOCKED   12

Top blocked stack group
OrderLock.acquire() 10 threads

Deadlock
none
~~~

필요하면 해당 stack group의 full frames를 확장한다.

## 5. Heap dump는 정말 필요할 때만 쓴다

Memory leak를 의심한다고 바로 heap dump부터 뜨는 것은 좋지 않다.

먼저 heap usage, GC pause, allocation profile, JFR 증거로 범위를 좁힐 수 있다.

heap dump는 크고 민감하며 운영 pause와 저장 비용을 유발할 수 있다.

그래서 가벼운 확인부터 시작해 필요한 경우에만 더 깊은 진단으로 내려가야 한다.

~~~text
metrics
→ profile
→ JFR
→ thread dump
→ heap dump
~~~

실제 순서는 장애에 따라 달라질 수 있지만 비용이 높은 증거를 자동 기본값으로 두지 않는 것이 핵심이다.

## 6. 권한도 단계적으로 올라가야 한다

기존 profile을 읽는 것과 운영 JVM에서 새 JFR을 시작하는 것은 다르다.

그래서 권한을 구분한다.

~~~text
OBSERVE_L0
metrics/logs/traces

OBSERVE_L1
existing profiles

CAPTURE_L2
JFR/thread dump

CAPTURE_L3
heap dump/deep diagnostics
~~~

Capture는 별도의 audit와 승인 정책을 둘 수 있다.

## 7. Agent가 모든 diagnostic을 쓰면 안 되는 이유

자율성은 tool 사용량이 많다는 뜻이 아니다.

좋은 Agent는 다음 질문을 한다.

> 지금 가진 증거로 competing 원인 후보를 구분할 수 있는가?

구분할 수 없다면 다음으로 가장 값싼 증거를 선택한다.

이 원칙은 디버깅 비용뿐 아니라 운영 안전성을 지킨다.

## 8. 어떤 진단을 먼저 선택할까

문제 유형에 따라 다음에 볼 자료가 달라진다.

CPU가 높다면 CPU profile이 먼저다.

CPU는 낮은데 latency가 높다면 wall profile이나 thread state가 더 유용할 수 있다.

memory가 계속 증가한다면 heap 사용량과 allocation profile을 먼저 본다.

~~~text
CPU 높음
→ CPU profile

CPU 낮음 + latency 높음
→ wall profile / thread

memory 증가
→ allocation / GC

deadlock 의심
→ thread dump
~~~

이 정도의 작은 선택 규칙만 있어도 Agent가 불필요한 진단을 많이 줄일 수 있다.

## 9. 진단 자체가 장애를 만들 수 있다는 점을 잊지 않는다

관측 도구는 공짜가 아니다.

상세 JFR 설정이나 heap dump는 CPU, I/O, pause, 저장공간에 영향을 줄 수 있다.

따라서 Agent가 '자료가 더 필요하다'는 이유만으로 무조건 실행하면 안 된다.

다음 질문을 먼저 한다.

> 지금 가진 자료로 원인 후보를 구분할 수 없는가?

구분할 수 없다면 그때 가장 부담이 작은 추가 진단을 선택한다.

## 10. 원본보다 요약을 먼저 본다

Thread dump와 JFR도 로그와 마찬가지다.

Agent에게 원본 수천 줄을 먼저 주지 않는다.

예를 들어 JFR 결과를 이렇게 요약할 수 있다.

~~~text
GC pause: 정상 범위
allocation: OrderDto 변환 구간 급증
lock contention: OrderLock.acquire 집중
socket timeout: 없음
~~~

Agent는 이 요약으로 다음 질문을 선택하고, 필요한 event만 더 자세히 볼 수 있다.

## 11. 여덟 번째 원칙

> 큰 dump 파일을 그대로 Agent의 입력으로 넣지 않는다.

그리고:

> 이미 있는 진단 자료를 읽는 것과 운영 서버에서 새 진단 자료를 만드는 권한은 나눈다.

여기까지가 Agent가 애플리케이션을 관측하기 위한 핵심 signal stack이다.

다음 Part에서는 이 signal들을 실제 Agent Tool로 노출할 때 어떤 interface와 policy가 필요한지 살펴본다.

### 참고 자료

출처 상세: [References](../references.md)

- [S-PYROSCOPE] Grafana Pyroscope
- JVM runtime 증거 research
- [S-TEMPO-AI] Grafana Tempo and AI
