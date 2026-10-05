# 15장. Lock Contention은 로그만으로 보이지 않는다

이번에는 API가 느리지만 로그에는 특별한 오류가 없다.

CPU도 높지 않다. DB도 정상이다. 모든 요청은 결국 200으로 끝난다.

이런 문제는 개발자를 더 답답하게 만든다. 실패했다는 흔적이 거의 없기 때문이다.

## 1. 처음에는 DB를 의심하기 쉽다

report API가 평소 300ms 안에 끝나는데 특정 시간대부터 2초가 넘는다.

로그에 예외는 없다. DB query도 대부분 20~30ms다.

소스코드만 보면 report 생성 로직이 무거워 보인다. Agent도 처음에는 SQL이나 CPU 사용을 의심할 수 있다.

하지만 Prometheus에서 CPU를 보면 35% 수준이다.

이럴 때는 질문을 바꿔야 한다.

> 계산하느라 느린 것이 아니라 기다리느라 느린 것은 아닐까?

## 2. trace에서 시간이 멈춘 위치를 찾는다

Tempo에서 느린 요청 하나를 본다.

~~~text
POST /report 2.4s
 └ generateReport 2.2s
~~~

문제 위치는 좁혀졌다. 하지만 generateReport 안에서 왜 2.2초가 걸렸는지는 아직 모른다.

여기서 CPU profile만 보면 답이 잘 안 나올 수 있다. 실제로 CPU를 거의 쓰지 않고 기다리고 있기 때문이다.

## 3. wall profile을 보면 기다린 시간이 보인다

Pyroscope wall profile에서 다음 패턴이 크게 보인다고 하자.

~~~text
OrderLock.acquire
  ↓
LockSupport.park
~~~

이제 lock을 기다리는 시간이 길다는 가능성이 생긴다.

중요한 점은 여기서도 바로 결론을 내리지 않는 것이다.

LockSupport.park는 여러 이유로 나타날 수 있다. 그래서 더 구체적인 자료가 필요하다.

## 4. thread dump로 같은 stack이 몰려 있는지 본다

thread dump를 요약해보니 다음과 같다.

~~~text
RUNNABLE 18
WAITING  71
BLOCKED  12

Top blocked stack
OrderLock.acquire() 10 threads

Deadlock
none
~~~

여러 요청 thread가 같은 위치에서 기다리고 있다.

이제 '한 요청이 우연히 늦었다'가 아니라 '여러 요청이 같은 lock 앞에서 줄을 섰다'는 사실을 확인할 수 있다.

## 5. JFR은 시간 흐름을 확인할 때 유용하다

thread dump는 한 순간의 사진에 가깝다.

JFR에서 같은 시간대의 monitor contention이나 thread park event를 보면 문제가 몇 초 동안 반복됐는지 확인할 수 있다.

이 단계까지 와야 lock contention이라는 원인 후보가 충분히 강해진다.

## 6. 소스코드로 돌아간다

이제 소스코드를 본다.

예를 들어 모든 report 생성을 하나의 전역 lock으로 감싸고 있었다고 하자.

~~~text
synchronized(globalLock) {
  generateReport();
}
~~~

한 요청이 report를 만드는 동안 나머지 요청은 기다린다.

traffic이 적을 때는 잘 드러나지 않지만 동시 요청이 늘면 p99가 빠르게 악화된다.

## 7. 흔한 잘못된 수정

이 상황에서 thread pool 크기만 늘리면 어떻게 될까?

기다리는 thread 수만 늘어날 수 있다.

CPU가 남아 있으니 worker를 늘리자는 판단은 자연스럽지만, 병목이 lock이라면 문제를 해결하지 못한다.

또 timeout만 늘리면 사용자는 더 오래 기다리게 된다.

증상을 완화하는 설정 변경과 원인을 고치는 수정은 구분해야 한다.

## 8. 수정 방법은 lock의 목적에 따라 달라진다

전역 lock이 정말 필요한지 먼저 본다.

가능한 수정은 상황마다 다르다.

- lock 범위를 줄인다.
- 오래 걸리는 작업을 lock 밖으로 옮긴다.
- 전역 lock을 key별 lock으로 나눈다.
- 불변 자료를 미리 계산해 공유한다.

중요한 것은 특정 패턴을 외우는 것이 아니라 왜 그 lock이 존재하는지 확인하는 것이다.

## 9. 수정 후 같은 자료를 다시 본다

같은 부하를 다시 건다.

Before:
~~~text
p99 2.4s
BLOCKED 12
OrderLock.acquire 10 threads
~~~

After:
~~~text
p99 320ms
BLOCKED 0~1
OrderLock.acquire hotspot 사라짐
~~~

이제 단위 테스트 통과보다 훨씬 강한 검증이 된다.

## 10. Agent가 따라야 할 순서

~~~text
느린 API 확인
→ DB와 CPU가 정상인지 확인
→ 느린 trace에서 위치 확인
→ wall profile로 대기 여부 확인
→ thread/JFR로 lock 경쟁 확인
→ 소스코드에서 lock 범위 확인
→ 수정
→ 같은 부하로 다시 확인
~~~

## 이 장의 한 문장

> CPU가 낮은데 느리다면 계산보다 대기를 먼저 의심해볼 수 있다.

> trace로 위치를 찾고 profile과 thread 정보로 이유를 확인한다.