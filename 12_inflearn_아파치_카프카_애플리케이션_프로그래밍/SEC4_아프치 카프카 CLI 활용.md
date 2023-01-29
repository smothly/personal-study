# SEC4 아파치 카프카 CLI 활용

---

## 4-1 카프카 CLI툴 소개

---

- `CLI`는 카프카를 운영할 때 토픽이나 피티션 개수 변경과 같은 명령을 실행할 때 자주 쓰임
- 선택옵션과 필수옵션이 있어, 기본값들과 설정항목의 내용들을 알고 있어야함

## 4-2 로컬에서 카프카 브로커 실행

---

- Java 설치
  - 설치 후 심볼릭 링크들 설정
  
  ```
  brew install java

  For the system Java wrappers to find this JDK, symlink it with
  sudo ln -sfn /opt/homebrew/opt/openjdk/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk.jdk

  openjdk is keg-only, which means it was not symlinked into /opt/homebrew,
  because macOS provides similar software and installing this software in
  parallel can cause all kinds of trouble.

  If you need to have openjdk first in your PATH, run:
    echo 'export PATH="/opt/homebrew/opt/openjdk/bin:$PATH"' >> ~/.zshrc

  For compilers to find openjdk you may need to set:
    export CPPFLAGS="-I/opt/homebrew/opt/openjdk/include"
  ```   

