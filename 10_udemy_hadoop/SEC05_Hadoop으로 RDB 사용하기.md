# SEC5 Hadoop으로 관계형 데이터 저장소 사용하기

## What is Hive?

---

- ![Hive architecture](https://image.slidesharecdn.com/jetlore-120711223401-phpapp01/85/spark-and-shark-lightningfast-analytics-over-hadoop-and-hive-data-25-320.jpg?cb=1342046123)
- Hive 특징
  - `SQL`을 사용하는 것이 Key!! 사람들이 익숙한 언어이기 때문임
  - SQL 쿼리를 MR로 분해하고 클러스터 전체에 어떻게 실행시킬지 알아냄
  - OLAP 쿼리에 유용
  - 빅데이터에 scalable하게 쿼리 가능
  - UDF, JDBC/ODBC 등 확장성이 좋음
- Hive 단점
  - OLTP에 적합하지 않음. MR을 내부적으로 사용하기 때문에 지연시간이 오래걸림
  - SQL로도 한계가 있음. 이럴때는 Pig나 Spark가 적절함
  - RDB가 아니여서 트랜잭션이나 레코드레벨을 지원하지 않음
- HiveQL
  - MySQL과 비슷함
  - 쿼리결과를 `view`(논리적 구조)로 저장하여 테이블처럼 활용할 수 있음
  - 저장과 파티셔닝을 지정할 수 있음

## 실습: Hive를 사용하여 가장 인기 있는 영화 찾기

---

- 테이블 업로드(u.data, u.item)
- VIEW 생성
  
```sql
CREATE VIEW IF NOT EXISTS topMovieIDs AS
SELECT movieID, COUNT(movieID) as ratingCount
FROM ratings
GROUP BY movieID
ORDER BY ratingCount DESC;
```

- 조회

```sql
SELECT n.title, ratingCount
FROM topMovieIDs t JOIN names n ON t.movieID = n.movieID;
```

- view 삭제

```sql
DROP view topMovieIDs;
```

## Hive 작동 방식

---

- **Schema on Read**
  - unstructured data를 읽는 순간 스키마를 지정함
  - 메타스토어에 실제 스키마 정보를 가지고 있음
  - RDB는 schema on write
- 데이터 저장 위치
  - HDFS에 분산 저장되어 있음. 파일의 소유권을 가지는 형태
  - 외부 테이블의 경우는 Hive가 소유권을 가지고 있지 않음
- 파티셔닝
  - 파티셔닝 필드에 따라 폴더링이 됨 ex) /customers/country=CA/
- Hive 사용법
  - CLI
  - query 파일 .hql
  - ambari/hue
  - jdbc/odbc
  - thrift service
  - oozie

## 실습: 평균 평점이 제일 높은 영화 찾기

---

- 10명 이상이 평가한 영화중 평균 평점이 높은 영화

```sql
CREATE VIEW IF NOT EXISTS avgRatings AS
SELECT movieID, AVG(rating) as avgRating, COUNT(movieID) as ratingCount
FROM ratings
GROUP BY movieID
ORDER BY avgRating DESC;

SELECT n.title, avgRating
FROM avgRatings t JOIN names n ON t.movieID = n.movieID
WHERE ratingCount > 10;
```

## MySQL과 Hadoop 통합하기

---

- MySQL
  - 오픈소스 RDB
  - OLTP 적합
  - 모놀리틱함
  - Hadoop에 import할 수 있음.
- Sqoop
  - ![sqoop아키텍처](https://t1.daumcdn.net/cfile/tistory/246A5C3A58479DB10D)
  - 각 매퍼가 RDB와 HDFS와 통신하며 import/export함
  - Hive에 직접 테이블 생성도 가능
  - 특정 column(pk, date 등) 기준으로 incremental import도 가능

## 실습 Saoop을 사용하여 MySQL <-> HDFS/Hive

---

### MySQL 설치 및 영화 데이터 가져오기

- 데이터 삽입 스크리트 다운로드

```shell
wget http://media.sundog-soft.com/hadoop/movielens.sql
```

- MySQL 접속 `mysql -u root -p` pw는 hadoop
- 테이블 생성 + 데이터 import

```sql
create database movielens;

show databases; -- 확인

set names 'utf8'; -- 인코딩 변경
set character set utf8;
use movielens;
source movielens.sql; -- 데이터 삽입

select * from movies limit 10; -- 확인
describe ratings;

select movies.title, count(ratings.movie_id) as ratingCount
from movies
inner join ratings
on movies.id = ratings.movie_id
group by movies.title
order by ratingCount;
```

- 현재 데이터는 작기 때문에 로컬에 MySQL설치하는 것이 이득일 수는 있으나, 대용량 데이터일 경우 Hadoop 클러스터의 이점을 가져갈 수 있음

### Sqoop을 사용하여 MySQL에서 HDFS/Hive로 데이터 가져오기

- localhost에 권한부여 `GRANT ALL PRIVILEGES ON movielens.* to ''@'localhost';`
- sqoop 명령어로 hdfs import 실행

```shell
sqoop import --connect jdbc:mysql://localhost/movielens --driver com.mysql.jdbc.Driver --table movies -m 1
```

- sqoop 명령어로 hive import 실행
- apps > hive > warehouse > movies에 위치함

```shell
sqoop import --connect jdbc:mysql://localhost/movielens --driver com.mysql.jdbc.Driver --table movies -m 1 --hive-import
```

### Sqoop을 사용하여 HadoopL에서 MySQL로 데이터 내보내기

- MySQL에 전달받을 테이블 생성

```sql
create table exported_movies (id INTEGER, title VARCHAR(255), releaseDate DATE);
```

- sqoop export 명령어
- delimeter는 기본적으로 ascii 1번으로 되어있음

```shell
sqoop export --connect jdbc:mysql://localhost/movielens --driver com.mysql.jdbc.Driver -m 1 --table exported_movies --export-dir /apps/hive/warehouse/movies --input-fields-terminated-by '\0001'
```

```sql
select * from exported_movies limit 10; -- 확인
```
