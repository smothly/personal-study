# SEC5 카프카 프로듀서 어플리케이션 개발

---

## 5-1 Kafak Producer 소개

---

- Producer Application은 카프카에 필요한 데이터를 선언하고 브로커의 특정 토픽의 파티션에 전송
- 리더 파티션을 가지고 있는 브로커와 직접 통신
- `Java` 라이브러리만 공식적으로 지원해서, Java사용을 권장함. 다른 언어로도 지원은 하나 안정성이나 최신 기능지원 등이 떨어짐
- 브로커로 데이터를 전송할 때 내부적으로 파티셔너, 배치 생성단계를 거침
- 내부 구조
  - ![프로듀서 내부구조](https://user-images.githubusercontent.com/37397737/218251229-792faced-8d8d-446f-9297-95480362365f.png)
  - `offset`은 메시지에서 지정하는게 아니라, 클러스터에 데이터가 생성될 때 지정됨
  - 토픽, 메시지 값만 있더라도 데이터 보낼때는 문제 없음
  - 자바 기준 설명
    - ProducerRecord: 생성하는 레코드, 오프셋은 미포함
    - send(): 레코드 전송 요청
    - Partitioner: 어느 파티션으로 전송할지 결정. 기본값으로 DefaultPartitioner로 설정됨
    - Accumulator: 배치로 묶어 전송할 데이터를 모으는 버퍼

## 5-2 Partitioner

---

- 파티셔너는 2가지 버전이 있음
  - UniformStickyPartitioner
    - 2.5.0 버전에서는 지정하지 않을경우 기본 설정
  - RoundRobinPartitioner
  - message key가 있을 때
    - **메시지 키의 해시값**과 파티션을 매핑하여 전송
    - 동일한 메시지 키일 경우 동일한 파티션으로 가게 됨
    - 파티션 개수가 늘어나면 동일한 메시지 키가 다른 파티션으로 갈 가능성이 있음
  - message key가 null일 때
    - RoundRobinPartitioner
      - 데이터가 들어오는 대로 파티션을 순회하면서 전송
      - Accumulator에서 묶이는 양이 적어서 전송 성능이 낮음
    - UniformStickyPartitioner
      - **Accumulator에서 레코드들이 배치로 묶일 때까지 기다렸다가 전송**
      - 배치로 묶이는 차이점만 있을 뿐 파티션은 순회하면서 전송
      - RoundRobinPartitioner보다 향상된 성능을 가짐
- 커스텀 파티셔너
  - Partitioner인터페이스를 제공하므로 인터페이스를 상속받은 사용자 정의 클래스에서 메시지 키 또는 메시지 값에 따른 파티셔닝 지정로직을 만들 수 있음
  - 


## 5-3 Producer 주요 옵션 소개

---

- 필수 옵션
  - bootstrap.servers: 카프카 클러스터에 속한 브로커의 호스트(이름:Port), **2개 이상의 브로커** 정보를 입력하여 가용성을 높임
  - key.serializer: 메시지 키를 직렬화하는 클래스 지정
  - value.serializer: 메시지 값을 직렬화하는 클래스 지정
    - string으로 직렬화/역직렬화 함. 이유는 console-consumer로 디버깅이나 운영상 이점때문, 네트워크나 메시지 크기에서는 불리하긴 함
- 선택 옵션(default 값이 있음) 
  - acks: 전송 성공 여부 확인하는 옵션. 0, 1, -1중 하나로 설정하며 기본값은 1
  - linger.ms: 배치를 전송하기 전까지 기다리는 최소 시간. 기본 값은 0
  - retries: 브로커로부터 에러를 받고 재전송을 시도하는 횟수. 기본값은 21474836547
  - max.in.flight.requests.per.connection: 한 번에 요청하는 최대 커넥션 개수. 설정된 값만큼 전달 요청 수행. 기본값은 5
  - partitioner.class: 레코드를 파티션에 전송할 때 적용하는 파티셔너 클래스 지정. 기본값은 DefaultPartitioner(UniformStickyPartitioner)
  - enable.idempotence: 멱등성 프로듀서로 동작할지 여부. 네트워크 장애가 발생했을 때 데이터 중복전송의 여부. 데이터 전송을 기본값은 false(2.5.0 버전만. 그 이후 버전은 true)
  - transactional.id: 레코드를 트랜잭션 단위로 묶을지 여부. 기본값은 null

## 5-4 ISR(In-Sync-Replicas)와 acks 옵션

---

- 리더 파티션에서 팔로워 파티션으로 데이터를 복제하는데 시간을 걸려 오프셋 차이가 발생할 수 있음
- `ack`로 얼마나 신뢰성 높게 저장할 지 지정가능
  - 복제 개수가 2이상임을 가정. 1일 때는 별차이가 없음
    - acks=0
      - **리더 파티션으로 데이터가 저장되었는지 확인하지 않음**
      - send라는 메소드를 호출하는 것만으로도 ack를 받음
      - 데이터가 몇번 째 오프셋에 저장되었는지도 리턴받지 못함
      - 속도는 가장 빠르나 신뢰도는 낮음 ex) gps
    - acks=1
      - **리더 파티션에만 정상적으로 적재되었는지 확인**
      - 정상적으로 적재될 때까지 재시도하여 신뢰도는 소폭 상승
      - 팔로워에는 데이터가 동기화되지 않을 수 있음
    - acks=-1(all)
      - **리더 파티션과 팔로워 파티션에 모두 정상적으로 적재되었는지 확인**
      - 속도가 현저하게 느림
      - 모든 파티션을 전부 체크하는게 아니라 `min.insync.replicas` 옵션에 따라 토픽 확인을 함.
      1로 설정하면 ack=1과 똑같고, 일반적으로 2까지만해도 안정성은 충분히 보장

