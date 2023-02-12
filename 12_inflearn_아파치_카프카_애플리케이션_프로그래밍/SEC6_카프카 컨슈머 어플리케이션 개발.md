# SEC6 카프카 컨슈머 애플리케이션 개발

---

## 6-1 Kafka Consumer 소개

---

- 카프카 브로커에 적재된 데이터를 가져와 처리함
- 컨슈머 내부구조
  - ![컨슈머 내부구조](https://user-images.githubusercontent.com/37397737/218251245-eed64f59-e74b-4a1a-8605-b3b6b9a72318.png)
  - Fetcher: 리더 파티션으로부터 레코드들을 미리 가져와서 대기
  - poll(): Fetcher에 있는 레코드들을 리턴하는 레코드
  - ConsumerRecords: 처리하고자 하는 레코드들의 모음. 오프셋이 포함되어 있음
- **커밋**을 통해 해당 오프셋까지 처리했음을 관리함
- 
## 6-2 Consumer Group

---

- 각 컨슈머 그룹끼리는 논리적으로 다른 로직을 가지고 있다고 판단함
- 토픽을 구독하는 형태로 각 컨슈머는 1개 이상의 파티션에 할당되어 데이터를 가져갈 수 있음
- 1개 컨슈머는 여러개의 파티션에 할당될 수 있으므로, 일반적으로는 파티션과 컨슈머 개수가 같은게 좋음
  - 컨슈머 개수 > 파티션 개수 => 여분의 컨슈머는 유휴 상태로 남게 됨
- **컨슈머 그룹 활용 이유**
  - 동기적으로 실행되는 수집 에이전트의 경우 하나에 장애가 나면 다른 수집 로직에 영향을 줄 수 있음
  - 용도에 따라 컨슈머 그룹으로 나누어 장애 대응에 유연하게 대응할 수 있고, 컨슈머의 스케일 아웃 등 리소스 관리도 유연하게 할 수 있음
  - 목적에 따라 컨슈머를 분리할 수 있음

## 6-3 Rebalancing

---

- Failover방식으로, 장애가 발생한 컨슈머에서 장애가 발생하지 않은 컨슈머로 파티션을 소유권을 넘김
- ![rebalancing](https://miro.medium.com/max/1400/1*aqisiCLbEBlk4RGOVbj6dw.jpeg)
- 두가지 상황에 따른 할당이 변경되는 과정이 있어 **데이터 처리 중 리밸런싱에 대응하는 코드를 작성해야 함**
  - 컨슈머가 추가되는 상황
  - 컨슈머가 제외되는 상황
- 파티션이 많을수록 리밸런싱하는 시간이 오래걸려, 장애와 비슷하게 되어 리밸런싱이 덜 발생하게 하고 대응할 수 있또록 준비해야 함

## 6-4 Commit

---

- ![커밋](https://velog.velcdn.com/images/dobby/post/f930dc73-2b70-4ecb-abfe-534d5c4e5c3a/image.png)
- 커밋을 하면 카프카 브로커 내부에서 `cosumer_offsets`에 기록 됨
- 커밋이 잘못 동작한다면 데이터 처리의 중복이 발생할 수 있어, **컨슈머에서 오프셋 커밋을 정상적으로 처리했는지 검증**해야함

## 6-5 Assignor

---

- **파티션 할당 정책은 컨슈머의 Assignor에 의해 결정**됨
  - RangeAssignor: 각 토픽에서 파티션을 숫자로 정렬, 컨슈머를 사전 순서로 정렬하여 할당 (2.5.0 버전의 기본값)
  - RoundRobinAssignor: 모든 파티션을 컨슈머에서 번갈아가면서 할당
  - StickyAssignor: 최대한 파티션을 균등하고 배분하면서 할당
- 일반적으로는 파티션 개수와 컨슈머 개수를 맞춰 사용하여 할당이 큰 의미가 없음

## 6-6 Consumer 주요 옵션 소개

---

- 필수 옵션
  - bootstrap.servers: 카프카 클러스터에 속한 브로커의 호스트 이름:포트 를 1개 이상 작성
  - key.deserializer: 메시지 키를 역직렬화하는 클래스
  - value.deserializer: 메시지 값을 역직렬화하는 클래스
- 선택 옵션
  - group_id: 컨슈머 그룹아이디를 지정, subscribe() 메소드로 토픽을 구독하여 사용할 때 이 옵션을 필수로 넣어야 함. 기본값은 null
  - auto.offset.reset: 컨슈머 오프셋이 없는 경우(= commit이 없는 경우) 어느 오프셋부터 읽을지 선택하는 옵션. 기본값은 latest
  - enable.auto.commit: 자동 커밋/수동 커밋 여부. 기본값은 true
  - auto.commit.interval.ms: 자동 커밋일 때 오프셋 커밋 간격. 기본값은 5000(5초)
  - max.poll.records: `poll()`메소드를 통해 반환되는 레코드 수. 기본값은 500
  - session.timeout.ms: 컨슈머가 브로커와 연결이 끊기는 최대 시간. 짧을수록 좋으나 너무 짧으면 리밸런싱이 잦을 수 있음. 기본값은 10000(10초)
  - heartbeat.interval.ms: 하트비트를 전송하는 시간 간격. 기본값은 3000(3초)
  - max.poll.interval.ms: `poll()` 메서드를 호출하는 간격의 최대 시간. 기본값은 300000(5분)
  - isolation.level: 트랜잭션 프로듀서가 레코드를 트랜잭션 단위로 보낼 경우 사용. 

## 6-7 auto.offset.reset

---

- 컨슈머 오프셋이 없는 경우(= commit이 없는 경우) 어느 오프셋부터 읽을지 선택하는 옵션. 기본값은 latest
  - latest: 가장 최근에 넣은 오프셋부터 읽기 시작
  - earliest: 가장 오래전에 넣은 오프셋부터 읽기 시작
  - none: 커밋 기록을 찾아보고 없으면 오류 반환
- 이미 토픽에 데이터가 있을 경우 새로 만든 컨슈머가 어떻게 시작할지 활용

## 6-8 Consumer Application 개발하기

---

- [consumer 기본 코드](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.4.2%20kafka-consumer/simple-kafka-consumer)
  - key, value 둘 다 StringDeserializer 사용
  - while true로 `poll` 해가며 처리
  - `bin/kafka-console-producer.sh --bootstrap-server my-kafka:9092 --topic test` 로 consumer 정상적으로 동작하는지 체크


## 6-9 수동 commit Consumer Application

---

- `poll()`메소드 호출 이후 `commitSync()` 메소드를 호출하면 오프셋 커밋을 명시적으로 수행 가능
- `poll()`메소드로 받은 레코드의 처리가 끝난 이후 `commitSync()` 메소드를 호출해야 함
- 동기 오프셋 커밋(레코드 단위)
  - 브로커로부터 응답을 받을 떄까지 기다림
  - 커밋 응답을 기다리는 동안 데이터 처리가 일시적으로 중단되어 짧은 간격으로 수행할 경우 브로커와 컨슈머에 무리가 감
- 비동기 오프셋 커밋
  - `commitAsync()` 메소드를 호출하여 사용
  - callback을 받을 수 있는데, 완료여부에 대한 대응을 할 수 있음 
- [consumer 수동 commit 코드](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.4.2%20kafka-consumer/kafka-consumer-sync-commit)
  - autocommit은 interval 을 통해 자동으로 commit함
  - autocommit=False일 경우 명시적으로 commit해줘야 함
- [consumer 수동 비동기 commit 코드](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.4.2%20kafka-consumer/kafka-consumer-async-commit)
  - callback을 통해 백그라운드 스레드에서 커밋을 수행하도록 함


## 6-10 Rebalanc listner Counsmer Application

---

- 리밸런스를 감지하기 위해 `ConsumerRebalanceListener` 인터페이스를 지원
- `ConsumerRebalanceListener` 인터페이스로 구현된 클래스는 `onPartitionAssgined()` ,`onPartitionRevoked()` 메소드로 이루어짐
  - `onPartitionAssgined()`: 리밸런스가 끝난 뒤에 파티션이 할당 완료되면 호출되는 메소드
  - `onPartitionRevoked()`: 리밸런스가 시작되기 전에 호출되는 메소드. 마지막으로 처리한 레코드를 커밋하기 위해 해당 메소드에 커밋을 구현하여 처리할 수 있음
- [consumer 리밸런싱을 처리하는 코드](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.4.2%20kafka-consumer/kafka-consumer-rebalance-listener)
  - `subscribe()`메소드 옵션으로 리밸런스 리스너를 넣게 됨
  - 여러개 consumer application을 띄워 하나를 종료하여 rebalancing과정 목격
  - 리밸런스 구현체
  
  ```java
  public class RebalanceListener implements ConsumerRebalanceListener {
    private final static Logger logger = LoggerFactory.getLogger(RebalanceListener.class);

    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        logger.warn("Partitions are assigned : " + partitions.toString());

    }

    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        logger.warn("Partitions are revoked : " + partitions.toString());
    }
  }
  ```

## 6-11 Partition assign Counsmer Application

---

- `assgin`메소드를 통해 특정 토픽에 특정 파티션에 직접 할당하도록 동작 가능
- group id가 필수 값이 아님
- [partition assign 하는 코드](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.4.2%20kafka-consumer/kafka-consumer-exact-partition)

  ```java
  consumer.assign(Collections.singleton(new TopicPartition(TOPIC_NAME, PARTITION_NUMBER)));
  ```

## 6-12 Counsmer Application 안전한 종료

---

- 정상적으로 종료되지 않을 경우 컨슈머는 session timeout이 발생할 때까지 컨슈머 그룹에 남음
- 컨슈머를 안전하게 종료하기 위해 `wakeup()`메소드를 지원
- `wakeup()` 메소드 실행 -> `poll()` 실행 시 `WakeupException` 예외 발생 -> 예외처리시 데이터 처리를 위해 사용한 자원들 해제
- try catch문으로 예외처리
- [셧다운을 처리하는 코드](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.4.2%20kafka-consumer/kafka-consumer-sync-offset-commit-shutdown-hook)

  ```java
  static class ShutdownThread extends Thread {
      public void run() {
          logger.info("Shutdown hook");
          consumer.wakeup();
  }
  ```

  ```java
  Runtime.getRuntime().addShutdownHook(new ShutdownThread());

  try {
      while (true) {
          ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
          for (ConsumerRecord<String, String> record : records) {
              logger.info("{}", record);
          }
          consumer.commitSync();
      }
  } catch (WakeupException e) {
      logger.warn("Wakeup consumer");
  } finally {
      logger.warn("Consumer close");
      consumer.close();
  }
  ```
## 6-1 ~ 12 퀴즈

---

- 1) 1개 컨슈머는 여러 개의 파티션에 할당될 수 있다 (O/X)
  - O
- 2) 하나의 파티션은 여러 컨슈머에 할당될 수 있다 (O/X)
  - X 동일한 컨슈머 그룹에서는 불가능, 다른 컨슈머 그룹에서는 가능
- 3) 파티션 개수보다 컨슈머 그룹의 컨슈머 개수가 많으면 오류가 발생한다 (O/X)
  - X 오류는 발생하지 않고, 여분의 consumer는 idle상태로 남은
