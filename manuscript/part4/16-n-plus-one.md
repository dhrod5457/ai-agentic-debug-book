# 16장. N+1은 느린 SQL 하나가 아니다

목록 API가 데이터가 적을 때는 빠른데, 데이터가 많아질수록 급격히 느려진다.

로그에는 특별한 timeout이 없다.

DB CPU도 크게 치솟지 않는다.

이럴 때 흔한 원인 중 하나가 N+1이다.

## 1. 먼저 한 요청 안에서 SQL이 몇 번 실행됐는지 본다

Tempo에서 느린 trace 하나를 연다.

~~~text
GET /orders 1.8s
 ├─ SELECT orders       22ms
 ├─ SELECT customer      9ms
 ├─ SELECT customer      8ms
 ├─ SELECT customer      9ms
 ├─ SELECT customer      8ms
 └─ ... 반복
~~~

각 SQL 하나만 보면 빠르다.

문제는 같은 종류의 query가 수십 번, 수백 번 반복된다는 점이다.

## 2. 느린 SQL만 찾으면 놓칠 수 있다

일반적인 slow query 분석은 오래 걸린 SQL을 찾는 데 강하다.

하지만 N+1에서는 각각의 query가 짧다.

그래서 다음 질문이 더 중요하다.

> 한 request에서 같은 query가 몇 번 실행됐는가?

## 3. query summary로 묶어 본다

원문 SQL 전체보다 query summary를 이용하면 같은 종류의 query를 묶기 쉽다.

예:
~~~text
SELECT orders        x1
SELECT customer      x120
~~~

이제 문제가 선명해진다.

## 4. 데이터 양과 호출 수를 비교한다

10건을 조회할 때 customer query가 10번, 100건을 조회할 때 100번이라면 관계가 거의 그대로 드러난다.

~~~text
rows=10   → child query 10
rows=50   → child query 50
rows=100  → child query 100
~~~

이런 패턴은 단일 SQL latency보다 훨씬 강한 증거다.

## 5. 소스코드에서 반복 접근을 찾는다

Repository 하나만 보는 것이 아니라 loop 안에서 lazy relation이나 추가 조회가 발생하는지 본다.

예를 들어 각 Order를 순회하면서 customer를 따로 조회하고 있을 수 있다.

## 6. SQL parameter는 굳이 보여줄 필요가 없다

이 문제를 찾는 데 customer ID 실제 값은 필요하지 않다.

query 종류와 호출 횟수만으로도 충분하다.

민감한 parameter를 Agent에게 넘기지 않아도 디버깅할 수 있다는 좋은 예다.

## 7. 수정 후 무엇을 확인할까

fetch join, batch fetch, bulk query 등으로 수정한 뒤 같은 요청을 다시 실행한다.

비교할 것은 다음이다.

~~~text
Before
DB spans = 121
latency = 1.8s

After
DB spans = 2
latency = 180ms
~~~

## 8. 이 장에서 기억할 것

> N+1은 '느린 SQL' 문제가 아니라 '너무 많은 SQL' 문제다.

> Agent가 query 시간뿐 아니라 한 요청 안의 반복 횟수를 볼 수 있어야 한다.