# SEC_9 클러스터에 데이터 제공하기

## Kafka 설명

---

- 스트리밍
  - 실시간으로 클러스터에 데이터를 넣어줌. 넣어주는 즉시 처리도 가능
  - 실시간 주식 거래 / 센서 데이터 등 다양한 곳에서 쓰임
  - 2가지 문제
    - 어떻게 데이터를 넣을지
    - 어떻게 데이터를 처리하지
- Kafka 설명
  - ![kafka architecture](https://daxg39y63pxwu.cloudfront.net/images/blog/apache-kafka-architecture-/image_7224627121625733881346.png)
  - pub/sub messageing system
  - producer가 topic이라고 불리는 stream에 데이터를 넣음
  - consumer는 topic을 구독하고 데이터를 받음
  - db에 접근할 수 있는 connector도 있고, prodce/consume하는 app들도 있음
  - stream processor를 사용하여 데이터를 받자마자 변형할 수 있음
- kafka scale 방법
  - 여러 서버에 여러 producer가 있음
  - consumer도 분산되어 있음. consumer 같은 그룹끼리는 데이터를 분산하여 처리. 다른 그룹끼리는 동일한 메시지를 가지고 있음

## 실습1 Kafka 설정 및 데이터 퍼블리싱

---

- ambari 들어가서 kafka 시작

```bash
cd /usr/hdp/current/kafka-broker/bin
./kafka-topic.sh --create --zookeeper sandbox.hortonworks.com:2181 --replication-factor 1 --partitions 1 --topic fred # 토픽 생성
./kafka-topic.sh --list --zookeeper sandbox.hortonworks.com:2181
./kafka-console-producer.sh --broker-list sandbox.hortonworks.com:6667 --topic fred # 타이핑 하는 라인이 meessage로 감

# 새 터미널 열어서 consume하기
cd /usr/hdp/current/kafka-broker/bin
./kafka-console-consumer.sh --bootstrap-server sandbox.hortonworks.com:6667 --zookeeper localhost:2181 --topic fred --from-beginning
```

## 실습2 웹로그 게시하기

---

- 내장된 kafka connector로 파일을 모니터하고 토픽으로 발행하기
- access_log_small.txt의 변경사항이 logout.txt에 저장됨

```bash
cd /usr/hdp/current/kafka-broker/conf
cp connect-standalone.properties ~/
cp connect-file-sink.properties ~/
cp connect-file-source.properties ~/

vi connect-standalone.properties
bootstrap.servers=sandbox.hortonworks.com:6667

vi connect-file-sink.properties
file=/home/maria_dev/logout.txt
topics=log-test

vi connect-file-source.properties
file=/home/maria_dev/access_log_small.txt
topic=log-test

wget http://media.sundog-soft.com/hadoop/access_log_small.txt

# 컨슈머 개시
cd /usr/hdp/current/kafka-broker/bin
./kafka-console-consumer.sh --bootstrap-server sandbox.hortonworks.com:6667 --zookeeper localhost:2181 --topic log-test

# file 모니터링
cd /usr/hdp/current/kafka-broker/bin
./connect-standalone.sh ~/connect-standalone.properties ~/connect-file-source.preperties ~/connect-file-sink.properties
```

## Flume 설명

---

- ![flume](https://cdn.educba.com/academy/wp-content/uploads/2019/11/Flume-Sink-1.png)
- Hadoop 생태계에서만 쓰이는 스트리밍
- source / channel(memory or file) / sink 로 구성됨
- channel에서 싱크 처리되는 순간 그 데이터는 삭제 됨(kafka와의 주 차이점)
- source 종류
  - avro
  - file
  - kafka
  - exec
  - thrift
  - http 등
- sink 종류
  - hdfs
  - hive
  - hvase
  - avro
  - kafka
  - elasticsearch 등
- agent끼리 연결하면서 확장 가능

## 실습1 Flume 설정 및 로그 게시하기

---

- source: netcat / channel: memory / sink: logger 로 테스트

```bash
wget media.sundog-soft.com/hadoop/example.conf # source sink channel의 정보가 있음

cd /usr/hdp/current/flume-server/
bin/flume-ng agent --conf conf --conf-file ~/example.conf --name a1 -Dflume.root.logger=INFO,console # log가 자동으로 출력 됨

telnet localhost 44444 # 아무 문장 입력
```

## 실습2 Flume으로 디렉토리 감시하며 HDFS에 저장하기

---

- source: spooldir / channel: memory / sink: hdfs로 테스트
- sppldir은 티렉토리에 새 파일을 timeintercepter를 통해 생성 날짜를 찍음
- hdfs에 10분 간격으로 새 서브디렉토리를 만듬

```bash
wget media.sundog-soft.com/hadoop/flumelogs.conf # source sink channel의 정보가 있음

# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = spooldir
a1.sources.r1.spoolDir = /home/maria_dev/spool
a1.sources.r1.fileHeader = true
a1.sources.r1.interceptors = timestampInterceptor
a1.sources.r1.interceptors.timestampInterceptor.type = timestamp

# Describe the sink
a1.sinks.k1.type = hdfs
a1.sinks.k1.hdfs.path = /user/maria_dev/flume/%y-%m-%d/%H%M/%S
a1.sinks.k1.hdfs.filePrefix = events-
a1.sinks.k1.hdfs.round = true
a1.sinks.k1.hdfs.roundValue = 10
a1.sinks.k1.hdfs.roundUnit = minute

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
####################################

# 파일 가져다 놓기
mkdir spool
# ambari에서 hdfs에 flume 폴더 만들기
bin/flume-ng agent --conf conf --conf-file ~/flumelogs.conf --name a1 -Dflume.root.logger=INFO,console # log가 자동으로 출력 됨

# 파일 생성 후 테스트
# 완료되면 source 파일에 .completed가 접미사로 붙음

```
