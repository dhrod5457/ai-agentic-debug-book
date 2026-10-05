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

특정 slow span에서 profile을 조회하면 전체 service profile보다 훨씬 좁은 evidence를 얻을 수 있다.

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

하지만 JFR도 원본 파일 전체를 model context에 넣는 대상은 아니다.

incident window의 event summary나 top contention 같은 projection을 제공하는 편이 낫다.

## 4. Thread dump가 유효한 장애

Thread dump는 특히 다음 문제에서 강하다.

- deadlock
- blocked thread
- executor starvation
- stuck request
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

먼저 heap usage, GC pause, allocation profile, JFR evidence로 범위를 좁힐 수 있다.

heap dump는 크고 민감하며 production pause와 저장 비용을 유발할 수 있다.

그래서 가벼운 확인부터 시작해 필요한 경우에만 더 깊은 진단으로 내려가야 한다.

~~~text
metrics
→ profile
→ JFR
→ thread dump
→ heap dump
~~~

실제 순서는 장애에 따라 달라질 수 있지만 비용이 높은 evidence를 자동 기본값으로 두지 않는 것이 핵심이다.

## 6. 권한도 단계적으로 올라가야 한다

기존 profile을 읽는 것과 production JVM에서 새 JFR을 시작하는 것은 다르다.

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

> 지금 가진 evidence로 competing hypothesis를 구분할 수 있는가?

구분할 수 없다면 다음으로 가장 값싼 evidence를 선택한다.

이 원칙은 디버깅 비용뿐 아니라 production 안전성을 지킨다.

## 8. 여덟 번째 원칙

> Dump를 Context로 착각하지 않는다.

그리고:

> 이미 있는 진단 자료를 읽는 것과 운영 서버에서 새 진단 자료를 만드는 권한은 나눈다.

여기까지가 Agent가 애플리케이션을 관측하기 위한 핵심 signal stack이다.

다음 Part에서는 이 signal들을 실제 Agent Tool로 노출할 때 어떤 interface와 policy가 필요한지 살펴본다.

### 주요 근거

- [S-PYROSCOPE] Grafana Pyroscope
- JVM runtime evidence research
- [S-TEMPO-AI] Grafana Tempo and AI