- [2.5.0 버전 설치 링크](https://kafka.apache.org/downloads) 
- `config/server.properties` 수정
  - listeners, advertised.listeners 속성 localhost로 수정
  - 데이터 위치 log.dirs data폴더로 수정
  - num.partitions=1개로 되어있음 확인
- zookeeper 실행
  - 테스트환경이라 zookeeper는 하나만 띄워짐
  
  ```
  bin/zookeeper-server-start.sh config/zookeeper.properties 
  ```

- 다음 broker 실행

  ```
  bin/kafka-server-start.sh config/server.properties
  ```

- 정상적으로 실행됐는지 확인
  - zookeper와 broker 총 2개의 프로세스가 띄워져 있음
  - port는 9092가 기본

  ```
  bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092 # api 속성들 확인
  bin/kafka-topics.sh --bootstrap-server localhost:9092 --list # 0개 나옴
  ```

- 테스트 편의를 위한 Hosts 설정

  ```
  sudo vi /etc/hosts
  # 마지막 줄에 적어 간편하게 사용하기 위함
  127.0.0.1 my-kafka
  ```

## 4-3 kafka-topics.sh

---

- 카프카 클러스터 정보와 토픽이름만으로 토픽 생성할 수 있음
- 파티션 개수, 복제 개수 등과 같이 다양한 옵션들도 있지만, 미설정하면 브로커에 설정된 기본값으로 생성됨
- 파티션 개수를 늘리기 위해서는 `--alter` 옵션을 활용. **줄이는 건 불가능**하여 `InvalidPartitionsException` 발생

```bash
# 기본 생성
bin/kafka-topics.sh --create \
  --bootstrap-server my-kafka:9092 \
  --topic hello.kafka

# 확인1
bin/kafka-topics.sh --describe\
  --bootstrap-server my-kafka:9092 \
  --topic hello.kafka
  
# 옵션을 활용하여 생성
bin/kafka-topics.sh --create \
  --bootstrap-server my-kafka:9092 \
  --partitions 10 \
  --replication-factor 1 \
  --topic hello.kafka2 \
  --config retention.ms=172800000

# 확인2
bin/kafka-topics.sh --describe\
  --bootstrap-server my-kafka:9092 \
  --topic hello.kafka2

# 임시 생성
bin/kafka-topics.sh --create \
  --bootstrap-server my-kafka:9092 \
  --topic test

# 파티션 개수 늘리기
bin/kafka-topics.sh --alter \
  --bootstrap-server my-kafka:9092 \
  --topic test \
  --partitions 5

# 확인3
bin/kafka-topics.sh --describe\
  --bootstrap-server my-kafka:9092 \
  --topic test

# 파티션 개수 줄이기 - 실패!!!
bin/kafka-topics.sh --alter \
  --bootstrap-server my-kafka:9092 \
  --topic test \
  --partitions 1

# Caused by: org.apache.kafka.common.errors.InvalidPartitionsException: Topic currently has 5 partitions, which is higher than the requested 1.
```

## 4-4 kafka-configs.sh

---

- 토픽의 일부 옵션을 설정
- `--alter`와 `--add-config` 옵션을 사용해서 토픽별로 옵션 설정 가능
- 브로커의 기본값들은 `--broker` `--all` `--describe`옵션을 사용하여 조회 가능

```bash
# replica개수 늘리기
bin/kafka-configs.sh --alter \
  --bootstrap-server my-kafka:9092 \
  --topic test \
  --add-config min.insync.replicas=2

# 확인
bin/kafka-topics.sh --describe\
  --bootstrap-server my-kafka:9092 \
  --topic test

# 옵션 값 확인
bin/kafka-configs.sh --describe \
  --bootstrap-server my-kafka:9092 \
  --broker 0 \
  --all
```

## 4-5 kafka-console-producer.sh

---

- 토픽에 데이터를 콘솔을 통해 넣을 수 있음
- 메시지가 key를 갖게 하려면 `key.separatoe`를 설정하고 `delimiter(기본값 \t)`도 맞춰줘야함
- 동일한 메시지 키의 경우 동일한 파티션에 들어감. 여기서의 핵심은 **동일한 키 내에서 순서를 지킬수 있게 됨!**
- 메시지 키가 null일 경우 Round Robin방식으로 들어감

```bash
# 한 줄이 하나의 레코드로 전송
bin/kafka-console-producer.sh \
  --bootstrap-server my-kafka:9092 \
  --topic hello.kafka

# 메시지를 key:value 형태로 전송
bin/kafka-console-producer.sh \
  --bootstrap-server my-kafka:9092 \
  --topic hello.kafka \
  --property "parse.key=true" \
  --property "key.separator=:"
```

## 4-6 kafka-console-consumer.sh

---

- 토픽에 전송된 데이터를 콘솔을 통해 확인
- `from-biginning` 옵션을 통해 처음 데이터 부터 확인할 수 있음
- `--max-massages` 옵션을 사용하여 최대 컨슘 메시지 개수를 설정할 수 있음
- `--partition` 옵션을 사용해서 특정 파티션만 컨슘할 수 있음
- `--group` 옵션을 사용하면 컨슈머 그룹을 기반으로 동작함. 어느 레코드까지 읽었는지에 대한 데이터를 브로커에 저장함

```bash
# 처음부터 확인하기
bin/kafka-console-consumer.sh \
  --bootstrap-server my-kafka:9092 \
  --topic hello.kafka \
  --from-beginning

# key를 구분하여 확인하기
# key가 없을 경우 null로 출력됨
bin/kafka-console-consumer.sh \
  --bootstrap-server my-kafka:9092 \
  --topic hello.kafka \
  --property print.key=true \
  --property key.separator="-" \
  --from-beginning

# 최대 consume 메시지 조절하기
bin/kafka-console-consumer.sh \
  --bootstrap-server my-kafka:9092 \
  --topic hello.kafka \
  --from-beginning \
  --max-messages 1

# 특정 파티션 조회
bin/kafka-console-consumer.sh \
  --bootstrap-server my-kafka:9092 \
  --topic hello.kafka \
  --from-beginning \
  --partition 1

# 그룹을 통해 조회
bin/kafka-console-consumer.sh \
  --bootstrap-server my-kafka:9092 \
  --topic hello.kafka \
  --from-beginning \
  --group hello-group

# 그룹을 통해 조회할 경우 커밋을 함
# offset파일을 확인하면 됨. __consumer_offsets은 자동으로 생성되는 토픽임
bin/kafka-topics.sh \
  --bootstrap-server my-kafka:9092 \
  --topic hello.kafka \
  --list __consumer_offsets

```
## 4-7 kafka-consumer-groups.sh

---

- 생성된 컨슈머 그룹 확인할 수 있음
- `--describe` 옵션을 사용하면 대상 토픽, 현재 오프셋, 마지막오프셋, 컨슈머 랙, 컨슈머 ID, 호스트를 알 수 있음. 컨슈머 상태를 조회할 수 있음!
  - `컨슈머 랙` = `마지막 오프셋 - 현재 오프셋` 으로 **지연**의 정도를 볼 수 있음
- `--reset-offsets`를 통해 오프셋 초기화 가능
  - reset 종류
    - --to-ealiest
    - --to-latest
    - --to-current
    - --to-datetime
    - --to-offset
    - --shift-by

```bash
# 생성된 컨슈머 그룹 확인
bin/kafka-consumer-groups.sh --list \
  --bootstrap-server my-kafka:9092

# 컨슈머 그룹 상세 확인
bin/kafka-consumer-groups.sh --describe \
  --bootstrap-server my-kafka:9092 \
  --group hello-group

# 컨슈머 그룹 리셋해보기
bin/kafka-consumer-groups.sh \
  --bootstrap-server my-kafka:9092 \
  --group hello-group \
  --topic hello.kafka \
  --reset-offsets --to-earliest --execute

# 다시 컨슘하기
bin/kafka-console-consumer.sh \
  --bootstrap-server my-kafka:9092 \
  --topic hello.kafka \
  --group hello-group
```

## 4-8 그 외 커맨드 라인 툴

---

### kafka-producer-perf-test.sh

- 프로듀서로 퍼포먼스를 측정할 때 사용 됨

```bash
bin/kafka-consumer-perf-test.sh \
  --bootstrap-server my-kafka:9092 \
  --topic hello.kafa \
  --num-records 10 \
  --throughput 1 \
  --record-size 100 \
  --print-metric
```

### kafka-consumer-perf-test.sh

- 컨슈머로 퍼포먼스를 측정할 때 사용 됨

```bash
bin/kafka-consumer-perf-test.sh \
  --bootstrap-server my-kafka:9092 \
  --topic hello.kafa \
  --messages 10 \
  --show-detailed-stats
```

### kafka-reassign-partitions.sh

- 특정 파티션에 몰릴 경우 reassign하여 리더 파티션의 위치를 변경할 수 있음
- ![reassign](https://user-images.githubusercontent.com/37397737/215327912-4e2caf00-1671-4fe7-b325-aed596a86ec1.png)
- 브로커에 `auto.leader.rebalance.enable` 옵션을 통해 자동으로 리밸런싱 됨. 백그라운드 스레드가 일정한 간격으로 리더의 위치를 파악하고 리더 파티션을 리밸런싱 함

### kafka-delete-record.sh

- 특정 offset까지의 record를 삭제하는 커맨드

### kafka-dump-log.sh

- 직접 dump를 떠서 상세 로그들을 확인할 수 있음

## 4-9 카프카 토픽을 만드는 두가지 방법

---

- 생성방법
  - 첫번째는 CLI통해 명시적으로 생성
  - 두번째는 카프카 컨슈머 또는 프로듀서가 생성되지 않은 토픽에 데이터 요청할 때 생성됨
- **유지보수하기에는 명시적으로 생성하여, 데이터의 특성에 따른 토픽을 만드는 것이 좋음**
- 데이터 처리량, 데이터 보관기관, 병렬 처리 등을 고려하여 토픽을 직접 생성하는 것이 좋음
- auto create는 disable가능함

## 4-10 카프카 브로커와 로컬 커맨드 라인 툴 버전을 맞춰야 하는 이유

---

- 브로커의 버전이 업그레이드 되면 CLI의 상세 옵션이 달라져서 버전 차이로 인한 명령어 오류 발생
- 실습에서는 2.5.0의 바이너리 패키지안에서 같이 사용

## 퀴즈

---

- 1) 토픽을 생성하기 위해서는 반드시 kafka-topics.sh를 사용해야 한다 (O /X)
  - X 프로듀서 컨슈머가 없는 토픽에 접근하면 자동으로 생성됨
- 2) kafka-topics.sh를 사용하여 토픽을 생성하면 복제 개수는 항상 1로 설정된다 (O/X)
  - X 브로커 설정에 따라 다름. replication factor 옵션을 통해 변경도 가능
- 3) kafka-console-producer.sh를 사용하여 메시지 키와 메시지 값이 담긴 레코드를 전송할 수 있다 (O/X)
  - O key separator 옵션을 통해 가능
- 4) kafka-console-consumer.sh를 사용하면 오프셋을 확인할 수 있다 (O/X)
  - X message key, value만 확인할 수 있음