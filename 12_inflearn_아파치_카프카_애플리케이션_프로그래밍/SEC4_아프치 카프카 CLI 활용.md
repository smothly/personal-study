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

## 4-4 kafka-configs.sh

---

## 4-5 kafka-console-producer.sh

---

## 4-6 kafka-console-consumer.sh

---

## 4-7 kafka-consumer-groups.sh

---

## 4-8 그 외 커맨드 라인 툴

---

## 4-9 카프카 토픽을 만드는 두가지 방법

---

## 4-10 카프카 브로커와 로컬 커맨드 라인 툴 버전을 맞춰야 하는 이유

---

## 퀴즈