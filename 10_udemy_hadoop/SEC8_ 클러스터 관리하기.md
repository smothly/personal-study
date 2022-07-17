# SEC_8 클러스터 관리하기

## Yarn

---

- hadoop 2에서 새로 생김
  - ![yarn위치](https://blog.kakaocdn.net/dn/dY5446/btqVuhsIGCO/rJHnz0lXFIPYfDftv8Umc1/img.png)
  - ![hadoop ecosystem](https://user-images.githubusercontent.com/60086878/111483142-de938580-8777-11eb-92c6-7527eeb4fdeb.png)
- resource negotiator(리소스 관리)
- Yarn 위에 어플리케이션(스파크와 테즈)를 개발하여 Acyclic 한 특징으로 성능을 올림
- 클러스터의 컴퓨팅 리소스를 관리
- 동작 방법
  - ![yarn 동작방법](https://nnkumar13home.files.wordpress.com/2018/11/fig04.png?w=616)
- 어플리케이션 프로세스를 관리
  - CPU 최적화
  - locality 최적화하여 네트워크 최소화
  - 다양한 스케줄링 방법
    - fifo, capacity, fair schedular
- 따로 실습은 없음, 스파크, 하이브, 피그 등을 사용할 때 내부적으로 Yarn은 동작했음

## Tez

---

- Hive, Pig, MR job을 빠르게 해줌
- DAG를 만들어 불필요한 종속성 제거 + 물리적 데이터 흐름을 최적화하여 분산작업에 더 효율적으로 구성됨

## 실습 Tez에서 Hive에서 성능상 이점 확인하기

---

- hive에 u.data 올리기
- 아래 쿼리를 tez, mr로 각각돌려보기
  - 10만개의 평점데이터지만, tez는 20초 mr은 70초 소요됨
```sql
DROP VIEW IF EXISTS topMovieIDs;

CREATE VIEW IF NOT EXISTS topMovieIDs AS
SELECT movieID, COUNT(movieID) as ratingCount
FROM ratings
GROUP BY movieID
ORDER BY ratingCount DESC;

SELECT n.name, ratingCount
FROM topMovieIDs t JOIN movielens.names n ON t.movie_id = n.movie_id;
```

## Mesos

---

- 데이터 센터에 있는 리소스를 관리하는 프로그램
- yarn과 비슷하지만 mesos는 hadoop에 한정되지 않음
- spark와 storm은 원래 mesos를 대상으로 개발됨
- `Myriad`라는 패키지를 사용해서 yarn과 통합할 수 있음
- 단일 스케줄러
- two-tired 시스템
  - 어플리케이션에 리소스를 제안함
  - 어플리케이션이 수락할지 말지 결정하고 스케줄 알고리즘을 지정
  - 대규모 작업을 다룰 때 확장성에 용이함
  - yarn은 분석 작업에 최적화. 데이터가 HDFS에 있으면 yarn이 좋음
- 요즘은 mesos 대체재로 k8s로 많이 사용함
- mesos에서 spark를 실행할때 단일로 제한이 되어 대부분 yarn에서 spark를 활용하는 것이 효율적임

## Zookeeper

---

- ![zookeeper아키텍처](https://hopetutors.com/wp-content/uploads/2019/08/zookeeper-architecture.png)
- 클러스터 사이의 동기화 되어야 하는 정보들을 유지시킴
  - 모든 클라이언트는 마스터 노드를 일관성있게 인지해야 함
  - 어던 워커가 무슨일 하고 있는지
- HA 보장
- Failover
- Failure modes
  - 마스터가 다운되면 백업으로 실행되는 마스터를 선별해 새 마스터로 지정
  - 워커가 다운되면 어플리케이션에 해당 사실 전달하고 작업을 다시 적절하게 분산
  - 네트워크가 실패하면 작업을 다시 적절하게 분산
- primitive하게 운영하기 위함
  - 마스터 선출
  - 크래시 탐지
  - 그룹관리: 어떤 작업자가 가용가능한지
  - 메타데이터: 작업 정보
- Zookeeper API
  - 분산 파일 시스템 같음. znode를 file로 생각하면 됨
  - 클러스터의 모든 노드에 데이터가 안정적이고 일관성 있게 저장되도록 보장
- notification
  - znode의 변화가 생기면 알람을 줌
- ensemble로 일관성 보장 개수로 설정한 개수만큼 각 서버간의 복제를 하고 완료 후 응답을 줌
  - 최소 5개의 서버와 quorum을 3개로 해줘야 네트워크 파티션이 발생해도 일관성을 보장할 수 있음

## 실습 Zookeeper 

---

- zookeeper에 znode를 생성해보고 마스터인지 살펴보기

```bash
cd /usr/hdp/current/zookeeper-client/bin
.zkCli.sh  # zookeeper와 연결

ls / 
create -e /testmaster "127.0.0.1:2223" # 임시 znode 생성
get /testmaster 
quit # 다시 접속하면 testmaster는 사라져 있음

create -e /testmaster "127.0.0.1:2225" # 임시 znode 생성. 1개까지밖에 생성 안됨
get /testmaster
# 마스터 노드가 죽으면 모든 클라이언트가 testmaster노드를 만들어 zookeeper가 그중에 하나를 택해 마스터 노드로 채택함
```

## Oozie

---

- ![oozie 예시](https://miro.medium.com/max/1200/1*aLyxKMabrnd29zbX0pjjTA.jpeg)
- hadoop task들을 실행시키고 스케줄링함
- DAG 워크플로 
- xml로 정의됨
- HDFS에 폴더를 생성하여 `workflow.xml` `job.preperties`을 위치시킴
  - job에는 namenode, jobtracker, queuename, libpath, wf path 등이 들어감
- http://localhsot:11000/oozie 에서 모니터링 가능
- 실행 예시

```bash
oozie job --oozie http://localhsot:11000/oozie -config /home/maria_dev/job.properties -run
```

- oozie coordinator: 스케줄, 타임아웃, 동시성, 대기 등 설정함
- oozie bundler: coordinator를 묶어주는 역할. 공통의 job들을 하나로 묶어 관리하여 편함

## 실습 Oozie

---

- Sqoop 추출 -> Hive 분석을 워크플로에 돌려보기

- Sqoop으로 MySQL 데이터 추출

```sql
mysql -u root -p # mysql 접속
wget http://media.sundog-soft.com/hadoop/movielens.sql -- 제목, 평정, 직업 등의 6개 테이블 insertr

set names 'utf8';
set character set 'utf8';

use movielens;
source movilens.sql; -- 데이터 삽입
```

- Oozie에서 Hive job 실행시키기

```sql
DROP TABLE movies;
CREATE EXTERNAL TABLE movies (movie_id INT, title STRING, release DATE) ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' LOCATION '/user/maria_dev/movies/';
INSERT OVERWRITE DIRECTORY '${OUTPUT}' SELECT * FROM movies WHERE release < '1940-01-01' ORDER BY release;
```

- oozie xml 파일

```xml
<?xml version="1.0" encoding="UTF-8"?>
<workflow-app xmlns="uri:oozie:workflow:0.2" name="old-movies">
    <start to="sqoop-node"/>
 
    <action name="sqoop-node">
        <sqoop xmlns="uri:oozie:sqoop-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <prepare>
                <delete path="${nameNode}/user/maria_dev/movies"/>
            </prepare>
 
            <configuration>
                <property>
                    <name>mapred.job.queue.name</name>
                    <value>${queueName}</value>
                </property>
            </configuration>
            <command>import --connect jdbc:mysql://localhost/movielens --driver com.mysql.jdbc.Driver --table movies -m 1 --username root --password hadoop</command>
        </sqoop>
        <ok to="hive-node"/>
        <error to="fail"/>
    </action>
  
    <action name="hive-node">
        <hive xmlns="uri:oozie:hive-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <prepare>
                <delete path="${nameNode}/user/maria_dev/oldmovies"/>
            </prepare>
            <configuration>
                <property>
                    <name>mapred.job.queue.name</name>
                    <value>${queueName}</value>
                </property>
            </configuration>
            <script>oldmovies.sql</script>
            <param>OUTPUT=/user/maria_dev/oldmovies</param>
        </hive>
        <ok to="end"/>
        <error to="fail"/>
    </action>
 
    <kill name="fail">
        <message>Sqoop failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
    </kill>
    <end name="end"/>
</workflow-app>
```

- job.properties

```yaml
nameNode=hdfs://sandbox-hdp.hortonworks.com:8020
jobTracker=http://sandbox-hdp.hortonworks.com:8032
queueName=default 
oozie.use.system.libpath=true
oozie.wf.application.path=${nameNode}/user/maria_dev
```

```bash
hadoop fs -put workflow.xml /user/maria_dev
hadoop fs -put oldmovies.xml /user/maria_dev
hadoop fs -put oldmovies.xml /usr/share/java/mysql-connector-java.jar /user/oozie/share/lib/lib_20161025075203 # 공유 라이브러리
```

- ambari 들어가서 oozie 재시작하기
- 터미널에서 oozie job 실행하기

```
oozie job --oozie http://localhsot:11000/oozie -config /home/maria_dev/job.properties -run
```

- 완료화면
- ![완료화면](https://community.cloudera.com/t5/image/serverpage/image-id/12315i1A7D5D0245D16A26?v=v2)

## Zeppelin

---

- 빅데이터 노트북 인터페이스
- ![zepplin 예시](https://zeppelin.apache.org/docs/0.6.0/assets/themes/zeppelin/img/notebook.png)
- Spark 실행 및 시각화 분석에 사용
- SparkSQL과 통합도 되어 SQL쿼리를 날릴 수 있음
- 데이터를 대화형으로 사용할 수 있음
- 노트처럼 포매팅하여 이쁘게 보일 수 있음
- 다양한 인터프리터들이 있음 ex) hive, spark, python, tajo, cassandra 등