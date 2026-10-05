# Spring Boot Failure Corpus Design v0.1

기준일: 2026-10-05

상태: Experiment Design Draft

## 1. 목적

이 corpus의 목적은 Coding Agent에게 runtime evidence를 제공하는 방식이 root-cause diagnosis와 patch/verification 성능에 실제 영향을 주는지 비교하는 것이다.

단순히 어려운 bug 모음을 만드는 것이 목적이 아니다. 각 failure는 서로 다른 evidence surface가 필요하도록 설계한다.

## 2. 공통 애플리케이션 형태

소규모지만 production debugging 패턴을 재현하는 Spring Boot 기반 multi-service sample을 사용한다.

필요 구성:
- API/Gateway 또는 단일 ingress
- 2~3 Spring Boot services
- relational DB
- Redis 또는 cache optional
- Prometheus
- Loki
- Tempo
- Pyroscope
- Grafana
- OpenTelemetry/Micrometer
- optional Kubernetes deployment profile

## 3. Failure Families

### F1. DB Connection Pool Exhaustion

증상:
- 일부 API p99 latency 급증
- timeout 또는 5xx

필수 evidence:
- HTTP latency metric
- Hikari active/pending metric
- representative trace
- DB acquisition/operation span
- pool warning log

의도된 root cause:
- connection leak 또는 지나치게 긴 transaction로 pool 고갈

잘못된 유혹:
- 느린 SQL 자체를 root cause로 오판

### F2. Lock Contention / Thread Starvation

증상:
- CPU는 높지 않지만 latency 증가
- request thread 대기

필수 evidence:
- request latency
- slow trace
- wall/lock profile
- JFR 또는 thread dump projection

잘못된 유혹:
- DB/network latency로 단정

### F3. Memory Pressure / Allocation Regression

증상:
- GC pause 증가
- heap usage 상승
- throughput 저하

필수 evidence:
- JVM heap/GC metrics
- allocation profile/JFR
- runtime version
- source diff

escalation:
- heap dump는 기본 gold path에 포함하지 않고 추가 evidence로만 허용

### F4. Bad Deployment Regression

증상:
- deployment 직후 error rate 증가

필수 evidence:
- error metric
- service.version/image digest
- deployment revision/time
- trace/log
- previous/current source diff

핵심 평가:
- Agent가 current main만 보고 엉뚱한 patch를 하지 않는가

### F5. Trace Context Propagation Break

증상:
- downstream 오류는 존재하지만 end-to-end trace가 끊김

필수 evidence:
- partial trace
- downstream independent trace/log
- client construction/config source

root cause:
- auto-configured traced HTTP client를 우회한 custom client 등

핵심 평가:
- Agent가 'evidence 없음'과 'correlation 깨짐'을 구분하는가

### F6. Pod Instability / OOM or Eviction

증상:
- intermittent 5xx
- restart 증가

필수 evidence:
- Pod/container state
- restart count
- Kubernetes Event
- memory metric
- image/deployment version

잘못된 유혹:
- application exception만 찾아 code bug로 오판

### F7. External HTTP Timeout + Retry Amplification

증상:
- 외부 dependency 장애가 내부 thread/connection 자원까지 압박

필수 evidence:
- outbound spans
- retry count/log
- thread/connection metric
- service dependency graph

핵심 평가:
- 최초 외부 장애와 내부 증폭 원인을 구분하는가

### F8. N+1 / Query Explosion

증상:
- 특정 list API만 데이터량에 따라 급격히 느려짐

필수 evidence:
- route latency
- DB span count per trace
- db.query.summary grouping
- sanitized query/source

핵심 평가:
- 단일 slow query가 아니라 query multiplicity를 발견하는가

## 4. 각 Failure의 Gold Artifact

각 scenario에는 다음을 고정한다.

- failure_id
- injected change/bug
- runtime identity
- incident time range
- user-visible symptom
- gold root cause
- minimum sufficient evidence set
- optional supporting evidence
- misleading evidence
- expected patch location
- reproduction command/workload description
- verification criteria

## 5. Evidence Leakage 방지

benchmark가 쉽게 뚫리지 않도록 다음을 피한다.

- filename에 root-cause 명시
- log message에 직접 'ROOT_CAUSE' 삽입
- issue description에 원인 포함
- git history에서 정답 patch를 바로 읽을 수 있는 상태

대신 evidence를 조합해야만 원인을 알 수 있게 한다.

## 6. 난이도 단계

Tier 1:
- single service
- one dominant signal
- direct stack/trace evidence

Tier 2:
- multi-signal correlation 필요
- distractor telemetry 존재

Tier 3:
- cross-service 또는 platform/runtime evidence 필요
- version mismatch/partial telemetry/sampling 같은 ambiguity 포함

## 7. Corpus의 역할

이 corpus는 일반 SWE benchmark를 대체하지 않는다.

측정 대상은 다음으로 한정한다.

> Application Runtime Evidence access 방식이 동일한 Coding Agent의 debugging trajectory와 결과를 어떻게 바꾸는가?