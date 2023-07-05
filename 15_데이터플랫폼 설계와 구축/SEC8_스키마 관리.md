# 스키마 관리

## 8.1 스키마 관리가 필요한 이유

---

- 데이터 구조가 미리 정의되어야 하는 DB들은 ETL 파이프라인을 수작업으로 변경해야하는 것이 필요
  - 컬럼명이 바뀔 경우 수집 중단
- `schema on read`
  - 데이터를 읽을 때 스키마를 추론함
  - HDFS(분산 파일 스토리지)에 파일을 그대로 저장하여 수집 프로세스를 단순화
  - 수집단에서 발생할 수 있는 **스키마 불일치 문제를 데이터 처리 파이프라인으로 내려보냄**
  - 수집의 장애가 없을 뿐 운천적인 해결은 불가능함
  - ![schema on read](https://drek4537l1klr.cloudfront.net/zburivsky/Figures/CH08_F04_Zburivsky.png)

## 8.2 스키마 관리 방식

---

- 소스와 타겟의 스키마 정보 전부 저장해야함
- 스키마를 계약으로 다루는 방법(schema as a contract)
  - 개발자가 스키마 정보를 중앙 레지스트리에 개시
  - 소비자는 최신 스키마 버전을 가져와 사용
  - 변경 전에 생성된 데이터도 변경된 스키마로 처리가 가능해야 함 ex) 기존 컬럼 NULL 적재
  - 잘 운영만 된다면 데이터 생산자 소비자가 명확히 구분되고 결합도도 없는 이상적인 상태
  - 전제 조건
    - 높은 수준의 개발 프로세스 성숙도
      - 이전 버전과 호환되는지 자동 확인
      - CI/CD 고도화 필요
    - third party 툴의 명확한 데이터 소유자
      - SaaS로 부터 직접 스키마 업데이트를 요청할 수 없음
- 데이터 플랫폼으로 스키마 관리
  - 내부데이터 소스와 third-party데이터의 통합 지점임
  - 변환 파이프라인에서 문제가 있는 경우가 많은데. 스키마를 최신 상태로 호스팅하기 위한 논리적인 장소가 됨
  - 이점
    - 장애가 발생하기 전에 스키마 변경 감지하여 대처 가능
    - 스키마 카탈로그를 최신화 할 수 있음. 데이터 검색에 중요
    - 스키마 변경 이력 확보 가능
  - 스키마 관리단계는 공통 데이터 변환할 때 주로 수행함
- 스키마 관리 모듈
  - 공통 데이터 변환 파이프라인에서 스키마 관리를 추가해야 함
  - 스키마 관리 절차
    - 스키마 없을 경우
      - 수신 데이터에서 스키마 추론
      - 레지스트리에 등록
    - 있을 경우
      - 레지스트리로부터 현재 스키마를 가져옴
      - 수신 데이터로부터 스키마를 추론
      - 스키마 비교. 이전버전과 호환성 유지 방식으로 신규 스키마를 만듬
      - 신규 버전을 레지스트리에 등록
- Spark 스키마 추론
  - nested json 형식도 쉽게 추론함
  - 주의사항
    - default로 1000개의 샘플로 추론함. 전체 배치로 실행하는 것이 좋으나 성능상 이슈가 있음 ex) rdbms 같은 경우 스키마가 있기 때문에 조금만 읽어도 됨
    - 컬럼명에 의존 함. csv는 헤더 옵션 지정해야 함
    - 일반화된 타입을 주로 사용. 일반화된 타입을 지정할 수 없는 경우(nested와 숫자 컬럼이 혼재)는 `_corrupt_record`라는 특수 필드를 두어 문제 행을 검사할 수 있음
  - 대부분 서비스에서는 Spark를 통한 스키마 추론을 사용하고 있음
  - 레지스트리에 avro 타입으로 저장하여도 많이 사용
  
  ```scala
  scala> val df = spark.read.json("/data/sample.json")
  df: org.apache.spark.sql.DataFrame = [_id: string, about: string ... 19 more 
  ➥ fields]
  
  scala> df.printSchema
  root
  |-- _id: string (nullable = true)
  |-- about: string (nullable = true)
  |-- address: string (nullable = true)
  |-- age: long (nullable = true)
  |-- balance: string (nullable = true)
  |-- company: string (nullable = true)
  |-- email: string (nullable = true)
  |-- eyeColor: string (nullable = true)
  |-- favoriteFruit: string (nullable = true)
  |-- friends: array (nullable = true)
  |    |-- element: struct (containsNull = true)
  |    |    |-- id: long (nullable = true)
  |    |    |-- name: string (nullable = true)
  |-- gender: string (nullable = true)
  |-- guid: string (nullable = true)
  |-- index: long (nullable = true)
  |-- isActive: boolean (nullable = true)
  |-- latitude: double (nullable = true)
  |-- longitude: double (nullable = true)
  |-- name: string (nullable = true)
  |-- phone: string (nullable = true)
  |-- picture: string (nullable = true)
  |-- registered: string (nullable = true)
  |-- tags: array (nullable = true)

  |    |-- element: string (containsNull = true)
  ```

