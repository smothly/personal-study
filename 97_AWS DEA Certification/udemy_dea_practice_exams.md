### AWS Glue

- 데이터 셔플링이 자주 발생하고 DPU의 부하가 높을 경우, cloudwatch, s3, etl스크립트 리팩토링을 통해 부하를 줄일 방법을 찾아야 함
- 작업 북마크는 증분 로드에 유용
- DataBrew
  - AWS Glue DataBrew의 데이터 프로파일링 기능은 데이터 품질 평가
  - 데이터를 정리하고 정규화할 수 있는 시각적 데이터 준비 도구
  - 이상 필터링, 데이터 표준형식 변환, 잘못된 값 수정
    - Null 제거, 누락된 값 교체, 스키마 불일치 수정, 함수 기반 열 생성 등이 포함
  - 다양한 소스의 데이터 세트에서 불일치와 이상 현상을 식별하는 데 중요
- AWS CloudWatch에서 AWS Glue 작업 지표를 모니터링하면 변환 문제를 해결하는 데 중요한 비효율적인 처리 및 데이터 유출 문제를 식별
- 작업 로그에는 변환 오류에 대한 귀중한 정보가 포함되는 경우가 많으며 작업 실패로 이어지는 특정 스크립트 또는 데이터 형식 문제를 강조
- Glue ETL 피벗변환은 행을 열로 변환
- Glue 'FLEX' 실행 클래스는 유연한 타이밍 요구 사항이 있는 작업을 위해 특별히 설계되었습니다. 리소스를 즉시 할당하지 않음으로써 리소스 사용을 최적
  - 야간 ETL 작업 또는 워크로드 처리를 위해 주말에 실행되는 작업
  - 일회성 대량 데이터 수집 작업
  - 테스트 환경 또는 사전 프로덕션 워크로드에서 실행되는 작업
  - 가변적인 시작 및 종료 시간을 허용하는 시간에 민감하지 않은 워크로드
- 사용자 지정 분류자를 갖춘 AWS Glue 크롤러는 특히 파일이 일관되지 않은 경우 다양한 데이터 스키마를 처리하는 효과적인 방법입니다. 이 솔루션은 다양한 스키마를 이해하고 정확한 쿼리를 위해 데이터 카탈로그를 업데이트하는 동적 방법을 제공합니다.
- AWS Glue DynamicFrames의 주요 이점 중 하나는 스키마 변형과 오류를 보다 원활하게 처리할 수 있다는 것입니다. DynamicFrames에는 고정된 스키마가 필요하지 않으며 데이터 이상 및 변형에 더 잘 견딥니다. 이는 반구조화된 데이터와 관련된 ETL 프로세스 또는 시간이 지남에 따라 스키마가 변경될 수 있는 데이터 소스를 처리할 때 특히 유용합니다.
- Glue Elastic Views는 여러 데이터 저장소에 걸쳐 구체화된 뷰 구축을 지원합니다.

### Amazon DynamoDB
- DynamoDB는 특히 전역 테이블을 사용하여 사용자 장바구니 데이터와 같은 상태 저장 트랜잭션을 처리하고 짧은 지연 시간, 고가용성 및 지역 간 복제를 제공하는 탁월한 선택
- 주로 주식 기호와 날짜를 기반으로 검색
  -  날짜를 파티션 키로, 주식 기호를 정렬 키로 사용하는 GSI를 도입하는 것이 가장 좋은 접근 방식
  -  역 인덱스는 특정 시나리오, 특히 기본 키의 일부가 아닌 특성에 대해 쿼리해야 하는 경우
- DAX는 읽기 대기 시간을 줄이는 데 유용하지만 쓰기 용량 제한 초과라는 핵심 문제를 해결하지 못합니다. 마이크로초 응답 시간을 제공하여 데이터베이스의 로드를 줄여 대규모 DynamoDB 테이블에 대한 빠른 읽기 성능을 제공하는 캐싱 서비스


