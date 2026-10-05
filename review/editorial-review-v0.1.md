# Editorial Review v0.1

기준일: 2026-10-05

## 1. 전체 판단

서문부터 22장, 맺음말까지 흐름은 완성됐다.

현재 강점:
- 중심 질문이 흔들리지 않는다.
- Prometheus/Grafana/Loki/Tempo가 제품 소개가 아니라 디버깅 흐름 안에서 등장한다.
- 14~18장의 실제 장애 사례가 책의 성격을 분명하게 만든다.
- 어려운 합성 용어를 줄이고 사례 뒤에 이름을 붙이는 방향으로 개선됐다.

현재 약점:
- 앞부분에 비해 후반부 장들이 짧아 개요처럼 느껴지는 곳이 있다.
- 일부 장은 '원칙 → 목록 → 결론' 패턴이 반복되어 리듬이 단조롭다.
- 운영 권한/검증/평가 장은 실제 상황 예시가 더 필요하다.
- Part II의 제품별 장은 사람이 하는 행동과 Agent가 하는 행동의 연결을 조금 더 보여줄 수 있다.

## 2. 분량 균형

상대적으로 충분:
- 1장
- 2장
- 3장
- 4장

중간:
- 5~13장
- 14장
- 16장
- 18장
- 22장

보강 우선:
- 15장 Lock Contention
- 17장 Platform Failure
- 19장 Production Access
- 20장 Diagnosis to Change
- 21장 Verification

## 3. 편집 원칙

분량을 늘리기 위해 개념을 더 넣지 않는다.

보강 방법은 다음 순서만 사용한다.

1. 실제 장애 상황 추가
2. 사람이 흔히 하는 잘못된 판단 추가
3. Agent가 어떤 순서로 확인하는지 추가
4. 수정 전후 비교 추가
5. 운영에서 주의할 경계 추가

## 4. 반복 표현 개선

현재 많은 장이 '이 장에서 기억할 것'으로 끝난다.

완전히 없애지는 않되 다음과 같이 변주한다.
- 한 문장 결론
- 다음 장애로 연결
- 운영 체크
- 실무에서의 판단

## 5. 용어 기준

본문에서 가능한 한 다음 한국어 표현을 우선한다.

- evidence → 증거 / 확인한 자료
- correlation → 연결
- scope → 범위
- runtime identity → 실행 버전
- query budget → 조회 제한
- hypothesis → 원인 후보 / 가설
- observability → 관측 / 관측 시스템

영어가 실무에서 더 익숙한 경우만 그대로 둔다.

## 6. 이번 편집 패스에서 즉시 수정할 장

- 15장: lock contention의 오판 사례와 Agent 조사 순서 보강
- 17장: OOMKilled와 eviction을 구분하는 흐름 보강
- 19장: 외부 LLM 데이터 경로, 최소권한 운영 사례 보강
- 20장: read → capture → dev mutation → prod mutation의 실제 승인 흐름 보강
- 21장: before/after 검증 실패 사례와 부분 해결 판정 보강

## 7. 다음 편집 패스

1. 5~8장 제품 설명 비율 재검토
2. 장간 중복 문장 제거
3. 장별 출처 표기 정규화
4. Part 사이 연결 문장 강화
5. 전체 manuscript 조립본 생성

## 8. 1차 편집 패스 결과

완료:
- 1~8장 어려운 용어 1차 완화
- 5~8장 실무 흐름 보강
- 9~12장 reader-friendly 톤으로 집필
- 15장 Lock Contention 보강
- 17장 Platform Failure 보강
- 19장 Production Access 보강
- 20장 권한 전환 보강
- 21장 검증 흐름 보강
- 장간 중복 주제 ownership 정의
- 2장/4장/9장 중복 설명 축소
- README의 잘못된 research link 수정

다음 편집 패스:
1. 13~18장 사이 사례 연결 문장 보강
2. 22장 평가 장의 실제 예시 보강
3. 전체 출처 연결 방식 통일
4. 서문/맺음말 최종 톤 조정
5. 통합 원고 생성


## 9. 문체·References·통합 원고 패스 완료

2026-10-05 추가 편집:
- 본문 전체에서 어려운 영어 혼용 표현을 2차 정리
- 코드 블록과 고유 기술 용어는 유지
- 어색한 자동 치환 표현 수동 교정
- 장별 `주요 근거`를 `참고 자료`로 통일
- 내부 연구 메모 이름을 공식 Source ID로 교체
- `manuscript/references.md` 생성
- 공식 문서와 peer-reviewed/preprint 구분
- `manuscript/BOOK.md`에 서문~22장~맺음말~References 통합

현재 다음 단계:
1. 출간 직전 freshness audit
2. 오탈자/문장 호흡 최종 교정
3. 필요 시 PDF/EPUB 등 publication artifact 생성


## 10. 출간 직전 교정 결과

완료:
- 전체 원고 2차 문체 통일
- 자동 용어 치환 후 발생한 조사/혼용 오류 수동 교정
- Grafana MCP / Tempo MCP / OpenTelemetry / Spring Boot / Oracle JDK / Kubernetes freshness audit
- Grafana Loki guardrail default=off 보정
- Tempo MCP 별도 활성화 및 LLM 응답 형식 안정성 주의 반영
- References에 확인일/안정성/default 상태 반영
- 최신 장별 원고에서 BOOK.md 재조립
- 최종 검사: 22장, References 1개, Source ID 누락 0개, 알려진 자동 치환 패턴 0개

현재 원고는 publication artifact 생성 전 최종 Markdown 기준본으로 사용한다.
