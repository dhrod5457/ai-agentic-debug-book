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

출처 상세: [References](../references.md)

- [S-PYROSCOPE] Grafana Pyroscope
- JVM runtime 증거 research
- [S-TEMPO-AI] Grafana Tempo and AI