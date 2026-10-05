# 7장. Loki에서 같은 execution의 로그를 찾는다

Trace를 통해 느린 request와 suspicious span을 찾았다.

이제 로그를 본다.

중요한 점은 로그를 처음 보는 것이 아니라는 것이다.

이미 service, time range, trace ID가 있다.

## 1. 로그 검색의 질은 scope에서 결정된다

나쁜 query는 전체 production 로그에서 timeout을 검색한다.

좋은 query는 특정 service, 특정 시간대, 특정 trace ID를 기준으로 검색한다.

~~~text
service=login-service
range=09:12:10~09:12:14
trace_id=abc
~~~

범위가 좁아질수록 irrelevant evidence가 줄어든다.

## 2. Label과 Structured Metadata를 구분한다

Loki에서는 모든 correlation key를 label로 만들면 안 된다.

service name이나 namespace처럼 cardinality가 낮은 값은 label에 잘 맞는다.

반면 trace_id, request_id처럼 요청마다 바뀌는 값은 cardinality가 매우 높다.

이런 값을 stream label로 만들면 저장과 query 성능이 나빠질 수 있다.

Grafana Loki는 structured metadata를 통해 이런 high-cardinality field를 다룰 수 있다.

Agent 편의를 위해 backend 구조를 망가뜨리지 않는 것이 중요하다.

## 3. 로그는 event evidence로 본다

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

## 4. Stack trace도 projection이 필요하다

긴 Java stack trace를 매번 전부 Agent에 줄 필요는 없다.

첫 응답에는 다음이 더 유용할 수 있다.

- exception type
- message
- root cause
- top application frames
- caused-by chain
- trace/span ID

Agent가 더 필요하면 full stack을 요청한다.

이것도 progressive disclosure다.

## 5. Loki query에도 예산이 필요하다

Grafana MCP는 Loki query에 대해 최대 scan bytes와 최대 effective time range를 제한할 수 있다.

또 모든 query에 label matcher를 강제로 추가해 production/staging 또는 특정 service 범위를 벗어나지 못하게 할 수 있다.

이 패턴은 매우 중요하다.

~~~text
Agent Query Budget
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
- query scope 오류
- trace propagation 깨짐
- retention 만료
- result truncation

따라서 tool result에는 sampled, truncated, retention, query scope 같은 metadata가 필요하다.

## 7. 일곱 번째 원칙

> 로그는 전체를 읽는 대상이 아니라 이미 좁혀진 execution을 설명하는 evidence로 사용한다.

그리고:

> high-cardinality correlation key와 stream indexing strategy를 분리한다.

다음 장에서는 로그와 trace만으로 부족한 CPU, lock, allocation 문제를 profile과 JVM diagnostics로 내려가 본다.

### 주요 근거

- [S-LOKI-METADATA] Loki Structured Metadata
- [S-LOKI-QUERY] Loki Query Best Practices
- [S-GRAFANA-MCP] Grafana MCP