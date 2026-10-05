# 원고 읽는 순서

출간용 통합 원고: [BOOK.md](BOOK.md)

통합 References: [references.md](references.md)

이 디렉터리는 1차 초고와 1차 편집이 반영된 본문입니다.

## 들어가며

- [들어가며 — Agent에게 애플리케이션을 보여주는 법](00-preface.md)

## Part I. 로그를 주는 것과 디버깅을 가능하게 하는 것은 다르다

1. [소스코드만 보는 Agent는 애플리케이션을 모른다](part1/01-source-code-is-not-runtime.md)
2. [로그 파일을 통째로 넣으면 왜 실패하는가](part1/02-raw-logs-are-not-context.md)
3. [디버깅 증거는 한 종류가 아니다](part1/03-evidence-pyramid.md)

## Part II. 관측 시스템을 Agent의 눈과 귀로 바꾼다

4. [OpenTelemetry를 상관관계의 뼈대로 사용한다](part2/04-opentelemetry-backbone.md)
5. [Prometheus로 문제 공간을 먼저 줄인다](part2/05-prometheus-narrowing.md)
6. [Tempo로 실제 실행 경로를 따라간다](part2/06-tempo-execution-path.md)
7. [Loki에서 같은 실행의 로그를 찾는다](part2/07-loki-correlated-logs.md)
8. [Pyroscope와 JVM 진단으로 더 깊이 내려간다](part2/08-pyroscope-jvm-diagnostics.md)

## Part III. 관측 데이터를 Agent가 직접 조회하게 만든다

9. [Grafana MCP는 어디까지 해결해 주는가](part3/09-grafana-mcp.md)
10. [Agent가 디버깅할 때 꼭 기억해야 할 것](part3/10-debug-session.md)
11. [Agent에게 주는 도구는 작고 제한적이어야 한다](part3/11-small-bounded-tools.md)
12. [로그를 찾았다고 원인을 찾은 것은 아니다](part3/12-evidence-and-reasoning.md)

## Part IV. Spring Boot 애플리케이션을 실제로 디버깅한다

13. [Spring Boot에 Agent가 읽을 수 있는 흔적을 남긴다](part4/13-spring-observability-baseline.md)
14. [DB Connection Pool이 바닥났을 때](part4/14-db-pool-exhaustion.md)
15. [Lock Contention은 로그만으로 보이지 않는다](part4/15-lock-contention.md)
16. [N+1은 느린 SQL 하나가 아니다](part4/16-n-plus-one.md)
17. [장애가 코드가 아닐 때](part4/17-platform-failure.md)
18. [이미 다른 버전이 운영 중이라면](part4/18-runtime-version.md)

## Part V. 운영 환경에서 안전하게 연결한다

19. [운영 관측 데이터를 Agent에게 열어도 되는가](part5/19-production-observability-access.md)
20. [원인을 찾은 뒤부터는 권한이 달라진다](part5/20-from-diagnosis-to-change.md)
21. [수정했다고 끝난 것이 아니다](part5/21-verification.md)
22. [Agentic Debugging을 어떻게 평가할까](part5/22-evaluation.md)

## 맺으며

- [더 많은 로그보다 더 좋은 관측 인터페이스](23-epilogue.md)

## 편집 상태

- 1차 초고: 완료
- 1차 가독성 편집: 완료
- 후반부 사례 보강: 완료
- 장간 중복 1차 정리: 완료
- 출처 포인터 1차 정리: 완료
- 문체 통일 2차 패스: 완료
- References 정규화: 완료
- 출간용 통합 원고: 완료
- Freshness audit: 완료
- 최종 오탈자/구조 검사: 완료