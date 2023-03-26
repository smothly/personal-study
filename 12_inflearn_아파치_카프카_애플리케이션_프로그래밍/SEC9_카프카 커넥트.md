# SEC9 카프카 커넥트

---

## 9-1 카프카 커넥트 소개

---

- ![kafka connect](https://images.ctfassets.net/gt6dp23g0g38/2nnjIy51SMV6IpKMSMAhYJ/35d001cc00a1f58e33037a05d315c750/inside-kafka-connect.jpg)
- `소스 커넥터`와 `싱크 커넥터`가 있음
- 데이터 파이프라인 생성 시 반복 작업을 줄이고 효율적인 전송을 이루기 위한 어플리케이션
- 커넥트 내부에 `커넥터`와 `태스크`가 있음
  - 커넥터는 태스크 관리
  - 태스크가 실질적 데이터 처리

## 9-2 커넥터

---

- 소스 커넥터
  - producer 역할
  - 저장소에서 데이터를 가져올 때 반복작업을 줄어들 수 있음 ex) MySQL 소스일 때 다양한 DB와 테이블에 커넥터만 연결하면 됨
- 싱크 커넥터
  - consumer 역할
  - 타겟에 데이터를 보낼때 반복적 작업 줄임
- 커넥터 플러그인
  - 2.6버전에는 미러메이커2, 파일 싱크, 파일 소스 커넥터를 기본으로 제공
  - 추가적인 커넥터는 커넥터 jar파일을 추가하여 사용해야 함
  - 오픈소스 커넥터
    - hdfs, s3, jdbc, elastic search 여러가지 커넥터 공개되어 있은
    - 라이센스 확인은 필요
- 컨버터, 트랜스폼
  - 커넥터에 옵션으루 추가할 수 있음
  - 데이터 처리를 더욱 풍부하게할 수 있음
  - 컨버터: 데이터 처리전에 스키마를 변경하게 도와줌 ex) json coverter, string converter, byterarayconverter
  - 트랜스폼: 각 메시지 단위로 데이터 간단하게 변환 ex) cast, drop, extract field

## 9-3 커넥트 배포 및 운영

---

- 실행 방법
  - 단일 모드 커넥트(standalone mode connect)
    - 1개 프로세스만 실행됨
    - SPOF가 있어 개발환경에 주로 사용
  - **분산 모드 커넥트(distributed mode connect)**
    - 커넥트가 2대 이상의 서버에서 클러스터 형태로 운영
    - 커넥트가 스레드로 실행 됨
    - 무중단으로 스케일 아웃가능
    - Failover가능
  - 커넥트 Rest API
    - 플로그인 종류, 태스크 상태, 커넥터 상태 등을 조회할 수 있음
    - rest api는 기본적으로 제공되고 커넥트와 태스크에 대해서 개발해야 함

## 9-4 단일모드 커넥트

---

- `connect-standalone.properties` 파일을 수정해야 함
  - bootstrap 서버 설정
  - converter
  - schema convert, serialize
  - offset flush 단위
  - plugin 위치
- 실행
  - 커넥트 설정파일과 커넥터 설정파일을 파라미터로 넣어야 함

## 9-5 분산모드 커넥트

---

- failover, scaleout이 용이하기 때문에 상용환경에서 주로 쓰임
- **동일한 group id**를 가져야 동일 클러스터에 묶임
- `connect-distributed.properties` 파일 수정
  - group id
  - converter, serializer
  - 3개의 토픽이 추가적으로 생성됨. group id 별로 3개 토픽이 있어야 함
    - offset
    - config
    - status
  - plugin path

## 9-6 커스텀 소스 커넥터

---