### AWS Kinesis
- Firehose는 즉각적인 처리가 중요하지 않는 시나리오에 더 적합 - 실시간 처리에는 부적합
- CloudWatch에서 PutRecord.Success, IncomingRecords 및 GetRecords와 같은 Kinesis Data Streams 지표를 모니터링하는 것이 실용적인 첫 번째 단계
- IteratorAgeMilliseconds는 스트림이 처리할 수 있는 것보다 더 높은 속도로 데이터를 수신하는지 또는 소비자 측에서 처리 지연이 있는지 이해하는 데 도움
- Amazon Kinesis Data Streams의 향상된 팬아웃 기능은 여러 소비자에게 높은 처리량과 낮은 지연 시간의 데이터 전송이 필요한 시나리오를 지원하도록 특별히 설계.  HTTP/2를 사용하여 Kinesis Data Streams의 성능을 크게 향상시켜 실시간 분석 및 대용량 IoT 데이터 처리에 적합하게 만듭니다. 이 설정은 AWS Lambda 및 Kinesis Data Analytics와 같은 여러 서비스에 대한 동시 데이터 배포 https://aws.amazon.com/ko/blogs/aws/kds-enhanced-fanout/

## AWS Lambda
- 예약된 동시성 한도를 높게 설정하면 Lambda 함수가 최대 로드를 처리할 수 있지만 사용량이 적은 시간에는 자동으로 축소되지 않으므로 잠재적인 비용 비효율성이 발생
- 배치 크기 조정은 데이터 버스트 처리 문제를 해결하는 직접적이고 최소한의 조정입니다. 호출당 처리되는 레코드 수를 최적화함으로써 팀은 피크 시간 동안 데이터 흐름을 더 잘 관리

### AWS RDS
- PSQL
  - Amazon RDS는 PostgreSQL을 지원하며 PostGIS 확장은 PostgreSQL 데이터베이스에 공간 및 지리 객체를 추가하여 복잡한 공간 쿼리를 보다 효율적으로 실행
  - GraphQL 서비스를 갖춘 AWS AppSync는 사용자별 데이터 검색에 최적화될 수 있는 강력하고 유연한 쿼리 기능을 제공하나 캐싱이나 실시간 제공은 어려움
- 공통
  - Amazon RDS 이벤트 구독은 직접적인 데이터 수정 또는 삭제 작업을 트리거하는 대신 RDS 인스턴스를 모니터링하기 위한 것
  - 암호화되지 않은 기존 RDS 데이터베이스를 암호화하려면 암호화되지 않은 데이터베이스의 스냅샷을 생성하고 암호화된 스냅샷을 복사한 다음 이 암호화된 스냅샷에서 RDS 인스턴스를 복원
- rds psql 장시간 트랜잭션 - 유휴 트랜잭션에 대한 시간 제한(idle_in_transaction_session_timeout)을 설정하면 장기 실행 트랜잭션이 성능 문제의 일반적인 원인인 불필요하게 잠금을 유지하는 것을 방지



### AWS S3
- Storage Gateway
  - 온프레미스를 클라우드 기반 스토리지와 연결하여, 온프레미스와 IT 환경과 AWS의 스토리지를 사용하는 서비스
파일 기반, 볼륨 기반 및 테이프 기반 솔루션 제공
- **PrivateLink**
  - VPC와 S3 버킷 간의 프라이빗 연결을 활성화하여 데이터가 퍼블릭 인터넷에 노출되지 않도록 보장
- 보안
  - SSE-S3는 고객 관리형 키에 대한 요구 사항을 준수하지 않는 Amazon S3 관리형 키
  - SSE-C는 고객 관리형 키를 사용하여 데이터를 암호화하지만, AWS는 암호화 키를 관리하지 않음
  - AWS 계정 경계에서 암호화를 처리하는 가장 안전하고 규정을 준수하는 방법은 키 관리에 AWS KMS를 사용하고 고객 관리 키로 S3 기본 암호화를 활성화. **계정 간에 CMK를 공유**하는 것은 여러 계정이 있는 AWS 환경에서 일반적인 관행입니다. 이는 키 관리 및 교차 계정 액세스에 대한 AWS 모범 사례와 일치
- Amazon S3 액세스 포인트는 S3의 공유 데이터 세트에 대한 대규모 데이터 액세스 관리를 단순화합니다.
- 클라이언트 측 암호화는 수동으로 관리해야 함


### AWS SageMaker
- Data Wrangler
  - SageMaker Data Wrangler는 데이터를 가져오고 준비하며 변환하고 기능화하고 분석하는 데 사용되는 엔드 투 엔드 솔루션을 제공
  - Data Wrangler는 기존 텍스트 편집기나 스프레드시트 애플리케이션과 같은 '찾기 및 바꾸기' 도구를 특별히 제공하지 않습니다.
- 계보 추적
  - Amazon SageMaker ML 계보 추적은 기계 학습 모델, 데이터 세트 및 작업에 대한 엔드 투 엔드 계보 정보를 제공하도록 특별히 설계되었습니다.
