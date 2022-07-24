# 클라우드 비용, 어떻게 줄일 수 있을까? (AWS builders 온라인 시리즈)

## 1. 데이터 수집

- 빌링 대시보드
- trusted advisor
- 비용탐색기

---

## 2. 비용 최적화

- right sizing
  - cloudwatch
  - cost exploer 를 통한 검색
- scheduling
  - 사용시간 분석 후 스케줄링
- pricing
  - 온디맨드(기본옵션)
  - 예약 인스턴스(RI)
    - 표준(1년 선결제)
    - 전환(유연성 있음)
  - SPOT 인스턴스
- storage
  - storage 수명주기 규칙

---

## 3. 팀플레이

- 기술팀과 사업팀의 비용 최적화를 위한 노력
- 리디북스 마이크로서비스처럼 비용도 각각 최적화를 위해 노력

---
---

# AWS, 최적의 비용 효율화 방법은?  (AWS builders 온라인 시리즈)

## 1. AWS의 비용 체계

- 온디맨드 식으로 사용으로의 전환
- 오버 프로비저닝에 대한 제거
- 비용 절감 / 직원 생산성 / 운영 탄력성 / 비즈니스 민첩성

## 2. AWS 비용효율적 사용

- right sizing
  - 각 용도에 맞는 인스턴스 타입 설정(t3, m4 등ㄴ)
  - cost explorer에서 리소스 최적화 권장 사항 보기
  - trusted advisor
- scheduling
- saving plans + 예약인스턴스
- 비용 고려한 아키텍처

## 3. AWS 비용 관리 방법

- 비용 예측
- 비용 측정 및 모니터링
- 비용 효율성을 위한 측정
