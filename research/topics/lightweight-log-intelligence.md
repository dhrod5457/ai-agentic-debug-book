# Topic — Lightweight Log Intelligence for Agentic Debugging

기준일: 2026-10-05

## 1. 왜 이 계층이 필요한가

Agentic Debugging에서 가장 나쁜 기본값은 애플리케이션이 생성한 raw log 전체를 LLM context에 직접 넣는 것이다.

문제는 다음과 같다.

- 로그 양이 많아 context budget을 빠르게 소모한다.
- 반복되는 값(IP, request id, user id, timestamp)이 의미 밀도를 낮춘다.
- 정상 로그가 대부분인 환경에서는 LLM 추론 비용이 낭비된다.
- 긴 로그 덤프는 관련 증거와 무관한 증거를 섞어 RCA 정확도를 떨어뜨릴 수 있다.
- 운영 로그에는 개인정보, secret, tenant 정보가 포함될 수 있다.

따라서 로그 경로는 다음처럼 분리하는 것이 적절하다.

~~~text
Application Logs
      ↓
Collection
OpenTelemetry / Alloy / Fluent Bit
      ↓
Normalization / Template Mining
Drain3
      ↓
Lightweight Detection
Rule / DeepLog / LogBERT / small Transformer
      ↓
Evidence Selection
anomalous window / representative sequence
      ↓
Agent Interface
LogQL / HTTP API / MCP / bounded tool
      ↓
LLM Debugging Agent
RCA → source inspection → patch → verification
~~~

핵심 원칙:

> LLM은 로그 저장소도, 1차 필터도 아니다. 작은 계층이 로그를 압축하고 좁힌 뒤 LLM은 선택된 증거를 추론한다.

## 2. Drain3 — 로그 템플릿 압축

- Repository: https://github.com/logpai/Drain3
- Type: OSS
- 역할: streaming log template mining

Drain3는 로그를 한 줄씩 처리하면서 값이 달라지는 위치를 wildcard로 치환해 템플릿 cluster를 만든다.

예:

~~~text
connected to 10.0.0.1
connected to 192.168.0.1

→ connected to <:IP:>
~~~

Agentic Debugging 의미:

- raw text의 반복을 줄일 수 있다.
- template ID sequence를 anomaly model 입력으로 사용할 수 있다.
- masking을 통해 IP, 숫자, 식별자 등의 변동성을 줄일 수 있다.
- online processing과 inference-only mode를 지원하므로 운영 파이프라인 앞단에 배치하기 쉽다.

주의:

- template mining은 anomaly detection이 아니다.
- template이 같아도 순서, 빈도, latency, 주변 trace가 다르면 장애일 수 있다.
- structured field와 correlation key(trace_id 등)를 template 처리 과정에서 잃으면 안 된다.

## 3. DeepLog — 매우 가벼운 sequence baseline

- Paper: DeepLog: Anomaly Detection and Diagnosis from System Logs through Deep Learning
- Venue: CCS 2017
- DOI: 10.1145/3133956.3134015
- 구조: LSTM 기반 log sequence modeling

DeepLog은 정상 로그 시퀀스를 학습하고 다음 이벤트의 패턴이 예상과 다를 때 이상으로 본다.

장점:

- 생성형 LLM보다 훨씬 작고 가볍다.
- sequence anomaly의 baseline으로 적합하다.
- 정상 패턴에서 벗어나는 순서 이상을 잡을 수 있다.

한계:

- 최신 Transformer 계열보다 표현력이 제한될 수 있다.
- 템플릿 품질과 window 구성에 민감하다.
- anomaly는 root cause가 아니다.

책에서는 "LLM을 쓰기 전에 얼마나 작은 모델로 범위를 줄일 수 있는가"의 baseline으로 사용한다.

## 4. LogBERT — 경량 self-supervised anomaly detector

- Paper: LogBERT: Log Anomaly Detection via BERT
- arXiv: 2103.04475
- Repository: https://github.com/HelenGuohx/logbert
- Datasets: HDFS, BGL, Thunderbird

LogBERT는 정상 로그 sequence의 패턴을 self-supervised task로 학습하고 정상 분포에서 벗어난 sequence를 anomaly로 탐지한다.

Agentic Debugging에서의 위치:

~~~text
Drain3 template sequence
          ↓
       LogBERT
          ↓
 normal ──┴── anomalous
             ↓
      Agent evidence candidate
~~~

장점:

- 생성형 LLM 없이 sequence context를 이용한다.
- DeepLog보다 Transformer 기반 contextual representation을 사용할 수 있다.
- HDFS/BGL/Thunderbird 공개 실험 구현이 있다.

한계:

- 원 논문과 공개 구현은 현재 production log schema나 modern OTel pipeline을 직접 다루지 않는다.
- 모델이 anomaly를 찾더라도 "왜 장애인가"는 별도 reasoning 단계가 필요하다.

## 5. LogGPT / LogLLM — 직접 LLM 탐지의 비교 기준

### LogGPT

- Repository: https://github.com/nokia-steward/LogGPT
- arXiv: 2309.14482
- 방식: next log prediction + RL fine-tuning

LogGPT는 로그를 sequence language처럼 보고 다음 로그 event를 예측하는 방향이다.

연구적 가치는 높지만 상시 1차 필터로는 DeepLog/LogBERT/small Transformer보다 무겁다.

### LogLLM

- Repository: https://github.com/guanwei49/LogLLM
- 방식: BERT semantic encoder와 LLM 기반 anomaly classification

HDFS, BGL, Liberty, Thunderbird 등 대규모 로그 데이터셋을 사용한다.

