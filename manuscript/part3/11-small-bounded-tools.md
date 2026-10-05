# 11장. Agent에게 주는 도구는 작고 제한적이어야 한다

Agent에게 운영 시스템 접근 권한을 줄 때 가장 쉬운 방법은 shell 하나를 주는 것이다.

curl도 할 수 있고, kubectl도 할 수 있고, SQL도 실행할 수 있다.

유연하다.

하지만 운영에서는 너무 넓다.

## 1. 도구가 넓으면 실수 범위도 넓어진다

예를 들어 Agent가 로그를 보기 위해 shell을 쓴다고 하자.

실수로 파일을 지울 수도 있고, 잘못된 서버에 접속할 수도 있고, 너무 넓은 조회를 실행할 수도 있다.

그래서 운영용 디버깅 도구는 목적을 작게 나누는 편이 낫다.

## 2. 질문 하나에 도구 하나

예를 들면 다음 정도다.

~~~text
지표 보기
느린 trace 찾기
특정 trace 로그 보기
Pod 상태 보기
배포 버전 보기
JFR 요약 보기
~~~

각 도구는 할 수 있는 일이 작다.

대신 감사와 제한이 쉬워진다.

## 3. 조회 범위를 도구가 강제한다

Agent가 매번 '30분만 검색해'라는 지시를 잘 지킬 것이라고 기대하지 않는다.

도구가 직접 제한한다.

~~~text
최대 시간 범위
최대 결과 수
최대 scan 크기
허용 environment
허용 service
~~~

이런 제한은 prompt보다 강하다.

## 4. 결과도 너무 많이 주지 않는다

도구는 결과 전체보다 먼저 요약을 줄 수 있다.

예를 들어 trace 조회 결과는 처음에:

- 가장 느린 span
- error span
- 서비스 이동
- 전체 duration

정도만 주고, 필요할 때 상세 span을 더 본다.

이 방식은 사람의 UI와 비슷하다.

처음부터 모든 세부정보를 펼쳐놓지 않는다.

## 5. JVM 진단 도구는 별도로 다룬다

기존 metric을 읽는 것과 thread dump를 새로 뜨는 것은 다르다.

그래서 JFR, thread dump, heap dump 같은 기능은 일반 조회 도구와 나누는 것이 좋다.

예:

~~~text
일반 조회
metrics / logs / traces

추가 진단
JFR / thread dump

고비용 진단
heap dump
~~~

## 6. Kubernetes도 읽기와 실행을 나눈다

Pod 상태를 보는 것과 Pod 안에 exec로 들어가는 것은 다르다.

기본 Agent에는 get/list 정도만 주고, exec나 restart는 별도 권한으로 둔다.

## 7. Grafana MCP는 좋은 기본 재료다

Prometheus, Loki, Tempo, Pyroscope 쪽은 Grafana MCP가 이미 많은 기능을 제공한다.

따라서 처음부터 새 MCP 서버를 전부 만들 필요는 없다.

필요한 것은 그 위에서 범위와 권한을 더 좁히는 일이다.

## 8. 범용 Python 도구는 어디에 쓸까

OpenRCA는 Python executor를 이용해 관측 데이터를 자유롭게 분석한다.

연구나 offline 분석에서는 매우 유연하다.

하지만 운영에서는 범위가 너무 넓을 수 있다.

그래서 이 책에서는:

~~~text
Production
작은 전용 도구 우선

Offline / Sandbox
필요하면 Python 분석 허용
~~~

정도로 나누는 편을 권한다.

## 9. 이 장에서 기억할 것

> Agent에게 강력한 도구 하나보다 작고 제한된 도구 여러 개를 주는 편이 운영에서는 안전하다.

> 중요한 제한은 prompt가 아니라 도구 자체에서 강제한다.

### 주요 근거

- [S-GRAFANA-MCP] Grafana MCP
- [S-OPENRCA] OpenRCA