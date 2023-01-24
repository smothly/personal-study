# SEC3 카프카 클러스터 운영

---

## 3-1 카프카 클러스터를 운영하는 여러가지 방법

---

- ![iaas vs paas vs saas](https://blog.kakaocdn.net/dn/5C0qf/btrndGrDNfB/ev2IHy68QBcq9GJk5QwNS1/img.png)
- on-premise
  - 사용자가 자체적으로 전산실에 서버를 직접 설치해 운영
  - 초기 도입 비용과 유지비수 비용 발생
  - 카프카는 물리장비와 네트워크를 설치하고 오픈소스 or 기업용 카프카(컨플루언트 플랫폼) 설치 및 운영
- iaas
  - 컴퓨팅을 리소스로 발급받아서 사용
  - 운영체제, 어플리케이션 등을 직접 설정 배포, 운영
  - 카프카는 컴퓨팅 리소스에 오픈소스 or 기업용 카프카 설치 및 운영
  - ex) ec2
- paas
  - 어플리케이션 개발 및 실행환경을 제공
  - ex) lambda, elastic beanstalk
- saas
  - 소프트웨어를 업체에서 관리하고 기능을 제
  - 카프카는 다양한 주변 생태계(ksql db, 모니터링 등)을 옵션으로 제공
  - ex) msk, dropbox, salesforce

## 3-2 SaaS형 아파치 카프카 소개

---

- confluent
  - 카프카 창시자인 제이크립새가 링크드인에서 퇴사하여 개발한 회사
  - 현재 50억 달러 가치
  - connecotr, ksqldb, restproxy 등 kafka생태계의 범위를 넓히는중
- confluent cloud vs confluent platform
  - ![confluent비교](https://dattell.com/wp-content/uploads/2020/07/Kafka-vs-Confluent-Table1.png)
  - confluent cloud가 다양한 기능 제공
  - confluent platform은 설치형 카프카 클러스터로 온프레미스 환경일 때 주로 사용
- msk
  - AWS Managed Streaming for Apache Kafka  
  - 카프카 클러스터를 생성, 업데이트, 삭제 등과 같은 운영 요소를 대시보드를 통해 제공
  - 카프카 특정 버전 설치 가능
  - AWS와 호환이 좋음

## 3-3 SaaS형 아파치 카프카 장점과 단점

---

- 장점
  - 인프라 관리의 효율화
    - 최소 3대의 브로커 운영, 2.x 버전까지는 최소 3대의 주키퍼 운영을 SaaS가 대부분 해줌
    - ex) broker recovery, broker scale out
  - monitoring dashboard 제공
    - broker의 지표들을 수집하고 적재하여 데이터를 시각화 해줌
  - 보안 설정
    - ssl, sasl, acl 등 다양한 종류의 보안 설정 가능
- 단점
  - 서비스 사용 비용
    - 직접 서버 발급 비용과 2배 이상 차이남. 하지만, **인력비용과 운영비용을 같이 고려해야 함**
  - 커스터마이징 제한
    - 서버의 최적화나 브로커 특정 옵션이 제한이 걸려있음
    - 멀티/하이브리드 클라우드 일 경우 설정이 불편
  - 클라우드의 종속성
    - (피치 못할 이슈로) 클라우드를 옮길 경우 까다로움

## 퀴즈

- 1) 컨플루언트 플랫폼은 클라우드에서 제공하는 SaaS형 카프카이다 (O/X) 
  - X SaaS는 confluent cloud
- 2) SaaS를 사용하면 비용이 반드시 절감된다 (O/X)
  - X 일반적인 경우 비용은 오히려 증가
- 3) SaaS형 카프카를 사용하더라도 주키퍼는 사용자가 운영해야 한다 (O/X)
  - X 업체에서 운영
- 4) AWS MSK는 컨플루언트의 대표적인 클라우드 서비스이다 (O/X)
  - X AWS의 대표적인 서비스
- 5) SaaS형 카프카를 사용하면 카프카에 대한 깊은 지식이 없어도 무관하다 (O/X)
  - X 