- SageMaker 모델 레지스트리는 ML 모델을 관리하고 버전 관리하는 데 중요하지만 다양한 ML 아티팩트 간의 관계 및 종속성을 이해하기 위한 시각적 탐색 기능을 제공하지 않습니다.

### 보안

- 여러 AWS 계정 간의 데이터 암호화 및 공유를 처리하는 가장 안전하고 효과적인 방법은 AWS Key Management Service(AWS KMS)를 이용하는 것입니다. 이 옵션은 파트너 회사(계정 B)의 암호화된 데이터 액세스를 허용하면서 암호화 키(고객 관리형 CMK)가 원래 계정(계정 A) 내에 유지되도록 보장하므로 정확


### 기타
- AWS Direct Connect는 온프레미스에서 AWS로의 전용 네트워크 연결을 제공. 
- DataSync는 AWS <-> 온프레미스 간 데이터 전송을 위한 서비스이나 JDBC/ODBC를 통해 직접 연결은 지원하지 않음
- SCP는 지리적 데이터 저장 및 처리 제한 사항을 준수하는 가장 효과적인 방법. AWS Backup과 교차리전 복제는 전사적 정책을 시행할 수 있는 기능은 없음
- 계층화된 샘플링은 특히 데이터세트에 별개의 하위 그룹이 있는 경우 데이터세트의 주요 특성을 포착하는 효율적인 방법입니다. 특정 특성을 기준으로 데이터 세트를 계층(또는 그룹)으로 나눈 다음 이러한 계층 내에서 무작위로 샘플링함으로써 컨설턴트는 데이터 세트의 모든 부분이 표시되도록 보장하여 통계적으로 더 중요한 결과를 얻을 수 있습니다.
- AWS Resource Access Manager(RAM)는 계정 간에 리소스를 공유하는 데 사용되지만 사용자 액세스 또는 권한을 관리하도록 설계되지 않았습니다.
- Amazon Macie는 기계 학습 및 패턴 일치를 사용하여 AWS에서 PII와 같은 민감한 데이터를 검색하고 보호하므로 이러한 맥락에서 이상적인 솔루션
- AWS Config는 S3 환경의 구성 및 변경 사항을 추적하는 데 적합
- AWS CloudTrail Lake는 CloudTrail 로그를 저장, 관리 및 분석하기 위한 최적화된 중앙 집중식 솔루션을 제공합니다. 이를 통해 최대 7년 동안 로그를 보존하고 쿼리할 수 있으며 이는 1년 동안의 데이터 분석에 대한 회사의 요구 사항에 잘 부합합니다.
- AWS Data Exchange  AWS 클라우드에서 직접 타사 데이터 세트를 쉽게 찾고, 구독하고, 사용할 수 있도록 해주는 서비스
- Amazon Managed Workflows for Apache Airflow(MWAA)는 Airflow 작업자의 자동 조정을 지원합니다.
이 기능을 사용하면 서비스는 작업 부하, 특히 대기열에 있는 작업 수와 평균 작업 실행 시간에 따라 작업자 수를 동적으로 늘리거나 줄일 수 있습니다.
이렇게 하면 확장을 위해 수동 개입 없이 워크플로를 효율적으로 실행할 수 있습니다.
- AWS Resource Access Manager는 계정 간 리소스 공유에 중점을 두고 있으며 백업 및 데이터 상주 관리용으로 특별히 설계되지 않았습니다.
- AWS Backup은 다양한 AWS 서비스 전반에 걸쳐 백업을 관리할 수 있는 중앙 집중식 솔루션을 제공합니다. 지역을 지정하고 이러한 설정을 여러 계정에 적용할 수 있는 기능을 통해 중앙 집중식 제어 및 데이터 상주 법률의 엄격한 준수 요구 사항에 완벽하게 부합합니다.

