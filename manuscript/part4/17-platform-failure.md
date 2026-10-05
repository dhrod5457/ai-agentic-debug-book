# 17장. 장애가 코드가 아닐 때

API가 간헐적으로 500을 반환한다.

애플리케이션 로그를 뒤져도 결정적인 예외가 없다.

그런데 Pod restart count는 계속 올라간다.

이럴 때 코드만 붙잡고 있으면 문제를 놓칠 수 있다.

## 1. 먼저 Pod 상태를 본다

Kubernetes에서 해당 workload의 상태를 확인한다.

~~~text
restartCount = 6
lastState.reason = OOMKilled
~~~

이 한 줄만으로도 조사 방향이 크게 바뀐다.

## 2. Event도 같이 본다

같은 시간대의 Event를 확인한다.

~~~text
Reason: OOMKilling
Action: Killing
~~~

다만 Event 하나만으로 모든 것을 확정하지 않는다.

Event는 보조 자료다.

Pod 상태와 memory metric을 같이 본다.

## 3. memory가 실제로 올라갔는지 확인한다

Prometheus에서 container memory를 본다.

장애 직전 limit에 가까워지고, 그 직후 Pod가 재시작했다면 흐름이 연결된다.

~~~text
memory usage ↑
  ↓
limit 근접
  ↓
OOMKilled
  ↓
Pod restart
  ↓
일부 요청 500
~~~

## 4. 그다음에야 애플리케이션 쪽 원인을 본다

OOM의 원인은 여러 가지다.

- memory leak
- 한 번에 너무 큰 데이터 처리
- 잘못된 cache
- heap limit 설정
- 최근 배포의 allocation 증가

이제 JVM metric과 profile, 최근 source diff를 확인한다.

## 5. 코드 문제가 아닌 경우도 있다

반대로 애플리케이션 memory 사용은 정상인데 Node memory pressure 때문에 eviction된 경우도 있다.

이때 Java heap만 분석하면 헛수고다.

그래서 platform 상태를 별도의 증거로 봐야 한다.

## 6. Agent에게 Kubernetes 권한을 어디까지 줄까

대부분의 조사에는 읽기 권한이면 충분하다.

~~~text
Pod status 조회
Event 조회
Deployment 조회
Node pressure 조회
~~~

Pod 안에 exec로 들어가거나 restart하는 권한은 별도로 나눈다.

## 7. 수정 후 확인

원인이 memory regression이었다면 수정 후 같은 부하에서 다음을 본다.

- memory peak
- GC
- restart count
- OOM Event
- API error rate

## 8. 이 장에서 기억할 것

> 애플리케이션 장애가 항상 애플리케이션 코드에서 시작되는 것은 아니다.

> Pod와 Node 상태도 코드와 같은 수준의 디버깅 자료로 봐야 한다.