- 소스 파일로부터 데이터 가져와서 토픽으로 넣는 역할
- 커스텀 소스 커넥터 디펜던시`implementation 'org.apache.kafka:connect-api:2.5.0'` 추가 필요
- Source Connector
  - 태스크 실행 전에 커넥터 설정파일 초기화
  - [파일 소스 커넥터 예시](https://github.com/bjpublic/apache-kafka-with-java/blob/master/Chapter3/3.6%20kafka-connector/simple-source-connector/src/main/java/com/example/SingleFileSourceConnector.java)
- Source Task
  - 소스로부터 데이터를 가져와 토픽으로 데이터를 보내는 역할
  - 토픽의 오프셋이 아닌 자체적으로 오프셋을 사용함
    - 소스에서 어디까지 읽었는지 저장하는 역할
    - 중복 적재를 방지함
    - ex) line num
  - [파일 소스 태스크 예시](https://github.com/bjpublic/apache-kafka-with-java/blob/master/Chapter3/3.6%20kafka-connector/simple-source-connector/src/main/java/com/example/SingleFileSourceTask.java)
    - `offsetStorageReader`를 통해 내부 번호를 기록함
    - `poll`이 실질적으로 데이터를 처리하고 리턴함
- 모두다 클래스로 jar파일로 압축하여야 함

  ```java
  jar {
      from {
          configurations.compile.collect { it.isDirectory() ? it : zipTree(it) }
      }
  }
  ```

## 9-7 커넥터 옵션값 설정시 중요도(Importance) 지정 기준

---

- 옵션값의 중요도를 Impotance enum 클래스로 지정할 수 있음
- HIGH, MEDIUM, LOW 3가지로 나누어져 있지만, 명시적으로만 사용함

## 9-8 카프카 싱크 커넥터

---

- 데이터를 토픽으로부터 가져와 타겟에 저장하는 역할 = consumer와 유사함
- 커스텀 커넥터는 jar파일로 플러그인 형태로 사용해야 함
- [싱크 커넥터 예제](https://github.com/bjpublic/apache-kafka-with-java/tree/master/Chapter3/3.6%20kafka-connector/simple-sink-connector/src/main/java/com/example)
  - 구성은 source connector와 비슷함
  - `start`에는 jdbc connection같이 initialize 하는 역할
  - `put` 레코드 데이터 처리. source connector에 `poll`과 비슷함
  - `flush` 커밋시점마다 호출되는 메소드로 offset 확인
  - `stop`에서 커넥트 종료를 잘 해줘야 함

## 9-9 분산모드 카프카 커넥트 실습

---

- `config/connect-distributed.properties` 에서 분산 커넥트 설정
  - `bootstrap.servers` 지정
  - `group id` 그대로
  - `converter`는 StringConverter 사용
  - connect-offsets, configs, status토픽도 설정하여야 함
  - `tasks.max`는 컨슈머처럼 파티션 개수와 똑같이 설정하는 게 일반적임
  
  ```bash
  bootstrap.servers=my-kafka:9092
  group.id=connect-cluster

  key.converter=org.apache.kafka.connect.storage.StringConverter
  value.converter=org.apache.kafka.connect.storage.StringConverter
  key.converter.schemas.enable=false
  value.converter.schemas.enable=false

  offset.storage.topic=connect-offsets
  offset.storage.replication.factor=1
  config.storage.topic=connect-configs
  config.storage.replication.factor=1
  status.storage.topic=connect-status
  status.storage.replication.factor=1
  offset.flush.interval.ms=10000
  ```

- `bin/connect-distributed.sh config/connect-distributed.properties`로 실행
- `curl`을 통해서 sink connector 생성 및 확인

  ```bash
  # 커넥터 플러그인 조회
  $ curl -X GET http://localhost:8083/connector-plugins

  # FileStreamSinkConnector 생성

  $ curl -X POST \
    http://localhost:8083/connectors \
    -H 'Content-Type: application/json' \
    -d '{
      "name": "file-sink-test",
      "config":
      {
        "topics":"test",
        "connector.class":"org.apache.kafka.connect.file.FileStreamSinkConnector",
        "tasks.max":1,
        "file":"/tmp/connect-test.txt"
      }
    }'

  # file-sink-test 커넥터 실행 상태 확인
  $ curl http://localhost:8083/connectors/file-sink-test/status

  # file-sink-test 커넥터의 태스크 확인
  $ curl http://localhost:8083/connectors/file-sink-test/tasks

  # file-sink-test 커넥터 특정 태스크 상태 확인
  $ curl http://localhost:8083/connectors/file-sink-test/tasks/0/status

  # file-sink-test 커넥터 특정 태스크 재시작
  $ curl -X POST http://localhost:8083/connectors/file-sink-test/tasks/0/restart

  # file-sink-test 커넥터 수정
  $ curl -X PUT http://localhost:8083/connectors/file-sink-test/config \
    -H 'Content-Type: application/json' \
    -d '{
        "topics":"test",
        "connector.class":"org.apache.kafka.connect.file.FileStreamSinkConnector",
        "tasks.max":1,
        "file":"/tmp/connect-test2.txt"
    }'

  # file-sink-test 커넥터 중지
  $ curl -X PUT http://localhost:8083/connectors/file-sink-test/pause

  # file-sink-test 커넥터 시작
  $ curl -X PUT http://localhost:8083/connectors/file-sink-test/resume

  # file-sink-test 커넥터 재시작
  $ curl -X POST http://localhost:8083/connectors/file-sink-test/restart

  # file-sink-test 커넥터 삭제
  $ curl -X DELETE http://localhost:8083/connectors/file-sink-test

  ```

## 9-10 카프카 커넥트를 운영하기 위한 웹페이지

---

- [카프카 커넥트를 웹페이지로 관리](https://github.com/kakao/kafka-connect-web)

```bash
# mac에서 내 ip 확인
$ ifconfig | grep "inet " | awk '{ print $2}'

# kafka-connect-ui 도커 실행
$ docker run --rm -it -p 8000:8000 \
          -e "CONNECT_URL=http://{{my-ip}}:8083" \
          landoop/kafka-connect-ui
```

## 퀴즈

---

- 1) 카프카 커넥트는 스레드이다 (O/X)
  - X 커넥터와 태스크가 스레드이고 커넥트는 워커단위로 프로세스로 실행된다.
- 2) 모든 오픈 소스 커넥터는 무료로 사용할 수 있다 (O/X)
  - X 라이센스 확인해야 함
- 3) 싱크 커넥터는 컨슈머 역할을 한다 (O/X)
  - O
- 4) 소스 커넥터는 컨슈머 역할을 한다 (O/X)
  - X
- 5) 태스크는 커넥터당 최대 10개까지 띄울 수 있다 (O/X)
  - X `tasks.max`에 따라 자유롭게 설정 가능. 일반적으로는 파티션 개수와 동일하게 가져감
