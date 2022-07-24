# SEC 10 데이터 스트림 분석하기

## Spark Streaming 소개

---

- ![스파크 스트리밍](https://spark.apache.org/docs/latest/img/streaming-arch.png)
- 실시간으로 들어온 데이터를 처리. 정확히는 마이크로 배치
- 데이터를 작은 RDD 조각으로 나눔
- Discretized Stream
  - ![dstream](https://i2.wp.com/techvidvan.com/tutorials/wp-content/uploads/sites/2/2019/11/Apache-Spark-Dstream-01.jpg?fit=1200%2C628&ssl=1)
  - 각 배치의 RDD를 만드는 작업
  - transform action을 적용함
  - RDD에서 할 수 있는 작업 대부분 가능
  - stateful하여 배치 간격을 초과해 상태 유지 가능
- windowing
  - ![window 예시](https://prateekvjoshi.files.wordpress.com/2015/11/2-windowed-processing.png)
  - 데이터는 1초에 한번씩 처리되고 1시간 동안의 인기 상품을 매 30분 찾는 예제
    - batch interval: 데이터를 RDD로 가져오는 주기 = 1초
    - slide interval: 데이터를 샘플링하는 주기 = 윈도 변화 계산 = 30분
    - window interval: 현재 시점에서 데이터를 얼마나 볼지 = 1시간
- python code

```python
ssc = StreamingContext(sc, 1) # batch interval 1초
hashtagCounts = hashtagKeyValues.reduceByKeyAndWindow(lambda x,y:x+y, lambda x,y: x-y, 300 , 1) # slide 1초 window 5분
```

- 최근에는 Dstream 대신 `Dataset API`를 주로 사용. 장점은 코드의 차이가 없어짐
- 실습에서는 Dstream을 사용할 예정


## 실습1 Spark Streaming을 사용하여 Flume으로 방행한 웹 로그 분석하기

---

- flume을 통해 spool을 이용하여 avro에 쓰고 avro에 쌓이는 데이터를 spark streaming으로 처리하기

```bash
wget media.sundog-soft.com/hadoop/sparkstreamingflume.conf # flume 설정 avro 9002에 출력함
wget media.sundog-soft.com/hadoop/SparkFlume.py # script 다운
```

- spark streaming 실행하기 

```python
import re

from pyspark import SparkContext
from pyspark.streaming import StreamingContext
from pyspark.streaming.flume import FlumeUtils

parts = [
    r'(?P<host>\S+)',                   # host %h
    r'\S+',                             # indent %l (unused)
    r'(?P<user>\S+)',                   # user %u
    r'\[(?P<time>.+)\]',                # time %t
    r'"(?P<request>.+)"',               # request "%r"
    r'(?P<status>[0-9]+)',              # status %>s
    r'(?P<size>\S+)',                   # size %b (careful, can be '-')
    r'"(?P<referer>.*)"',               # referer "%{Referer}i"
    r'"(?P<agent>.*)"',                 # user agent "%{User-agent}i"
]
pattern = re.compile(r'\s+'.join(parts)+r'\s*\Z')

def extractURLRequest(line):
    exp = pattern.match(line)
    if exp:
        request = exp.groupdict()["request"]
        if request:
           requestFields = request.split()
           if (len(requestFields) > 1):
                return requestFields[1]


if __name__ == "__main__":

    sc = SparkContext(appName="StreamingFlumeLogAggregator")
    sc.setLogLevel("ERROR")
    ssc = StreamingContext(sc, 1)

    flumeStream = FlumeUtils.createStream(ssc, "localhost", 9092)

    lines = flumeStream.map(lambda x: x[1])
    urls = lines.map(extractURLRequest)

    # Reduce by URL over a 5-minute window sliding every second
    urlCounts = urls.map(lambda x: (x, 1)).reduceByKeyAndWindow(lambda x, y: x + y, lambda x, y : x - y, 300, 1)

    # Sort and print the results
    sortedResults = urlCounts.transform(lambda rdd: rdd.sortBy(lambda x: x[1], False))
    sortedResults.pprint()

    ssc.checkpoint("/home/maria_dev/checkpoint")
    ssc.start()
    ssc.awaitTermination()
```

- flume 실행하기 log txt 생성하기

```bash
cd /usr/hdp/current/flume-server
bin/flume-ng agent --conf conf --conf-file ~/sparkStreamingFlume.conf --name a1
```

## 실습2 spark streaming 코드 변경

---

- 5초의 간격으로 5분간의 HTTP status code 집계하기

```python
import re

from pyspark import SparkContext
from pyspark.streaming import StreamingContext
from pyspark.streaming.flume import FlumeUtils

parts = [
    r'(?P<host>\S+)',                   # host %h
    r'\S+',                             # indent %l (unused)
    r'(?P<user>\S+)',                   # user %u
    r'\[(?P<time>.+)\]',                # time %t
    r'"(?P<request>.+)"',               # request "%r"
    r'(?P<status>[0-9]+)',              # status %>s
    r'(?P<size>\S+)',                   # size %b (careful, can be '-')
    r'"(?P<referer>.*)"',               # referer "%{Referer}i"
    r'"(?P<agent>.*)"',                 # user agent "%{User-agent}i"
]
pattern = re.compile(r'\s+'.join(parts)+r'\s*\Z')

def extractURLRequest(line):
    exp = pattern.match(line)
    if exp:
        request = exp.groupdict()["status"]
        if request:
           requestFields = request.split()
           if (len(requestFields) > 1):
                return requestFields[1]


if __name__ == "__main__":

    sc = SparkContext(appName="StreamingFlumeLogAggregator")
    sc.setLogLevel("ERROR")
    ssc = StreamingContext(sc, 1)

    flumeStream = FlumeUtils.createStream(ssc, "localhost", 9092)

    lines = flumeStream.map(lambda x: x[1])
    statusCodes = lines.map(extractURLRequest)

    # Reduce by URL over a 5-minute window sliding every second
    statusCounts = statusCodes.map(lambda x: (x, 1)).reduceByKeyAndWindow(lambda x, y: x + y, lambda x, y : x - y, 300, 5)

    # Sort and print the results
    sortedResults = urlCounts.transform(lambda rdd: rdd.sortBy(lambda x: x[1], False))
    sortedResults.pprint()

    ssc.checkpoint("/home/maria_dev/checkpoint")
    ssc.start()
    ssc.awaitTermination()
```

## Apache storm

---

- storm은 개별 이벤트를 처리함
- 용어
  - ![스톰 용어](https://t1.daumcdn.net/cfile/tistory/252B8E3454C4517E21)
  - stream: tuple(그룹지어진 데이터)들이 흐르는 개념
  - spout: 소스와 연결하는 곳
  - bolt: 실제 데이터를 처리
  - 위의 요소들을 원하는 대로 조합할 수 있음. 조합된 걸 토폴로지라고 함
- 스톰 아키텍처
  - ![스톰 아키텍처](https://phoenixnap.com/kb/wp-content/uploads/2022/05/apache-storm-architecture.png)
  - nimbus: job tracker, SPOF이지만 요즘은 고가용성이 보장됨
  - zookeeper: 고가용성
  - supervisor: 작업자와 작업 프로세스를 실행하는 주체
- java로 개발됐지만, bolt에서 다른 언어 사용 가능
- 2개의 API 제공L storm core(low) at least once / trident(high) exactly once
- spark streaming과의 차이점
  - realtime이라면 storm. spark streaming은 마이크로 배치
  - spark streaming은 mllib이나 graphx와 통합할 수 있음
  - tumbling window는 이벤트가 겹칠 수가 없음 sliding window는 가능
  - kafka+storm은 자주 사용

## 실습 storm으로 단어 세기

---

- 구성할 기능
  - spout: 랜덤 문장 생성기
  - bolt: 단어 분리
  - bolt: 각 단어 카운트
- ambari 들어가서 storm 켜기
```bash
cd /usr/hdp/current/storm-client
cd contrib/storm-starter/src/jvm/org/apache/storm/starter
vim WordCountTopology.java # 예시 코드 보기

storm jar /usr/hdp/current/storm-client/contrib/storm-starter/storm-starter-topologies-*.jar org.apache.storm.starter.WordCountTopology wordcount 

cd /usr/hdp/current/storm-client/logs/workers-artifacts/wordcount-4-/6700
tail -f worker.log
```
- ambari ui에서 topology결과 모니터링

## Flink 

---

- 스트리밍의 이벤트별로 작업
- 독립적인 클러스터에서 실행 가능하여 확장성이 큼
- Fault-tolerant: exactly-once, state snapshot
- 차이점
  - storm보다 빠름
  - high-level api 제공
  - scale와 친화적
  - spark처럼 본인만의 에코시스템을 가짐
  - 데이터를 반은 순간이 아닌 이벤트 시간을 기준으로 처리할 수 있음(유연한 윈도 시스템)
- 플링크 아키텍처
  - ![플링크 아키텍처](https://www.researchgate.net/profile/Mehmet-Aktas-7/publication/329621212/figure/fig4/AS:812521086795787@1570731531478/Apache-Flink-Architecture.jpg)
  - spark처럼 독자적인 에코시스템을 가짐

## 실습 Flink로 단어 세기

---

- flink.apache.org 들어가서 설치

```bash
wget 주소
tar -xvf
cd flink-1.2.0/conf
vi flink-conf.yaml # 포트 8081 -> 8082로 수정
```

- example 실행

```bash
./bin/flink run examples/streaming/SocketWindowWordCount.jar --port 9000
nc -l 9000 # tcp 포트 9000에 전달. 터미널에 문장 입
cd log # log 확인
./bin/stop-local.sh
```