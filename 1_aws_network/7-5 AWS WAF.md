# 7-5 AWS WAF

## AWS WAF 개요

#### WAF란?

- Web Application Firewall
- SQL Injection, XSS, CSRF 공격등과 같이 웹서비스에 대한 공격을 탐지하고 차단하는 기능 수행
- 페이로드 분석 및 패턴 기반의 필터링

#### AWS WAF 소개

- 보안 규칙과 사용자 정의의 특정 트래픽 패턴을 필터링하는 규칙을 생성하여 제어
- ALB, cloudfront, API Gateway에 배포 가능

#### 주요 기능

- 웹 트래픽 필터링
  - 필터링 규칙 생성 및 적용
- 자동화 및 유지관리
  - 규칙 통합 및 유지 관리
  - Cloudformation을 통한 자동 배포 및 프로비저닝 가능
- 가시성 보장
  - Cloudwatch와 통합되어 다양한 지표를 제공
- Firewall Manager와 통합
  - WAF를 중앙에서 구성 및 관리 가능

---

## AWS WAF 구성

#### 구성

- Web ACL, Rule, Statement로 구성
  - Web ACL
    - 최상위 컴포넌트
    - Rule을 최대 1000개 까지 생성
  - Rule
    - Web ACL의 하위 컴포넌트
    - 기준과 작업을 포함
    - 최대 5개의 statements 설정 가능
  - Statements
    - 웹 필터링의 상세조건을 정의
      - Inspect: Inspection 대상을 정의
      - Match Condition: Inspection 대샹에 대한 분류 방법
      - Transformation: Match의 추가적인 옵션 부여
      - Action: 필터링 동작방식을 정의 허용/거부/카운트

#### 장점

- 민첩한 보안
- 동적 확장
- 효율적인 비용
- 손쉬운 배포 및 유지관리
- API를 통한 자동화
