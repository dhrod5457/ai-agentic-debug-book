# Deep Dive — Grafana MCP as an Agent Observability Gateway

기준일: 2026-10-05

## 결론

Grafana MCP는 이 책에서 가정한 Agent Debug Evidence Gateway에 가장 가까운 현재 오픈소스 구현이다.

중요한 점은 단순히 Prometheus/Loki/Tempo를 MCP로 감쌌다는 것이 아니다. Tool exposure, query execution, write operation, datasource scope를 서로 다른 control로 나눠 Agent 권한을 제한한다.

## 1. Tool Surface

공식 문서와 저장소에서 확인한 주요 debugging 관련 tool category:

- Prometheus: query_prometheus, histogram query
- Loki: query_loki_logs, query patterns, stats/label analysis
- Tempo: TraceQL search, trace-derived metrics, trace fetch/diff, attribute exploration
- Pyroscope: profile query
- Dashboard: summary/property/panel query
- Datasource discovery
- Alert/Incident/OnCall 계열

전체 dashboard JSON보다 summary/property를 우선하도록 공식 문서가 안내한다. 이는 context window를 고려한 interface design 사례다.

## 2. 세 개의 독립 Control

### Tool exposure
`--enabled-tools`, `--disable-<category>`로 Agent에게 보이는 capability 자체를 줄인다.

### Query execution
`--disable-query`는 datasource query를 실행하는 tool을 제거한다. metadata/discovery는 남길 수 있다.

### Write capability
`--disable-write`는 create/update 계열을 제거한다. query는 기본적으로 남는다.

이 구조는 다음 원칙으로 일반화할 수 있다.

Capability Discovery ≠ Query Authority ≠ Mutation Authority

## 3. Read-only가 단순한 이름이 아닌 이유

raw SQL/Influx query는 datasource credential이 쓰기 가능하면 query 자체가 mutation이 될 수 있으므로 read-only mode에서 제거된다.

따라서 Agent gateway의 read-only는 HTTP method나 tool 이름으로 판정하면 안 된다. backend semantic까지 고려해야 한다.

원칙 후보:
- Read-only는 tool semantic + backend credential을 함께 검증한다.
- arbitrary query language는 특별 취급한다.

## 4. Loki Query Guardrail

저장소에서 확인되는 주요 guardrail:

- single query 최대 scan bytes
- 최대 effective time range
- 모든 native Loki query에 강제로 AND하는 label matcher
- query execution 전체 disable 가능

예를 들어 production/staging처럼 Agent가 읽을 수 있는 stream을 label matcher로 제한할 수 있다.

이 구현은 책의 중요한 패턴 후보다.

Agent Query Budget = time range + scan bytes + result limit + scope

## 5. RBAC

Grafana MCP tool은 Grafana service account와 RBAC permission/scope를 사용한다.

예:
- datasources:query + datasources:uid:<uid>
- dashboards:read + dashboards:uid:<uid>

즉 Agent에게 organization 전체 datasource 권한을 줄 필요가 없다.

원칙 후보:
- Agent identity에 observability 전용 service account를 사용한다.
- datasource/service/environment 단위 최소권한을 우선한다.

## 6. Debugging System에 그대로 쓰기 어려운 점

Grafana MCP는 범용 Grafana assistant interface다. Coding Agent debugging workflow 자체를 강제하지는 않는다.

부족한 계층:
- incident scope contract
- evidence provenance
- deploy/commit version correlation
- hypothesis/evidence distinction
- patch/reproduction/verification loop
- evidence budget accounting

따라서 책에서는 Grafana MCP를 대체하지 않고 위에 Debug Workflow Layer를 둔다.

구조:
Grafana MCP / Tempo MCP
  ↓
Debug Policy Gateway
  ↓
Evidence Selection
  ↓
Coding Agent

## 7. 실전 baseline 권고

첫 실습 baseline은 다음처럼 잡을 수 있다.

- Grafana MCP
- disable-write
- production datasource UID만 필요한 범위로 허용
- Loki enforced matchers로 environment/service scope 제한
- Loki scan byte/time range 제한
- Agent가 먼저 metadata discovery 후 bounded query 실행
- Tempo trace와 Prometheus metric은 query limit을 별도로 둠

이 baseline에서 custom Debug Gateway가 추가로 필요한지 실험한다.