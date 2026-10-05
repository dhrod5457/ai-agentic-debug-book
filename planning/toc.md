# 목차 v0.2

기준일: 2026-10-05

## 이 책이 묻는 질문

> 소스코드만 보는 Coding Agent에게 실제 애플리케이션의 실행 상태를 어떻게 보여줄 것인가?

이 책은 AI에게 로그를 많이 넣는 방법을 설명하지 않는다.

대신 사람이 평소 하던 디버깅 과정을 Agent도 따라갈 수 있게 만드는 방법을 다룬다.

---

# Part I. 로그를 주는 것과 디버깅을 가능하게 하는 것은 다르다

## 1장. 소스코드만 보는 Agent는 애플리케이션을 모른다
- 코드와 실제 실행 상태는 다르다
- stack trace 하나로 충분하지 않은 이유
- 같은 요청의 정보를 서로 연결해야 하는 이유

## 2장. 로그 파일을 통째로 넣으면 왜 실패하는가
- 로그가 많다고 더 잘 찾는 것은 아니다
- 모델이 읽는 내용과 전체 관측 데이터는 다르다
- 모든 로그를 넣기보다 필요한 자료를 찾아보게 한다

## 3장. 디버깅 증거는 한 종류가 아니다
- 지표, trace, 로그, profile
- DB와 JVM 정보
- Kubernetes와 배포 상태
- 가벼운 확인부터 깊은 진단까지

---

# Part II. 관측 시스템을 Agent의 눈과 귀로 바꾼다

## 4장. OpenTelemetry를 상관관계의 뼈대로 사용한다
- 같은 요청의 로그와 trace 연결
- Spring Boot에서 trace 전파
- Collector에서 filtering과 민감정보 처리
- 실행 버전도 함께 남긴다

## 5장. Prometheus로 문제 공간을 먼저 줄인다
- 언제, 어느 서비스가 이상한지 확인
- 평소 값과 비교
- 지표에서 실제 느린 요청으로 내려가기

## 6장. Tempo로 실제 실행 경로를 따라간다
- 느린 요청 하나 찾기
- 어디에서 시간이 쓰였는지 확인
- 정상 요청과 느린 요청 비교
- trace가 끊긴 경우도 단서로 보기

## 7장. Loki에서 같은 실행의 로그를 찾는다
- trace ID로 로그 범위 줄이기
- 반복 로그를 패턴으로 묶어 보기
- 긴 stack trace는 필요한 만큼만 펼치기

## 8장. Pyroscope와 JVM 진단으로 더 깊이 내려간다
- CPU를 쓰는지 기다리는지 구분
- profile과 trace 연결
- JFR과 thread dump
- heap dump는 정말 필요할 때만 사용

---

# Part III. 관측 데이터를 Agent가 직접 조회하게 만든다

## 9장. Grafana MCP는 어디까지 해결해 주는가
- Agent가 metrics, logs, traces를 직접 조회
- 읽기 전용과 조회 제한
- Grafana MCP가 해결하지 않는 디버깅 절차

## 10장. Agent가 디버깅할 때 꼭 기억해야 할 것
- 문제 범위
- 운영 버전
- 무엇을 확인했는지 기록
- 사실과 추측 분리
- 재현과 재검증

## 11장. Agent에게 주는 도구는 작고 제한적이어야 한다
- 강력한 shell 하나보다 작은 전용 도구
- 시간 범위와 결과 크기 제한
- JVM/Kubernetes 권한 분리

## 12장. 로그를 찾았다고 원인을 찾은 것은 아니다
- 같이 나타난 것과 원인 구분
- 여러 원인 후보를 하나씩 시험
- 반대 증거도 확인
- 찾지 못한 것과 없는 것을 구분

---

# Part IV. Spring Boot 애플리케이션을 실제로 디버깅한다

## 13장. Spring Boot에 Agent가 읽을 수 있는 흔적을 남긴다
- traceId/spanId
- 구조화 로그
- DB span
- JVM 지표
- service.version

## 14장. DB Connection Pool이 바닥났을 때
- SQL timeout을 보고도 SQL부터 고치지 않는 이유
- Hikari 지표와 trace
- 긴 transaction 찾기
- 같은 부하로 재검증

## 15장. Lock Contention은 로그만으로 보이지 않는다
- CPU는 낮은데 latency가 높은 경우
- wall profile
- thread dump와 JFR
- 전역 lock 수정과 재검증

## 16장. N+1은 느린 SQL 하나가 아니다
- 한 요청의 SQL 반복 횟수 보기
- query summary
- 민감한 parameter 없이도 원인 찾기

## 17장. 장애가 코드가 아닐 때
- Pod restart와 OOMKilled
- eviction과 Node pressure
- Kubernetes Event와 resource metric 함께 보기

## 18장. 이미 다른 버전이 운영 중이라면
- service.version
- image digest
- deployment revision
- 현재 저장소와 운영 버전 비교

---

# Part V. 운영 환경에서 안전하게 연결한다

## 19장. 운영 관측 데이터를 Agent에게 열어도 되는가
- 최소권한
- Agent 전용 계정
- tenant 범위
- 민감정보와 외부 LLM
- query 비용과 감사 기록

## 20장. 원인을 찾은 뒤부터는 권한이 달라진다
- 읽기와 진단 자료 생성
- 로컬 수정과 운영 수정
- 임시 완화와 근본 수정
- 승인과 rollback

## 21장. 수정했다고 끝난 것이 아니다
- 같은 workload로 다시 확인
- 올바른 버전이 배포됐는지 확인
- 처음 장애 신호를 다시 보기
- 부분 해결도 기록

## 22장. Agentic Debugging을 어떻게 평가할까
- 소스만 제공한 경우와 비교
- raw logs와 관측 도구 비교
- 정답뿐 아니라 증거·비용·안전성 평가
- 반복 실행과 실패 원인 분류

---

# 맺으며. 더 많은 로그보다 더 좋은 관측 인터페이스

> Agent에게 모든 정보를 넣는 것이 아니라, 필요한 증거를 안전하게 찾고 확인할 수 있는 길을 만들어야 한다.