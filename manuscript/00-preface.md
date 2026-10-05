# 들어가며 — Agent에게 애플리케이션을 보여주는 법

코딩 Agent는 소스코드를 잘 읽는다. 파일을 찾고, 함수를 따라가고, 테스트를 실행하고, 오류 메시지를 보고 수정안을 만든다. 로컬에서 재현 가능한 버그라면 이것만으로도 꽤 많은 문제를 해결할 수 있다.

문제는 애플리케이션이 실행 중일 때 시작된다.

운영에서 특정 API의 응답 시간이 갑자기 3초를 넘는다. 간헐적으로 500 응답이 발생한다. 특정 배포 이후부터 오류율이 올라간다. 어떤 요청은 정상인데 어떤 요청은 같은 코드에서 실패한다. 로그에는 SQL timeout이 보이지만 데이터베이스 CPU는 정상이다. Pod는 재시작됐지만 애플리케이션 stack trace에는 특별한 예외가 없다.

사람은 이런 문제를 소스코드만 보고 풀지 않는다.

Grafana를 열어 지표를 본다. Prometheus에서 오류율과 지연시간을 확인한다. Tempo에서 느린 trace를 찾는다. Loki에서 같은 trace ID를 가진 로그를 검색한다. 필요하면 profiler를 열고, thread dump를 뜨고, Kubernetes Event와 배포 revision을 확인한다. 데이터베이스 connection pool과 slow query를 비교한다. 마지막에는 장애가 발생한 버전과 현재 checkout된 코드가 같은지도 확인한다.

그런데 Coding Agent에게는 종종 이 중 아무것도 주어지지 않는다.

대신 "운영에서 느리대. 코드 봐줘."라는 요청이나 application.log 전체가 전달된다.

이 책은 이 간극을 다룬다.

> 소스코드만 보는 Coding Agent에게 실제 애플리케이션의 실행 상태를 어떻게 보여줄 것인가?

이 질문을 풀기 위해 새로운 APM을 만들 필요는 없다. 이미 소프트웨어 산업은 오랫동안 관측 시스템 문제를 해결해 왔다. OpenTelemetry는 logs, metrics, traces를 공통 맥락으로 연결한다. Prometheus는 metric을 조회한다. Loki는 log를 저장하고 검색한다. Tempo는 distributed trace를 찾는다. Pyroscope는 continuous profile을 제공한다. Grafana는 이 신호들을 사람이 탐색할 수 있게 묶는다.

2026년에는 이 경계가 한 단계 더 움직였다. Grafana와 Tempo는 Agent가 관측 데이터를 직접 질의할 수 있는 MCP 인터페이스까지 제공하기 시작했다.

따라서 질문은 더 이상 "AI가 로그를 읽을 수 있는가?"가 아니다.

> Agent에게 어떤 증거를, 어떤 범위로, 어떤 순서로, 어떤 권한 아래에서 조회하게 해야 하는가?

이 책에서는 이를 실행 증거(Runtime Evidence)라고 부른다. 이 용어는 외부 표준이 아니라 이 책의 설명을 위해 정리한 표현이다.

실행 증거에는 로그만 들어가지 않는다.

~~~text
Request
  ├─ trace
  ├─ span
  └─ HTTP/RPC

Application
  ├─ structured log
  ├─ exception
  └─ state transition

Database
  ├─ query summary
  ├─ latency/error
  └─ connection pool

Runtime
  ├─ JVM
  ├─ JFR
  ├─ thread
  └─ profile

Platform
  ├─ Pod
  ├─ Deployment
  ├─ Event
  └─ Node

Artifact
  ├─ service.version
  ├─ image digest
  ├─ deployment revision
  └─ source commit
~~~

이 정보는 모두 같은 가치와 비용을 가지지 않는다. Prometheus metric 조회는 저렴하지만 heap dump는 비싸다. trace는 요청 경로를 보여주지만 원인을 자동으로 알려주지는 않는다. Kubernetes Event는 힌트지만 canonical truth가 아니다. SQL text는 유용하지만 parameter에는 개인정보가 들어갈 수 있다.

그래서 이 책은 "더 많은 정보"보다 "더 좋은 관측 인터페이스"를 설계하는 데 집중한다.

책 전체에서 반복해서 지킬 경계는 다음과 같다.

~~~text
Raw Log ≠ Debug Context
Dashboard ≠ Agent Interface
Telemetry Storage ≠ Model Context
Correlation ≠ Root Cause
Anomaly ≠ Root Cause
Evidence Retrieval ≠ Reasoning
Observation Authority ≠ Mutation Authority
Test Pass ≠ Incident Resolution
~~~

1부에서는 왜 소스코드와 로그만으로는 부족한지 설명한다. 2부에서는 OpenTelemetry, Prometheus, Tempo, Loki, Pyroscope를 Agent의 눈과 귀로 바꾸는 방법을 본다. 3부에서는 Grafana MCP를 포함한 실제 Agent interface와 그 위의 Debug Workflow를 설계한다. 4부에서는 Spring Boot 장애를 실제 사례로 따라간다. 5부에서는 운영 권한, 비용, 보안, 검증, 평가를 다룬다.

소스코드는 프로그램이 무엇을 하도록 작성되었는지를 보여준다.

실행 증거는 프로그램이 실제로 무엇을 했는지를 보여준다.

Agentic Debugging은 이 두 세계를 연결하는 일에서 시작한다.