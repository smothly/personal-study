Athena 파티션 프로젝션을 사용하면 파티션 값의 범위와 각 파티션 열의 프로젝션 유형을 지정합니다. 그러면 Athena에서 인 메모리 연산을 수행하여 파티션 값을 계산할 수 있습니다. 이 솔루션은 세부적으로 분할된 테이블의 쿼리 런타임을 큰 폭으로 줄이고 파티션 관리를 자동화


DataSync는 온프레미스와 Amazon S3와 같은 여러 AWS 서비스 간에 대량의 데이터를 온라인으로 이동합니다. DataSync는 스크립팅 복사 작업과 같은 많은 태스크를 자동으로 처리합니다. DataSync는 지속적인 데이터 배포 및 데이터 파이프라인은 물론 여러 대상 버킷 간에 데이터를 분할하는 데 적합


S3 게이트웨이 엔드포인트를 사용하면 추가 비용 없이 VPC에서 Amazon S3에 액세스할 수 있습니다. S3 게이트웨이 엔드포인트를 만들면 라우팅 테이블이 S3 IP 접두사 목록으로 업데이트되어 새로 만든 엔드포인트에서 VPC 트래픽을 S3로 직접 라우팅합니다.
S3 버킷 정책은 AWS Glue가 VPC를 사용하여 S3 버킷에 액세스하는 것을 허용하지 않습니다. 거부 정책이 이미 존재하고 요청을 허용하기 위해 수정이 필요한 경우 S3 버킷 정책이 필요할 수 있습니다.


Redshift는 단일 데이터 원본에서 테이블을 업데이트하기 위한 단일 병합 또는 upsert 명령을 지원하지 않습니다. 하지만 스테이징 테이블을 생성하여 병합 작업을 수행할 수 있습니다.


DynamoDB에 대한 I/O 요청의 균형이 맞지 않으면 ‘핫’ 파티션이 발생하여 I/O 용량이 제한되고 비효율적으로 사용될 수 있습니다. 요청 내에서 일관성을 높이기 위해 파티션 키를 변경하는 솔루션으로 ‘핫’ 파티션을 방지할 수 있습니다. SENSOR_ID를 사용하는 솔루션은 파티션 키로 BUILDING_ID를 사용하는 솔루션보다 요청 수가 적습니다

Athena는 삭제 작업을 수행하기 위한 Hudi 테이블 쓰기는 지원하지 않습니다.

Amazon Redshift의 DDM(동적 데이터 마스킹)을 사용하면 데이터 웨어하우스에서 민감한 데이터를 보호할 수 있습니다. 데이터베이스에서 데이터를 변환하지 않고 쿼리 시 사용자에게 민감한 데이터가 표시되는 방식을 조작할 수 있습니다. 특정 사용자나 역할에 사용자 정의 난독화 규칙을 적용하는 마스킹 정책을 통해 데이터에 대한 액세스를 제어

OpenSearch Service는 데이터를 인덱싱, 검색 및 시각화하는 데 사용되는 관리형 서비스입니다. UltraWarm은 핫 스토리지에 비해 비용을 절감할 수 있는 스토리지 티어입니다. m은 최대 3PB까지 확장할 수 있습니다. UltraWarm은 캐싱을 사용하여 데이터를 검색할 때 빠른 대화형 환경을 보장

태스크 상태의 Resource 필드에는 Step Function SDK 통합의 Amazon Resource Name(ARN)과 해당 API 이름이 포함됩니다. Resource 필드는 필수 사항


보안 그룹을 삭제하기 전에 VPC의 인스턴스 또는 기타 리소스와 연결된 상태에서 보안 그룹을 분리해야 삭제할 수 있습니다.
VPC의 보안 그룹에 있는 규칙에서 보안 그룹을 참조하는 경우, 보안 그룹을 삭제하기 전에 먼저 규칙을 제거해야 합니다.

Macie는 Amazon S3에 저장된 민감한 데이터만 검색할 수 있습니다. Macie는 다른 AWS 서비스에서 민감한 데이터를 검색할 수 없습니다. 또한 Macie는 데이터 마스킹 기술을 제공하지 않습니다.

DataBrew로 데이터 마스킹 기술을 사용하여 PII 데이터에 대한 데이터 변환을 수행
AWS Glue Studio를 사용하여 다양한 데이터 집합과 파일 형식(예: CSV, Apache Parquet, JSON)에서 PII를 감지

푸시다운은 소스에 더 가까운 데이터를 검색하는 데 사용할 수 있는 최적화 기법입니다. 따라서 이 솔루션은 시간과 처리되는 데이터의 양을 줄일 수 있습니다. 푸시다운 조건자를 정의하는 솔루션은 AWS Glue 작업이 읽는 파일 수를 줄입니다. 이 솔루션을 사용하면 데이터를 덜 읽어 메모리 부족 오류를 해결하는 데 도움이 됩니다.

Glue catalogPartitionPredicate 옵션을 사용한 서버 측 필터링은 AWS Glue 데이터 카탈로그의 파티션 인덱스를 활용합니다. 이 옵션은 세부적으로 파티셔닝된 테이블을 스캔할 때 처리 및 응답 시간을 향상

boundedSize 파라미터를 사용한 워크로드 분할을 통해 각 AWS Glue ETL(추출, 변환 및 로드) 작업에서 실행되는 데이터의 양을 제어할 수 있습니다. 이 솔루션은 작업에서 너무 많은 데이터를 처리하여 메모리 부족 오류가 발생하는 경우를 방지합

거버넌스 모드를 사용하면 잠금을 제거할 수 있는 특정 AWS Identity and Access Management(AWS IAM) 권한이 있는 AWS 계정을 지정할 수 있습니다. 그러나 잠금을 제거하면 데이터가 수정되거나 삭제될 위험이 있습니다. 버킷을 생성할 때 객체 잠금 및 S3 버전 관리를 사용

규정 준수 모드를 사용하면 어떤 사용자도 잠금을 제거할 수 없습니다. 루트 계정조차도 잠금을 제거할 수 없습니다. 따라서 이 솔루션은 데이터를 수정하거나 삭제하는 것을 방지