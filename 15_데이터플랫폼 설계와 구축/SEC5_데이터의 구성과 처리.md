# 데이터의 구성과 처리

- 처리계층은 스토리지에서 데이터를 읽고 변환, 변환한 결과를 다시 스토리지에 저장
  - 배치모드, 스트리밍모드 둘 다 처리 가능해야 함
  - 다양한 유형의 비즈니스 로직 적용
  - 데이터 플랫폼과 Interactive하게 작업하기

## 5.1 데이터 플랫폼에서 처리 계층을 별도로 분리한다는 것

---

- DW에서 처리하면 SQL의 이점을 얻을 수 있음
  - 스토리지 분리 -> 확장성, 비용 절감, 유연성 및 유지보수성
  - 최신 클라우드에서는 DW 외부에서 운영되어야 함을 규정함
  - 데이터 처리 부분에서의 장단점 비교

    |---|Data Lake|Data Warehouse|
    |---|---|---|
    |유연성|다른 사용자나 **시스템 전달 용도로 데이터**를 사용할 수 있음|DW에서만 사용 가능|
    |생산성|**테스트 프레임워크나 라이브러리를 통해 코드 개발 속도** 증대 가능. 스파크의 확장성 사용 가능|표준화된 언어하나로 사용 가능|
    |데이터 거버넌스|데이터를 일관성 있게 관리할 수 있음|데이터 정의가 레이크에서 사용하는 경우 상충될 수 있음|
    |플랫폼 간 이식성|스파크 코드로 이식성이 좋음|ANSI-SQL은 이식 가능. 나머지는 테스트와 마이그레이션 작업 필요|
    |성능|처리량이 늘어도 DW에 간섭이 없음|처리 부하가 있을 수 있음|
    |처리 속도|실시간 분석 가능|일부 실시간 분석 가능|
    |비용|컴퓨팅 사용량만 한정한다면 훨씬 저렴|계약 조건에 따라 비용이 발생|
    |재사용성|재사용 가능한 기능이나 모듈을 사용하여 반복되는 작업의 처리가 쉬워짐|프로시저나 함수로 가능|

  - 변화를 수반할 것인지 최소화할 것인지의 문제
  - 클라우드로 유연성을 극대화하기 위해서는 DW에서 처리하는 것은 최선이 아님

## 5.2 데이터 처리 스테이지

---

