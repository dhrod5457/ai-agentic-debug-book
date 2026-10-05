# 5장. Prometheus로 문제 공간을 먼저 줄인다

운영 장애를 만났을 때 Agent가 가장 먼저 해야 할 일은 로그를 검색하는 것이 아닐 수 있다.

먼저 범위를 줄여야 한다.

어느 service가 문제인지, 언제 시작됐는지, error인지 latency인지, 특정 endpoint인지 전체 시스템인지부터 알아야 한다.

Prometheus는 이 단계에 잘 맞는다.

## 1. Metric은 root cause보다 scope를 알려준다

예를 들어 로그인 지연이 보고됐다.

Agent가 처음부터 login-service 로그 10만 줄을 읽는 대신 다음을 확인한다.

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

이제 investigation scope가 훨씬 작아졌다.

Metric은 답을 주지 않았지만 무엇을 다음에 볼지 정해줬다.

## 2. Agent에게 Dashboard를 보여줄 필요는 없다

사람은 Grafana graph를 보는 것이 편하다. Agent는 숫자와 구조화된 query result가 더 직접적이다.

Prometheus HTTP API는 instant query와 range query를 JSON으로 반환한다.

따라서 Agent tool은 다음 정도면 충분할 수 있다.

~~~text
query_metric(query, time)
query_metric_range(query, start, end, step)
get_exemplars(query, start, end)
~~~

중요한 것은 PromQL 문법보다 tool boundary다.

## 3. Broad query를 허용하면 안 된다

Agent가 임의 PromQL을 사용할 수 있다고 해서 무제한 range와 series를 읽게 할 필요는 없다.

Tool gateway는 다음을 제한할 수 있다.

- datasource
- environment
- service
- time range
- returned series
- sample count
- query timeout

Agent에게 query capability를 주는 것과 observability backend 전체를 맡기는 것은 다르다.

## 4. Exemplars가 중요한 이유

Metric은 aggregate다.

~~~text
p99 = 3.1s
~~~

이 값만으로는 어떤 request가 3초 걸렸는지 알 수 없다.

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

## 5. anomaly를 root cause로 부르지 않는다

Prometheus에서 connection pending이 증가했다고 connection pool이 반드시 원인인 것은 아니다.

upstream timeout 때문에 transaction이 길어져 결과적으로 pool이 고갈됐을 수도 있다.

Metric 결과는 hypothesis를 만드는 evidence다.

~~~text
Evidence
Hikari pending ↑

Hypothesis
connection acquisition bottleneck

Next
slow trace에서 connection acquire span 확인
~~~

이 구분을 지켜야 한다.

## 6. Prometheus의 역할은 다음 query를 더 좋게 만드는 것이다

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

즉 Prometheus는 Agent가 다음 evidence를 더 정확하게 찾게 하는 첫 번째 narrowing layer다.

## 7. 다섯 번째 원칙

> Aggregate signal에서 시작하되 concrete execution으로 내려갈 연결 고리를 남긴다.

다음 장에서는 이 concrete execution을 Tempo와 TraceQL로 따라간다.

### 주요 근거

- [S-PROM-API] Prometheus HTTP API
- [S-PROM-EXEMPLAR] Prometheus Exemplars
- [S-GRAFANA-MCP] Grafana MCP