# 21장. 수정했다고 끝난 것이 아니다

Agent가 코드를 수정했다.

테스트도 통과했다.

문제가 해결됐다고 말해도 될까?

아직 한 단계가 남았다.

처음 장애를 확인했던 신호가 실제로 좋아졌는지 봐야 한다.

## 1. 처음 증상으로 돌아간다

DB pool 장애였다면 처음 본 것은 p99와 pending connection이었다.

수정 후 같은 부하에서 다시 본다.

~~~text
Before
p99 3.1s
pending 37
connection acquire 2.8s

After
p99 240ms
pending 1
connection acquire 8ms
~~~

이 비교가 있어야 실제 incident 해결을 말할 수 있다.

## 2. 테스트 통과와 운영 해결은 다르다

단위 테스트는 로직이 맞는지 확인한다.

통합 테스트는 여러 컴포넌트가 함께 동작하는지 확인한다.

하지만 운영 장애는 latency, concurrency, memory, thread, platform 상태 때문에 생길 수 있다.

따라서 test pass만으로는 부족하다.

## 3. 수정한 코드가 실제로 배포됐는지도 확인한다

의외로 자주 빠지는 단계다.

Agent가 commit B에서 수정했고 CI도 통과했다.

그런데 production은 여전히 commit A image를 실행하고 있을 수 있다.

그래서 검증 전에 다음을 확인한다.

~~~text
expected version = B
deployed version = B
image digest = expected digest
~~~

버전 확인이 없으면 좋은 결과가 우연히 다른 배포 때문인지 구분하기 어렵다.

## 4. 가능하면 같은 workload로 다시 확인한다

수정 전에는 100 concurrent users였는데 수정 후에는 5명으로 시험했다면 비교가 어렵다.

가능한 한 같은 조건을 맞춘다.

- 입력 데이터
- 요청 수
- 동시성
- timeout
- 환경 설정

완전히 같게 만들 수 없다면 차이를 기록한다.

## 5. 하나의 숫자만 좋아졌다고 끝내지 않는다

예를 들어 latency를 줄이기 위해 cache를 추가했다.

p99는 좋아졌지만 stale data 오류가 늘 수 있다.

또 timeout을 줄여 latency는 좋아 보이지만 error rate가 올라갈 수 있다.

그래서 기능 결과와 장애 신호를 같이 본다.

예:
~~~text
latency 개선
+ error rate 유지
+ 데이터 정확성 유지
+ resource 사용량 허용 범위
~~~

## 6. 부분 해결도 있다

수정 후 다음처럼 나올 수 있다.

~~~text
p99 3.1s → 700ms
pending 37 → 5
error rate 정상
~~~

좋아졌지만 목표가 300ms라면 완전한 해결은 아니다.

또 이런 경우도 있다.

~~~text
p99 정상
하지만 GC pause 여전히 높음
~~~

이 경우 다른 문제가 남았을 수 있다.

Agent는 성공/실패 둘 중 하나만 선택하기보다 부분 해결을 기록할 수 있어야 한다.

## 7. 잘못된 수정은 before/after에서 드러난다

connection pool 문제에서 pool size만 크게 늘렸다고 하자.

초기에는 p99가 좋아질 수 있다.

하지만 traffic을 더 올리면 pending이 다시 증가한다.

근본 원인이 긴 transaction이라면 병목을 뒤로 미룬 것뿐이다.

그래서 검증은 한 번의 happy path가 아니라 원래 문제가 나타나던 조건에서 해야 한다.

## 8. 검증 결과도 증거로 남긴다

수정 전후 자료는 코드 리뷰와 incident 회고에 유용하다.

예:
~~~text
Fix
외부 API 호출을 transaction 밖으로 이동

Before
p99 3.1s / pending 37

After
p99 240ms / pending 1

Regression
기존 login 기능 테스트 통과
~~~

이 정도만 남아도 왜 이 patch가 효과가 있었는지 나중에 다시 확인할 수 있다.

## 9. 다음 장애를 위한 규칙으로 바로 고정하지 않는다

이번에 같은 증상이 connection pool 문제였다고 다음번에도 그렇다고 단정하면 안 된다.

환경과 버전, 원인이 달라질 수 있다.

과거 incident는 참고자료이지 현재의 사실이 아니다.

## 10. Agent가 따라야 할 검증 순서

~~~text
patch
→ 테스트
→ 올바른 버전 배포 확인
→ 같은 workload
→ 처음 증상 재확인
→ 부작용 확인
→ 잔여 이상 기록
~~~

## 이 장의 한 문장

> 처음 문제를 발견한 신호를 수정 후 다시 본다.

> 테스트 통과와 장애 해결은 같은 말이 아니다.