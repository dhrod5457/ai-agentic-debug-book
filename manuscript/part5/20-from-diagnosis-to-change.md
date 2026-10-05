# 20장. 원인을 찾은 뒤부터는 권한이 달라진다

Agent가 원인 후보를 찾았다.

이제 코드를 고치면 된다.

여기서부터는 상황이 달라진다.

관측하는 것과 시스템을 바꾸는 것은 위험 수준이 다르다.

## 1. 로컬 수정과 운영 수정은 같은 일이 아니다

Agent가 로컬 repository를 수정하고 테스트를 돌리는 것은 비교적 안전하다.

하지만 production deployment를 바꾸는 것은 다른 권한이다.

그래서 다음처럼 나누는 편이 좋다.

~~~text
운영 데이터 읽기
로컬 코드 수정
테스트 실행
staging 배포
production 배포
~~~

각 단계는 별도 권한으로 본다.

## 2. JFR을 뜨는 것도 변경이다

코드를 수정하지 않더라도 production JVM에서 새 JFR이나 heap dump를 만드는 것은 영향을 줄 수 있다.

그래서 diagnostic capture도 단순 read와 분리한다.

## 3. 자동 수정은 작은 범위부터 시작한다

처음부터 production auto-remediation을 목표로 하지 않는다.

더 현실적인 순서는 다음과 같다.

~~~text
원인 후보 제시
→ patch 생성
→ 테스트
→ staging 검증
→ 사람 승인
→ production 반영
~~~

## 4. rollback도 함께 생각한다

Agent가 수정안을 만들었다면 실패했을 때 되돌릴 수 있어야 한다.

그래서 production action에는 최소한 다음이 필요하다.

- 변경 전 버전
- 변경 후 버전
- rollback 경로
- 검증 조건

## 5. 승인도 모든 작업에 붙이지 않는다

사소한 read query까지 사람이 승인하면 쓸 수 없는 시스템이 된다.

반대로 production restart와 deployment 변경은 자동 허용하기 어렵다.

작업 위험도에 따라 approval 수준을 다르게 둔다.

## 6. 이 장에서 기억할 것

> 보는 권한과 바꾸는 권한을 분리한다.

> Agent의 자율성은 production 변경 권한을 많이 주는 것으로 측정하지 않는다.