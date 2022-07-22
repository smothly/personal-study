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