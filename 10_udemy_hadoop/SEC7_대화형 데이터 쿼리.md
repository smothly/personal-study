# SEC7 대화형 데이터 쿼리

## Drill 개요

---

- 다양한 쿼리 엔진
  - Apache drill: 다양한 DB에 SQL쿼리를 날릴 수 있음
  - phoenix: Hbase위에 동작함 
  - presto: drill과 비슷. facebook에서 만듬. cassandra와 소통 가능
- drill 설명
  - nosql 또는 data 파일에 대한 SQL 엔진 - hive, mnongo db, hbase, json, parqeut, s3 등
  - real sql(not sql like) - ODBC/JDBC 드라이버 사용해서 다른 툴들과 연결 가능
  - 내부적으로는 비관계형 데이터베이스여서 대용량 데이터를 합칠 때는 비효율적임
  - 시스템의 요구사항을 명확히 파악하고 설계된 용도에서 벗어나면 안됨
  - 데이터 소스를 불러오거나 변형 없이 쿼리를 날릴 수 있는게 가장 큰 장점
  - 다른 DB들과 데이터를 합칠 수 있음

## 실습 Drill 설정

---

- 몽고디비 다시 실행
- 몽고디비 u.user 올리기
  - `shell spark-submit --packages org.mongodb.spark:mongo-spark:mongo-spark-connector_2.11:2.0.0 MongoSpark.py`
- hdfs에 hive로 u.data 올리기
- drill 설치하기
  - 1.12 버전 설치
  
  ```shell
  wget http://archive.apache.org/dist/drill/drill-1.12.0/apache-drill-1.12.0.tar.gz
  tar -xvf apache-drill-1.12.0.tar.gz
  cd apache-drill-1.12.0
  bin/drillbit.sh start -Ddrill.exec.port=8765 # 외부화 통신하기 위해 이미 열려있는 port로 설정합
  ```

- `127.0.0.1:8765` Drill UI 접속
- hive, mongo enable 시키기

## 실습 Drill로 여러 DB 쿼리하기

---

- 쿼리 사용

```sql
SHOW DATABASES;
SELECT * FROM hive.movilens.ratings LIMIT 10;
SELECT * FROM mongo.movilens.users LIMIT 10;
-- 각 직업별 평점
SELECT u.occupation, count(*)
FROM hive.movielens.ratings r
JOIN mongo.movielens.users u
ON r.user_id = u.user_id
GROUP BY u.occupation;
```

- 종료
  - `bin/drillbit.sh stop`
  - `mongodb 종료`

## Phoenix 개요

---

- 오직 Hbase와만 통신
- Fast, low-latency OLTP 지원
- 세일즈포스에서 개발한 오픈소스
- JDBC Connector를 노출
- UDF 지원
- 통합 Jar파일을 가지고 있어 Spark, hive, pig 등과 통합 가능
- HBase에 최적화 되어 있음
- phoenix client가 Hbase API와 통신하며 데이터를 가져옴
- 사용법
  - CLI
  - API for java
  - JDBC Driver
  - Phoenix query server
  - jar

## 실습 Phoenix 설치 및 HBase 쿼리

---

- hbase를 먼저 실행해 놓기 - admin으로 접속해서 ambari에서 실행
- phoenix 실행

```shell
cd /user/hdp/current/phoenix-client
cd bin
python sqlline.py
```

```sql
!tables
create table if not exists us_population(
    state char(2) not null,
    city varchar not null,
    population bigint,
    constraint my_pk primary key (state,city)
);

upsert into us_population values ('NY', 'New York', 8143197); -- insert를 지원하지 않음.
upsert into us_population values ('CA', 'Los Angeles', 3844829);

select * from us_population where state='CA';
drop table us_population;
```

## 실습 Phoenix와 Pig 통합하기

---

```sql
create table users(
    userid integer not null,
    age integer,
    gender char(1),
    occupation varchar,
    zip varchar
    constraint pk primary key (userid)
);
```

```shell
cd /home/maria_dev
wget http://media.sundog-soft.com/hadoop/ml-100k/u.user
cd ..
wget http://media.sundog-soft.com/hadoop/phoenix.pig
```

- pig는 MR기반으로 동작하여 이런 테스트 작업에는 적합하지 않음

```sql
pig 스크립트
```

## Presto 개요

---

- Drill과 유사함
- 다양한 데이터 소스를 지원하여 데이터가 어디에 있든 쿼리를 실행할 수 있음
- SQL 구문과 유사
- OLAP에 적합
- JDBC, CLI, 태블로 Interface
- Cassandra 지원!!
- PB이상의 데이터를 가진 페이스북에서 사용중임
- 솔루션을 굳이 선택할 필요 없이 다양한 데이터 소스를 하나의 소스처럼 사용할 수 있음

## 실습 Prosto 설치 및 Hive 쿼리

---

- presto 설치

```shell
wget https://repo1.maven.org/maven2/com/facebook/presto/presto-server/0.165/presto-server-0.165.tar.gz
tar -xvf presto-server-0.165.tar.gz

cd presto-server-0.165

wget http://media.sundog-soft.com/hadoop/presto-hdp-config.tgz # 설정파일 다운로드 (포트, memory 등 설정)
tar -xvf presto-hdp-config.tgz # presto, log, node, hive, jvm 등 각 설정파일들 담겨져 있음

cd bin

wget https://repo1.maven.org/maven2/com/facebook/presto/presto-cli/0.165/presto-cli-0.165-executable.jar # 실행파일 다운로드
mv presto-cli-0.165-executable.jar presto
chmod +x presto

cd ../presto-server-0.165
bin/launcher start # 시작

# 127.0.0.1:8090 접속(모니터링 화면)
bit/presto --server 127.0.0.1:8090 --catalog hive # CLI가 hive 디비에 연결

bin/launcher stop # 종료
```

- presto CLI에서 실행. 모니터링화면에서 쿼리 모니터링 가능

```sql
show tables from default;
select * from defaults.ratings limit 10;
select * from defaults.ratings where rating=5 limit 10;
select count(*) from defaults.ratings where rating=1;
```

## 실습 Presto를 사용해서 Hive 및 Cassandra에 쿼리하기

---

- cassandra 시작

```shell
service cassndra start
nodetool enablethrift # presto와 연결하기 위해 thrift 서비스 활성화
cqlsh --cqlversion="3.4.0" # CLI 접속
```

- presto cassandra 연결

```shell
cd presto-server-0.165/etc/catalog
vi cassandra.properties # 카산드라 정보 입룍
# connector.name=cassandra
# cassandra.contact-points=127.0.0.1

cd ../..
bin/launcger start
bin/presto --server 127.0.0.1:8090 --catalog hive,cassandra # CLI 접속
```

- 쿼리 날리기

```sql
describe cassandra.movielens.users;
select * from cassandra.movielens.users limit 10;
select * from hive.default.ratings limit 10;

select u.occupation, count(*)
from hive.default.ratings r
join cassandra.movielens.users u
on r.user_id = u.user_id
group by u.occupation;
```