- 4) 안정적인 데이터 처리를 위해 session.timeout.ms는 heartbeat.interval.ms보다 작은 값이여야 한다 (O/X)
  - X 말이 안되는 가정임. 기본값은 session10초 heartbeat3초임
- 5) 컨슈머를 안정적으로 종료하기 위해 session.timeout.ms을 사용한다 (O/X)
  - Wakeup 훅 사용


## 6-13 Multithread Consumer Application

---

- 1thread -> 1consumer
- 1process에 여러 counsumer thread를 위치시킬 수 있음
  - ![multi thread 사진](https://howtoprogram.xyz/wp-content/uploads/2016/05/model2.png)
- 데이터를 병렬처리하기 위해 파티션 개수와 컨슈머 개수를 동일하게 맞추는 것이 가장 좋은 방법
- 멀티 스레딩: 배포는 간단해지나 스레드 장애시 다른 스레드에 영향을 끼치기 때문에 이 부분을 생각해야함
- 멀티 프로세싱: 배포 자동화나 여러 프로세스를 띄우는데 무리없는 k8s, ecs환경이라면 멀티프로세싱이 나음

## 6-14 Consumer lag

---

- **컨슈머 랙 = 최신 오프셋(LOG-END-OFFSET) - 컨슈머 오프셋(CURRENT-OFFSET)**
  - ![consumer lag](https://miro.medium.com/max/1400/1*yOgr0yiTSGI-XHtTQlatXA.png)
- 컨슈머가 정상적으로 동작하는지 확인할 수 있기 때문에 필수적인 컨슈머 모니터링 지표
  - 컨슈머 장애 확인
  - 파티션 개수 정하기
  - ex) 데이터양이 증가할 경우 파티션과 컨슈머를 늘릴 수 있는 판단의 지표
- 1개 토픽에 3개의 파티션, 1개의 컨슈머 그룹이 토픽을 구독하면 랙은 총 3개
- producer 처리량 > consumer 처리량이면 컨슈머 랙이 늘어남

## 6-15 How to consumer lag monitoring

---

- 확인하는 방법 3가지
  - 카프카 명령어 수행
    - 컨슈머 랙을 일회성으로 확인하기에 좋음. 지표를 지속적으로 모니터링하기에는 부족
    ```bash
    bin/kafka-consumer-groups.sh --bootstrap-server my-kafka:9092 --group my-group --describe
    ```
  - metrics() 메서드 사용
    - 한계점
      - **컨슈머가 정상 동작할 경우**에만 확인할 수 있음
      - 모든 컨슈머 어플리케이션에 랙 모니터링 코드를 중복해서 작성해야 함
      - 카프카 third party 어플리케이션(logstash, fluentd, telegraf)에는 모니터링 코드를 추가할 수 없음
  - 외부 모니터링 툴 사용
    - 모든 토픽의 모든 컨슈머에 대한 지표를 트래킹할 수 있음
    - Datadog, Confluent Control Center와 같은 카프카 종합 모니터링 툴 사용 가능
    - Burrow는 오픈소스

## 6-16 Burrow

---

- 컨슈머 랙 체크 툴
- 다수의 카프카 클러스터를 연결하여 모니터링할 수 있음
- REST API 기반
- 컨슈머 랙 이슈 판별
  - 단순한 threshold를 가지고 이슈를 판정짓는거는 의미없음
  - `Evaluation`방식을 사용하여 sliding window계산을 통해 문제가 생긴 파티션과 컨슈머 상태(OK, WARNING, ERROR) 표현
  - `WARNING`은 컨슈머 랙이 지속적으로 늘어나는 경우 발생
  - `ERROR`는 컨슈머 오프셋이 멈춘 경우 파티션은 STALLED, 컨슈머는 ERROR
- Burrow를 통한 모니터링 예시
  - ![monitroing example](https://user-images.githubusercontent.com/37397737/218304640-859b49cb-1512-4b25-926e-ae792c4e86eb.png)

## 6-3 ~ 16 퀴즈

---

- 1) 컨슈머 랙은 토픽별로 1개 측정할 수 있다 (O/X)
  - X 컨슈머 그룹 개수에 따라 배수가 되어 가변적임
- 2) 파티션 개수가 50개인 토픽의 컨슈머 랙은 50개가 측정 된다 (O/X)
  - X 컨슈머 그룹 개수에 따라 배수가 되어 가변적임
- 3) 컨슈머 랙을 모니터링하는 가장 좋은 방법은 외부 모니터링  툴을 사용하는 것이다 (O/X)
  - O
- 4) 프로듀서의 전송량이 초당 10개, 컨슈머의 처리량이 초당 1개, 파티션이 20개이면 컨슈머 랙이 증가한다 (O/X)
  - X 컨슈머가 20개로 초당 20개의 데이터를 처리할 수 있어 증가하지 않음