### IaC & CI/CD
- SAM은 매개변수 파일을 사용하여 다양한 구성을 처리할 수 있는 기능과 여러 계정 및 지역에 대한 배포가 용이하다는 점 때문에 효율적인 선택이 됩니다. 또한 AWS SAM은 안전한 배포를 위해 기본적으로 롤백 기능과 트래픽 이동을 지원합니다.
- AWS SAM은 AWS CloudFormation을 확장하여 애플리케이션의 서버리스 구성 요소를 정의하는 단순화된 방법을 제공합니다. AWS Lambda, Amazon API Gateway, Step Functions 및 DynamoDB 테이블과 같은 리소스를 
- AWS CodePipeline은 빌드, 테스트, 배포 등 여러 단계가 포함된 워크플로를 조정하는 데 이상적입니다. 소스 제어를 위해 GitHub와 통합할 수 있으며, 구축 및 테스트를 위해 AWS CodeBuild를 통합할 수 있습니다.
- CodePipeline은 조건 기반 배포에 대한 팀의 요구 사항과 일치하는 정의된 기준에 따라 작업의 조건부 실행을 지원
- AWS CodeCommit으로 전환하면 AWS 내 통합이 간소화될 수 있지만 특히 GitHub를 사용하여 복잡한 조건 기반 배포 전략을 처리할 수 있는 CI/CD 솔루션에 대한 팀의 요구 사항은 해결되지 않습니다. Code commit == github
- AWS CodeBuild는 소스 코드를 컴파일하고, 테스트를 실행하고, 배포 패키지를 생성하는 완전 관리형 빌드 서비스
- AWS CodePipeline은 CodeCommit에서 최신 코드를 가져오고, CodeBuild로 이를 구축한 다음 Amazon EMR에 배포하는 것을 포함하여 전체 CI/CD 파이프라인을 오케스트레이션합니다.
- CodeDeploy는 주로 Amazon EMR 클러스터가 아닌 EC2 및 온프레미스 시스템과 같은 인스턴스에 대한 애플리케이션 배포를 관리
- CodeDeploy는 본질적으로 GitHub 이벤트에 직접 연결된 고급 조건 기반 배포 전략(예: 특정 분기, 태그 배포 또는 커밋 메시지 기반 배포)을 제공하지 않습니다.
- CloudFormation의 매핑은 동일한 템플릿 내에서 환경별 구성을 정의하는 데 이상적인 키-값 쌍입니다. 이를 통해 단일 템플릿이 입력 매개변수에 따라 동작을 조정할 수 있으므로 별도의 템플릿이 필요 없이 환경(개발, 스테이징, 프로덕션)에 따라 EMR 클러스터와 같은 리소스 구성을 동적으로 설정할 수 있습니다.
- StackSets는 계정과 지역 전체에 걸쳐 여러 스택을 관리하는 강력한 도구이지만, 동일한 지역이나 계정 내에서 다양한 환경 구성을 관리하기보다는 이러한 차원에 걸쳐 일관되게 배포하는 데 주로 사용됩니다.


### Redshift
- ROWS BETWEEN과 결합된 AVG() 함수를 사용하면 Redshift 내에서 직접 이동 평균을 계산
- 'sales_region' 과 같이 원래 배포 키가 데이터 편향으로 이어지는 경우는 균등 분산
- Join성능이 낮아지면 키분산
- Amazon Redshift Query Performance Insights는 쿼리 성능에 대한 포괄적인 보기를 제공하므로 데이터 엔지니어는 오래 실행되거나 문제가 있는 쿼리를 빠르게 식별할 수 있습니다. 이는 개별 쿼리와 전체 작업 부하의 성능 특성을 이해하는 데 도움이 됩니다.
- 반면 Amazon Redshift Advisor는 배포 스타일 변경, 정렬 키 추가 등 Redshift 클러스터의 성능을 최적화하기 위한 자동화된 권장 사항을 제공
- Amazon Redshift의 행 수준 보안을 사용하면 데이터베이스 관리자는 역할이나 팀과 같은 사용자 속성을 기반으로 데이터베이스 테이블의 행에 대한 액세스를 제어하는 ​​보안 정책을 설정할 수 있습니다. 따라서 데이터 공유 요구 사항이 복잡하고 사용자 ID 또는 역할과 밀접하게 연결되어 있는 상황에 이상적인 선택입니다.
- VACUUM
  - VACUUM SORT ONLY -  테이블 내 데이터의 물리적 레이아웃을 최적화하여 특히 잦은 업데이트 및 삽입으로 인해 데이터가 정렬되지 않은 경우 쿼리 성능을 크게 향상
  - VACUUM DELETE ONLY - 테이블에서 삭제된 행을 정리하여 테이블의 물리적 저장소를 최적화하고 쿼리 성능을 향상
  - VACUUM FULL - 테이블의 물리적 저장소를 최적화하고 쿼리 성능을 향상시키는 데 사용되는 가장 포괄적인 VACUUM 명령
