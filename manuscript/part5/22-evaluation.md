# 22장. Agentic Debugging을 어떻게 평가할까

Agent가 디버깅을 잘한다고 말하려면 무엇을 측정해야 할까?

최종 정답률만 보면 부족하다.

우연히 맞힐 수도 있고, 너무 많은 로그를 읽고 비싼 query를 남발할 수도 있기 때문이다.

## 1. 네 가지 조건을 비교한다

이 책에서 제안한 실험은 단순하다.

~~~text
A. 소스코드만 제공

B. 소스 + raw log bundle

C. 소스 + Grafana/Tempo 조회 도구

D. C + 디버깅 순서와 기록 규칙
~~~

같은 Agent와 같은 장애에서 차이를 본다.

## 2. 원인을 맞혔는가

가장 기본적인 지표다.

- root cause component
- root cause reason

둘을 나눠 볼 수 있다.

## 3. 제대로 고쳤는가

원인을 맞혀도 patch가 틀릴 수 있다.

그래서 다음도 본다.

- correct patch
- reproduction success
- regression test
- incident signal 개선

## 4. 필요한 증거를 실제로 봤는가

Agent가 connection pool 문제를 맞혔지만 pool metric도 trace도 보지 않았다면 우연일 수 있다.

그래서 각 scenario마다 최소한 봐야 할 증거를 정해둘 수 있다.

## 5. 얼마나 많이 읽었는가

같은 정답이라면 적은 query와 적은 token으로 찾는 쪽이 운영에서 더 낫다.

측정 후보:
- token
- query 수
- 반환 데이터 크기
- scan bytes
- 진단 단계 수

## 6. 안전하게 조회했는가

정답을 맞혔더라도 production 전체를 무제한 검색했다면 좋은 시스템이 아니다.

그래서 다음도 기록한다.

- 범위 밖 query
- 너무 넓은 시간 범위
- 민감정보 노출
- 불필요한 heap/JFR capture
- mutation attempt

## 7. 실패 이유를 분류한다

점수만 보면 어디를 개선해야 할지 모른다.

실패를 다음처럼 나눌 수 있다.

~~~text
필요한 자료를 못 찾음
자료는 찾았지만 연결 못 함
자료를 잘못 해석함
잘못된 버전의 코드 봄
도구를 잘못 사용함
검증 부족
~~~

이 분류가 다음 시스템 개선으로 이어진다.

## 8. 한 번의 성공을 믿지 않는다

Agent 실행은 변동성이 있다.

그래서 같은 scenario를 여러 번 실행하고 단일 성공률과 반복 신뢰성을 함께 본다.

## 9. 이 장에서 기억할 것

> Agentic Debugging 평가는 정답뿐 아니라 과정, 비용, 안전성, 검증까지 함께 봐야 한다.

> 좋은 디버깅 시스템은 왜 맞았는지를 다시 설명할 수 있어야 한다.