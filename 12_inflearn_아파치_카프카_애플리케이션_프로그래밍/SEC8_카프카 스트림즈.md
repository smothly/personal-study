# SEC8 카프카 스트림즈

---

## 8-1 카프카 스트림즈 소개

---

- 카프카에서 공식적으로 지원하는 자바 라이브러리
- **토픽에 적재된 데이터를 실시간으로 변환**하여 다른 토픽에 적재하는 라이브러리
- ![카프카 스트림즈](https://blog.kakaocdn.net/dn/uF8L1/btqEmp9yJ41/RSrS24sAhlRy1KxLpl8431/img.jpg)
- **exactly once할 수 있도록 fault tolerant system을 가지고 있어 데이터 처리 안정성이 매우 뛰어남**
- 실시간 스트림 처리를 할 필요성이 있다면 1순위로 고려할 것
- 스트림 데이터 처리에 다양한 기능을 **스트림즈 DSL로 제공**하고 **프로세서 API를 사용**하여 기능 확장할 수 있음
- 프로듀서 컨슈머의 조합으로 구현은 가능하나, exactly once fault tolerant system을 구축하기 쉽지 않음
- 토픽이 서로 다른 클러스터일 경우 지원하지 않음
- 스트림즈 내부 구조
  - 내부적으로 스레드를 1개 이상 생성할 수 있으며, 스레드는 1개 이상의 태스크를 가짐
  - **`태스크`는 스트림즈 어플리케이션을 실행하면 생기는 데이터 처리 최소 단위**
  - 3개의 파티션일 경우 3개의 태스크가 생김
  - 스케일 아웃은 스트림즈 스레드(or 프로세스)개수를 늘리면 됨
    - ![스케일 아웃](https://user-images.githubusercontent.com/37397737/218309200-ece3dd59-a166-4dca-a6b6-c5c8662a8cd9.png)
  - 토폴로지(Topology)
    - 2개 이상의 노드들과 선으로 이루어진 집합
    - ring, star, **tree** 형이 있음
    - 프로세스와 스트림
      - ![구조](https://docs.confluent.io/platform/current/_images/streams-architecture-topology.jpg)
      - 소스 프로세서: 토픽에서 데이터를 가져오는 역할
      - 스트림 프로세서: 데이터 처리하는 역할
      - 싱크 프로세서: 데이터를 특정 토픽으로 저장하는 역할
- 구현하는 방법 2가지
  -  **스트림즈 DSL(Domain Specific Language)**: 다양한 기능들을 자체 API로 제공
  -  **프로세서 API**: DSL에서 제공하지 않는 일부 기능들의 경우 프로세서 API를 사용 ex) 메시지 값에 따라 토픽을 가변적으로 전송, 일정한 시간 간격으로 데이터 처리

## 8-2 스트림즈DSL

---

- 추상화한 3가지 개념
  - KStream
  - KTable
  - GlobalKTable

## 8-3 KStream, KTable, GlobalKTable

---

- KStream
  - 레코드의 흐름을 표현한 것
  - 메시지 키와 메시지 값으로 구성
  - 컨슈머로 토픽을 구독하는 것과 동일한 선상
- KTable
  - ![KTable 예시](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSLiKMH6GWktUxAWlyaYmAmgDf4v-4byawxgQIztBTkcLKOzWUDU7xXbrlnq5c4-ET4PPk&usqp=CAU)
  - 메시지 키를 기준으로 묶음
  - 메시지 키 기준으로 **가장 최신의 레코드**
  - 데이터가 업데이트 됨. key-value스토어 처럼 사용
- co-partitioning
  - KStream과 Ktable을 조인할려면 `co-partitioning`되어 있어야 함
  - 2개 데이터의 **파티션 개수**가 동일, **파티셔닝 전략**(메시지 키가 동일하게 들어가는 것)을 동일하게 맞추는 작업
    - ex) andy가 동일한 파티션에 들어가면 동일한 태스크에서 다른 토픽내 파티션끼리 조인 가능함
    - 만약 전제조건이 다를 경우 `TopologyException` 발생
- GlobalKTable
  - kstream과 ktable은 각각의 파티션이 태스크에 할당돼서 사용
  - GlobalKTable은 파티션에 있는 모든 데이터로 Metrialized View로 태스크에서 활용 됨
  - 전부 GlobalKTable로 하면 데이터가 비대해져 어플리케이션에 부담이 되게 됨

## 8-4 스트림즈 주요 옵션 소개

---

- 필수 옵션
  - bootstrap.servers: 브로커의 호스트 이름. 2개이상 작성하여 HA구성
  - application.id: 스트림즈 어플리케이션 고유 ID. 다른 로직을 가짐녀 다른 ID
- 선택 옵션
  - default.key.serde: 메시지 키를 직렬화, 역직렬화 클래스 지정. 기본값은 `Serdes.ByteArray().getClass().getName()`
  - default.value.serde: 메시지 값을 직렬화, 역직렬화 클래스 지정. 기본값은 `Serdes.ByteArray().getClass().getName()`
  - num.stream.threads: 프로세싱 시리행 시 스레드 개수를 지정. 기본값은 1
  - state.dir: 상태기반 데이터를 처리할 때 데이터를 저장할 디렉토리 지정. 기본값은 `/tmp/kafka-streams`. Rocks DB에 저장되게 되는데 활용하는 경우는 드뭄

## 8-5 스트림즈 어플리케이션 개발하기

---

- [필터링 스트림즈 어플리케이션](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.5%20kafka-streams/kafka-streams-filter)
  - 데이터중 문자열의 길이가 5보다 큰경우만 필터링하는 스트림 프로세서 만들기
  - `filter()`메소드 활용

    ```java
        public static void main(String[] args) {

            Properties props = new Properties();
            props.put(StreamsConfig.APPLICATION_ID_CONFIG, APPLICATION_NAME);
            props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
            props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
            props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

            StreamsBuilder builder = new StreamsBuilder();
            KStream<String, String> streamLog = builder.stream(STREAM_LOG);
    //        KStream<String, String> filteredStream = streamLog.filter(
    //                (key, value) -> value.length() > 5);
    //        filteredStream.to(STREAM_LOG_FILTER);

            streamLog.filter((key, value) -> value.length() > 5).to(STREAM_LOG_FILTER);


            KafkaStreams streams;
            streams = new KafkaStreams(builder.build(), props);
            streams.start();

        }
    ```

## 8-6 KStream, KTable 조인 스트림즈 어플리케이션

---

- KTable과 KStream은 메시지 키를 기준으로 조인할 수 있음
  - ex) KTable에는 주소와 같이 최신값만 있는 데이터 + KStream에는 주문관련 데이터