## 5-5 Producer Application 개발하기

---

- [git code](https://github.com/bjpublic/apache-kafka-with-**java**) clone 받기
- `apache-kafka-with-java/Chapter3/3.4.1 kafka-producer/simple-kafka-producer` IntelliJ에서 오픈하기
- gradle에서 kafka2.5.0버전 확인하기

  ```gradle
  dependencies {
      implementation 'org.apache.kafka:kafka-clients:2.5.0'
      compile  'org.slf4j:slf4j-simple:1.7.30'
  }
  ```

- [producer 코드](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.4.1%20kafka-producer/simple-kafka-producer)
  - project settings > Modules > main > source path를 src/main/java로 잡아줘야 함
  - ![setting참고](https://user-images.githubusercontent.com/37397737/216829149-4dbf5ab9-ff01-402d-ac66-52e0bbc13d63.png)

- 확인

```bash
bin/kafka-console-consumer.sh --bootstrap-server my-kafka:9092 --topic test --from-beginning
``` 

## 5-6 Message Key를 가진 Producer Application

---

- [key value producer 코드](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.4.1%20kafka-producer/kafka-producer-key-value)

```java
ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, "Pangyo", "Pangyo");
producer.send(record);
```

- 확인

```bash
bin/kafka-console-consumer.sh --bootstrap-server my-kafka:9092 --topic test --property print.key=true --from-beginning
``` 

## 5-7 Partition 번호를 지정한 Producer Application

---


- [Partition 번호 지정 producer 코드](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.4.1%20kafka-producer/kafka-producer-exact-partition)

```java
int partitionNo = 0;
ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, partitionNo, "Pangyo", "Pangyo");
producer.send(record);
```

- 확인

```bash
bin/kafka-console-consumer.sh --bootstrap-server my-kafka:9092 --topic test --property print.key=true --from-beginning
``` 
## 5-8 Custom Partitioner Producer Application

---

- custom partitioner
  - CustomPartitioner인터페이스를 구현함
  - 메시지 키가 Pangyo일 경우 0번 파티션으로 가도록 설정

  ```java
  public class CustomPartitioner  implements Partitioner {

      @Override
      public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes,
                          Cluster cluster) {

          if (keyBytes == null) {
              throw new InvalidRecordException("Need message key");
          }
          if (((String)key).equals("Pangyo"))
              return 0;

          List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
          int numPartitions = partitions.size();
          return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
      }


      @Override
      public void configure(Map<String, ?> configs) {}

      @Override
      public void close() {}
  }
  ```

- [Custom partitioner producer 코드](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.4.1%20kafka-producer/kafka-producer-custom-partitioner)
  - 특정 데이터가 특정 파티션으로 보내야 할 때가 있음
  - config에 위에 생성한 custom partitioner클래스를 설정함

```java
configs.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, CustomPartitioner.class);

KafkaProducer<String, String> producer = new KafkaProducer<>(configs);

ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, "Pangyo", "Pangyo");
```

## 5-9 Record 전송 결과를 확인하는 Producer Application

---

- `send()`메소드는 Future객체를 반환함
- `RecordMetafata`의 비동기 결과를 표현하는 것으로 카프카 브로커에 정상적으로 적재되었는 지에 대한 데이터가 포함
- `get()`메소드를 사용하여 프로듀서로 보낸 데이터의 결과를 동기적으로 가져올 수 있음

- [응답 출력하는 producer 코드](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.4.1%20kafka-producer/kafka-producer-sync-callback)
  - `[main] INFO com.example.ProducerWithSyncCallback - test-2@2`: test토픽에 2번 파티션에 2번 오프셋으로 저장되었다는 뜻
  - `acks=1`이기 때문에 동기적으로 응답을 받을 수 있었음
  - `acks=0`이면 `test-2@-1`처럼 오프셋을 알 수 없음

  ```java
  ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, "Pangyo", "Pangyo");
          try {
              RecordMetadata metadata = producer.send(record).get();
              logger.info(metadata.toString());
          } catch (Exception e) {
              logger.error(e.getMessage(),e);
          } finally {
              producer.flush();
              producer.close();
          }
  ```
## 5-10 Producer Application의 안전한 종료

---

- Accumulator에 저장되어 있는 데이터를 `flush` `close`를 통해 완전히 보내고 종료하여야 함

```java
producer.close()
```

## 퀴즈

---

- 1) 프로듀서에서 데이터를 전송할 때 반드시 배치로 묶어 전송한다 (O/X)
  - X Accumulator가 배치로 묶어 전송하긴 하지만 배치사이즈가 적거나 하나의 레코드가 크거나 하면 하나씩 전송될 수도 있음 
- 2) 메시지 키를 지정하지 않으면 RoundRobinPartitioner로 파티셔너가 지정된다 (O/X)
  - X 메시지 키와 상관없이 옵션으로 지정되는 사항이고 UniformStickyPartitioner가 default임
- 3) ISR은 복제 개수가 2 이상일 경우에만 존재한다 (O/X)
  - X 1인 경우에도 존재
- 4) acks가 all(-1)일 경우 데이터 전송 속도가 가장 빠르다 (O/X)
  - X 신뢰도 가장 높아지고 데이터가 가장 느려짐. acks=0일 때 가장 빠름
- 5) min.insync.replicas=3, 복제 개수가 2일 때 가장 신뢰도 높게 데이터를 전송할 수 있다 (O/X)
  - X 전제조건자체가 틀림.
  - `acks=all`, `min.insync.replicas=2`, `replication factor=3`일 경우 신뢰도가 가장 높음
    - `min.insync.replicas=3`이 아닌 이유는 팔로워 파티션 중 하나라도 장애가 발생하게 되면 더 이상 토픽에 데이터를 수신/송신할 수 없게 될 가능성이 있어 신뢰도와 지속운영까지 겸비하여 2로 설정이 좋음