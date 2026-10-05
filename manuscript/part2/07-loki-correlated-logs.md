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

서비스 name이나 namespace처럼 cardinality가 낮은 값은 label에 잘 맞는다.

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

### 주요 근거

- [S-LOKI-METADATA] Loki Structured Metadata
- [S-LOKI-조회] Loki 조회 Best Practices
- [S-GRAFANA-MCP] Grafana MCP