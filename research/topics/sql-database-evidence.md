# Topic — SQL and Database Evidence for Agentic Debugging

기준일: 2026-10-05

## 핵심 질문

> Agent가 SQL/DB 문제를 디버깅하려면 query text와 parameter를 어디까지 보여줘야 하는가?

## 1. OpenTelemetry DB semantic conventions

현재 OpenTelemetry SQL database semantic conventions는 `db.query.summary`, `db.query.text`, database system/server, returned rows, error type 등을 표준 attribute로 정의한다.

특히 중요한 privacy rule:
- non-parameterized query text는 literal value를 redaction하는 sanitization 없이는 기본 수집하지 않는 것을 권고한다.
- query parameter value는 PII/민감정보가 있을 수 있으므로 기본 수집하지 않는 것을 권고한다.

## 2. Agent에는 Query Summary를 우선한다

`db.query.summary`는 low-cardinality이고 dynamic/sensitive data를 포함하지 않아야 한다.

따라서 Agent evidence 단계는 다음처럼 설계하는 것이 안전하다.

1. db.query.summary
2. duration / error.type / returned_rows
3. sanitized db.query.text
4. parameter value는 특별 승인/정책 없이는 제공하지 않음

## 3. SQL evidence hierarchy

Metric:
- connection pool active/pending
- query latency histogram
- error rate
- transaction wait

Trace:
- DB client span
- db.query.summary
- duration
- server address
- error.type

Log:
- timeout/deadlock/constraint violation
- driver/pool warning

DB-side:
- slow query / execution plan
- lock/wait state
- connection/session state

Agent는 애플리케이션-side evidence만 보고 DB root cause를 단정하면 안 된다.

## 4. query text가 root cause가 아닐 수 있다

동일 SQL이 느려져도 원인은:
- lock wait
- pool exhaustion
- bad plan
- network
- DB CPU/I/O
- transaction scope
- retry amplification
일 수 있다.

따라서 SQL text와 execution context를 함께 봐야 한다.

## 5. Tool surface 후보

- get_slow_db_spans(service, range, limit)
- group_db_queries_by_summary(service, range)
- get_sanitized_query(span_id)
- get_db_errors(range)
- get_pool_metrics(service, range)
- get_db_wait_summary(range)

arbitrary production SQL execution tool은 observability tool과 분리한다.

## 6. 설계 원칙 후보

### DB-P1. Query Summary First
원문 SQL보다 low-cardinality summary와 latency/error evidence를 먼저 조회한다.

### DB-P2. Parameters Default Deny
query parameter 값은 기본적으로 Agent evidence에서 제외한다.

### DB-P3. App Span ≠ DB State
애플리케이션 DB span과 실제 DB-side state를 구분한다.

### DB-P4. Read Observability ≠ SQL Execution
Agent에게 DB telemetry 조회 권한을 주는 것과 임의 SQL 실행 권한을 주는 것을 분리한다.