# Topic — Kubernetes Runtime and Deployment Evidence

기준일: 2026-10-05

## 핵심 질문

> 애플리케이션 장애가 코드가 아니라 배포/Pod/Node/Kubernetes 상태에서 발생했을 때 Agent가 어떤 evidence를 봐야 하는가?

## 1. Kubernetes는 application 밖의 failure evidence를 제공한다

Kubernetes 공식 observability 문서는 metrics, logs, traces를 cluster 내부 상태와 application 상태를 이해하기 위한 핵심 signal로 설명한다.

Kubernetes 1.37 기준 control-plane components도 OTLP로 trace를 export할 수 있다.

## 2. Events는 유용하지만 canonical truth가 아니다

Kubernetes Events API는 Event를 cluster state change에 대한 report로 정의하지만 limited retention, evolving trigger/message 특성 때문에 best-effort supplemental data로 취급하라고 명시한다.

따라서 Agent는 Event 하나만으로 root cause를 확정하면 안 된다.

원칙 후보:
Kubernetes Event = Supporting Evidence, not Source of Truth

## 3. Agent에 필요한 Kubernetes evidence

Pod:
- phase/state
- readiness
- restart count
- container state/reason
- node
- image/imageID

Deployment/ReplicaSet:
- desired/current/available replicas
- rollout status
- revision
- image
- generation

Events:
- reason
- type
- action
- regarding/related object
- reportingController
- occurrence count
- lastObservedTime

Node/Resource:
- CPU/memory pressure
- eviction/OOM
- scheduling failure

## 4. Revision correlation

Kubernetes는 ReplicaSet에 `deployment.kubernetes.io/revision` annotation을 관리하며 Pod template이 바뀔 때 revision을 증가시킨다.

이 값은 rollout history와 rollback에 사용된다.

Agentic Debugging에서는 incident 시점의 deployment revision과 current workspace source를 연결하는 보조 identifier로 쓸 수 있다.

## 5. Image Digest가 version correlation에서 중요하다

OpenTelemetry container semantic conventions는 `container.image.id`와 `container.image.repo_digests`를 제공한다.

특히 runtime/environment 사이 동일 이미지를 식별할 때 image digest가 tag보다 강한 identifier가 된다.

## 6. Tool surface 후보

- get_pod_status(service, time)
- get_k8s_events(resource, range)
- get_rollout_history(deployment)
- get_deployment_revision(time)
- get_container_image_digest(pod)
- get_restart_history(pod)
- get_node_pressure(node, range)

## 7. 권한

대부분의 debugging 단계에서는 Kubernetes read-only API면 충분하다.

`exec`, rollout restart, scale, patch, delete는 별도의 mutation/capture authority로 분리한다.

## 8. 설계 원칙 후보

### K8S-P1. Application Failure ≠ Application Code Failure
Pod/Node/deployment evidence를 source-code 분석과 분리해 확인한다.

### K8S-P2. Event ≠ Truth
Event는 보조 evidence이며 object status/metrics/logs/traces와 교차 검증한다.

### K8S-P3. Immutable Artifact Identity
tag보다 image digest와 deployment revision을 우선한다.

### K8S-P4. Read API ≠ Exec
cluster observation 권한과 container shell/rollout mutation 권한을 분리한다.