- 실시간 데이터 파이프라인에서의 스키마 관리
  - 메시지 마다 처리를 해야 하므로 스키마 추론의 제약이 있음
  - protobuf나 avro와 같은 바이너리 형식을 사용하여 메시지의 크기를 최소화해야 함
- 스키마 변경 모니터링
  - default값을 0으로 바꾸고 새 컬럼을 추가하면 집계는 되지만 값이 0으로 나오는 논리적 오류가 발생함.
  - 그래서 경보 메커니즘이 중요
  - 파이프라인 처리내역에 스키마 변경 이벤트도 추가해야 함
    - 로그 집계하여 스키마가 변경 되면 데이터 플랫폼에 알리는 식으로 경보 메커니즘 제공 가능
    - 스키마 변경의 영향을 받는 데이터 소스를 알고 있으면 다운스트림 변환과 리포트들을 쉽게 파악할 수 있음
  - ![스키마 변경 모니터링](https://drek4537l1klr.cloudfront.net/zburivsky/Figures/CH08_F09_Zburivsky.png)

## 8.3 스키마 레지스트리 구현

---

- 스키마를 표현하고 저장하는 방법
- 스키마 정의를 저장, 가져오기, 업데이트할 수 있는 데이터 베이스

### 아파치 아브로 스키마

- 자체 내장 스키마를 갖고 있음. 대부분의 데이터 타입은 표현 가능함
- 스키마를 avro포맷으로 저장하기

  ```scala
  import org.apache.spark.sql.avro.SchemaConverters
  val df = spark.read.json("/data/sample.json")
  
  val avroSchema = SchemaConverters.toAvroType(df.schema, false, 
  ➥ "sampleUserProfile")
  avroSchema: org.apache.avro.Schema = {"type":"record","name":
  ➥ "sampleUserProfile","fields":[{"name":"_id","type":["string","null"]} ...
  ```

- 스키마의 변경내역도 추적 가능
- 스키마의 저장 위치 = 스키마 레지스트리
- JSON 데이터임

### 스키마 레지스트리 솔루션

- aws/gcp/azure data catalog
  - 기존 데이터 소스의 자동 검색에 중점을 두고 있음
  - 고유의 방식으로 스키마 표현
- Glue가 그나마 일반적인 파일 형식으로 된 스키마 검색과 API를 통한 스키마 업데이트 지원
  - 히스토리는 지원하지 않음
- confluent registry가 유일한 솔루션이나 kafka를 같이 설치해야 함

### 메타데이터 계층의 스키마 레지스트리

- API 방식으로 지원하는 것이 효율적
- ![schema registry](https://drek4537l1klr.cloudfront.net/zburivsky/Figures/CH08_F10_Zburivsky.png)
- 스키마 레지스트리의 주요 작업
  - 데이터 소스의 현재 버전을 가져온다.
  - 존재하지 않는 경우 데이터 소스에 대한 신규 스키마를 생성한다. 
  - 기존 데이터 소스에 대한 신규 버전의 스키마를 추가한다.
- 저장되어야할 속성
  - id
  - 버전
  - 스키마(avro 포맷의 텍스트)
  - 생성 타임스탬프
  - 최종 수정 타임스탬프

## 8.4 스키마 진화 시나리오

---

- 데이터 구조 변경을 어떻게 다루는지
- default값이 있으면 이전/이후 스키마와 둘다 호환 됨
- 이전 버전 스키마를 유지해야하는 경우는 스키마가 변경되더라도 기존 데이터는 읽을 수 있어야 함
  - 하지만, 재처리를할 때 신규 스키마를 사용할 경우 과거 데이터 집계는 망가짐

### 스키마 호환성 규칙

- 이전 버전 호환성 = backward-compatibility
- 이후 버전 호환성 = forward-compatibility
- avro에서 스키마에 컬럼을 추가하는 것은 forward임
  - 이전버전의 스키마로 최신의 데이터를 처리하는 것
  - 이것은 오류 미발생에도 도움을 줌
- 타입 변경은 추가/삭제와 비슷함
- 컬럼 타입 변경은 데이터 손실이 없다는 제한 하에 데이터 타입을 승격 함
- 최대한 avro에서 제공하는 자동 데이터 타입 변환을 구현하는 것이 좋음

### 스키마 진화와 데이터 변환 파이프라인

- 특정 컬럼으로 계산하는 경우 동작이 중단되지 않으려면 이전 버전의 스키마를 계속 사용해야 함
- 디폴트 값이 있으면 오류는 발생하지만 논리적인 장애, 디폴트 값이 없으면 파이프라인 장애가 발생
- 디폴트 값이 있는 신규 컬럼 추가/삭제가 가장 안전한 스키마 변경 작업
  - 스파크 스키마 추론 기능을 사용하면 null값이 default이기 때문에 파이프라인이 중단될 가능성은 적음
- 경험적으로는 **파이프라인을 최신 버전이 아닌 이전 버전의 스키마로 유지하는 것이 안정적이고 회복력 있음** = forward
- 신규 버전 전환은 모든 변경이 이뤄진 후에 하는 것이 좋음
- 논리적인 오류는 스키마 변경에 대한 알람을 구축하는 것으로 해결해야 한다. 아니면 잘못된 리포트를(NULL로 집계된) 마주칠 수 있을 것임!

## 8.5 스키마 진화와 데이터 웨어하우스

---

- 위에 내용은 데이터 변환 파이프라인에서 스키마 변경을 처리하는 방법
- DW에서는 스키마를 조정하거나 적용하지 않아야 함
- 외부 스키마 레지스트리와 통합되지 않음
- 공통 변환 파이프라인에서 스키마 관리 모듈을 사용해 스키마 변경을 관리할 수 있음!
- 스키마 변경 자동화
  - avro 데이터 타입을 DW 타입으로 매핑하는 모듈을 개발해야 함
  - 테이블을 삭제하고 신규 테이블 생성해서 하는 방법도 있으나 데이터가 작을 경우에만 해당함
  - 대부분의 DDL은 테이블에 Lock이 걸려서 어떻게 동작하는지 제대로 이해하고 실행할 것

### 클라우드 데이터 웨어하우스의 스키마 관리 기능

- redshift, synapse는 전통적인 DW
  - 둘 다 스키마 추론 기능은 제공하지 않음
  - 일반적인 RDBMS와 비슷하지만 columnar DW임
- bigquery
  - 기존 파일의 스키마를 기반으로 자동으로 수행됨
  - alter table은 미지원함 (지금은 지원함)

## 요약

---

- RDB/DW 같은 경우 스키마가 변경됨녀 ETL이 중단되는 경우가 많음
- 컬럼 추가, 삭제, 컬럼명 변경, 컬럼 데이터 타입 변경이 있음
- 데이터 통제가 안되는 3rd-party 툴은 스키마 관리가 어려워질 수 있음
- 데이터 플랫폼에서 스키마 관리
  - ETL장애 발생전에 스키마 변경을 미리 감지하여 대응과 회복력 있는 ETL 구축
  - 스키마 정보를 관리 유지함으로써 데이터 검색 및 셀프 서비스에도 도움
  - 스키마 변경 이력을 관리함으로써 기존 데이터 처리 및 문제 해결에도 도움이 됨
- 스키마 레지스트리
  - 데이터 변환 파이프라인이 최신 버전을 가져오기
  - 스키마의 시간에 따른 변경 내용 추적하기
- 데이터 변환 파이프라인에 스키마 관리를 추가하기
  - 스키마 레지스트리의 존재 여부 확인 후 추론하여 추가하기
  - 이미 존재하는 경우는 현재 스키마와 추론된 스키마를 비교하여 신규 버전을 레지스트리에 개시
- 실시간 스키마 레지스트리는 메시지마다 처리하기 때문에 스키마 버전을 관리하기 어려워 수작업으로 개시/관리해야 함
- 스키마 변경 처리를 자동화할 수 없는 경우도 있으니 경보 메커니즘을 꼭 만들 것
- 파이프라인 설정 정보에 파이프라인 관련 정보가 있으니 영향받는 다운스트림 변환과 리포트를 쉽게 변환할 수 있음
- 스키마 레지스트리 서비스로는 glue가 있으나 제약이 많음. 파일 -> NoSQL -> API 형태로 스키마 레지스트리를 업데이트 해나갈 것
