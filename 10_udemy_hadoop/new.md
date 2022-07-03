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






# SEC_7 클러스터 관리하기

## Yarn 설명

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