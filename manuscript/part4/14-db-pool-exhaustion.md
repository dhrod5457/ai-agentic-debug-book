# 14장. DB Connection Pool이 바닥났을 때

월요일 오전, 로그인 API가 간헐적으로 3초씩 멈춘다.

로그에는 SQL timeout이 보인다.

첫 느낌은 'DB가 느려졌나?'다.

Agent도 소스코드만 보면 같은 추측을 하기 쉽다.

하지만 실제 원인은 connection pool일 수 있다.

## 1. 먼저 증상을 본다

Prometheus에서 로그인 API p99를 본다.

~~~text
09:08 180ms
09:10 900ms
09:12 3.1s
09:18 220ms
~~~

같은 시간대의 DB CPU는 정상이다.

반면 Hikari pending connection이 올라간다.

~~~text
active 20
idle 0
pending 37
~~~

이제 'DB 자체가 느리다'는 가설보다 'connection을 얻기 어렵다'는 가설이 강해진다.

## 2. 느린 요청 하나를 본다

Tempo에서 느린 trace를 찾는다.

~~~text
POST /login                 3.1s
 └ authenticate             3.0s
    ├ acquireConnection     2.8s
    └ SELECT user           34ms
~~~

여기서 중요한 건 34ms와 2.8초의 차이다.

SQL은 빠르다.

기다린 곳은 connection acquire다.

## 3. 같은 trace의 로그를 확인한다

Loki에서 trace ID를 검색한다.

~~~text
connection acquisition timeout
active=20 idle=0 pending=37
~~~

이제 세 가지 증거가 같은 방향을 가리킨다.

- API 지연
- pool pending 증가
- connection acquire 지연

## 4. 그래도 아직 원인은 하나 더 남아 있다

pool이 부족한 이유는 여러 가지다.

- max pool size가 너무 작음
- transaction이 너무 김
- connection leak
- 외부 호출을 transaction 안에서 오래 기다림

그래서 소스코드를 본다.

예를 들어 transaction 안에서 외부 API를 호출하고 있었다고 하자.

~~~text
transaction 시작
  ↓
DB 조회
  ↓
외부 API 2초 대기
  ↓
transaction 종료
~~~

이 동안 connection이 잡혀 있으면 traffic이 몰릴 때 pool이 빠르게 바닥난다.

## 5. 잘못된 첫 수정

이 상황에서 pool size만 20에서 50으로 늘리면 증상은 잠시 좋아질 수 있다.

하지만 원인이 긴 transaction이라면 병목을 뒤로 미룬 것뿐이다.

그래서 Agent는 '증상 완화'와 '원인 수정'을 구분해야 한다.

## 6. 재현한다

테스트 환경에서 같은 traffic을 만든다.

관찰 항목은 단순하다.

- p99
- active/pending connection
- connection acquire 시간

문제가 재현되면 transaction 범위를 줄이거나 외부 호출을 transaction 밖으로 옮긴다.

## 7. 수정 후 처음 증상을 다시 본다

같은 부하를 다시 건다.

~~~text
Before
p99 3.1s
pending 37
acquire 2.8s

After
p99 240ms
pending 0~2
acquire 8ms
~~~

이제야 장애가 해결됐다고 말할 수 있다.

## 8. Agent는 어떤 순서로 움직였나

~~~text
지연시간 확인
→ pool 지표 확인
→ 느린 trace 확인
→ 같은 trace 로그 확인
→ 소스코드에서 긴 transaction 확인
→ 재현
→ 수정
→ 같은 지표로 재검증
~~~

## 9. 다음 사례에서는 로그가 거의 도움이 되지 않는다

이번 사례는 metric, trace, log가 비교적 잘 맞아떨어졌다.

하지만 모든 장애가 이렇게 친절하지는 않다.

다음 장에서는 오류 로그도 없고 CPU도 높지 않은데 요청만 느린 상황을 본다.

그때는 profile과 thread 상태가 더 중요해진다.

## 10. 이 장에서 기억할 것

> timeout 로그가 보인다고 SQL부터 고치지 않는다.

> 어디에서 기다렸는지 trace로 먼저 구분한다.

### 참고 자료

출처 상세: [References](../references.md)

- [S-PROM-API] Prometheus HTTP API
- [S-TEMPO-API] Tempo HTTP API
- [S-SPRING-OBS] Spring Boot 관측 시스템