- ![processing](https://drek4537l1klr.cloudfront.net/zburivsky/Figures/CH05_F02_Zburivsky.png)
- 원본 -> 스테이징 
  - 공통 데이터 처리
    - 파일 포맷 표준화
    - 스키마 차이 해결
    - 중복 제거
    - 데이터 품질 검사
    - 데이터 변환
- 스테이징 -> 결과물
  - 비즈니스 로직
  - 사용자 정의 결과
  - 유효성 검사
- 위와 같은 데이터 처리를 각자 격리된 환경에서 할 수 있다는 점

## 5.3 클라우드 스토리지 구성

---

- 일관된 원칙을 구성하여 데이터 레이크 구축
- ![a](https://i0.wp.com/contenteratechspace.com/wp-content/uploads/2019/05/data-lake-architecture.jpg?fit=1024%2C388&ssl=1)
- 데이터 구성 단계
  - 1. 랜딩영역
    - 처리되기 전 일시적인 영역
    - 원시 데이터가 필요하면 별도 아카이빙
  - 2. 스테이징 영역
    - 공통 변환 과정
      - avro format 변경
      - 스키마 준수
      - 데이터 품질 검사
  - 3. 아카이브 영역
    - 재처리 할 경우 필요
    - 신규 파이프라인 테스트할 경우 필요
  - 4. 프로덕션 영역
    - 비즈니스 로직 적용
      - parquet포맷으로 변경
  - 5. pass-through 적용
    - 비즈니스로직 적용없이 DW에 바로 적용
  - 6. 클라우드 DW와 프로덕션 영역
    - 리포트나 분석목적의 데이터 세트 생성
  - 6. 실패 영역
    - 실패 원인 파악을 위함
    - 재처리할려면 실패 스토리지에서 랜딩으로 재삽입함으로 해결

### 클라우드 스토리지 컨테이너와 폴더

- 버킷이라 불리며 아래와 같은 설정 가능
  - 엑세스 및 보안
  - 컨테이너 스토리지 티어
  - 각 단계 별 권한과 티어
- 폴더 명명 규칙
  - namespace
  - 파이프라인명
  - 데이터 소스명
  - 배치ID - UUID, ULID
  - 랜딩 컨테이너
    - /landing/etl/sales_oracle_ingest/customers/01DFDDFDFKJLJALSJS
    - /landing/etl/marketing_ftp_ingest/customers/01DFDDFDFKJLJALSJS
  - 스테이징 컨테이너
    - 수집 시간 별로 파티셔닝 필요. 파티셔닝 조회 가능
    - 특정 변환 작업이 실행된 시간
    - /staging/etl/sales_oracle_ingest/customers/year=2018/month=91/day=03/01dsadasdhjkasd -> 인코딩값
    - /staging/etl/marketing_ftp_ingest/customers/year=2018/month=91/day=03/01dsadasdhjkasd -> 인코딩값
- 스트리밍 데이터 구성
  - 클릭스트림의 데이터를 15분마다 저속 스토리지로 flush하기로 함
  - /landing/etl/clickstream_ingest/clicks/01DFDDFDFKJLJALSJS
  - 뒤에 과정은 배치와 동일

## 5.4 공통 데이터 처리 단계

---

### 파일 포맷 변환

- 원본 포맷을 유지하고 저장 + 단일 통합 포맷으로 저장
- binary file format
  - avro(row-based) parquet(column-based)
  - 텍스트 기반 파일보다 좋은 점
    - 인코딩을 통한 디스크 공간 절약으로 인한 비용 및 속도
    - 스키마 사용 강제하기
    - col based가 좋은 점
      - row-based일 경우 블록단위로 읽을 때 row단위로 읽게 됨
      - 일반적인 분석 워크로드의 경우 컬럼 기반으로 질의 함
      - 압축률이 좋음. 한 블록에 같은 데이터 타입이 저장되기 때문
        - ![row vs column](https://help.sap.com/doc/6a504812672d48ba865f4f4b268a881e/Cloud/en-US/loio7156b2d7b35b4b2abda3c00057ddc210_LowRes.png)
  - avro
    - 스키마를 각 파일 내부에 포함하여 컬럼 정의와 타입을 신속하게 가져옴
    - 스키마가 변경되더라도 최신의 avro format만 사용
    - adhoc 탐색에 적합
  - parquet
    - 분석 쿼리에 적합
    - 압축률 좋음
    - DW에 원활하게 적재 가능
- 스파크 사용하여 파일 포맷 변환
  - datetime같은걸로 명명규칙을 세세하게 하면 됨
  - 명명규칙에 대한 정보는 파라미터로 받아서 처리 해야 함
  - 스키마 추론하는 스파크 기능 사용
    - 복잡한 스키마는 활용 못할 수도
  
  ```python
  # SparkSession 생성
  import datetime
  from pyspark.sql import SparkSession
  spark = SparkSession.builder ... # we omit Spark session creation for brevity
  
  namespace = “ETL”
  pipeline_name = “click_stream_ingest”
  source_name = “clicks”
  batch_id = “01DH3XE2MHJBG6ZF4QKK6RF2Q9”
  current_date = datetime.datetime.now()
  in_path = f“gs://landing/{namespace}/{pipeline_name}/{source_name}/{batch_id}/*”
  out_path = f”gs://staging/{namespace}/{pipeline_name}/{source_name}/year=
  ➥ {current_date.year}/month={current_date.month}/day={current_date.day}/
  ➥ {batch_id}”
  
  clicks_df = spark.read.json(in_path)
  clicks_df = spark.write.format(“avro”).save(out_path)
  ```

- 데이터 중복 제거
  - 1. 논리적으로 같은 데이터인지 확인 하기 master data management라는 기술이 계속 발전 됨
  - 2. 결제 ID같이 특정 속성이 고유값을 갖게 할려면 어떻게 해야할까
  - PK가 있어도 여전히 문제
    - 1. 신뢰할 수 없는 데이터 소스와 반복 수집
    - 2. DW의 고유성 강제화 부족
  - DW든 스티리밍이든 파일이든 장애가 발생할 경우 중복이 생길 가능성이 농후함
  - Spark에서 중복 제거
    - 배치 단위 내에서의 중복은 제거하기 간단함 ex) drop duplicates
    - 글로벌 단위에서는 데이터를 조인하여서 중복을 제거해야 함
      - 데이터가 커짐에 따라 전역적으로 제거하는 것은 리소스에 부담이 됨.
      - 연, 월, 주 단위로 중복 제거 범위를 결정할 것

    ```python
    from pyspark.sql import SparkSession
  
    spark = SparkSession.builder ... # we omit Spark session creation for brevity
    
    namespace = “ETL”
    pipeline_name = “users_csv_ingest”
    source_name = “users”
    batch_id = “01DH3XE2MHJBG6ZF4QKK6RF2Q9”
    in_path = f“gs://landing/{namespace}/{pipeline_name}/{source_name}/{batch_id}/*”
    
    staging_path = f”gs://staging/{namespace}/{pipeline_name}/{source_name}/*”
    
    incoming_users_df = spark.read.format(“csv”).load(in_path)
    staging_users_df = spark.read.format(“avro”).load(staging_path)
    
    
    incoming_users_df.createOrReplaceTempView(“incomgin_users”)
    staging.users_df.createOrReplaceTempView(“staging_users”)
    
    users_deduplicate_df = \
    spark.sql(“SELECT * FROM incoming_users u1 LEFT JOIN staging_users u2 ON 

    ➥ u1.user_id =  u2.user_id WHERE u2.user_id IS NULL”)
    ```

### 데이터 품질 검사

- 우려 사항
  - 데이터 레이크를 신뢰할 수 없음
  - 데이터 소스마다 품질 수준이 상이함
- 공톰 품질 검사 항목
  - 특정 컬럼에 대한 값의 길이는 정의된 범위 내에 있어야 함
  - 숫자 값 범위
  - 값의 필수 여부
  - 값의 특정 패턴 ex) email
  
  ```python
  users_df = spark.read.format(“csv”).load(in_path)
  bad_user_rows = users_df.filter(“length(email) > 100 OR username IS NULL”)
  users_df = users_df.subtract(bad_user_rows)
  ```

- 데이터 품질 고려 사항
  - 중요도 확인
    - username같은 거는 없어도 처리에는 문제가 없지만 중요한 정보일 수 있음
    - salary가 음수면은 처리에 문제가 생김
  - 데이터 품질 이슈에 대한 경보 알림
    - 데이터 소비자에게 경보를 보내야 함
  - 잘못된 행 제거 또는 배치 실패
    - 품질 검사가 실패했을 경우는 실패 디렉토리로 옮기기


## 5.5 설정 가능한 파이프라인

- 소스, 스토리지 위치, 스키마 정보, 중복제거 컬럼들을 파라미터화 해야함. 이 정보는 메타데이터 저장소에 저장되어야 함
- **워크플로에서 스토리지의 랜딩 영역을 모니터링 하고 명명규칙을 통해 파라미터들을 추출**할 수 있도록 해야함
- ![파라미터화 과정](https://drek4537l1klr.cloudfront.net/zburivsky/Figures/CH05_F11_Zburivsky.png)


## 요약

---

- 처리 계층에서는 비즈니스 로직, 데이터 유효성 검사, 데이터 변환이 수행되는 핵심계층
- 레이크에서 처리하는 것이 향후 아키텍처로 좋음
- 스테이지를 거쳐가며 데이터가 구조화되며 유용성이 증가
- 공통 처리단계 - 중복 제거, 표준 품질 검사 작업 등
- 설계 원칙이 중요 - Landing영역에서의 명명 규칙 - 향후의 파라미터가 됨
- 바이너리 포맷을 사용하기
- DW에서 고유성 강제가 불가능 하니 중복 제거도 필수
- 데이터 품질은 전적으로 소스 영역을 따르고 처리 부분에서 품질 검사 기능을 구현하는 것이 좋음
