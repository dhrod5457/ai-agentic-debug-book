# 20장. 원인을 찾은 뒤부터는 권한이 달라진다

Agent가 원인 후보를 찾았다.

이제 코드를 고치면 된다.

여기서부터는 상황이 달라진다.

운영 데이터를 보는 것과 실제 시스템을 바꾸는 것은 위험 수준이 다르기 때문이다.

## 1. 조사 단계에서는 대부분 읽기만 해도 된다

예를 들어 connection pool 문제를 조사하는 동안 Agent는 다음만 해도 충분하다.

- Prometheus 지표 조회
- Tempo trace 조회
- Loki 로그 조회
- 배포 버전 확인

이 단계에서는 운영을 바꿀 이유가 없다.

## 2. 진단 자료를 새로 만드는 순간 권한이 한 단계 올라간다

기존 profile을 읽는 것과 운영 JVM에서 새 thread dump나 JFR을 뜨는 것은 다르다.

heap dump는 더 무겁다.

그래서 단순 조회와 진단 자료 생성도 분리한다.

~~~text
읽기
metrics / logs / traces / 기존 profile

추가 진단
thread dump / JFR

고비용 진단
heap dump / deep DB diagnostics
~~~

Agent가 필요하다고 판단했다고 자동 실행할 필요는 없다.

운영 영향과 민감정보 위험에 따라 승인 단계를 둘 수 있다.

## 3. 로컬 수정과 운영 수정은 완전히 다른 권한이다

Agent가 로컬 repository를 수정하고 테스트를 돌리는 것은 비교적 안전하다.

하지만 운영 배포를 바꾸는 것은 다르다.

그래서 작업을 다음처럼 나누는 편이 낫다.

~~~text
운영 데이터 읽기
  ↓
로컬 코드 수정
  ↓
테스트 실행
  ↓
staging 반영
  ↓
production 반영
~~~

각 단계는 같은 버튼의 강도 차이가 아니라 서로 다른 권한으로 본다.

## 4. 실제로는 원인보다 '급한 완화'가 먼저 필요할 때도 있다

예를 들어 connection pool이 고갈돼 서비스가 거의 멈췄다.

근본 원인은 긴 transaction이지만 수정과 배포에는 시간이 필요하다.

운영자는 임시로 traffic을 줄이거나 인스턴스를 늘리거나 문제 기능을 끌 수 있다.

이런 조치는 근본 원인 수정이 아니다.

그래서 Agent 기록에도 다음을 구분하는 편이 좋다.

~~~text
Mitigation
지금 장애를 줄이는 조치

Fix
원인을 없애는 수정
~~~

임시 조치가 성공했다고 문제 해결로 기록하면 다음 장애 때 같은 문제가 반복된다.

## 5. 운영 변경은 되돌릴 수 있어야 한다

Agent가 수정안을 만들고 staging에서 검증했다고 하자.

그래도 운영에서는 예상하지 못한 문제가 생길 수 있다.

그래서 운영 action에는 최소한 다음이 필요하다.

- 변경 전 버전
- 변경 후 버전
- rollback 경로
- 배포 후 확인할 신호
- 중단 조건

예를 들어 error rate가 일정 수준을 넘으면 자동 rollback하는 식이다.

## 6. 승인도 위험한 작업에 집중한다

모든 Prometheus 조회마다 사람이 승인해야 한다면 시스템을 쓸 수 없다.

반대로 운영 restart나 배포 변경을 무조건 자동 허용하기도 어렵다.

작업을 위험도에 따라 나눈다.

~~~text
낮음
metrics/logs/traces 조회

중간
JFR/thread dump 생성
staging 재시작

높음
production restart
configuration 변경
deployment
~~~

승인은 높은 위험 작업에 집중한다.

## 7. Agent가 만든 수정도 변경 이유가 보여야 한다

단순히 diff만 남기지 않는다.

좋은 수정 기록에는 다음 연결이 있어야 한다.

~~~text
확인한 증거
→ 선택한 원인 후보
→ 바꾼 코드
→ 기대하는 변화
~~~

예:
~~~text
connection acquire 2.8s
+ 긴 transaction 확인
→ 외부 API 호출을 transaction 밖으로 이동
→ pending connection 감소 기대
~~~

이렇게 해야 코드 리뷰도 쉬워진다.

## 8. 작은 자동화부터 시작한다

처음부터 운영 auto-remediation을 목표로 할 필요는 없다.

현실적인 발전 순서는 다음과 같다.

~~~text
1. 원인 후보와 근거 제시
2. patch 제안
3. 로컬 테스트 자동화
4. staging 검증 자동화
5. production 변경은 승인 후 실행
~~~

운영 경험이 쌓인 일부 안전한 작업만 나중에 자동화 범위를 넓힐 수 있다.

## 이 장의 한 문장

> 보는 권한과 바꾸는 권한을 분리한다.

> Agent의 자율성은 운영 변경 권한을 많이 주는 것으로 측정하지 않는다.

### 참고 자료

출처 상세: [References](../references.md)

- [S-GRAFANA-MCP] Grafana MCP
- [S-ORACLE-JCMD] Oracle JDK 25 — jcmd/JFR