이 책에서는 LogGPT/LogLLM을 "모든 로그를 LLM 계열 모델에 직접 넣는 접근"의 비교군으로 다룬다.

## 6. LogRAIL — 작은 탐지기 + LLM 재검증

- Paper: LogRAIL: A Retrieval-Augmented LLM Reverification Layer for Log Anomaly Detection
- IEEE Access, 2026
- Repository: https://github.com/Choiwongwang/LogRAIL

LogRAIL은 두 단계로 동작한다.

~~~text
All log windows
      ↓
Stage 1 — LogFormer
      ↓
near-threshold / suspicious windows only
      ↓
Vector DB + LLM reverification
      ↓
final anomaly decision
~~~

공개 repository의 AOSP Android 로그 결과에서:

- DeepLog F1: 0.8639
- LogBERT F1: 0.8856
- Stage 1 LogFormer F1: 0.9086
- LogRAIL precision-oriented F1: 0.9273
- LogRAIL recall-oriented F1: 0.9248

이 결과는 "전체 로그를 LLM으로 처리하지 않고, 작은 모델로 먼저 좁힌 뒤 경계 사례만 LLM에 보내는 구조"의 직접적인 근거다.

주의:

- 특정 dataset/실험 조건의 결과이므로 일반적인 production 정확도로 일반화하면 안 된다.
- Stage 2는 Llama 3 8B Instruct를 사용하므로 완전한 경량 시스템은 아니다.
- 하지만 expensive model 호출 범위를 near-threshold case로 제한하는 구조가 중요하다.

## 7. 이 책에서 권장할 Reference Pattern

### Tier 0 — deterministic filter

- severity
- exception type
- error rate
- latency threshold
- known failure signature

### Tier 1 — template mining

- Drain3
- structured field preservation
- trace_id/span_id/request_id correlation key 보존

### Tier 2 — lightweight anomaly detection

우선순위 후보:

1. DeepLog — 가장 단순한 sequence baseline
2. LogBERT — 경량 Transformer baseline
3. small Transformer encoder — 운영 환경 맞춤형 후보

### Tier 3 — evidence construction

Agent에 전달할 것은 전체 로그가 아니라 다음이다.

- incident time window
- anomalous template sequence
- representative raw log samples
- first/last occurrence
- frequency delta
- service/version
- trace_id/span_id
- related metric anomaly
- nearby deployment/config change

### Tier 4 — LLM reasoning

LLM 역할:

- anomalous evidence 해석
- source/deployment/config와 상관관계 확인
- competing hypothesis 생성
- 추가 query 선택
- root cause 후보 정리

### Tier 5 — debugging agent

- source inspection
- reproduction
- patch
- test
- before/after telemetry verification

## 8. 모델 선택 기준

"가볍고 성능이 좋은 모델"은 parameter count만으로 판단하지 않는다.

측정해야 할 항목:

- model size / memory
- CPU inference latency
- GPU dependency
- events/sec 처리량
- template parser 비용
- false-positive rate
- recall
- adaptation cost
- retraining frequency
- concept drift sensitivity
- anomalous window reduction ratio
- LLM 호출 절감률
- 최종 RCA 정확도 변화

특히 이 책에서는 anomaly F1만 보지 않고 최종 Agent workflow에 미치는 영향까지 평가한다.

## 9. 책에서 연결해야 할 설명 순서

이 주제는 독립적인 ML 장으로 분리하기보다 다음 흐름에 끼워 넣는다.

~~~text
애플리케이션 디버깅의 기본
    ↓
로그에 무엇을 남겨야 하는가
    ↓
OpenTelemetry 기반 수집
    ↓
Loki 등에 저장
    ↓
Drain3로 구조화 / 템플릿화
    ↓
경량 모델로 이상 구간 탐지
    ↓
LogQL/API/MCP로 Agent가 해당 구간 조회
    ↓
Metrics/Trace/Profile과 correlation
    ↓
LLM이 RCA 수행
    ↓
Source patch
    ↓
동일 telemetry로 재검증
~~~

이 흐름을 유지하면 독자는 "관측 도구 설명"과 "LLM 연결 설명"을 별개의 기술로 배우지 않고 하나의 실제 debugging pipeline으로 이해할 수 있다.

## 10. Engineering Principles 후보

### P16. Raw Logs Are Not Agent Context
원본 로그 저장과 Agent에게 제공할 debugging context는 분리한다.

### P17. Compress Before Reasoning
template mining, aggregation, anomaly filtering으로 evidence를 줄인 뒤 LLM reasoning을 수행한다.

### P18. Preserve Correlation Keys
로그 템플릿화 과정에서도 trace_id/span_id/service.version 같은 debugging key는 잃지 않는다.

### P19. Cheap Detection, Expensive Reasoning
상시 탐지는 작은 모델이나 deterministic rule이 담당하고 LLM은 선택된 사례에만 사용한다.

### P20. Anomaly Is a Retrieval Hint, Not a Root Cause
anomaly score는 Agent가 어디를 볼지 정하는 retrieval signal이며 root cause 판정 자체가 아니다.

## 11. 후속 실험 제안

Spring Boot 예제 애플리케이션에서 같은 장애 corpus를 대상으로 다음을 비교한다.

A. raw logs → LLM
B. Drain3 → representative logs → LLM
C. Drain3 → LogBERT/small Transformer → anomalous windows → LLM
D. C + Prometheus/Tempo correlation → LLM Agent

측정:

- LLM input token 수
- log bytes scanned
- anomaly recall
- root cause accuracy
- time to first correct hypothesis
- query count
- GPU/CPU cost
- final patch success
- before/after verification success
