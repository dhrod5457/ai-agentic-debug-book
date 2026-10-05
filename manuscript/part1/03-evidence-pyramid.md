# 3장. 디버깅 증거는 한 종류가 아니다

애플리케이션 장애가 발생하면 가장 먼저 로그를 찾는 습관이 있다. 로그는 중요하다. 하지만 로그가 모든 장애를 설명하지는 않는다.

성능 저하는 로그에 아무것도 남기지 않을 수 있다. CPU hotspot은 stack trace로 나타나지 않는다. Pod eviction은 애플리케이션 로그보다 Kubernetes 쪽이 더 정확하다. 잘못된 배포는 exception보다 version metadata가 더 중요한 증거일 수 있다.

Agentic Debugging에서 중요한 것은 '로그 접근'보다 'evidence surface'다.

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

Metric은 root cause를 직접 말해주지 않는다. 대신 문제 공간을 줄인다.

예를 들어 login p99는 상승했는데 DB CPU는 정상이고 Hikari pending이 급증했다면 DB 자체보다 connection acquisition 쪽을 먼저 볼 이유가 생긴다.

그래서 metric의 역할은 무엇이 문제인지 확정하는 것보다 어디를 볼지 결정하는 것에 가깝다.

## 2. Traces — 실제 execution path를 보여준다

Metric이 집계라면 trace는 구체적인 request다.

~~~text
POST /checkout 2.8s
 ├─ inventory 40ms
 ├─ payment 2.6s
 │   └─ external API 2.5s
 └─ save order 20ms
~~~

이제 slow request가 어디서 시간을 썼는지 볼 수 있다.

하지만 trace 역시 root cause 그 자체는 아니다. external API span이 느린 이유가 network, upstream saturation, retry, DNS, connection pool, timeout configuration 중 무엇인지는 추가 evidence가 필요하다.

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

Pyroscope처럼 trace와 profile을 연결할 수 있으면 Agent는 특정 request의 느린 구간에서 어떤 코드가 시간을 사용했는지 더 직접적으로 좁힐 수 있다.

## 5. JVM diagnostics — 더 깊이 들어갈 때 사용한다

Java 애플리케이션에서는 JFR과 thread dump가 강력하다. 하지만 이 evidence는 일반 metric query보다 비싸다.

Thread dump는 deadlock, blocked thread, thread pool starvation을 보여준다. JFR은 allocation, GC, lock, socket I/O, thread park 같은 runtime event를 보여준다. Heap dump는 retained object와 memory leak를 추적하는 데 강하다.

여기서 중요한 것은 '가능하면 다 수집'이 아니다. 필요한 경우 단계적으로 내려간다.

## 6. Database evidence — SQL만 보면 부족하다

다음 SQL이 2.8초 걸렸다고 보인다고 하자.

실제로 DB execution은 30ms였고 connection을 얻는 데 2.7초 걸렸다면 SQL 튜닝은 잘못된 수정이다.

DB debugging에는 connection acquire, query execute, lock wait, network, pool state, DB resource를 구분해야 한다.

OpenTelemetry의 DB semantic convention도 query summary와 query text를 구분한다. Agent에게는 low-cardinality query summary와 duration/error를 먼저 주는 편이 안전하다. parameter 값은 기본적으로 숨기는 것이 낫다.

## 7. Platform evidence — 코드가 문제가 아닐 수 있다

Kubernetes 환경에서는 애플리케이션 장애가 platform에서 시작될 수 있다.

~~~text
Pod restart
OOMKilled
Evicted
FailedScheduling
ImagePullBackOff
~~~

Kubernetes Event는 중요한 힌트다. 하지만 Event 하나를 canonical truth로 보면 위험하다. Pod status, container state, resource metric, deployment revision을 함께 봐야 한다.

## 8. Artifact identity — 어떤 코드가 실제로 실행됐는가

이 evidence는 자주 빠진다.

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

일 수 있기 때문이다. Runtime version을 확인하지 않은 patch는 논리적으로 불완전하다.

## 9. Evidence Pyramid

모든 evidence를 같은 비용으로 취급하지 말자.

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

이를 Evidence Escalation이라고 부른다. 외부 표준이 아니라 이 책의 설계 synthesis다.

## 10. 왜 escalation이 필요한가

첫째는 운영 비용 때문이다. heap dump는 metric query와 같은 행위가 아니다.

둘째는 보안 때문이다. heap에는 사용자 데이터와 credential fragment가 있을 수 있다.

셋째는 권한 때문이다. 기존 JFR 파일을 읽는 것과 production JVM에서 새 recording을 시작하는 것은 다른 권한이다.

그래서 다음을 분리해야 한다.

> Diagnostic Capture ≠ Diagnostic Read

## 11. 좋은 Agent는 무엇을 더 볼지가 아니라 무엇을 아직 안 봐도 되는지 안다

Agent가 자율적이라고 해서 모든 tool을 사용해야 하는 것은 아니다.

좋은 debugging trajectory는 보통 metric에서 service를 좁히고, trace에서 operation을 좁히고, log/profile에서 원인 후보를 좁힌 뒤 그래도 구분이 안 될 때 JFR/thread로 내려간다.

반대로 첫 단계에서 heap dump부터 요청한다면 시스템 설계가 잘못된 것이다.

## 12. 세 번째 원칙

> Evidence는 종류마다 역할과 비용이 다르다.

그리고:

> 낮은 비용의 evidence로 먼저 좁히고, 필요한 경우에만 더 비싼 evidence로 escalation한다.

다음 장에서는 이 여러 signal을 하나의 execution으로 연결하는 기반인 OpenTelemetry를 살펴본다.

### 주요 근거

- [S-PROM-API] Prometheus HTTP API
- [S-TEMPO-API] Tempo HTTP API
- [S-PYROSCOPE] Grafana Pyroscope
- [S-SPRING-OBS] Spring Boot Observability