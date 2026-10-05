# 18장. 이미 다른 버전이 운영 중이라면

운영에서 NullPointerException이 발생했다.

stack trace에는 UserService.java 142번째 줄이라고 나온다.

Agent가 현재 main branch를 열어 142번째 줄을 본다.

그런데 그 줄에는 문제가 없다.

이런 상황은 생각보다 쉽게 생긴다.

운영 코드와 현재 저장소 코드가 다르기 때문이다.

## 1. line number를 믿기 전에 버전을 확인한다

먼저 장애 trace나 로그에서 서비스.version을 본다.

~~~text
service.version = a81c92f
~~~

현재 workspace는:
~~~text
git commit = b115e91
~~~

다르다.

이제 142번째 줄을 그대로 비교하면 안 된다.

## 2. image digest도 확인한다

tag는 바뀔 수 있다.

~~~text
myapp:latest
~~~

같은 이름이라도 다른 image일 수 있다.

가능하면 digest처럼 바뀌지 않는 식별자를 사용한다.

~~~text
sha256:ab34...
~~~

## 3. 배포 revision을 같이 본다

Kubernetes에서는 배포 revision을 통해 어느 rollout에서 문제가 시작됐는지 확인할 수 있다.

~~~text
revision 41 정상
revision 42 error rate 증가
~~~

이제 recent source diff와 연결하기 쉽다.

## 4. Agent가 해야 할 첫 행동이 바뀐다

버전이 다르면 바로 수정를 만들지 않는다.

먼저 다음 중 하나를 한다.

- 해당 commit checkout
- 해당 tag/branch 찾기
- 두 버전 diff 확인
- source artifact를 찾지 못하면 불확실하다고 표시

## 5. 오래된 장애도 있다

운영에서 이미 새 버전이 배포돼 문제가 사라졌는데 과거 장애를 분석하고 있을 수도 있다.

이때 현재 관측 데이터와 과거 로그를 섞으면 잘못된 결론이 나온다.

시간과 버전을 함께 봐야 한다.

## 6. 설정 버전도 중요하다

코드는 같아도 설정이 다르면 동작이 달라질 수 있다.

예:
- connection pool size
- feature flag
- timeout
- retry count

그래서 가능하면 config version이나 배포 configuration diff도 함께 확인한다.

## 7. 수정 후 배포 버전까지 확인한다

수정가 만들어졌다면 실제로 그 수정가 들어간 image가 배포됐는지 확인해야 한다.

테스트 결과와 운영 결과 사이에도 version 연결이 필요하다.

## 8. 이 장에서 기억할 것

> 운영 장애를 고치기 전에 지금 보고 있는 코드가 실제 운영 코드인지 먼저 확인한다.

> tag보다 commit과 image digest처럼 바뀌지 않는 식별자가 더 믿을 만하다.

### 주요 근거

- runtime/source version correlation research
- OpenTelemetry 서비스/container semantic conventions
