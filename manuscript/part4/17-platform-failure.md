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

## 이 장의 한 문장

> 애플리케이션 장애가 항상 애플리케이션 코드에서 시작되는 것은 아니다.

> Pod와 Node 상태도 코드와 같은 수준의 디버깅 자료로 봐야 한다.