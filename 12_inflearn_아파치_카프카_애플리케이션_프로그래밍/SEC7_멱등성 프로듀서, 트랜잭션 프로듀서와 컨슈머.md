# SEC7 멱등성 프로듀서, 트랜잭션 프로듀서와 컨슈머

---

## 7-1 멱등성 프로듀서

---

- 멱등성: 여러 번 연산을 수행해도 동일한 결과
- 기본 프로듀서의 동작 방식은 `at least once`를 지원. 중복된 데이터가 적재 될 수 있음
- 0.11.0 이후 버전 부터는 `enable.idempotence`옵션을 사용하여 `exactly once delivery`를 지원.  3.0.0 부터는 **true(acks=all)**가 default임
- 동작
  - 데이터를 브로커로 전달할때 **PID(Producer unique ID), Sequence Number를 함께 전달**
  - ![멱등성 프로듀서 예시](https://blog.kakaocdn.net/dn/Bf2AH/btrDyGC6eqp/KOalY5qbaVinUaAo5r0X8k/img.png)

    ```java
    configs.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
    ```

- 한계
  - **동일한 세션에서만 정확히 한번 전달 보장**. PID가 달라지면 동일한지 판단 불가
  - retries=integer.MAX_VALUE로 설정 + acks=all로 설정됨
  - 프로듀서가 여러 번 전송하되 브로커가 여러 번 전송된 데이터를 확인 후 적재하지 않음
- 오류 발생: Seq num이 일정하지 않을 경우 `OutOfOrderSequenceException` 발생

## 7-2 트랜잭션 프로듀서, 컨슈머

---

- 데이터를 저장할 경우 동일한 atomic을 만족시키기 위해 사용됨
- atomic을 만족시킨다는 의미는 다수의 데이터를 동일 트랜잭션으로 묶어 전체 데이터를 처리 or 처리하지 않음으로 하기
  - ![트랜잭션 컨슈머](https://gunju-ko.github.io//assets/img/posts/kafka-transaction/read_commit1.png)
- 트랜잭션 컨슈머는 저장된 레코드를 보고 트랜잭션 commit되었음을 확인하고 데이터를 가져감
- 트랜잭션 프로듀서 설정
  - `transaction.id`를 고유한 ID값으로 설정하면 자동으로 트랜잭션이 됨
  - init, begin, commit 순서대로 수행함
  - begin과 commit사이에 복수의 레코드를 전송하는 형태

    ```java
    KafkaProducer<String, String> producer = new KafkaProducer<>(configs);
    producer.initTransactions();
    producer.beginTransaction();
    String messageValue = "testMessage";
    ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, messageValue);
    producer.send(record);
    logger.info("{}", record);
    producer.flush();


    producer.commitTransaction();
    //producer.abortTransaction();

    producer.close();
    ```
- 트랜잭션 컨슈머 설정
  - `isolation.level`을 read_committed로 설정. 기본값은 read_uncommitted임
  - 커밋이 완료된 레코드들만 읽어 처리

    ```java
    configs.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");

    KafkaConsumer<String, String> consumer = new KafkaConsumer<>(configs);
    ```


## 퀴즈

---

- 1) 멱등성 프로듀서는 리더 파티션이 있는 브로커와 반드시 여러번 통신한다 (O/X)
  - X 반드시는 아님
- 2) read_committed로 설정한 컨슈머는 커밋이 완료된 레코드만 읽는다 (O/X)
  - O
- 3) 멱등성 프로듀서는 애플리케이션이 종료될 때 커밋을 수행한다 (O/X)
  - X 명시적으로 commit 수행하고 close로 안전하게 종료해야 함