- [실습](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.5%20kafka-streams/kstream-ktable-join)
  - 스트림 데이터 join을 위한 토픽 생성
    - **파티션 개수를 똑같이 맞춰야 함!**
    - address
    - order
    - order_join
  - 자동으로 메시지 키 값을 기준으로 Join하며 특정 값을 활용하여 값을 만들어 낼 수 있음
  - produce할 때 키값을 꼭 명시하기
  - 속도가 필요할 때는 태스크와 파티션의 개수를 스케일 아웃함으로써 해결 가능

    ```java
    public static void main(String[] args) {

        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, APPLICATION_NAME);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();
        KTable<String, String> addressTable = builder.table(ADDRESS_TABLE);
        KStream<String, String> orderStream = builder.stream(ORDER_STREAM);

        orderStream.join(addressTable, (order, address) -> order + " send to " + address).to(ORDER_JOIN_STREAM);

        KafkaStreams streams;
        streams = new KafkaStreams(builder.build(), props);
        streams.start();

    }
    ```

## 8-7 KStream, GlobalKTable 조인 스트림즈 어플리케이션

---

- KStream과 KTable은 **코파티셔닝**이 되어 있어 조인할 수 있었으
- **코파티셔닝**이 안되어 있을 경우 리파티셔닝 하여 **코파티셔닝** 형태로 하거나, KTable로 사용하는 토픽을 GlobalKTable로 선언하여 사용
- [실습](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.5%20kafka-streams/kstream-globalktable-join)
  - addres_v2 토픽을 생성하는데 해당 토픽을 파티션 2개로 생성하여 코파티셔닝이 안되게 함
  - globalKTable에 많은 데이터가 있으면 스트림즈 어플리케이션에 부하가 발생
  
    ```java
    public static void main(String[] args) {

        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, APPLICATION_NAME);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();
        GlobalKTable<String, String> addressGlobalTable = builder.globalTable(ADDRESS_GLOBAL_TABLE);
        KStream<String, String>   = builder.stream(ORDER_STREAM);

        orderStream.join(addressGlobalTable,
                (orderKey, orderValue) -> orderKey,
                (order, address) -> order + " send to " + address)
                .to(ORDER_JOIN_STREAM);

        KafkaStreams streams;
        streams = new KafkaStreams(builder.build(), props);
        streams.start();

    }
    ```

## 8-8 스트림즈DSL의 윈도우 프로세싱

---