- REDSHIFT DATA API 데이터 API는 쿼리 결과의 페이지 매김 및 임시 저장을 처리하여 대규모 결과 세트 관리 프로세스를 단순화하므로 분석 팀이 복잡한 결과 세트 관리를 처리하지 않고도 데이터를 더 쉽게 검색하고 처리
- 모니터링 쿼리
  - STL_QUERY_METRICS: 쿼리의 리소스 소비를 이해하는 데 유용하지만 잠재적인 성능 문제를 구체적으로 강조하거나 최적화 권장 사항을 제공하지는 않습니다.
  - STL_ALERT_EVENT_LOG: 비효율적인 조인 또는 과도한 데이터 검색과 같은 잠재적인 성능 문제를 감지한 쿼리를 식별하도록 맞춤화. 권장사항도 포함되어 있음
- Redshift 테이블 내에 행 수준 보안을 구현하여 사용자 역할에 따라 데이터 행에 대한 액세스를 제어하여 분석가가 분석과 관련된 데이터만 볼 수 있도록 합니다.
- WLM은 메모리 및 동시성을 관리하는 데 유용하지만 특정 쿼리의 실행 속도를 직접적으로 높이는 것보다는 리소스 할당에 더 중점

### LakeFormation
- 민감한 데이터에 태그를 지정하고 분석가의 역할에 권한 태그를 할당함으로써 컨설턴트는 엄격한 권한 관리
- Lake Formation은 행 수준 보안을 직접 지원하지 않습니다. 이는 스토리지 계층이나 데이터 소비 서비스(예: 뷰를 사용하는 Redshift) 내에서 관리
- 자동 데이터 마스킹 기능은 이 시나리오에서 중요한 데이터를 저장용으로 변환하는 것보다 액세스 제어에 더 적합
- 데이터 공유를 위해 특별히 설계된 기능도 포함합니다. 이를 통해 세부적인 액세스 제어를 생성하여 외부 파트너가 액세스할 수 있는 정확한 데이터 위치를 정의
- 데이터 액세스 정책을 정의하고 시행할 수 있는 중앙 집중식 장소를 제공하여 보안을 단순화할 뿐만 아니라 누가 어떤 데이터에 액세스할 수 있는지 관리하기 위한 세분화된 액세스 제어를 지원
- Lake Formation에는 특정 "셀 수준" 보안 기능이 없습니다. 주로 이 시나리오에 더 적합한 행 및 열 수준 보안에 중점을 둡니다.

### ElastiCache
- 캐싱 전략
  - 지연 로딩 전략
    - 읽기 작업이 많고 자주 업데이트되지 않는 데이터 시나리오에 이상적
    - 캐시 유지 관리 오버헤드를 최소화하고 가장 많이 요청된 데이터만 캐시되도록 합니다.
  - Write-Around 캐싱은 ElastiCache에서 일반적으로 사용되는 전략이 아님
  - Memcached와 함께 무효화 전략을 사용하는 것은 캐시와 데이터베이스 간의 데이터 일관성이 중요한 시나리오에서 유용



### Step Function
- 'Map' 상태는 단일 반복자를 적용하여 항목 컬렉션을 처리합니다. 여러 항목을 개별적으로 처리할 수 있지만 본질적으로 각 항목 내의 데이터 속성을 기반으로 다양한 처리 경로에 대한 조건부 라우팅을 제공하지는 않습니다.
- '통과' 상태는 입력을 출력으로 전달하거나 고정 데이터를 주입할 수 있지만 조건을 평가하거나 입력 데이터를 기반으로 결정을 내리는 기능은 없습니다


### Athena

- Amazon Athena에서 Apache Spark를 시작하려면 먼저 Spark 지원 작업 그룹
- 기본은 60분. 쿼리 결과를 재사용할 수 있는 최대 기간을 120분으로 설정함으로써 팀은 이 2시간 내에 동일한 쿼리가 다시 실행될 경우 Athena가 새 데이터 검색을 수행하지 않도록 할 수 있습니다.
- 작업 그룹 전체 설정이 적용되면 개별 사용자는 권한에 관계없이 쿼리에 대해 해당 설정을 재정의할 수 없습니다. 작업 그룹 설정은 클라이언트 측 설정보다 우선합니다. 쿼리 결과 위치, 데이터 암호화 설정, 쿼리 실행 제한 등 다양한 쿼리 실행 매개변수 포함
- alter table add partition 이 명령은 테이블에 파티션을 추가하는 데 사용되지만 각 파티션을 수동으로 지정해야 합니다.
모든 새 파티션을 자동으로 검색하고 추가하는 "MSCK REPAIR TABLE" 명령과 달리 이는 많은 수의 파티션에 대해 비실용적이고 시간이 많이 걸릴 수 있습니다.

