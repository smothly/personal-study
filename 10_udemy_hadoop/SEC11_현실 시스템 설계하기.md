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