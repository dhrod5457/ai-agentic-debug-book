# 5장. Prometheus로 문제 공간을 먼저 줄인다

운영 장애를 만났을 때 Agent가 가장 먼저 해야 할 일은 로그를 검색하는 것이 아닐 수 있다.

먼저 범위를 줄여야 한다.

어느 서비스가 문제인지, 언제 시작됐는지, error인지 latency인지, 특정 endpoint인지 전체 시스템인지부터 알아야 한다.

Prometheus는 이 단계에 잘 맞는다.

## 1. Metric은 원인보다 먼저 문제 범위를 알려준다

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

Agent에게 조회 권한를 주는 것과 관측 데이터 저장소 전체를 맡기는 것은 다르다.

## 4. Exemplars가 중요한 이유

Metric은 aggregate다.

~~~text
p99 = 3.1s
~~~

이 값만으로는 어떤 요청이 3초 걸렸는지 알 수 없다.

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