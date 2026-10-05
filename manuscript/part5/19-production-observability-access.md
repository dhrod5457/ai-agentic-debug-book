# 19장. 운영 관측 데이터를 Agent에게 열어도 되는가

지금까지는 Agent가 Prometheus, Loki, Tempo, Pyroscope를 직접 조회하는 흐름을 만들었다.

여기서 자연스럽게 다음 질문이 나온다.

> 운영 데이터를 Agent에게 직접 보여줘도 괜찮을까?

기술적으로 가능하다는 것과 운영에서 허용해도 된다는 것은 다른 문제다.

## 1. 가장 작은 권한에서 시작한다

처음부터 운영 전체를 보여줄 필요는 없다.

예를 들어 login-service 장애만 조사한다면 Agent에게 필요한 것은 다음 정도다.

~~~text
environment = production
service = login-service
time = 09:10~09:20
~~~

이 범위를 벗어나는 조회는 gateway가 막을 수 있다.

## 2. 기본은 읽기 전용이다

첫 단계에서 필요한 것은 대부분 이런 정보다.

- 지표
- 로그
- trace
- profile
- Pod 상태
- 배포 버전

이 단계에서는 read-only 권한으로 충분하다.

대시보드 수정, alert 변경, restart, 배포 같은 기능은 따로 둔다.

## 3. Agent 전용 계정을 쓴다

사람 계정을 공유하지 않는다.

Agent 전용 service account를 따로 만들고 필요한 datasource와 environment만 허용한다.

이렇게 하면 Agent가 실수해도 영향 범위를 줄일 수 있다.

## 4. 로그에 있는 민감정보는 생각보다 많다

운영 로그에는 다음 값이 들어갈 수 있다.

- Authorization header
- cookie
- 사용자 식별자
- 요청 body
- SQL parameter
- 내부 URL

이런 값을 prompt 직전에만 가리는 것으로 충분하지 않을 수 있다.

가능하면 OpenTelemetry Collector 같은 수집 단계에서부터 제거하거나 mask한다.

## 5. 외부 모델을 쓰면 데이터가 어디로 가는지 다시 본다

사내 Agent가 외부 SaaS LLM을 호출한다면, Loki에서 가져온 로그 일부가 외부 provider로 전송될 수 있다.

따라서 다음 질문이 필요하다.

- 어떤 데이터 등급까지 외부 전송이 허용되는가?
- 원문 대신 요약만 보낼 수 있는가?
- tenant/user 식별자를 제거했는가?
- trace 자체에 payload가 들어 있지 않은가?

도구 연결이 가능하다고 해서 데이터 반출이 자동으로 허용되는 것은 아니다.

## 6. 멀티테넌트라면 tenant 경계를 조회에 강제한다

한 고객의 장애를 조사하는 Agent가 다른 고객 로그를 읽어서는 안 된다.

좋은 구조는 Agent가 tenant 조건을 기억하기를 기대하지 않는다.

gateway가 모든 조회에 tenant matcher를 붙인다.

~~~text
사용자 질문
  ↓
Agent query
  ↓
강제 조건
tenant=A, service=login-service
  ↓
Loki/Tempo/Prometheus
~~~

## 7. 조회 비용도 운영 비용이다

Agent가 반복해서 넓은 로그 검색을 하면 관측 시스템 자체가 느려질 수 있다.

따라서 다음을 제한한다.

- 최대 시간 범위
- 최대 scan 크기
- 최대 결과 수
- 조회 timeout
- 호출 빈도

실제 Grafana MCP의 Loki guardrail이 좋은 참고 사례다. 현재 기본 모드는 `off`이므로 운영에서는 `enforce` 설정 여부를 명시적으로 확인해야 한다.

## 8. 결과 크기도 제한한다

조회는 작아도 결과가 매우 클 수 있다.

그래서 처음에는 요약과 상위 몇 개만 반환하고, Agent가 필요할 때 더 요청하게 할 수 있다.

중요한 것은 결과가 잘렸다면 반드시 표시하는 것이다.

## 9. 감사 기록은 나중에 필요해진다

Agent가 어떤 조회를 실행했는지 남겨야 한다.

장애가 끝난 뒤 다음을 확인할 수 있어야 한다.

- 어떤 데이터를 봤는가
- 어느 시간대를 조회했는가
- 어떤 범위를 벗어나려 했는가
- 어떤 결과가 잘렸는가
- 민감정보 차단이 동작했는가

## 10. 작은 운영 예시

login-service만 조사하는 Agent profile을 생각해보자.

~~~text
허용
- production login-service metrics
- production login-service logs
- 관련 Tempo trace
- 해당 deployment status

차단
- 다른 서비스 로그
- 24시간 초과 query
- raw SQL 실행
- Pod exec
- deployment 변경
~~~

이 정도만 해도 실제 debugging의 대부분을 시작할 수 있다.

## 이 장의 한 문장

> 운영 관측 권한은 처음부터 최소한으로 준다.

> 읽기 권한, 데이터 범위, 조회 비용, 민감정보를 각각 따로 통제한다.

### 참고 자료

출처 상세: [References](../references.md)

- [S-GRAFANA-MCP] Grafana MCP
- [S-OTEL-TRANSFORM] OpenTelemetry — Transforming telemetry