- 특정 시간에 대응하여 취합 연산을 처리할 때 활용
- 4가지 지원 - 모두 메시지 키를 기준으로 취합되어 동일한 파티션에 동일한 키가 존재해야 함
  - 텀블링 윈도우
    - ![텀블링 윈도우](https://img1.daumcdn.net/thumb/R800x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbsNRyV%2Fbtreh2rJVnX%2Fk5h2cROdwAz629RtJ7lZIK%2Fimg.png)
    - 서로 겹치지 않은 윈도우를 특정 간격으로 지속적으로 처리할 때 사용
    - ex) 매 5분간 접속한 고객의 수를 측정
    - 예제 코드

    ```java
    StreamsBuilder builder = new StreamsBuilder();
    KStream<String, String> stream = builder.stream(TEST_LOG);
    KTable<Windowed<String>, Long> countTable = stream.groupByKey()
            .windowedBy(TimeWindows.of(Duration.ofSeconds(5)))
            .count();
    countTable.toStream().foreach(((key, value) -> {
        log.info(key.key() + " is [" + key.window().startTime() + "~" + key.window().endTime() + "] count : " + value);
    }));

    KafkaStreams streams = new KafkaStreams(builder.build(), props);
    streams.start();
    ```

  - 호핑 윈도우
    - 일정 시간 간격으로 겹치는 윈도우 연산
  - 슬라이딩 윈도우
    - 호핑윈도우와 유사하지만 데이터의 정확한 시간(Timestamp)을 바탕으로 윈도우 사이즈에 포홤되는 데이터를 모두 연산에 포함시킴
      - 시스템 시간: 서버가 돌아가는 시간
      - 레코드 시간: Timestamp
  - 세션 윈도우
    - ![세션 윈도우](https://kafka.apache.org/20/images/streams-session-windows-02.png)
    - **동일 메시지 키의 데이터를 한 세션에 묶어** 연산할 때 사용
    - 세션의 최대 만료시간에 따라 윈도우 사이즈가 달라짐(**=윈도우 사이즈 가변적**)
- **주의사항!**
  - 커밋(기본값 30초)을 수행할 때 윈도우 사이즈가 종료되지 않아도 **중간 정산 데이터를 출력**함
  - **동일 윈도우 시간에 upsert하는 방식**으로 처리
  - ![동일 윈도우 처리](https://user-images.githubusercontent.com/37397737/226159247-d32650df-70be-4e87-8aef-5e4c503a24ab.png)

## 8-9 스트림즈DSL의 Queryable store

---

- KTable은 토픽의 데이터를 로컬의 rockesDB에 Meterialized View로 만들어 사용하기 때문에, **메시지 키, 값을 기반으로 `KeyValueStore`로 사용**가능

```java
private static ReadOnlyKeyValueStore<String, String> keyValueStore;

StreamsBuilder builder = new StreamsBuilder();
KTable<String, String> addressTable = builder.table(ADDRESS_TABLE, Materialized.as(ADDRESS_TABLE));
KafkaStreams streams;
streams = new KafkaStreams(builder.build(), props);
streams.start();

TimerTask task = new TimerTask() {
    public void run() {
        if (!initialize) {
            keyValueStore = streams.store(StoreQueryParameters.fromNameAndType(ADDRESS_TABLE,
                    QueryableStoreTypes.keyValueStore()));
            initialize = true;
        }
        printKeyValueStoreData();
    }
};
Timer timer = new Timer("Timer");
long delay = 10000L;
long interval = 1000L;
timer.schedule(task, delay, interval);
```

## 8-10 프로세서 API

---

- 스트림즈 DSL(더 추상화)보다는 투박하지만, 토폴로지를 기반으로 데이터를 처리하기 때문에 동일한 역할을 함
- `Processor` 또는 `Transformer` 인터페이스로 구현한 클래스가 필요
  - Processor 인터페이스는 일정 로직이 이루어진 뒤 다음 프로세서로 데이터가 넘어가지 않을 때 사용
  - [Processor 사용 예제](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.5%20kafka-streams/simple-kafka-processor/src/main/java/com/example)
    - processer 클래수 구현
    - 토폴리지 사용
  - Transformer 인터페이스는 일정 로직이 이루어진 뒤 다음 프로세서로 데이터를 넘길 때 사용

## 8-11 카프카 스트림즈와 스파크 구조적 스트리밍 비교

---

| 항목 | Kafka Streams | Spark Structured Streaming |
| --- | ------- | ------- |
| Deployment | standalone java process | Spark Executor(Yarn Cluster) |
| Streaming Source | Kafka only | Socker, File system, Kafka ... |
| Execution Model | Masterless | Driver+Executor |
| Fault-tolerance | StateStore, backed by changelog | RDD Cache |
| Syntax | Low level Processor API, High level DSL | SparkSQL |
| Semantics | Simple | Rich |

- 카프카 스트림즈: 카프카 토픽을 input으로 하는 경량 프로세싱 애플리케이션 개발
- 스파크 스트리밍: 카프카 및 하둡 생태계를 input으로 한 복잡한 프로세싱 개발

## 퀴즈

---

- 1) 카프카 스트림즈는 반드시 자바로 작성해야 한다 (O/X)
  - X 공식적인건 자바지만, JVM으로 동작하는 언어(scala, kotlin)도 동작함
  - 다른 언어는 third-pary tool 사용
- 2) 스트림즈DSL을 사용하면 반드시 KStream을 사용해야 한다 (O/X)
  - X ktable, globalktable도 활용 가능
- 3) KStream은 토픽을 구체화된 뷰로 활용하는 것이다 (O/X)
  - X kstream은 컨슈머와 유사, ktable, globalktable이 구체화된 뷰
- 4) GlobalKTable을 사용하면 토픽의 모든 데이터를 한개 태스크에서 활용할 수 있다 (O/X)
  - O ktable은 파티션과 태스크가 1대1 매핑임
- 5) KTable과 KStream을 join하기 위해서는 반드시 코파티셔닝 되어 있어야 한다 (O/X)
  - O 파티션 개수, 파티셔닝 전략 동일 되어야 함
- 6) 코파티셔닝이랑 파티션의 개수와 메시지 키에 대한 파티셔닝 전략이 동일한 것을 뜻한다 (O/X)
  - O