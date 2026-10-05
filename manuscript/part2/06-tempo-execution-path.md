# 6장. Tempo로 실제 실행 경로를 따라간다

Metric에서 login-service의 특정 시간대가 문제라는 사실까지 좁혔다.

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

예를 들어 한 trace에서 payment-service가 느렸다면 다음 질문이 필요하다.

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

2026년 현재 Tempo는 Agent용 MCP endpoint와 LLM용 간소화 응답을 제공한다. MCP 서버는 Tempo 설정에서 별도로 활성화해야 한다.

이 변화는 중요하다.

관측 데이터 저장소가 더 이상 사람의 UI만을 위한 저장소가 아니라 machine reasoning client를 직접 고려하기 시작했다는 뜻이다.

하지만 Tempo MCP가 원인을 보장하는 것은 아니다.

Tempo는 증거를 제공한다. 원인 후보를 세우고 코드를 수정하는 일은 다른 책임이다.

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

원본 trace는 그대로 보관하되, Agent에게는 먼저 필요한 부분만 보여주는 편이 낫다. LLM용 간소화 응답은 형식이 바뀔 수 있으므로 안정적인 프로그램 연동에는 표준 JSON이나 MCP 인터페이스를 우선한다.

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

- context propagation이 빠졌다.
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