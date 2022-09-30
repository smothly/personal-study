# SEC1 환경설정 + Hadoop 개요

## 환경설정

---

- [VM설치](https://www.virtualbox.org/wiki/Downloads>)
- [Hortonworks Data Platform 2.5.0 설치](https://www.cloudera.com/downloads/hortonworks-sandbox.html)
- [IMDB 영화 데이터 다운로드](https://files.grouplens.org/datasets/movielens/ml-100k.zip)
- VM에 HDP 설치완료
- Ambari에 들어가 Data import 후 Hive로 데이터 조회

```SQL
-- 영화별 평점 개수 세기
select movie_id, count(movie_id) as ratingCount
from ratings
group by movie_id
order by ratingCount desc;

- ![image](https://user-images.githubusercontent.com/37397737/168478789-dcb080f3-9683-4bf4-8297-f7a331a5590c.png)

```

```SQL
-- 평점개수가 가장 많았던 영화 이름: 스타워즈
select name
from movie_names
where movie_id = 50;
```

## Hadoop 개요 및 역사

---

- 정의
  - An open source software platform for distributed storage and distributed processing of very large data sets on computer clusters built from commodity hardware.
  - 여러 컴퓨터위에서 데이터를 분산 처리 및 저장하는 오픈 소스 플랫폼
- 역사
  - GFS와 MapReduce 2003-2004 구글논문 기반
  - 2006년 이후 급속 성장을 이룸
- 왜 하둡을 써야하는가?
  - 데이터가 너무 큼
  - single point of failure을 방지 함
  - scale out이 좋은 시대

## Hadoop 생태계 개요

---

- ![하둡생태계](https://ducmanhphan.github.io/img/hadoop/architecture/core-hadoop-system.png)
  - HDFS
    - 분산파일 시스템
    - GFS의 하둡버전
    - 백업 복사본을 사용한 fail over
  - Yarn
    - 리소스 교섭자(resource negotiator)
    - 데이터 처리 부분을 담당
    - 컴퓨터 클러스터의 리소스를 관리하는 시스템
  - MapReduce
    - 매퍼는 클러스터에 분포되어 있는 데이터를 효율적으로 동시에 변형
    - 리듀서는 매퍼에서 변형한 데이터를 집계
  - Pig
    - MR위에 구축된 기술로 SQL 스타일의 구문을 사용
    - 코드말고 스크립트를 읽음
  - Hive
    - Pig처럼 MR위에 구축됨
    - 실제 SQL 쿼리를 받고 파일 시스템에 분산된 데이터를 데이터베이스처럼 처리
  - Ambari
    - 클러스터와 어플리케이션의 상태를 볼 수 있음
  - Mesos
    - Hadoop은 아니고 Yarn과 같은 리소스 교섭자
    - yarn과는 다른 방식으로 사용
  - Spark
    - MR과 동일 선상
    - 클러스터의 데이터를 신속하고 효율적으로 처리하고 싶을 때 사용
  - TEZ
    - directed acycle graph를 사용하여 MR일을 효율적으로 함. 쿼리 실행을 더 효율적으로 세우기 때문
    - 보통 Hive와 함게 사용되며 성능을 가속화함
  - HBASE
    - NoSQL 데이터 베이스
    - 트랜잭션의 수가 큰 아주 빠른 데이터 베이스
    - OLTP 트랜잭션을할 때 유용
  - Apache Storm
    - 스티리밍 데이터 실시간 처리
  - Oozie
    - 클러스터의 작업을 스케줄링
    - ex) hive -> pig -> spark -> hbase
  - Zookeper
    - 클러스터의 모든 것을 조직화하는 기술
    - 노드의 상태 추적, 클러스터의 공유 상태 확인
  - 데이터 수집
    - Sqoop
      - RDBMS의 데이터(ODBC, JDBC)를 HDFS형태로 변환
    - Flume
      - 대규모 웹로그를 안정적으로 클러스터로 불러옴
    - Kafka
      - 모든 종류의 데이터를 수집해 Hadoop 클러스터에 보냄
  - 외부 데이터 저장소
    - Hadoop과 연결될 수 있는 저장소들
    - MySQL
    - Cassandra
    - mongoDB
  - Query Engine
    - 쿼리를 실행하고 클러스터로부터 데이터를 추출하는 작업
    - Apache Drill
      - NoSQL데이터에 SQL을 날릴 수 있음
    - Hue
      - Hive, Hbase에 호환성이 좋음
    - Presto
      - 전체 클러스터에 쿼리
    - Zeppelin
      - 노트북으로 클러스터에 접근
    - Phoneix
      - drill과 비슷하지만 ACID를 보장함
