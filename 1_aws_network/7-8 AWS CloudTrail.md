# 7-8 AWS CloudTrail

## AWS CloudTrail 개요
- AWS 계정의 관리, 규정 준수, 운영 및 위험 감사를 지원하는 서비스
- AWS에서 발생한 모든 행위에 대한 이벤트를 한 곳으로 모아 분석, 리소스 변경 추적, 문제 해결 가능

#### CloudTrail 이벤트
- 관리 이벤트: AWS 계정의 리소스에 대해 수행되는 관리 작업에 대한 정보를 제공
- 데이터 이벤트: 리소스 상에서 수행되는 리소스 작업에 대한 정보를 제공
- Insights 이벤트: AWS 계정에서 비정상적은 활동을 캡처합니다.
- 90일까지 확인 가능

#### 추적
- s3와 cloudwatch logs에 cloudtrail 이벤트를 제공
- 추적을 사용하여 이벤트를 필터링 및 암호화 알림 설정도 가능

---

## AWS CloudTrail 모범 사례 조치 사항
- 모든 리전 활성화
- 무결성 확인을 위해 Validation 활성화
- S3에 저장할 경우 퍼블릭 접근 차단
- S3에 저장할 경우 엑세스 로그 설정
- Cloudwatch에서 같이 모니터링
