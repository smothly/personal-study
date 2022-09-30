# SEC 11 현실 시스템 설계하기

## 다양한 시스템 소개

---

- Impala
  - hadoop 위에 구축된 병렬 SQL 엔진
  - 항상 실행되고 있어 start-up 과정이 없음
  - hive보단 빠르지만 다양성이 부족함
  - cloudera를 사용중이라면 고려할만한 쿼리 엔진
- Accumulo
  - Hbase clone
  - 더 나은 보안을 가짐
  - 셀 기반 엑세스 통제
- Redis
  - in-memory data store
  - 캐시에 최적화 됨
  - 디스크에 써서 지속적 데이터 저장소로도 사용 가능
- Ignite
  - in-memory data febric
  - ACID를 제공해서 데이터 베이스 가까움
  - SQL지원
- Elastic Search
  - 분산 검색 검색 및 분석 엔진
  - 실시간 검색 쿼리로 주로 쓰임
  - kibana와 많이 결합함
- Kinesis
  - 카프카의 AWS버전
  - AWS의 다양한 서비스와 결합 가능
- Apache NiFi
  - data의 directed graph 구축
  - kafka, hdfs, hive 와 결합
- Falcon
  - oozie위에서 움직이는 데이터 거버넌스 엔진
  - nifi처럼 데이터 처리 그래프를 만들지만, 하둡으로 들어온 데이터만 만ㄷ블 수 있음
  - oozie에 워크플로가 많을 때 고려
- Apache Slider
  - yarn 클러스터에 분산하려는 범용 어플리케이션을 위한 도구
  - app 모니터링
## 다양한 도구 결합하기

---

- 여태까지 배웟던 하둡 생태계 요소들 훑기

## 요구 사항 이해하기

---

- end-user의 니즈로부터 어떤게 필요한지 알아야 함
  - 액세스 패턴
    - 대량의 분석 쿼리
    - 신속하게 계산해야 하는 트랜잭션
    - 가용성이 필요한지 일관성이 필요한지 
    - 데이터 양은 어느정도 되는지
  - 현재의 인력과 시스템 파악
  - 데이터 유지 정책
  - 데이터 보안
  - Latency
  - Timeleness
- 최소한의 저항, 기존 시스템 활용, 미래에 이 정보를 어떻게 활용할 지 등 최대한 간단한 솔루션으로 생각

## 예제: 웹 서버 로그를 사용하고 베스트 셀러 추적하기

---

- 10개의 인기 상품을 추적하는 시스템을 개발
  - 인기상품의 의미, 어느 기간 동안인지 등의 요구사항 파악이 필요
- cassandra 선택
  - Latency가 중요하므로 Distributed NoSQL로 데이터 전송해야함
  - 초 단위의 시기적절함(일관성)은 필요없음. 하지만 시스템이 다운되지 않을 고가용성이 필요함
- Spark streaming선택
  - cassandra와 호환성이 좋음
  - 슬라이드 윈도의 시간에 대한 컴퓨팅
  - 
- Kafka or Flume
  - HDFS 사용하거나 Log4j 사용하고 있다면 Flume
- 보안
  - 개인 식별 정보를 어떻게 다룰 것인지 점검받아야 함
- 최종 아키텍처
  - 구매 서버 - Flume + zookeeper - spark streaming - cassandra - web server

## 예제: 웹사이트에 추천 영화 제공하기

---

- 대규모 웹사이트여서 가용성과 파티션 저항성이 중요
- 추천시스템은 인공 지능이나 머신러닝이 필요하기 때문에 Spark MLlib 사용
- 모든 사용자의 추천을 미리 계산하는 것은 비효율적 => 즉석으로 추천을 계산하는 방법 = colloaborate filtering
  - 비슷한 영화의 목록을 가져옴
  - 비슷한 영화는 하루에 한번 업데이트 함
  - 영화 사이의 관계에 대한 정보
- 최종 아키텍처
  - 하둡을 사용한다는 가정하에
  - flume - spark streaming - hbase - sprak(mllib) - hdfs 끼리 연결

---

## 실습

- 가정
  - 대량의 트래픽 웹사이트
  - google analytics와 비슷하지만 커버 불가능한 서비스라고 가정
- 요구사항
  - 웹사이트의 세션 개수 보기
  - 특정 사이트와 교류한 횟수
  - 하루에 한 번 전날 데이터로 레포트 만들기
  - 세션 = 한시간 유지하는 같은 IP 주소로 부터의 트래픽
  - 오직 내부 분석 용도
- 최종 아키텍처
  - 가용성이나 대용량 트랜잭션 등 어려운 조건들은 없음
  - web server - kafka - spark streaming - hive - hdfs
  - sqoop을 통해서 DB로 직접 데이터