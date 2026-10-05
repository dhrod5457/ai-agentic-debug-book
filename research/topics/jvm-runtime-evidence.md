# Topic — JVM Evidence: JFR, Thread Dumps, Heap and Profiles

기준일: 2026-10-05

## 핵심 질문

> Spring Boot/Java 장애에서 logs/metrics/traces로 원인이 충분히 좁혀지지 않을 때 Agent에게 어떤 JVM runtime evidence를 추가로 제공할 것인가?

## 1. JFR은 production-side evidence source가 될 수 있다

Oracle JDK 25 문서에서 JFR은 실행 중 JVM의 상세 runtime event를 기록하는 공식 기능이다. `jcmd`의 `JFR.start`, `JFR.check`, `JFR.dump`, `JFR.stop` 명령은 낮은 impact로 기록을 제어할 수 있다.

`default` recording은 continuous recording에 적합한 low-overhead 설정으로 설명되고, `profile` 설정은 더 많은 데이터를 수집하는 profiling 목적이다.

Agentic Debugging 의미:
- 항상 heap dump를 뜨는 것보다 저비용 continuous JFR을 기본 evidence로 둘 수 있다.
- incident window만 `JFR.dump begin/end`로 잘라 Agent 분석 대상으로 넘길 수 있다.
- event type을 제한해 evidence volume을 제어할 수 있다.

## 2. JFR은 '파일 전체 전달' 대상이 아니다

JFR은 매우 많은 event를 담을 수 있다. Oracle 문서도 필요한 event와 duration threshold를 제한해야 overhead를 낮출 수 있다고 설명한다.

따라서 Agent interface는 `.jfr` 파일 자체보다 query/projection을 제공하는 것이 적절하다.

예상 Tool:
- summarize_jfr(time_range)
- list_jfr_events(type, range, limit)
- get_gc_pause_events(range)
- get_lock_contention(range)
- get_thread_park_events(range)
- get_socket_io_events(range)
- get_allocation_hotspots(range)

## 3. Thread Dump

`jcmd`는 JVM diagnostic command surface를 제공한다. Thread dump는 다음 failure에서 직접적 evidence가 된다.

- deadlock
- thread pool starvation
- blocking I/O
- lock contention
- request thread exhaustion

Agent에게는 전체 thread dump보다 다음 projection을 우선 제공한다.

- thread state histogram
- blocked/waiting top stacks
- identical stack groups
- deadlock cycle
- executor/thread-pool별 active/idle pattern
- application package frames

## 4. Heap Dump는 고비용/고민감 evidence다

Heap evidence는 memory leak, retained object, cache blow-up에서 강력하지만 수집/저장 비용과 개인정보 노출 위험이 크다.

JFR dump의 GC-root path도 Oracle 문서상 pause를 일으킬 수 있어 memory leak 의심 시에만 사용하도록 주의한다.

원칙 후보:
- JVM Evidence Escalation Level을 둔다.
- low-cost telemetry → JFR summary → thread dump → heap dump 순으로 escalation한다.

## 5. Pyroscope와 JFR의 관계

Grafana Pyroscope Java integration은 JFR format을 지원하며 CPU, allocation, lock 등 여러 profiling event를 지속적으로 전송할 수 있다.

또 Java Span Profiles는 OpenTelemetry span ID를 profile sample과 연결한다. 따라서 특정 slow trace span의 CPU/wall profile을 조회할 수 있다.

이 구조는 Agent가 service-wide profile보다 incident execution scope에 가까운 profile을 조회하게 해준다.

흐름:
slow trace span
→ pyroscope.profile.id / span_id
→ span-specific CPU or wall profile
→ hot method / source line

## 6. JFR와 Pyroscope를 함께 보는 이유

Pyroscope는 continuous profile과 trace correlation에 강하고, JFR은 JVM event taxonomy와 incident-window diagnostics에 강하다.

둘은 대체 관계보다 complementary evidence로 보는 것이 맞다.

## 7. 설계 원칙 후보

### JVM-P1. Expensive Evidence Escalation
가벼운 signal로 먼저 좁힌 뒤 고비용 diagnostic을 요청한다.

### JVM-P2. Dump ≠ Context
heap/thread/JFR dump 원본을 prompt에 넣지 않고 분석 projection을 만든다.

### JVM-P3. Execution Correlation
가능하면 trace/span/time/service/version과 JVM evidence를 연결한다.

### JVM-P4. Capture Authority ≠ Read Authority
JFR/heap dump 생성 권한은 단순 관측 조회보다 높은 위험 권한으로 취급한다.