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

출처 상세: [References](#references)

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

출처 상세: [References](#references)

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

출처 상세: [References](#references)

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

출처 상세: [References](#references)

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

출처 상세: [References](#references)

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

출처 상세: [References](#references)

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

출처 상세: [References](#references)

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

출처 상세: [References](#references)

- [S-PYROSCOPE] Grafana Pyroscope
- JVM runtime 증거 research
- [S-TEMPO-AI] Grafana Tempo and AI


---

# Part III. 관측 데이터를 Agent가 직접 조회하게 만든다

# 9장. Grafana MCP는 어디까지 해결해 주는가

지금까지는 사람이 Grafana를 열고 Prometheus, Tempo, Loki, Pyroscope를 따라가며 장애를 찾는 과정을 살펴봤다.

그렇다면 Agent도 같은 데이터를 직접 볼 수 있을까?

이 질문에 가장 현실적인 답 중 하나가 Grafana MCP다.

## 1. 화면을 보여주는 대신 도구를 연결한다

사람은 그래프를 보고 클릭하면서 탐색한다. Agent에게 같은 화면을 보여줄 필요는 없다.

대신 이런 식의 도구를 줄 수 있다.

~~~text
지표 조회
로그 검색
trace 검색
profile 조회
대시보드 요약
~~~

Grafana MCP는 이 역할을 실제로 제공한다.

중요한 점은 'AI 전용 모니터링 시스템'을 새로 만들지 않아도 된다는 것이다.

이미 운영 중인 관측 시스템을 Agent가 사용할 수 있게 연결하면 된다.

## 2. Agent는 사람보다 더 쉽게 과하게 조회할 수 있다

사람은 Grafana에서 대충 10분, 30분 범위를 보고 검색한다.

Agent는 잘못된 지시를 받으면 30일 전체 로그를 검색할 수도 있다.

그래서 Agent에게 조회 기능을 줄 때는 처음부터 제한이 필요하다.

예를 들면:

- 한 번에 볼 수 있는 시간 범위
- 최대 로그 스캔 크기
- 조회 가능한 datasource
- 조회 가능한 environment
- 쓰기 기능 비활성화

Grafana MCP는 실제로 이런 제한을 둘 수 있다.

## 3. Grafana MCP에서 읽기 기능만 남길 수 있다

운영 장애를 조사할 때는 대부분 데이터를 보는 것만으로 시작할 수 있다.

Grafana MCP는 쓰기 기능을 끄고 조회 기능만 남기는 구성이 가능하다.

이 장에서는 이것을 제품이 제공하는 안전장치의 한 예로만 보자.

운영에서 읽기, 진단 자료 생성, 실제 변경 권한을 어떻게 나눌지는 20장에서 자세히 다룬다.

## 4. '조회'라는 이름만 믿지는 않는다

SQL처럼 조회 도구처럼 보여도 backend 권한에 따라 데이터를 바꿀 수 있는 기능이 있다.

따라서 read-only는 tool 이름이 아니라 실제 동작과 credential까지 함께 보고 판단해야 한다.

## 5. Grafana MCP가 해주는 것

Grafana MCP는 우리가 원하는 많은 부분을 이미 해결한다.

- metrics 조회
- logs 검색
- trace 검색
- profile 조회
- datasource 권한 제한
- read-only 구성
- 조회 범위 제한

이 정도만 있어도 Agent는 소스코드 밖의 실제 운영 정보를 훨씬 잘 볼 수 있다.

## 6. 그래도 빠져 있는 것이 있다

하지만 Grafana MCP가 디버깅 전체를 대신해 주지는 않는다.

예를 들어 Agent가 로그를 찾았다고 하자.

그 다음 질문은 여전히 남는다.

- 이 로그가 정말 원인인가?
- 다른 가설은 없는가?
- 지금 보는 운영 버전과 소스코드 버전이 같은가?
- 수정 전에 재현했는가?
- 수정 후 같은 문제가 사라졌는가?

Grafana MCP는 데이터를 가져오는 도구다.

디버깅의 순서를 정하고, 증거를 정리하고, 수정과 검증까지 이어가는 일은 별도 구조가 필요하다.

## 7. 이 책에서 추가하려는 한 겹

그래서 이 책에서는 Grafana MCP 위에 아주 얇은 흐름을 하나 더 둔다.

~~~text
Grafana MCP
  ↓
문제 범위 확인
  ↓
필요한 증거 수집
  ↓
원인 후보 정리
  ↓
수정
  ↓
다시 확인
~~~

이 흐름을 다음 장에서 하나의 디버깅 세션 규칙으로 정리한다.

## 8. 이 장에서 기억할 것

> Grafana MCP는 Agent에게 운영 데이터를 보여주는 좋은 출발점이다.

> 하지만 데이터를 볼 수 있게 하는 것과 디버깅 절차를 만드는 것은 다른 일이다.

### 참고 자료

출처 상세: [References](#references)

- [S-GRAFANA-MCP] Grafana MCP
- [S-TEMPO-AI] Tempo and AI

# 10장. Agent가 디버깅할 때 꼭 기억해야 할 것

사람이 장애를 조사하다 보면 자연스럽게 몇 가지를 메모한다.

언제부터 문제가 있었는지, 어느 서버인지, 어떤 버전이 배포돼 있었는지, 지금까지 무엇을 확인했는지, 어떤 가설이 틀렸는지 같은 것들이다.

Agent도 이 기록이 필요하다.

그렇지 않으면 같은 로그를 다시 찾고, 이미 틀린 가설을 되풀이하고, 다른 버전의 코드를 수정할 수 있다.

## 1. 먼저 문제의 범위를 적는다

디버깅을 시작할 때 최소한 다음은 알아야 한다.

~~~text
어느 환경인가
어느 서비스인가
언제 문제가 있었나
사용자는 무엇을 겪었나
~~~

이 범위가 없으면 Agent는 필요 이상으로 넓게 검색한다.

## 2. 지금 보고 있는 코드가 맞는지 확인한다

운영에서 장애가 난 버전과 현재 저장소의 코드가 다를 수 있다.

그래서 수정 전에 다음을 확인한다.

~~~text
운영 service.version
운영 image digest
배포 revision
현재 git commit
~~~

같지 않다면 먼저 차이를 본다.

이 한 단계만으로도 엉뚱한 수정를 많이 막을 수 있다.

## 3. 무엇을 봤는지 기록한다

Agent가 metric을 보고, trace를 보고, 로그를 봤다면 그 결과를 남겨야 한다.

중요한 것은 원문 전체를 저장하는 것이 아니라 어떤 자료를 보고 어떤 사실을 확인했는지 남기는 것이다.

예:

~~~text
E1
09:10~09:18 login p99 상승

E2
같은 시간 Hikari pending 증가

E3
느린 trace에서 connection acquire 2.8초
~~~

이렇게 두면 나중에 결론이 어디서 나왔는지 다시 확인할 수 있다.

## 4. 사실과 추측을 섞지 않는다

다음 두 문장은 다르다.

~~~text
사실
connection acquire가 2.8초 걸렸다.

추측
connection pool 고갈이 원인일 수 있다.
~~~

Agent는 이 둘을 따로 기록해야 한다.

## 5. 반대 증거도 적는다

사람은 자기가 만든 가설에 맞는 정보만 보고 싶어지는 경향이 있다. Agent도 비슷한 실수를 할 수 있다.

그래서 가설마다 반대 증거를 함께 적는다.

~~~text
가설
DB 서버가 느리다.

지지
API latency 상승

반대
DB CPU 정상
실제 SQL 실행 34ms
~~~

이렇게 하면 첫 가설에 너무 빨리 고정되는 것을 줄일 수 있다.

## 6. 수정 전에 가능하면 재현한다

원인 후보를 찾았다고 바로 코드를 바꾸지 않는다.

가능하면 같은 실패를 다시 만들거나 최소한 같은 증상이 나타나는 조건을 확인한다.

재현이 어렵다면 무엇이 재현되지 않았는지 명시한다.

## 7. 수정 후에는 처음 증상을 다시 본다

테스트가 통과했다고 장애가 해결됐다고 말하면 안 된다.

처음 문제가 p99 latency였다면 수정 후 p99를 다시 본다.

처음 문제가 Pod restart였다면 restart가 멈췄는지 본다.

처음 문제가 DB pool pending이었다면 그 값이 정상으로 돌아왔는지 확인한다.

이것이 가장 간단하면서도 강력한 검증 방법이다.

## 8. 이 흐름에 이름을 붙인다면

이 책에서는 이런 한 번의 조사 기록을 'Agent Debug Session Contract'라고 부르겠다.

이 이름을 외울 필요는 없다.

핵심은 다음이다.

~~~text
문제 범위
→ 실행 버전
→ 확인한 증거
→ 원인 후보
→ 재현
→ 수정
→ 다시 확인
~~~

## 9. 이 장에서 기억할 것

> Agent가 무엇을 생각했는지보다 무엇을 확인했는지가 더 중요하다.

> 수정은 증거와 연결되어야 하고, 검증은 처음 증상으로 돌아가야 한다.

### 참고 자료

출처 상세: [References](#references)

- [S-OPENRCA] OpenRCA
- [S-BTS-AGENTBENCH] BTS-AgentBench

# 11장. Agent에게 주는 도구는 작고 제한적이어야 한다

Agent에게 운영 시스템 접근 권한을 줄 때 가장 쉬운 방법은 shell 하나를 주는 것이다.

curl도 할 수 있고, kubectl도 할 수 있고, SQL도 실행할 수 있다.

유연하다.

하지만 운영에서는 너무 넓다.

## 1. 도구가 넓으면 실수 범위도 넓어진다

예를 들어 Agent가 로그를 보기 위해 shell을 쓴다고 하자.

실수로 파일을 지울 수도 있고, 잘못된 서버에 접속할 수도 있고, 너무 넓은 조회를 실행할 수도 있다.

그래서 운영용 디버깅 도구는 목적을 작게 나누는 편이 낫다.

## 2. 질문 하나에 도구 하나

예를 들면 다음 정도다.

~~~text
지표 보기
느린 trace 찾기
특정 trace 로그 보기
Pod 상태 보기
배포 버전 보기
JFR 요약 보기
~~~

각 도구는 할 수 있는 일이 작다.

대신 감사와 제한이 쉬워진다.

## 3. 조회 범위를 도구가 강제한다

Agent가 매번 '30분만 검색해'라는 지시를 잘 지킬 것이라고 기대하지 않는다.

도구가 직접 제한한다.

~~~text
최대 시간 범위
최대 결과 수
최대 scan 크기
허용 environment
허용 service
~~~

이런 제한은 prompt보다 강하다.

## 4. 결과도 너무 많이 주지 않는다

도구는 결과 전체보다 먼저 요약을 줄 수 있다.

예를 들어 trace 조회 결과는 처음에:

- 가장 느린 span
- error span
- 서비스 이동
- 전체 duration

정도만 주고, 필요할 때 상세 span을 더 본다.

이 방식은 사람의 UI와 비슷하다.

처음부터 모든 세부정보를 펼쳐놓지 않는다.

## 5. JVM 진단 도구는 별도로 다룬다

기존 metric을 읽는 것과 thread dump를 새로 뜨는 것은 다르다.

그래서 JFR, thread dump, heap dump 같은 기능은 일반 조회 도구와 나누는 것이 좋다.

예:

~~~text
일반 조회
metrics / logs / traces

추가 진단
JFR / thread dump

고비용 진단
heap dump
~~~

## 6. Kubernetes도 읽기와 실행을 나눈다

Pod 상태를 보는 것과 Pod 안에 exec로 들어가는 것은 다르다.

기본 Agent에는 get/list 정도만 주고, exec나 restart는 별도 권한으로 둔다.

## 7. Grafana MCP는 좋은 기본 재료다

Prometheus, Loki, Tempo, Pyroscope 쪽은 Grafana MCP가 이미 많은 기능을 제공한다.

따라서 처음부터 새 MCP 서버를 전부 만들 필요는 없다.

필요한 것은 그 위에서 범위와 권한을 더 좁히는 일이다.

## 8. 범용 Python 도구는 어디에 쓸까

OpenRCA는 Python executor를 이용해 관측 데이터를 자유롭게 분석한다.

연구나 offline 분석에서는 매우 유연하다.

하지만 운영에서는 범위가 너무 넓을 수 있다.

그래서 이 책에서는:

~~~text
Production
작은 전용 도구 우선

Offline / Sandbox
필요하면 Python 분석 허용
~~~

정도로 나누는 편을 권한다.

## 9. 이 장에서 기억할 것

> Agent에게 강력한 도구 하나보다 작고 제한된 도구 여러 개를 주는 편이 운영에서는 안전하다.

> 중요한 제한은 prompt가 아니라 도구 자체에서 강제한다.

### 참고 자료

출처 상세: [References](#references)

- [S-GRAFANA-MCP] Grafana MCP
- [S-OPENRCA] OpenRCA

# 12장. 로그를 찾았다고 원인을 찾은 것은 아니다

Agent가 metric을 보고, trace를 보고, 로그까지 찾았다.

이제 원인을 알았다고 말해도 될까?

아직 아니다.

운영 디버깅에서 가장 위험한 순간은 정보가 없을 때가 아니라, 그럴듯한 정보가 하나 보였을 때다.

## 1. 같이 나타났다고 원인은 아니다

다음 두 현상이 같은 시간에 일어났다.

~~~text
API latency 상승
CPU 상승
~~~

CPU가 원인일 수도 있다.

반대로 retry가 폭증하면서 CPU가 같이 올라간 결과일 수도 있다.

그래서 Agent는 '같이 나타남'과 '원인'을 구분해야 한다.

## 2. 한 번에 하나의 가설을 시험한다

예를 들어 로그인 지연 문제에서 다음 가설이 있다고 하자.

~~~text
H1 DB가 느리다
H2 connection pool이 부족하다
H3 외부 인증 API가 느리다
~~~

각 가설을 구분할 수 있는 질문을 만든다.

H1을 보려면 실제 SQL 실행 시간을 본다.

H2를 보려면 connection acquire 시간과 pool pending을 본다.

H3을 보려면 outbound span을 본다.

이렇게 하면 Agent가 막연한 추측을 반복하지 않는다.

## 3. 맞는 증거만 찾지 않는다

H2를 의심한다고 Hikari pending만 계속 찾으면 안 된다.

반대 자료도 본다.

예를 들어 pool pending은 증가했지만 connection acquire는 빠르다면 H2는 약해진다.

가설은 증거가 쌓이면 강해지고, 반대 증거가 나오면 약해져야 한다.

## 4. 정보를 못 찾은 것과 정보가 없는 것은 다르다

이 구분은 매우 중요하다.

Agent가 error log를 못 찾았다.

가능성은 두 가지 이상이다.

~~~text
실제로 error log가 없다
검색 범위가 틀렸다
로그가 잘렸다
sampling됐다
trace 연결이 끊겼다
~~~

그래서 '찾지 못함'을 곧바로 '없음'으로 해석하면 안 된다.

## 5. OpenRCA가 보여준 중요한 점

OpenRCA에서는 관측 데이터를 직접 분석하는 Agent가 반복해서 작은 분석을 수행한다.

핵심은 모델이 한 번에 정답을 말하는 것이 아니다.

~~~text
질문
→ 분석
→ 결과
→ 다음 질문
~~~

이 반복이 실제 디버깅과 닮아 있다.

## 6. 우연히 맞힌 답을 구분해야 한다

Agent가 원인을 맞혔다.

하지만 관련 metric도 trace도 보지 않았다.

이 결과를 성공으로만 기록하면 실험이 왜곡된다.

그래서 평가할 때는 '정답을 맞혔는가'와 '필요한 증거를 확인했는가'를 따로 봐야 한다.

이 책에서는 이런 경우를 편의상 '운 좋게 맞힌 진단'으로 구분한다.

## 7. 좋은 디버깅 기록은 다시 읽을 수 있다

다른 개발자가 Agent의 결과를 보고 다음 질문에 답할 수 있어야 한다.

- 왜 이 원인을 선택했는가?
- 어떤 증거를 봤는가?
- 어떤 다른 가능성을 버렸는가?
- 무엇은 아직 모르는가?
- 수정 후 무엇이 좋아졌는가?

이 질문에 답할 수 없다면 결과가 맞더라도 운영에서 신뢰하기 어렵다.

## 8. Part III를 정리하면

여기까지의 구조는 생각보다 단순하다.

~~~text
운영 데이터
  ↓
Grafana MCP 같은 조회 도구
  ↓
작고 제한된 질문
  ↓
확인한 사실
  ↓
원인 후보
  ↓
재현
  ↓
수정
  ↓
같은 증상 다시 확인
~~~

다음 Part부터는 이 흐름을 Spring Boot 애플리케이션에 실제로 적용한다.

## 9. 이 장에서 기억할 것

> 증거는 사실이고, 원인은 해석이다.

> 찾지 못한 것과 존재하지 않는 것을 구분해야 한다.

### 참고 자료

출처 상세: [References](#references)

- [S-OPENRCA] OpenRCA
- [S-RCA-REALWORLD-2026] Real-world 관측 데이터 RCA
- [S-BTS-AGENTBENCH] BTS-AgentBench

---

# Part IV. Spring Boot 애플리케이션을 실제로 디버깅한다

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

## 3. 서비스.version을 반드시 남긴다

운영 장애에서 가장 위험한 실수 중 하나는 다른 버전의 코드를 고치는 것이다.

그래서 최소한 다음은 남겨야 한다.

~~~text
service.name
service.version
deployment.environment.name
~~~

가능하면 image digest와 배포 revision도 함께 둔다.

## 4. HTTP 호출의 trace가 끊기지 않게 한다

서비스 A가 서비스 B를 호출한다.

trace가 잘 이어지면 한 요청으로 보인다.

~~~text
A /login
  ↓
B /user
~~~

하지만 직접 만든 HTTP client 때문에 맥락 propagation이 빠지면 두 trace가 분리된다.

Agent는 downstream 호출 자체가 없었던 것처럼 오해할 수 있다.

그래서 auto-configured client를 사용하거나 trace propagation을 명시적으로 확인해야 한다.

## 5. DB 호출도 trace에 남긴다

DB 문제를 찾으려면 적어도 다음은 보여야 한다.

- 조회 summary
- duration
- error type
- connection acquire와 조회 execute의 구분

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

요청 body, Authorization header, cookie, SQL parameter 같은 값은 민감할 수 있다.

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

### 참고 자료

출처 상세: [References](#references)

- [S-SPRING-OBS] Spring Boot 관측 시스템
- [S-OTEL-LOGS] OpenTelemetry Logging Specification

# 14장. DB Connection Pool이 바닥났을 때

월요일 오전, 로그인 API가 간헐적으로 3초씩 멈춘다.

로그에는 SQL timeout이 보인다.

첫 느낌은 'DB가 느려졌나?'다.

Agent도 소스코드만 보면 같은 추측을 하기 쉽다.

하지만 실제 원인은 connection pool일 수 있다.

## 1. 먼저 증상을 본다

Prometheus에서 로그인 API p99를 본다.

~~~text
09:08 180ms
09:10 900ms
09:12 3.1s
09:18 220ms
~~~

같은 시간대의 DB CPU는 정상이다.

반면 Hikari pending connection이 올라간다.

~~~text
active 20
idle 0
pending 37
~~~

이제 'DB 자체가 느리다'는 가설보다 'connection을 얻기 어렵다'는 가설이 강해진다.

## 2. 느린 요청 하나를 본다

Tempo에서 느린 trace를 찾는다.

~~~text
POST /login                 3.1s
 └ authenticate             3.0s
    ├ acquireConnection     2.8s
    └ SELECT user           34ms
~~~

여기서 중요한 건 34ms와 2.8초의 차이다.

SQL은 빠르다.

기다린 곳은 connection acquire다.

## 3. 같은 trace의 로그를 확인한다

Loki에서 trace ID를 검색한다.

~~~text
connection acquisition timeout
active=20 idle=0 pending=37
~~~

이제 세 가지 증거가 같은 방향을 가리킨다.

- API 지연
- pool pending 증가
- connection acquire 지연

## 4. 그래도 아직 원인은 하나 더 남아 있다

pool이 부족한 이유는 여러 가지다.

- max pool size가 너무 작음
- transaction이 너무 김
- connection leak
- 외부 호출을 transaction 안에서 오래 기다림

그래서 소스코드를 본다.

예를 들어 transaction 안에서 외부 API를 호출하고 있었다고 하자.

~~~text
transaction 시작
  ↓
DB 조회
  ↓
외부 API 2초 대기
  ↓
transaction 종료
~~~

이 동안 connection이 잡혀 있으면 traffic이 몰릴 때 pool이 빠르게 바닥난다.

## 5. 잘못된 첫 수정

이 상황에서 pool size만 20에서 50으로 늘리면 증상은 잠시 좋아질 수 있다.

하지만 원인이 긴 transaction이라면 병목을 뒤로 미룬 것뿐이다.

그래서 Agent는 '증상 완화'와 '원인 수정'을 구분해야 한다.

## 6. 재현한다

테스트 환경에서 같은 traffic을 만든다.

관찰 항목은 단순하다.

- p99
- active/pending connection
- connection acquire 시간

문제가 재현되면 transaction 범위를 줄이거나 외부 호출을 transaction 밖으로 옮긴다.

## 7. 수정 후 처음 증상을 다시 본다

같은 부하를 다시 건다.

~~~text
Before
p99 3.1s
pending 37
acquire 2.8s

After
p99 240ms
pending 0~2
acquire 8ms
~~~

이제야 장애가 해결됐다고 말할 수 있다.

## 8. Agent는 어떤 순서로 움직였나

~~~text
지연시간 확인
→ pool 지표 확인
→ 느린 trace 확인
→ 같은 trace 로그 확인
→ 소스코드에서 긴 transaction 확인
→ 재현
→ 수정
→ 같은 지표로 재검증
~~~

## 9. 다음 사례에서는 로그가 거의 도움이 되지 않는다

이번 사례는 metric, trace, log가 비교적 잘 맞아떨어졌다.

하지만 모든 장애가 이렇게 친절하지는 않다.

다음 장에서는 오류 로그도 없고 CPU도 높지 않은데 요청만 느린 상황을 본다.

그때는 profile과 thread 상태가 더 중요해진다.

## 10. 이 장에서 기억할 것

> timeout 로그가 보인다고 SQL부터 고치지 않는다.

> 어디에서 기다렸는지 trace로 먼저 구분한다.

### 참고 자료

출처 상세: [References](#references)

- [S-PROM-API] Prometheus HTTP API
- [S-TEMPO-API] Tempo HTTP API
- [S-SPRING-OBS] Spring Boot 관측 시스템


# 15장. Lock Contention은 로그만으로 보이지 않는다

이번에는 API가 느리지만 로그에는 특별한 오류가 없다.

CPU도 높지 않다. DB도 정상이다. 모든 요청은 결국 200으로 끝난다.

이런 문제는 개발자를 더 답답하게 만든다. 실패했다는 흔적이 거의 없기 때문이다.

## 1. 처음에는 DB를 의심하기 쉽다

report API가 평소 300ms 안에 끝나는데 특정 시간대부터 2초가 넘는다.

로그에 예외는 없다. DB 조회도 대부분 20~30ms다.

소스코드만 보면 report 생성 로직이 무거워 보인다. Agent도 처음에는 SQL이나 CPU 사용을 의심할 수 있다.

하지만 Prometheus에서 CPU를 보면 35% 수준이다.

이럴 때는 질문을 바꿔야 한다.

> 계산하느라 느린 것이 아니라 기다리느라 느린 것은 아닐까?

## 2. trace에서 시간이 멈춘 위치를 찾는다

Tempo에서 느린 요청 하나를 본다.

~~~text
POST /report 2.4s
 └ generateReport 2.2s
~~~

문제 위치는 좁혀졌다. 하지만 generateReport 안에서 왜 2.2초가 걸렸는지는 아직 모른다.

여기서 CPU profile만 보면 답이 잘 안 나올 수 있다. 실제로 CPU를 거의 쓰지 않고 기다리고 있기 때문이다.

## 3. wall profile을 보면 기다린 시간이 보인다

Pyroscope wall profile에서 다음 패턴이 크게 보인다고 하자.

~~~text
OrderLock.acquire
  ↓
LockSupport.park
~~~

이제 lock을 기다리는 시간이 길다는 가능성이 생긴다.

중요한 점은 여기서도 바로 결론을 내리지 않는 것이다.

LockSupport.park는 여러 이유로 나타날 수 있다. 그래서 더 구체적인 자료가 필요하다.

## 4. thread dump로 같은 stack이 몰려 있는지 본다

thread dump를 요약해보니 다음과 같다.

~~~text
RUNNABLE 18
WAITING  71
BLOCKED  12

Top blocked stack
OrderLock.acquire() 10 threads

Deadlock
none
~~~

여러 요청 thread가 같은 위치에서 기다리고 있다.

이제 '한 요청이 우연히 늦었다'가 아니라 '여러 요청이 같은 lock 앞에서 줄을 섰다'는 사실을 확인할 수 있다.

## 5. JFR은 시간 흐름을 확인할 때 유용하다

thread dump는 한 순간의 사진에 가깝다.

JFR에서 같은 시간대의 monitor contention이나 thread park event를 보면 문제가 몇 초 동안 반복됐는지 확인할 수 있다.

이 단계까지 와야 lock contention이라는 원인 후보가 충분히 강해진다.

## 6. 소스코드로 돌아간다

이제 소스코드를 본다.

예를 들어 모든 report 생성을 하나의 전역 lock으로 감싸고 있었다고 하자.

~~~text
synchronized(globalLock) {
  generateReport();
}
~~~

한 요청이 report를 만드는 동안 나머지 요청은 기다린다.

traffic이 적을 때는 잘 드러나지 않지만 동시 요청이 늘면 p99가 빠르게 악화된다.

## 7. 흔한 잘못된 수정

이 상황에서 thread pool 크기만 늘리면 어떻게 될까?

기다리는 thread 수만 늘어날 수 있다.

CPU가 남아 있으니 worker를 늘리자는 판단은 자연스럽지만, 병목이 lock이라면 문제를 해결하지 못한다.

또 timeout만 늘리면 사용자는 더 오래 기다리게 된다.

증상을 완화하는 설정 변경과 원인을 고치는 수정은 구분해야 한다.

## 8. 수정 방법은 lock의 목적에 따라 달라진다

전역 lock이 정말 필요한지 먼저 본다.

가능한 수정은 상황마다 다르다.

- lock 범위를 줄인다.
- 오래 걸리는 작업을 lock 밖으로 옮긴다.
- 전역 lock을 key별 lock으로 나눈다.
- 불변 자료를 미리 계산해 공유한다.

중요한 것은 특정 패턴을 외우는 것이 아니라 왜 그 lock이 존재하는지 확인하는 것이다.

## 9. 수정 후 같은 자료를 다시 본다

같은 부하를 다시 건다.

Before:
~~~text
p99 2.4s
BLOCKED 12
OrderLock.acquire 10 threads
~~~

After:
~~~text
p99 320ms
BLOCKED 0~1
OrderLock.acquire hotspot 사라짐
~~~

이제 단위 테스트 통과보다 훨씬 강한 검증이 된다.

## 10. Agent가 따라야 할 순서

~~~text
느린 API 확인
→ DB와 CPU가 정상인지 확인
→ 느린 trace에서 위치 확인
→ wall profile로 대기 여부 확인
→ thread/JFR로 lock 경쟁 확인
→ 소스코드에서 lock 범위 확인
→ 수정
→ 같은 부하로 다시 확인
~~~

## 11. 느린 SQL과 느린 요청은 다른 문제다

여기서는 SQL도 CPU도 큰 문제가 아니었다.

다음 장에서는 반대로 DB 호출이 많이 보이지만 각 SQL은 빠른 상황을 살펴본다.

한 쿼리가 느린 것이 아니라 같은 쿼리가 너무 많이 반복되는 N+1 문제다.

## 이 장의 한 문장

> CPU가 낮은데 느리다면 계산보다 대기를 먼저 의심해볼 수 있다.

> trace로 위치를 찾고 profile과 thread 정보로 이유를 확인한다.

### 참고 자료

출처 상세: [References](#references)

- [S-PYROSCOPE] Grafana Pyroscope
- JVM runtime 증거 research


# 16장. N+1은 느린 SQL 하나가 아니다

목록 API가 데이터가 적을 때는 빠른데, 데이터가 많아질수록 급격히 느려진다.

로그에는 특별한 timeout이 없다.

DB CPU도 크게 치솟지 않는다.

이럴 때 흔한 원인 중 하나가 N+1이다.

## 1. 먼저 한 요청 안에서 SQL이 몇 번 실행됐는지 본다

Tempo에서 느린 trace 하나를 연다.

~~~text
GET /orders 1.8s
 ├─ SELECT orders       22ms
 ├─ SELECT customer      9ms
 ├─ SELECT customer      8ms
 ├─ SELECT customer      9ms
 ├─ SELECT customer      8ms
 └─ ... 반복
~~~

각 SQL 하나만 보면 빠르다.

문제는 같은 종류의 조회가 수십 번, 수백 번 반복된다는 점이다.

## 2. 느린 SQL만 찾으면 놓칠 수 있다

일반적인 slow 조회 분석은 오래 걸린 SQL을 찾는 데 강하다.

하지만 N+1에서는 각각의 조회가 짧다.

그래서 다음 질문이 더 중요하다.

> 한 요청에서 같은 조회가 몇 번 실행됐는가?

## 3. 조회 summary로 묶어 본다

원문 SQL 전체보다 조회 summary를 이용하면 같은 종류의 조회를 묶기 쉽다.

예:
~~~text
SELECT orders        x1
SELECT customer      x120
~~~

이제 문제가 선명해진다.

## 4. 데이터 양과 호출 수를 비교한다

10건을 조회할 때 customer 조회가 10번, 100건을 조회할 때 100번이라면 관계가 거의 그대로 드러난다.

~~~text
rows=10   → child query 10
rows=50   → child query 50
rows=100  → child query 100
~~~

이런 패턴은 단일 SQL latency보다 훨씬 강한 증거다.

## 5. 소스코드에서 반복 접근을 찾는다

Repository 하나만 보는 것이 아니라 loop 안에서 lazy relation이나 추가 조회가 발생하는지 본다.

예를 들어 각 Order를 순회하면서 customer를 따로 조회하고 있을 수 있다.

## 6. SQL parameter는 굳이 보여줄 필요가 없다

이 문제를 찾는 데 customer ID 실제 값은 필요하지 않다.

조회 종류와 호출 횟수만으로도 충분하다.

민감한 parameter를 Agent에게 넘기지 않아도 디버깅할 수 있다는 좋은 예다.

## 7. 수정 후 무엇을 확인할까

fetch join, batch fetch, bulk 조회 등으로 수정한 뒤 같은 요청을 다시 실행한다.

비교할 것은 다음이다.

~~~text
Before
DB spans = 121
latency = 1.8s

After
DB spans = 2
latency = 180ms
~~~

## 8. 다음부터는 코드 밖도 본다

14~16장은 애플리케이션 코드 안에서 원인을 찾을 수 있는 사례였다.

하지만 운영 장애는 JVM이나 Kubernetes, 배포 상태에서 시작될 수도 있다.

다음 장에서는 애플리케이션 로그보다 Pod 상태가 더 중요한 경우를 본다.

## 9. 이 장에서 기억할 것

> N+1은 '느린 SQL' 문제가 아니라 '너무 많은 SQL' 문제다.

> Agent가 조회 시간뿐 아니라 한 요청 안의 반복 횟수를 볼 수 있어야 한다.

### 참고 자료

출처 상세: [References](#references)

- SQL/database 증거 research
- [S-TEMPO-API] Tempo HTTP API


# 17장. 장애가 코드가 아닐 때

API가 간헐적으로 500을 반환한다.

애플리케이션 로그를 뒤져도 결정적인 예외가 없다. 그런데 Pod restart count는 계속 올라간다.

이럴 때 코드만 붙잡고 있으면 문제를 놓칠 수 있다.

## 1. 먼저 '애플리케이션이 죽은 것'인지 확인한다

해당 workload의 Pod 상태를 본다.

~~~text
restartCount = 6
lastState.reason = OOMKilled
~~~

이 한 줄만으로도 조사 방향이 크게 바뀐다.

이제 NullPointerException보다 memory를 먼저 봐야 한다.

## 2. Event는 힌트다

같은 시간대의 Kubernetes Event를 본다.

~~~text
Reason: OOMKilling
Action: Killing
~~~

하지만 Event 하나로 결론을 내리지는 않는다.

Event는 보조 자료다. retention도 제한적이고 message도 완전한 원인 설명이 아니다.

Pod 상태와 memory metric을 같이 본다.

## 3. OOMKilled와 eviction은 다르다

둘 다 Pod가 사라지거나 재시작될 수 있지만 조사 방향은 다르다.

### 컨테이너 OOM

애플리케이션 container가 memory limit을 넘어서 죽는다.

확인할 것:
- container memory usage
- memory limit
- JVM heap/native memory
- allocation profile

### Node pressure에 의한 eviction

애플리케이션 자체는 limit 안에 있어도 Node 전체 memory가 부족해 Pod가 쫓겨날 수 있다.

확인할 것:
- Node memory pressure
- eviction event
- 다른 workload의 사용량

이 둘을 구분하지 않으면 Java heap만 며칠 동안 분석할 수 있다.

## 4. 시간 순서를 맞춘다

Prometheus와 Kubernetes 상태를 같은 시간축으로 놓는다.

예를 들어:
~~~text
09:11 memory 70%
09:12 memory 88%
09:13 limit 근접
09:13:20 OOMKilled
09:13:25 Pod restart
09:13~09:14 5xx 증가
~~~

이렇게 보면 5xx가 코드 예외 때문에 시작된 것인지 Pod 재시작의 결과인지 구분하기 쉬워진다.

## 5. 그다음에야 애플리케이션 내부를 본다

컨테이너 OOM이 맞다면 원인은 여전히 여러 가지다.

- memory leak
- 한 번에 너무 큰 데이터 처리
- 무제한 cache
- 최근 배포의 allocation 증가
- heap/container limit 설정 불일치

이제 JVM metric, allocation profile, 최근 source diff를 본다.

## 6. 배포 직후라면 버전도 함께 본다

revision 41까지 정상이고 revision 42부터 memory가 증가했다면 최근 변경과 연결할 수 있다.

~~~text
revision 41
memory peak 55%

revision 42
memory peak 95%
OOMKilled
~~~

이때 image digest와 source commit까지 맞추면 어떤 변경을 봐야 하는지 훨씬 명확해진다.

## 7. Agent에게 Kubernetes 권한을 어디까지 줄까

대부분의 조사에는 읽기 권한이면 충분하다.

~~~text
Pod status 조회
Event 조회
Deployment 조회
ReplicaSet/revision 조회
Node pressure 조회
~~~

Pod 안에 exec로 들어가거나 restart, scale, delete를 하는 권한은 별도로 둔다.

읽기와 조치를 한 묶음으로 주지 않는다.

## 8. 수정 후에는 애플리케이션과 플랫폼을 같이 확인한다

memory regression을 고쳤다면 같은 부하에서 다음을 본다.

- memory peak
- GC
- Pod restart count
- OOM/eviction Event
- API error rate

memory만 정상이라고 끝내지 않는다. 사용자가 겪었던 5xx도 사라졌는지 확인한다.

## 9. Agent가 따라야 할 순서

~~~text
5xx 증가
→ Pod restart 확인
→ 종료 이유 확인
→ Event와 resource metric 교차 확인
→ OOM인지 eviction인지 구분
→ runtime/source version 확인
→ JVM/profile 또는 Node 상태 분석
→ 수정
→ restart + error rate 재검증
~~~

## 10. 마지막으로 '어떤 코드가 실행 중이었는가'를 확인한다

Pod와 Node 상태까지 봤다면 한 가지 질문이 더 남는다.

이 장애가 어느 배포 버전에서 발생했는가?

다음 장에서는 운영 trace의 버전과 현재 저장소의 코드가 다를 때 어떤 실수가 생기는지 살펴본다.

## 이 장의 한 문장

> 애플리케이션 장애가 항상 애플리케이션 코드에서 시작되는 것은 아니다.

> Pod와 Node 상태도 코드와 같은 수준의 디버깅 자료로 봐야 한다.

### 참고 자료

출처 상세: [References](#references)

- Kubernetes runtime 증거 research
- [S-OTEL-SERVICE] OpenTelemetry Service semantic conventions
- [S-OTEL-K8S] OpenTelemetry Kubernetes resource mapping


# 18장. 이미 다른 버전이 운영 중이라면

운영에서 NullPointerException이 발생했다.

stack trace에는 UserService.java 142번째 줄이라고 나온다.

Agent가 현재 main branch를 열어 142번째 줄을 본다.

그런데 그 줄에는 문제가 없다.

이런 상황은 생각보다 쉽게 생긴다.

운영 코드와 현재 저장소 코드가 다르기 때문이다.

## 1. line number를 믿기 전에 버전을 확인한다

먼저 장애 trace나 로그에서 서비스.version을 본다.

~~~text
service.version = a81c92f
~~~

현재 workspace는:
~~~text
git commit = b115e91
~~~

다르다.

이제 142번째 줄을 그대로 비교하면 안 된다.

## 2. image digest도 확인한다

tag는 바뀔 수 있다.

~~~text
myapp:latest
~~~

같은 이름이라도 다른 image일 수 있다.

가능하면 digest처럼 바뀌지 않는 식별자를 사용한다.

~~~text
sha256:ab34...
~~~

## 3. 배포 revision을 같이 본다

Kubernetes에서는 배포 revision을 통해 어느 rollout에서 문제가 시작됐는지 확인할 수 있다.

~~~text
revision 41 정상
revision 42 error rate 증가
~~~

이제 recent source diff와 연결하기 쉽다.

## 4. Agent가 해야 할 첫 행동이 바뀐다

버전이 다르면 바로 수정를 만들지 않는다.

먼저 다음 중 하나를 한다.

- 해당 commit checkout
- 해당 tag/branch 찾기
- 두 버전 diff 확인
- source artifact를 찾지 못하면 불확실하다고 표시

## 5. 오래된 장애도 있다

운영에서 이미 새 버전이 배포돼 문제가 사라졌는데 과거 장애를 분석하고 있을 수도 있다.

이때 현재 관측 데이터와 과거 로그를 섞으면 잘못된 결론이 나온다.

시간과 버전을 함께 봐야 한다.

## 6. 설정 버전도 중요하다

코드는 같아도 설정이 다르면 동작이 달라질 수 있다.

예:
- connection pool size
- feature flag
- timeout
- retry count

그래서 가능하면 config version이나 배포 configuration diff도 함께 확인한다.

## 7. 수정 후 배포 버전까지 확인한다

수정가 만들어졌다면 실제로 그 수정가 들어간 image가 배포됐는지 확인해야 한다.

테스트 결과와 운영 결과 사이에도 version 연결이 필요하다.

## 8. 이 장에서 기억할 것

> 운영 장애를 고치기 전에 지금 보고 있는 코드가 실제 운영 코드인지 먼저 확인한다.

> tag보다 commit과 image digest처럼 바뀌지 않는 식별자가 더 믿을 만하다.

### 참고 자료

출처 상세: [References](#references)

- [S-OTEL-SERVICE] OpenTelemetry Service semantic conventions
- [S-OTEL-K8S] OpenTelemetry Kubernetes resource mapping
- [S-OTEL-SERVICE] OpenTelemetry Service semantic conventions
- [S-OTEL-K8S] OpenTelemetry Kubernetes resource mapping



---

# Part V. 운영 환경에서 안전하게 연결한다

# 19장. 운영 관측 데이터를 Agent에게 열어도 되는가

지금까지는 Agent가 Prometheus, Loki, Tempo, Pyroscope를 직접 조회하는 흐름을 만들었다.

여기서 자연스럽게 다음 질문이 나온다.

> 운영 데이터를 Agent에게 직접 보여줘도 괜찮을까?

기술적으로 가능하다는 것과 운영에서 허용해도 된다는 것은 다른 문제다.

## 1. 가장 작은 권한에서 시작한다

처음부터 운영 전체를 보여줄 필요는 없다.

예를 들어 login-서비스 장애만 조사한다면 Agent에게 필요한 것은 다음 정도다.

~~~text
environment = production
service = login-service
time = 09:10~09:20
~~~

이 범위를 벗어나는 조회는 gateway가 막을 수 있다.

## 2. 기본은 읽기 전용이다

첫 단계에서 필요한 것은 대부분 이런 정보다.

- 지표
- 로그
- trace
- profile
- Pod 상태
- 배포 버전

이 단계에서는 read-only 권한으로 충분하다.

대시보드 수정, alert 변경, restart, 배포 같은 기능은 따로 둔다.

## 3. Agent 전용 계정을 쓴다

사람 계정을 공유하지 않는다.

Agent용 서비스 account를 따로 만들고 필요한 datasource와 environment만 허용한다.

이렇게 하면 Agent가 실수해도 영향 범위를 줄일 수 있다.

## 4. 로그에 있는 민감정보는 생각보다 많다

운영 로그에는 다음 값이 들어갈 수 있다.

- Authorization header
- cookie
- 사용자 식별자
- 요청 body
- SQL parameter
- 내부 URL

이런 값을 prompt 직전에만 가리는 것으로 충분하지 않을 수 있다.

가능하면 OpenTelemetry Collector 같은 수집 단계에서부터 제거하거나 mask한다.

## 5. 외부 모델을 쓰면 데이터가 어디로 가는지 다시 본다

사내 Agent가 외부 SaaS LLM을 호출한다면, Loki에서 가져온 로그 일부가 외부 provider로 전송될 수 있다.

따라서 다음 질문이 필요하다.

- 어떤 데이터 등급까지 외부 전송이 허용되는가?
- 원문 대신 요약만 보낼 수 있는가?
- tenant/user 식별자를 제거했는가?
- trace 자체에 payload가 들어 있지 않은가?

도구 연결이 가능하다고 해서 데이터 반출이 자동으로 허용되는 것은 아니다.

## 6. 멀티테넌트라면 tenant 경계를 조회에 강제한다

한 고객의 장애를 조사하는 Agent가 다른 고객 로그를 읽어서는 안 된다.

좋은 구조는 Agent가 tenant 조건을 기억하기를 기대하지 않는다.

gateway가 모든 조회에 tenant matcher를 붙인다.

~~~text
사용자 질문
  ↓
Agent query
  ↓
강제 조건
tenant=A, service=login-service
  ↓
Loki/Tempo/Prometheus
~~~

## 7. 조회 비용도 운영 비용이다

Agent가 반복해서 넓은 로그 검색을 하면 관측 시스템 자체가 느려질 수 있다.

따라서 다음을 제한한다.

- 최대 시간 범위
- 최대 scan 크기
- 최대 결과 수
- 조회 timeout
- 호출 빈도

실제 Grafana MCP의 Loki guardrail이 좋은 참고 사례다.

## 8. 결과 크기도 제한한다

조회는 작아도 결과가 매우 클 수 있다.

그래서 처음에는 요약과 상위 몇 개만 반환하고, Agent가 필요할 때 더 요청하게 할 수 있다.

중요한 것은 결과가 잘렸다면 반드시 표시하는 것이다.

## 9. 감사 기록은 나중에 필요해진다

Agent가 어떤 조회를 실행했는지 남겨야 한다.

장애가 끝난 뒤 다음을 확인할 수 있어야 한다.

- 어떤 데이터를 봤는가
- 어느 시간대를 조회했는가
- 어떤 범위를 벗어나려 했는가
- 어떤 결과가 잘렸는가
- 민감정보 차단이 동작했는가

## 10. 작은 운영 예시

login-서비스만 조사하는 Agent profile을 생각해보자.

~~~text
허용
- production login-service metrics
- production login-service logs
- 관련 Tempo trace
- 해당 deployment status

차단
- 다른 서비스 로그
- 24시간 초과 query
- raw SQL 실행
- Pod exec
- deployment 변경
~~~

이 정도만 해도 실제 debugging의 대부분을 시작할 수 있다.

## 이 장의 한 문장

> 운영 관측 권한은 처음부터 최소한으로 준다.

> 읽기 권한, 데이터 범위, 조회 비용, 민감정보를 각각 따로 통제한다.

### 참고 자료

출처 상세: [References](../references.md)

- [S-GRAFANA-MCP] Grafana MCP
- 관측 데이터 governance research


# 20장. 원인을 찾은 뒤부터는 권한이 달라진다

Agent가 원인 후보를 찾았다.

이제 코드를 고치면 된다.

여기서부터는 상황이 달라진다.

운영 데이터를 보는 것과 실제 시스템을 바꾸는 것은 위험 수준이 다르기 때문이다.

## 1. 조사 단계에서는 대부분 읽기만 해도 된다

예를 들어 connection pool 문제를 조사하는 동안 Agent는 다음만 해도 충분하다.

- Prometheus 지표 조회
- Tempo trace 조회
- Loki 로그 조회
- 배포 버전 확인

이 단계에서는 운영을 바꿀 이유가 없다.

## 2. 진단 자료를 새로 만드는 순간 권한이 한 단계 올라간다

기존 profile을 읽는 것과 운영 JVM에서 새 thread dump나 JFR을 뜨는 것은 다르다.

heap dump는 더 무겁다.

그래서 단순 조회와 진단 자료 생성도 분리한다.

~~~text
읽기
metrics / logs / traces / 기존 profile

추가 진단
thread dump / JFR

고비용 진단
heap dump / deep DB diagnostics
~~~

Agent가 필요하다고 판단했다고 자동 실행할 필요는 없다.

운영 영향과 민감정보 위험에 따라 승인 단계를 둘 수 있다.

## 3. 로컬 수정과 운영 수정은 완전히 다른 권한이다

Agent가 로컬 repository를 수정하고 테스트를 돌리는 것은 비교적 안전하다.

하지만 운영 배포를 바꾸는 것은 다르다.

그래서 작업을 다음처럼 나누는 편이 낫다.

~~~text
운영 데이터 읽기
  ↓
로컬 코드 수정
  ↓
테스트 실행
  ↓
staging 반영
  ↓
production 반영
~~~

각 단계는 같은 버튼의 강도 차이가 아니라 서로 다른 권한으로 본다.

## 4. 실제로는 원인보다 '급한 완화'가 먼저 필요할 때도 있다

예를 들어 connection pool이 고갈돼 서비스가 거의 멈췄다.

근본 원인은 긴 transaction이지만 수정과 배포에는 시간이 필요하다.

운영자는 임시로 traffic을 줄이거나 인스턴스를 늘리거나 문제 기능을 끌 수 있다.

이런 조치는 원인 fix가 아니다.

그래서 Agent 기록에도 다음을 구분하는 편이 좋다.

~~~text
Mitigation
지금 장애를 줄이는 조치

Fix
원인을 없애는 수정
~~~

임시 조치가 성공했다고 문제 해결로 기록하면 다음 장애 때 같은 문제가 반복된다.

## 5. 운영 변경은 되돌릴 수 있어야 한다

Agent가 수정안을 만들고 staging에서 검증했다고 하자.

그래도 운영에서는 예상하지 못한 문제가 생길 수 있다.

그래서 운영 action에는 최소한 다음이 필요하다.

- 변경 전 버전
- 변경 후 버전
- rollback 경로
- 배포 후 확인할 신호
- 중단 조건

예를 들어 error rate가 일정 수준을 넘으면 자동 rollback하는 식이다.

## 6. 승인도 위험한 작업에 집중한다

모든 Prometheus 조회마다 사람이 승인해야 한다면 시스템을 쓸 수 없다.

반대로 운영 restart나 배포 변경을 무조건 자동 허용하기도 어렵다.

작업을 위험도에 따라 나눈다.

~~~text
낮음
metrics/logs/traces 조회

중간
JFR/thread dump 생성
staging 재시작

높음
production restart
configuration 변경
deployment
~~~

승인은 높은 위험 작업에 집중한다.

## 7. Agent가 만든 수정도 변경 이유가 보여야 한다

단순히 diff만 남기지 않는다.

좋은 수정 기록에는 다음 연결이 있어야 한다.

~~~text
확인한 증거
→ 선택한 원인 후보
→ 바꾼 코드
→ 기대하는 변화
~~~

예:
~~~text
connection acquire 2.8s
+ 긴 transaction 확인
→ 외부 API 호출을 transaction 밖으로 이동
→ pending connection 감소 기대
~~~

이렇게 해야 코드 리뷰도 쉬워진다.

## 8. 작은 자동화부터 시작한다

처음부터 운영 auto-remediation을 목표로 할 필요는 없다.

현실적인 발전 순서는 다음과 같다.

~~~text
1. 원인 후보와 근거 제시
2. patch 제안
3. 로컬 테스트 자동화
4. staging 검증 자동화
5. production 변경은 승인 후 실행
~~~

운영 경험이 쌓인 일부 안전한 작업만 나중에 자동화 범위를 넓힐 수 있다.

## 이 장의 한 문장

> 보는 권한과 바꾸는 권한을 분리한다.

> Agent의 자율성은 운영 변경 권한을 많이 주는 것으로 측정하지 않는다.

### 참고 자료

출처 상세: [References](../references.md)

- [S-GRAFANA-MCP] Grafana MCP
- JVM runtime 증거 research


# 21장. 수정했다고 끝난 것이 아니다

Agent가 코드를 수정했다.

테스트도 통과했다.

문제가 해결됐다고 말해도 될까?

아직 한 단계가 남았다.

처음 장애를 확인했던 신호가 실제로 좋아졌는지 봐야 한다.

## 1. 처음 증상으로 돌아간다

DB pool 장애였다면 처음 본 것은 p99와 pending connection이었다.

수정 후 같은 부하에서 다시 본다.

~~~text
Before
p99 3.1s
pending 37
connection acquire 2.8s

After
p99 240ms
pending 1
connection acquire 8ms
~~~

이 비교가 있어야 실제 장애 해결을 말할 수 있다.

## 2. 테스트 통과와 운영 해결은 다르다

단위 테스트는 로직이 맞는지 확인한다.

통합 테스트는 여러 컴포넌트가 함께 동작하는지 확인한다.

하지만 운영 장애는 latency, concurrency, memory, thread, platform 상태 때문에 생길 수 있다.

따라서 test pass만으로는 부족하다.

## 3. 수정한 코드가 실제로 배포됐는지도 확인한다

의외로 자주 빠지는 단계다.

Agent가 commit B에서 수정했고 CI도 통과했다.

그런데 운영은 여전히 commit A image를 실행하고 있을 수 있다.

그래서 검증 전에 다음을 확인한다.

~~~text
expected version = B
deployed version = B
image digest = expected digest
~~~

버전 확인이 없으면 좋은 결과가 우연히 다른 배포 때문인지 구분하기 어렵다.

## 4. 가능하면 같은 workload로 다시 확인한다

수정 전에는 100 concurrent users였는데 수정 후에는 5명으로 시험했다면 비교가 어렵다.

가능한 한 같은 조건을 맞춘다.

- 입력 데이터
- 요청 수
- 동시성
- timeout
- 환경 설정

완전히 같게 만들 수 없다면 차이를 기록한다.

## 5. 하나의 숫자만 좋아졌다고 끝내지 않는다

예를 들어 latency를 줄이기 위해 cache를 추가했다.

p99는 좋아졌지만 stale data 오류가 늘 수 있다.

또 timeout을 줄여 latency는 좋아 보이지만 error rate가 올라갈 수 있다.

그래서 기능 결과와 장애 신호를 같이 본다.

예:
~~~text
latency 개선
+ error rate 유지
+ 데이터 정확성 유지
+ resource 사용량 허용 범위
~~~

## 6. 부분 해결도 있다

수정 후 다음처럼 나올 수 있다.

~~~text
p99 3.1s → 700ms
pending 37 → 5
error rate 정상
~~~

좋아졌지만 목표가 300ms라면 완전한 해결은 아니다.

또 이런 경우도 있다.

~~~text
p99 정상
하지만 GC pause 여전히 높음
~~~

이 경우 다른 문제가 남았을 수 있다.

Agent는 성공/실패 둘 중 하나만 선택하기보다 부분 해결을 기록할 수 있어야 한다.

## 7. 잘못된 수정은 before/after에서 드러난다

connection pool 문제에서 pool size만 크게 늘렸다고 하자.

초기에는 p99가 좋아질 수 있다.

하지만 traffic을 더 올리면 pending이 다시 증가한다.

근본 원인이 긴 transaction이라면 병목을 뒤로 미룬 것뿐이다.

그래서 검증은 한 번의 happy path가 아니라 원래 문제가 나타나던 조건에서 해야 한다.

## 8. 검증 결과도 증거로 남긴다

수정 전후 자료는 코드 리뷰와 장애 회고에 유용하다.

예:
~~~text
Fix
외부 API 호출을 transaction 밖으로 이동

Before
p99 3.1s / pending 37

After
p99 240ms / pending 1

Regression
기존 login 기능 테스트 통과
~~~

이 정도만 남아도 왜 이 수정가 효과가 있었는지 나중에 다시 확인할 수 있다.

## 9. 다음 장애를 위한 규칙으로 바로 고정하지 않는다

이번에 같은 증상이 connection pool 문제였다고 다음번에도 그렇다고 단정하면 안 된다.

환경과 버전, 원인이 달라질 수 있다.

과거 장애는 참고자료이지 현재의 사실이 아니다.

## 10. Agent가 따라야 할 검증 순서

~~~text
patch
→ 테스트
→ 올바른 버전 배포 확인
→ 같은 workload
→ 처음 증상 재확인
→ 부작용 확인
→ 잔여 이상 기록
~~~

## 이 장의 한 문장

> 처음 문제를 발견한 신호를 수정 후 다시 본다.

> 테스트 통과와 장애 해결은 같은 말이 아니다.

### 참고 자료

출처 상세: [References](../references.md)

- [S-OPENRCA] OpenRCA
- [S-BTS-AGENTBENCH] BTS-AgentBench


# 22장. Agentic Debugging을 어떻게 평가할까

Agent가 디버깅을 잘한다고 말하려면 무엇을 측정해야 할까?

최종 정답률만 보면 부족하다.

우연히 맞힐 수도 있고, 너무 많은 로그를 읽고 비싼 조회를 남발할 수도 있기 때문이다.

## 1. 네 가지 조건을 비교한다

이 책에서 제안한 실험은 단순하다.

~~~text
A. 소스코드만 제공

B. 소스 + raw log bundle

C. 소스 + Grafana/Tempo 조회 도구

D. C + 디버깅 순서와 기록 규칙
~~~

같은 Agent와 같은 장애에서 차이를 본다.

## 2. 원인을 맞혔는가

가장 기본적인 지표다.

- 원인 component
- 원인 reason

둘을 나눠 볼 수 있다.

## 3. 제대로 고쳤는가

원인을 맞혀도 수정가 틀릴 수 있다.

그래서 다음도 본다.

- correct 수정
- reproduction success
- regression test
- 장애 signal 개선

## 4. 필요한 증거를 실제로 봤는가

Agent가 connection pool 문제를 맞혔지만 pool metric도 trace도 보지 않았다면 우연일 수 있다.

그래서 각 scenario마다 최소한 봐야 할 증거를 정해둘 수 있다.

## 5. 얼마나 많이 읽었는가

같은 정답이라면 적은 조회와 적은 token으로 찾는 쪽이 운영에서 더 낫다.

측정 후보:
- token
- 조회 수
- 반환 데이터 크기
- scan bytes
- 진단 단계 수

## 6. 안전하게 조회했는가

정답을 맞혔더라도 운영 전체를 무제한 검색했다면 좋은 시스템이 아니다.

그래서 다음도 기록한다.

- 범위 밖 조회
- 너무 넓은 시간 범위
- 민감정보 노출
- 불필요한 heap/JFR capture
- mutation attempt

## 7. 실패 이유를 분류한다

점수만 보면 어디를 개선해야 할지 모른다.

실패를 다음처럼 나눌 수 있다.

~~~text
필요한 자료를 못 찾음
자료는 찾았지만 연결 못 함
자료를 잘못 해석함
잘못된 버전의 코드 봄
도구를 잘못 사용함
검증 부족
~~~

이 분류가 다음 시스템 개선으로 이어진다.

## 8. 가상의 비교 결과로 보면 더 쉽다

아래 숫자는 실제 실험 결과가 아니라 평가 방법을 설명하기 위한 예시다.

예를 들어 같은 connection pool 장애를 네 조건에서 10회씩 실행했다고 하자.

| 조건 | 원인 진단 | 올바른 수정 | 같은 신호로 재검증 | 평균 조회 수 |
|---|---:|---:|---:|---:|
| A. 소스만 | 3/10 | 2/10 | 1/10 | 0 |
| B. 소스 + raw logs | 5/10 | 3/10 | 1/10 | 0 |
| C. 관측 조회 도구 | 8/10 | 7/10 | 5/10 | 12 |
| D. 조회 도구 + 조사 순서 | 8/10 | 7/10 | 8/10 | 7 |

이런 결과가 나왔다면 단순히 D가 최고라고 끝내지 않는다.

C와 D의 원인 진단률이 같다면 조사 순서를 강제한 효과는 진단 정확도보다 **검증률과 조회 효율**에서 나타났다고 해석할 수 있다.

반대로 D가 조회 수는 줄였지만 정답률도 낮아졌다면 제한이 지나쳤을 수 있다.

평가는 설계를 칭찬하기 위한 숫자가 아니라 어디가 실제로 도움이 됐는지 찾는 도구다.

## 9. 한 번의 성공을 믿지 않는다

Agent 실행은 변동성이 있다.

그래서 같은 scenario를 여러 번 실행하고 단일 성공률과 반복 신뢰성을 함께 본다.

## 10. 이 장에서 기억할 것

> Agentic Debugging 평가는 정답뿐 아니라 과정, 비용, 안전성, 검증까지 함께 봐야 한다.

> 좋은 디버깅 시스템은 왜 맞았는지를 다시 설명할 수 있어야 한다.

### 참고 자료

출처 상세: [References](../references.md)

- [S-OPENRCA] OpenRCA
- [S-BTS-AGENTBENCH] BTS-AgentBench
- [S-RCA-REALWORLD-2026] Real-world 관측 데이터 RCA


---

# 맺으며 — 더 많은 로그보다 더 좋은 관측 인터페이스

이 책은 AI에게 로그를 잘 읽히는 방법에서 시작했다.

하지만 끝까지 따라오면 질문이 조금 달라진다.

문제는 로그 형식이 아니었다.

문제는 실행 중인 애플리케이션과 Coding Agent 사이에 어떤 연결을 만들 것인가였다.

사람은 이미 오래전부터 이런 방식으로 디버깅해 왔다.

지표를 보고 범위를 줄인다.

느린 요청 하나를 찾는다.

같은 요청의 로그를 본다.

필요하면 profile과 JVM 상태를 본다.

배포 버전을 확인한다.

가설을 세우고 틀린 가설을 버린다.

수정한 뒤 처음 증상을 다시 확인한다.

Agent에게 필요한 것도 크게 다르지 않다.

~~~text
소스코드
  +
실행 중에 남은 증거
  +
안전하게 조회할 수 있는 작은 도구
  +
수정 전후를 확인하는 습관
~~~

Prometheus, Loki, Tempo, Pyroscope, OpenTelemetry, Grafana는 이 문제를 해결하기 위한 좋은 재료다.

새로운 AI 전용 모니터링 시스템을 처음부터 만들 필요는 없다.

이미 있는 관측 시스템을 Agent가 이해할 수 있는 방식으로 연결하면 된다.

그리고 가장 중요한 것은 연결 뒤의 규칙이다.

- 너무 많이 보지 않는다.
- 같은 요청의 정보를 묶는다.
- 사실과 추측을 나눈다.
- 운영 버전을 확인한다.
- 보는 권한과 바꾸는 권한을 나눈다.
- 수정 후 처음 증상을 다시 본다.

결국 핵심은 단순하다.

> Agent에게 더 많은 로그를 주는 것이 아니라, 필요한 증거를 스스로 찾고 확인할 수 있는 길을 만들어야 한다.

소스코드는 프로그램이 어떻게 만들어졌는지를 보여준다.

관측 데이터는 프로그램이 실제로 어떻게 움직였는지를 보여준다.

두 세계가 연결될 때 Coding Agent는 비로소 실제 애플리케이션을 디버깅할 수 있다.

---

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
- 사용 위치: db.query.summary, db.query.text, SQL parameter 수집과 민감정보 처리

### [S-OTEL-SERVICE] OpenTelemetry — Service semantic conventions

- 유형: 공식 사양
- URL: https://opentelemetry.io/docs/specs/semconv/resource/service/
- 사용 위치: service.name, service.version, service.instance.id

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
- 사용 위치: Tempo MCP, Agent용 trace 조회, LLM-oriented 응답

### [S-PYROSCOPE] Grafana Pyroscope

- 유형: 공식 문서 / 오픈소스
- URL: https://grafana.com/docs/pyroscope/latest/
- 사용 위치: continuous profiling, trace와 profile 연결

### [S-GRAFANA-MCP] Grafana — MCP server

- 유형: 공식 문서 / 오픈소스
- URL: https://grafana.com/docs/grafana/latest/developer-resources/mcp/
- 저장소: https://github.com/grafana/mcp-grafana
- 사용 위치: Prometheus/Loki/Tempo/Pyroscope를 Agent가 직접 조회, read-only와 query guardrail

### [S-SPRING-OBS] Spring Boot — Observability

- 유형: 공식 문서
- URL: https://docs.spring.io/spring-boot/reference/actuator/observability.html
- 사용 위치: Micrometer Observation, tracing, Spring Boot 관측 구성
- 주의: Spring Boot 버전별 지원 범위를 출간 전 재확인

### [S-ORACLE-JCMD] Oracle JDK 25 — The jcmd Command

- 유형: 공식 문서
- URL: https://docs.oracle.com/en/java/javase/25/docs/specs/man/jcmd.html
- 사용 위치: JFR.start, JFR.check, JFR.dump, JVM 진단 명령과 영향도
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

