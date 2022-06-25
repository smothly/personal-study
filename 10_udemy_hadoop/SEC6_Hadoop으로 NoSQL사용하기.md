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

## Cassandra 개요

---

- Distributed NoSQL
- 마스터 노드가 없고 SPOF가 없음
- Hbase처럼 트랜잭션 쿼리에 최적화되어 있음
- CQL이라는 쿼리 언어를 가지고 있음
- CAP Theorem
  - 아래 3개중 2개만 보장할 수 있음
  - 보통 partition-tolerance는 기본으로 가지고, c와 a중 하나를 택하게 된지만 a를 우선시하게 된다.
  - 그렇기 때문에 조정 가능한 일관성이라고 함
  - consistency 일관성: 무조건 응답을 받는다
  - availability 가용성: DB가 항상 작동하고 신뢰할 수 있음
  - partition-tolerrance 파티션 저항성: 데이터가 쉽게 나눠지고 클러스터에 분산할 수 있음
  - ![다른 DB와의 비교](https://raw.githubusercontent.com/ippontech/blog-usa/master/images/2016/11/CapTheorem.jpg)
- 어떻게 일관성대신 가용성을 얻을 수 있을까?
  - ![카산드라 아키텍처](https://cassandra.apache.org/_/_images/diagrams/apache-cassandra-diagrams-01.jpg)
  - 마스터 노드가 없는 대신 `가십 프로토콜` `시니처 기술` 을 사용하여 매초마다 서로 간 소통하며 누가 어떤 데이터를 가지고 있고, 복사본을 가지고 있는지 추적함
  - 노드들은 서로 소통하며 데이터를 복사하고 클러스터를 구성할 때 지정한 중복수준에 따라 백업 보사본을 어떤 노드가 가질지 결정
  - 각 원은 특정 키의 범위를 가지고 있음
- Cassandra노드에 replication을 가지도록 하면 한쪽은 데이터 분석 한쪽은 서비스 하면서 사용 가능
- CQL
  - 조인 없음
  - PK기반으로 쿼리가 동작함
  - 모든 테이블은 `keyspace`에 존재해야함. database같은 것임
- Spark와 통합
  - DataFrame으로서 cassandra 테이블을 읽을 수 있음
  - Spark로 cassandra에 I/O 할 수 있음

## 실습 Cassandra 설치

---

```shell
yum install scl-utils # python 여러 버전 사용
yum install centos-release-scl-rh

yum install python27 # python 2.7 설치

scl enable python27 bash # python 버전 전환
python -V # 버전 확인

cd /etc/yum.repos.d # 카산드라 리소스를 받기 위한 repo 생성
vi datastax.repo # 아래 내용 입력
# [datastax]
# name = DataStax Repo for Apache Cassandra
# baseurl = http://rpm.datastax.com/community
# enabled = 1
# gpgcheck = 9

yum install dsc30 # 카산드라 설치

pip install cqlsh # cql 쓰기 위한 라이브러리 설치

service cassandra start # 카산드라 시작

cqlsh --cqlversion="3.4.0" # cql버전 맞춰서 접속
```

```sql
CREATE CERSPACE movielens WITH  replication = {'class': 'SimpleStrategy', 'replication_factor': '1'} AND durable_writes = truel-- 키스페이스 만들기, 실제로는 복제나 지속적 쓰기 설정은 멀티 클러스터에 걸맞게 생성해야함
USE movielens;
CREATE TABLE users (user_id int, age int, gender text, occupation text, zip text, PRIMARY KEY (user_id));
DESCRIBE TABLE users;
SELECT * FROM users;
```

## 실습 Spark출력을 Cassandra에 쓰기

---

```shell
export SPARK_MAJOR_VERSION=2 # dataets를 사용하기 때문에 v2.0을 사용 
spark-submit --packages datastax:spark-cassandra-connector:2.0.0-M2-s_2.11 CassandraSpark.py
service cassandra stop
```

- dataset으로 불러오기만 하면 평소에 Spark 사용하는 것처럼 하면 됨

```python
from pyspark.sql import SparkSession
from pyspark.sql import Row
from pyspark.sql import functions

def parseInput(line):
    fields = line.split('|')
    return Row(user_id = int(fields[0]), age = int(fields[1]), gender = fields[2], occupation = fields[3], zip = fields[4])

if __name__ == "__main__":
    # Create a SparkSession
    spark = SparkSession.builder.appName("CassandraIntegration").config("spark.cassandra.connection.host", "127.0.0.1").getOrCreate()

    # Get the raw data
    lines = spark.sparkContext.textFile("hdfs:///user/maria_dev/ml-100k/u.user")
    # Convert it to a RDD of Row objects with (userID, age, gender, occupation, zip)
    users = lines.map(parseInput)
    # Convert that to a DataFrame
    usersDataset = spark.createDataFrame(users)

    # Write it into Cassandra
    usersDataset.write\
        .format("org.apache.spark.sql.cassandra")\
        .mode('append')\
        .options(table="users", keyspace="movielens")\
        .save()

    # Read it back from Cassandra into a new Dataframe
    readUsers = spark.read\
    .format("org.apache.spark.sql.cassandra")\
    .options(table="users", keyspace="movielens")\
    .load()

    readUsers.createOrReplaceTempView("users")

    sqlDF = spark.sql("SELECT * FROM users WHERE age < 20")
    sqlDF.show()

    # Stop the session
    spark.stop()
```

- 카산드라는 일관성보단 가용성이 필요한 어플리케이션에 좋은 선택
- 데이터가 즉각적으로 전파되지 않아도 되는 데이터에 유용

## MongoDB 개요

---

- 문서 기반의 데이터 모델
- CAP 이론에 따르면 가용성보단 일관성을 중요시함
  - ![다른 DB와의 비교](https://raw.githubusercontent.com/ippontech/blog-usa/master/images/2016/11/CapTheorem.jpg)
- 단일 마스터 노드와 통신하며, 마스터 노드가 다운되면 다운타임(읽을수는 있지만 쓸 수는 없음)이 생김
- 스키마를 사용하지 않는 특성
  - 스키마를 강제할 수는 있지만 필수는 아님
  - 각 document가 스키마가 다를 수 있음
  - 자동으로 ID를 만들어 PK 처럼 활용. 원하는 필드로 index를 만들 수도 있고 샤딩할 수도 있음
  - 유연하게 저장할 수 있지만 어떻게 활용할 지 고민하고 저장해야 함
  - ![document 예시](https://mblogthumb-phinf.pstatic.net/MjAxODA2MThfMTg4/MDAxNTI5MzI5NzAzODI5.o74rpsjh3OatVI19l7knPJ_cI_V0GeASO0tgRPv1SGQg.6aI9aDyyXR6tqE-pu4d3vWTnxqGNInNscvY0DeBY7uMg.PNG.ijoos/image.png?type=w800)
- 구성 단위
  - Databases
  - Collections
  - Documents
- Replication Sets
  - 단일 마스터 아키텍처, 일관성을 우선시
  - primary가 다운되면 secondary중 하나가 대신함
  - 복제할 때 가장 빨리 복구할 수 있는 secondary를 택함. 물론 특정 노드 선택도 가능
  - 짝수의 서버를 가질 수 없음. primary를 설정할 때 과반수의 동의가 필요하기 때문
    - 결정권자 노드를 선택할 수는 있음
    - 하나의 mongo db 인스턴스를 사용하기 위해서 적어도 3개의 서버를 가져야 함
  - 레플리카 셋은 내구성만을 다룸
  - delayed seconday를 다뤄 사람의 실수로 다운됐을 때 보험으로서 복구를 할 수 있음
- Sharding
  - 다수의 레플리카 세트를 가지고, 각 레플리카 세트는 색인화된 일정 범위를 가짐
  - ![샤딩](https://webimages.mongodb.com/_com_assets/cms/kyc0ez5ijz9zx9iwz-image2.png?auto=format%252Ccompress)
  - `index`는 레플리카간의 정보의 양을 균형있게 하고, 시스템에 부하도 분산시킴
  - `mongos`라는 프로세스를 실행하며 각 서버의 collection들과 통신함.
  - `mongos`는 밸런서의 역할도 수행함. (index의 역할과도 비슷)
  - 시간이 지남에 따라 auto-sharding을 함
  - 효과적인 sharding을 위해서는 `cardinality`가 높은 필드를 선택하는게 좋음
- 장점
  - 매우 유연하여 대부분 형태의 데이터 저장 가능
  - js interpreter 지원
  - 여러 index 지원(여러개는 비추천, 텍스트, 좌표기반 검색 가능)
  - 내재된 집계기능으로 MR실행이나 GridFS라는 고유의 파일 시스템도 가짐(like HDFS)
  - Spark등 Hadoop과 통합할 수 있음
  - SQL connector를 사용할 수는 있으나 Join 및 Normalize 불가능

## 실습 MongoDB 설치 및 데이터 가져오기

- ambari + mongo connector 다운로드

```shell
su root
cd /var/lib/ambari-server/resources/stacks/HDP/2.5/services
git clone https://github.com/nikunjness/mongo-ambari.git # mongo-ambari connector 가져오기
sudo service ambari restart
```

- ambari 들어가서 service에서 mongodb 설치 및 시작
- python script에서 Spark로 mongo 사용하기 위해 라이브러리 설치

```shell
pip install pymongo
```

```shell
export SPARK_MAJOR_VERSION=2 # dataets를 사용하기 때문에 v2.0을 사용 
spark-submit --packages org.mongodb.spark:mongo-spark-connector_2.11:2.0.0 MongoSpark.py
```

```python
from pyspark.sql import SparkSession
from pyspark.sql import Row
from pyspark.sql import functions

def parseInput(line):
    fields = line.split('|')
    return Row(user_id = int(fields[0]), age = int(fields[1]), gender = fields[2], occupation = fields[3], zip = fields[4])

if __name__ == "__main__":
    # Create a SparkSession
    spark = SparkSession.builder.appName("MongoDBIntergrationIntegration").getOrCreate()

    # Get the raw data
    lines = spark.sparkContext.textFile("hdfs:///user/maria_dev/ml-100k/u.user")
    # Convert it to a RDD of Row objects with (userID, age, gender, occupation, zip)
    users = lines.map(parseInput)
    # Convert that to a DataFrame
    usersDataset = spark.createDataFrame(users)

    # Write it into Cassandra
    usersDataset.write\
        .format("com.mongodb.spark.sql.DefaultSource")\
        .options("uri","mongodb://127.0.0.1/movielens.users")\
        .mode('append')\
        .save()

    # Read it back from Cassandra into a new Dataframe
    readUsers = spark.read\
    .format("com.mongodb.spark.sql.DefaultSource")\
    .options("uri","mongodb://127.0.0.1/movielens.users")\
    .load()

    readUsers.createOrReplaceTempView("users")

    sqlDF = spark.sql("SELECT * FROM users WHERE age < 20")
    sqlDF.show()

    # Stop the session
    spark.stop()
```

## 실습 mongo shell 활용하기

---

- 몽고쉘 활용하기
- 
```sql
use movielens
db.users.find({user_id: 100})
db.users.explain().find( {user_id: 100} ) -- 쿼리가 어떤걸 하는지 설명해줌
db.users.createindex({user_id: 1}) -- 매번 검색하기 때문에 index를 생성. IXSCAN으로 변경됨
db.users.aggregate( [
  {$group: {_id: {occupation: "$occupation"}, avgAge: { $avg: "$age"}}
]) -- 각 직업별 평균 연령
db.users.count()
db.getCollectionInfos()
db.users.drop()
exit
```