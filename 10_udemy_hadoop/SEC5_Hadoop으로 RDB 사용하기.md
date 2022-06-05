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
FROM avgRatings t JOIN names n ON t.movieID = n.movieID;
WHERE ratingCount > 10;
```
