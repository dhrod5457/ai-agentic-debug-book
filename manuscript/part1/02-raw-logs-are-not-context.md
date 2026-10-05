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

그런데 production 규모에서는 이 접근이 빠르게 무너진다. 문제는 단순히 context window 크기만이 아니다.

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

사람이 보더라도 correlation ID가 없다면 읽기 어렵다. 멀티스레드 애플리케이션, 여러 Pod, 여러 service로 가면 로그는 더 뒤섞인다.

시간순으로 나열된 텍스트는 causal order가 아니다. 로그를 많이 제공한다고 request boundary가 복원되는 것은 아니다.

## 2. irrelevant evidence가 추론을 방해한다

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

Context는 storage가 아니다. 모델이 읽을 수 있다고 해서 모든 정보가 같은 중요도로 처리되는 것은 아니다.

OpenRCA가 흥미로운 이유도 여기에 있다. 이 benchmark의 RCA-agent baseline은 방대한 telemetry를 model context에 넣지 않고 Python으로 필요한 부분을 검색하고 분석한다.

데이터는 실행 환경에 남긴다. 모델에는 결과를 가져온다.

## 3. Model Context는 Telemetry Working Set이 아니다

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

> Model Context ≠ Telemetry Working Set

Telemetry는 query 가능한 외부 상태로 남겨두는 편이 낫다. Agent는 필요한 evidence만 단계적으로 가져온다.

## 4. 로그 전체를 넣으면 시간 범위도 흐려진다

incident가 09:10~09:15에 발생했다고 하자. 그런데 하루치 로그를 모두 넣으면 00:00부터 23:59까지의 사건이 함께 들어간다.

Agent가 우연히 다른 시간대의 같은 exception을 발견하면 잘못된 가설을 만들 수 있다.

실제 debugging에서는 time range가 매우 중요하다. 그래서 첫 질문은 보통 '언제부터 언제까지 문제가 있었는가?'다.

Agent query에도 같은 제약이 들어가야 한다. Grafana MCP의 Loki guardrail이 최대 effective time range를 두는 이유도 기술적으로 같은 문제와 맞닿아 있다.

## 5. 로그 전체는 보안 경계도 무너뜨린다

로그에는 생각보다 많은 정보가 들어간다.

- Authorization header
- cookie
- session ID
- user ID
- email
- DB bind value
- request body
- internal URL
- tenant identifier

사람이 Grafana에서 필요한 query만 보는 것과 로그 파일 전체를 외부 모델 context로 보내는 것은 보안적으로 전혀 다른 행위다.

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

저장하면 안 되는 값은 Collector에서 제거한다. 저장된 데이터 중 Agent가 볼 수 있는 범위는 Query Gateway에서 다시 제한한다.

이를 다음처럼 구분할 수 있다.

> Collection Governance ≠ Retrieval Governance

## 6. 오래된 로그는 현재 코드와 맞지 않을 수 있다

운영 장애 로그는 어제 배포된 이미지에서 발생했고, Agent가 보고 있는 repository는 오늘 main일 수 있다.

~~~text
Incident Runtime
commit A

Current Workspace
commit B
~~~

그런데 이 사실을 모른 채 stack trace의 line number만 따라가면 엉뚱한 코드를 수정할 수 있다.

따라서 Debug Context에는 로그뿐 아니라 runtime identity가 필요하다.

최소한 service.name, service.version, deployment.environment, image digest, deployment revision을 확인해야 한다. 가능하면 source commit까지 연결한다.

## 7. Push Context에서 Pull Evidence로

기존 방식은 telemetry를 모아서 prompt에 넣는 것이다.

더 나은 방식은 Agent가 telemetry backend에 질문하는 것이다.

~~~text
Telemetry Backend
      ↑
    query
      ↑
Agent
~~~

Agent는 먼저 문제 시간대의 login-service p99 latency를 묻는다. 그 결과를 보고 Hikari pending connection을 묻는다. 다음에는 3초 이상 걸린 trace 몇 개를 찾고, 그중 하나의 trace ID로 로그를 검색한다.

이것이 progressive narrowing이다. 넓게 보고 좁혀간다.

## 8. Evidence Retrieval도 무제한이면 안 된다

Pull 방식이라고 자동으로 안전해지는 것은 아니다. Agent가 30일 전체 로그를 검색하도록 놔두면 observability backend 자체에 부담을 줄 수 있다.

그래서 query budget이 필요하다.

~~~text
max_time_range = 30m
max_scan_bytes = 2GB
max_log_lines = 1000
max_traces = 20
~~~

실제 Grafana MCP에도 Loki query의 최대 scan bytes와 time range를 제한하고 특정 label matcher를 모든 query에 강제로 추가하는 guardrail이 있다.

이것은 단순한 성능 최적화가 아니라 Agent interface 설계의 일부다.

## 9. 결과가 잘렸다면 알려야 한다

Agent에게 'No errors found'라는 결과가 돌아왔다고 하자.

실제로는 100000 lines 중 앞 1000줄만 반환된 결과라면 결론은 완전히 달라진다.

그래서 tool response에는 query result뿐 아니라 한계도 들어가야 한다.

~~~text
sampled=true
truncated=true
result_count=1000
continuation_available=true
~~~

OpenRCA의 Executor도 큰 DataFrame 결과가 잘린 경우 observation bias 가능성을 명시적으로 경고한다.

결과의 불완전성을 숨기지 않는 것이 중요하다.

## 10. 두 번째 원칙

> Model Context ≠ Telemetry Working Set

그리고:

> Push Context보다 Pull Evidence가 확장 가능하다.

Agent에게 모든 로그를 읽게 하지 않는다. 대신 scope → aggregate → representative execution → local evidence → source 순서로 필요한 evidence를 가져오게 한다.

다음 장에서는 이 evidence가 로그만으로 구성되지 않는 이유를 살펴본다.

### 주요 근거

- [S-OPENRCA] OpenRCA
- [S-GRAFANA-MCP] Grafana MCP
- [S-LLM4LOG] LLM4Log
- [S-OTEL-COLLECTOR] OpenTelemetry Collector