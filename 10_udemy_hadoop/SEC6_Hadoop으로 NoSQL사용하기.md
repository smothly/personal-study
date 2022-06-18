# SEC6 Hadoop으로 비관계형 데이터 저장소 사용하기

## Why NoSQL?

---

- 방대한 양의 트랜잭션을 RDB로 처리하기에는 무리
- 수평적으로 무한 확장 가능
- 빠른 Failover
- Random access to planet-size data
- 진정한 빅데이터는 수평적으로 확장할 수 있어야 함
- RDB의 성능 극대화
  - 비정규화
  - 인메모리 캐싱 구성
  - Master/slave 구조 (write/read 분리)
  - 샤딩
  - 구체화된 뷰
  - procedure 제거
  - 성능을 극대화 하면 관리 포인트가 많아지는 문제 발생
- 간단한 API로 조회할 수 있는거에 SQL이 필요할까? 의 관점에서 시작됨
- 분석은 하이브, 스파크 등등을 사용하고 일반적인 어플리케이션에는 MySQL로도 충분함. 고로 상황에 맞는 DB를 선택해야 함

## HBASE

---

- ![HBASE아키텍처](https://cdn.educba.com/academy/wp-content/uploads/2019/06/hbase-architecture-1.jpg)
- HDFS 위에 구축됨
- 구글의 BigTable 기반으로 만들어짐. 전세계 웹페에지 링크들을 저장했어야 함
- 쿼리언어가 없음. API가 있고 CRUD 작업할 수 있음
- zookeeper는 감사자를 감시함
- 지역서버간에 파티션이 나위어져 있고, 마스터 서버가 무엇이 어디에 있는지 추적하며 데이터 자체는 HDFS에 저장되어 있음
- Unique key로 Row 단위의 빠른 접근
- 각 행마다 column families를 가지고 있음. column family는 많은 수의 컬럼을 가지고 있을 수 있음
- **sparse한 데이터에 유용**하고 컬럼이 없으면 스토리지를 사용하지 않음
- 셀 개념도 있고, 여러 버전으로 가지고 있음
- ![셀과 컬럼개념](https://www.cloudduggu.com/hbase/data-model/hbase_table.png)
- HBase 접근 방법
  - shell
  - java api
  - spark, hive, pig
  - rest service 간단하게 사용
  - thrift service 최대 성능. 동기화의 문제가 생김
  - avro service 최대 성능. 동기화의 문제가 생김

## 실습1 영화 평점을 HBase로 가져오기

---

- 스키마
  - userID 고유의 키
  - rating이라는 column family
  - column family는 모든 평점을 포함하게 됨
- python client - rest - hbase - hdfs 의 구조를 갖게 됨
- VM의 virtual box REST API와 통신할 8000포트를 추가해줍니다.
- admin으로 ambari 접속해서 hbase start
- 터미널 접속하여 hbase rest start
  
  ```shell
  /usr/hdp/current/hbase-master/bin/hbase-daemon.sh start rest -p 8000 --infoport 8001
  ```

- Hbase rest와 통신해서 데이터 올리기

  - 아래와 같이 스키마 구성
  
  ```python
  batch.update(userID, {'rating': {movieID: rating}})
  ```

## 실습2 HBase를 Pig와 함께 사용하여 대규모 데이터 가져오기

---

- `u.user` 폴더 HDFS에 올리기
- HBase 테이블 만들기
  - `hbase shell`로 접속
  - `create 'users', 'userinfo'` Hbase에 스키마 생성
  - pig 스크립트 다운로드 `wget http://media.sundog-soft.com/hadoop/hbase.pig`

    ```sql
    users = LOAD '/user/maria_dev/ml-100k/u.user'
    USING PigStorage('|')
    AS (userID:int, age:int, gender:chararray, occupation:chararray, zip:int);

    STORE users INTO 'hbase://users'
    USING org.apache.pig.backend.hadoop.hbase.HBaseStorage (
    'userinfo:age,userinfo:gender,userinfo:occupation,userinfo:zip');
    ```