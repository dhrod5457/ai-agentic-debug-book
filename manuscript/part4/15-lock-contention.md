# 15장. Lock Contention은 로그만으로 보이지 않는다

이번에는 API가 느리지만 로그에는 특별한 오류가 없다.

CPU도 높지 않다.

DB도 정상이다.

이런 문제는 오히려 더 답답하다.

## 1. trace에서 이상한 시간을 찾는다

report API의 trace를 본다.

~~~text
POST /report 2.4s
 └ generateReport 2.2s
~~~

문제 위치는 좁혀졌지만 왜 2.2초인지 아직 모른다.

## 2. CPU가 정상이라면 기다리고 있을 수 있다

CPU 사용률은 35% 수준이다.

그런데 요청은 느리다.

이럴 때는 '무언가를 계산하느라 느리다'보다 '무언가를 기다리느라 느리다'는 가능성을 본다.

## 3. profile을 본다

Pyroscope wall profile을 보면 LockSupport.park와 특정 application method가 크게 보인다고 하자.

~~~text
OrderLock.acquire
  ↓
LockSupport.park
~~~

이제 lock contention 가능성이 생긴다.

## 4. 더 확인해야 하면 thread dump나 JFR로 내려간다

thread dump를 요약한다.

~~~text
BLOCKED 12
WAITING 71

Top blocked stack
OrderLock.acquire() 10 threads
~~~

JFR에서도 같은 시간대에 monitor contention이 보인다.

이제 원인 후보가 훨씬 강해졌다.

## 5. 소스코드를 본다

예를 들어 모든 report 생성을 하나의 synchronized 블록으로 감싸고 있었다고 하자.

~~~text
synchronized(globalLock) {
  generateReport();
}
~~~

한 요청이 오래 걸리면 나머지 요청도 줄을 선다.

CPU는 높지 않은데 latency는 크게 올라갈 수 있다.

## 6. 왜 로그에는 아무것도 없었나

예외가 아니기 때문이다.

모든 요청은 결국 성공했다.

다만 오래 기다렸다.

이런 장애는 log만 보면 거의 보이지 않는다.

## 7. 수정과 검증

lock 범위를 줄이거나 key별 lock으로 바꾼 뒤 같은 부하를 다시 건다.

Before/After로 다음을 비교한다.

- p99
- blocked thread count
- wall profile
- lock contention event

## 8. 이 장에서 기억할 것

> CPU가 낮은데 느리다면 계산이 아니라 대기를 의심해볼 수 있다.

> trace로 위치를 찾고 profile과 thread 정보로 이유를 확인한다.