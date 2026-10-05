# Topic — Runtime to Source Version Correlation

기준일: 2026-10-05

## 문제

Agent가 production incident의 trace를 보고 현재 main branch 코드를 수정한다고 해도, incident가 다른 binary/image/commit에서 발생했다면 잘못된 patch가 나올 수 있다.

따라서 Runtime Evidence에는 source/version identity가 포함되어야 한다.

## 1. OpenTelemetry 표준 속성

OpenTelemetry service semantic conventions에서 `service.version`은 stable recommended attribute이며 예시로 semantic version과 commit-like hash 모두 가능하다.

`deployment.environment.name`도 stable attribute다.

container semantic conventions는 다음 identifier를 제공한다.

- container.image.id
- container.image.name
- container.image.repo_digests
- container.image.tags

## 2. Kubernetes에서 service.version 결정

OpenTelemetry의 Kubernetes resource mapping guidance는 `service.version`을 다음 우선순위로 계산할 수 있다고 제안한다.

1. `resource.opentelemetry.io/service.version` annotation
2. `app.kubernetes.io/version` label
3. image tag/digest 조합

## 3. 권고 Version Tuple

이 책에서는 설명을 위해 다음을 Runtime Version Tuple 후보로 둔다.

- service.name
- service.version
- deployment.environment.name
- deployment.id or Kubernetes revision
- container.image.repo_digest
- git.commit.sha
- config version

`git.commit.sha`는 이 문서에서 book synthesis로 사용하는 필드이며 OpenTelemetry stable standard attribute라고 단정하지 않는다. 실제 수집에서는 CI/CD annotation/custom resource attribute로 넣을 수 있다.

## 4. Debugging Workflow에서의 사용

Incident evidence를 읽기 전에 Agent는 먼저 묻는다.

- 이 trace는 어떤 service.version에서 나왔는가
- 어떤 image digest가 실행 중이었는가
- 어느 deployment revision인가
- 현재 checkout commit과 일치하는가

불일치하면:
- 해당 commit/tag를 checkout
- 또는 diff를 계산
- 또는 stale incident임을 명시

## 5. 왜 image tag만으로 부족한가

mutable tag를 사용하면 동일 tag가 다른 image를 가리킬 수 있다.

따라서 production incident correlation에는 digest가 더 강하다.

## 6. 설계 원칙 후보

### VER-P1. Runtime Evidence Without Version Is Incomplete
trace/log/metric에 실행 artifact identity가 없으면 source correlation이 불완전하다.

### VER-P2. Mutable Label ≠ Artifact Identity
tag/name보다 digest/commit 같은 immutable identifier를 우선한다.

### VER-P3. Workspace Version Check Before Patch
Agent는 patch 전에 incident runtime과 workspace version 차이를 확인한다.