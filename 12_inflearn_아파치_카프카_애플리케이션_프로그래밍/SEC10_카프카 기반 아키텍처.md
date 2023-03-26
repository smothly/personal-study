# SEC10 카프카 활용 아키텍처 사례 분석

---

## 10-1 카프카 활용 아키텍처 사례 분석

---

- 스마트 메시지의 카프카 스트림즈 활용
  - [발표 관련 정리 자료](https://inspirit941.tistory.com/m/436)
  - one by one 데이터 삽입시 mongodb에 부하가 많이 생김
  - bulk는 aggregation 작업이 포함되지만 mongodb에는 부하가 덜 감
  - **Redis 1분 로그를 Window취합하여 적재하고 map 구문을 통해 redis 유저 데이터와 결합 처리**
- [넷플릭스 키스톤 프로젝트](https://netflixtechblog.com/keystone-real-time-stream-processing-platform-a3ee651812a)
  - ![넷플릭스사례](https://miro.medium.com/v2/resize:fit:1400/0*IPlVeq8BVU0aoUY0)
  - 첫번째 단계의 **라우팅 용도**로 모든 데이터를 수집하고 S3, ES, Consumer카프카로 데이터 전송
  - 두번째단계의 카프카는 Flink와 다른 컨슈머로 실시간 분석을 진행함
- [라인 쇼핑 플랫폼 사례](https://engineering.linecorp.com/ko/blog/line-shopping-platform-kafka-mongodb-kubernetes/)
  - 카프카를 중추신경으로 중요하게 활용
  - mongo DB CDC 활용
- [11번가 주문/결제 시스템 적용 사례](https://www.deview.kr/2019/schedule/305)
  - DB의 병목현상을 해소하기 위해 도입
  - 카프카를 Source of Truth로 정의
  - 메시지를 Meterialized View로 구축하여 배치 데이터처럼 활용
- 아키텍처 적용 방법
  - 카프카 커넥트
    - **반복적인 파이프라인**을 만들어야할 경우 **분산 모드 커넥트**를 설치하고 운영
    - 오픈소스 커넥터를 활용하되는 필요시 커스텀 커넥터 개발하여 운영
    - rest pai로 통신하여 웹 화면을 개발하거나 오픈소스 활용하는 것을 추천
  - 카프카 스트림즈
    - stateful, stateless 프로세싱 기능이 들어 있으므로 카프카의 토픽의 데이터처리시 선택
  - 카프카 컨슈머, 프로듀서
    - 단일성 파이프라인이나 개발환경일 경우 직접 개발하는게 나음
  - 정리
    - 프로듀서로 데이터 처음 유입 OR 커넥트 형식으로 유입
    - 토픽 기반 데이터 프로세싱은 카프카 스트림즈로 처리
    - 반복된 파이프라인은 분사모드 카프카 커넥트로 운영, 단발적 처리는 컨슈머로 개발
