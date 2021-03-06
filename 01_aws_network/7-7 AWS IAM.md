# 7-7 AWS IAM

## AWS IAM 개요

- AWS 서비스와 리소스에 대한 사용자에 대한 접근 제어와 관리
- AWS 사용자 및 그룹을 만들고 관리하거나 권한을 통해 AWS 리소스에 대한 엑세스를 허용하거나 거부할 수 있음

---

## IAM 세부 요소

#### IAM 사용자와 그룹

- IAM 사용자
  - AWS 서비스와 상호 작용하는 사람 또는 서비스
  - IAM 사용자는 하나의 AWS 계정에만 연동됨
- IAM 그룹
  - IAM 사용자들의 모임
  - 한번에 여러 사용자에게 같은 정책을 연결하는 방법

#### IAM 정책(Policy)

- AWS의 리소스에 대해 어떤 대상이 어떤 ACtion을 수행할 수 있는지 정의
- 자격 기반의 정책: 사용자, 그룹, 역할이 수행할 수 있는 Action을 지정
- 리소스 기반의 정책: AWS 리소스가 수행할 수 있는 Action을 지정
- IAM 정책 구성 요소
  - JSON 으로 구성
  - Effect: 접근을 허용 또는 거부
  - Principal: 누가 어떤 것을 수행할 수 있는 여부를 정의
  - Action: 무엇을 할 수 있는지 정의
  - Resource: 정책이 적용되는 리소스가 무엇인지 정의
  - Condition: 정책이 사용되는 조건이 무엇인지 정의
  - ![예시](https://blog.cloudthat.com/assets/iam_policy_example.jpg)

#### IAM 역할(Role)

- 리소스에 대한 액세스를 부여하는 권한 세트
- 정의된 권한을 사용자나 서비스로 위임함
- 신뢰 정책: AWS 리소스를 소유한 계정과 리소스에 접근하는 사용자 간에 신뢰 설정
- 권한 정책: 사용자에게 리소스에서 원하는 작업을 수행하는 데 필요한 권한을 부여

---

## IAM 주요 기능

- 강화된 보안
- 세부적인 제어 관리
- 임시 자격 증명
- 엑세스 분석
- 외부 자격 증명 활용
