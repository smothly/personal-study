# CH9 지식을 행동으로 옮기기

---

## 25강 데이터 활용의 현장

--- 

- 빅데이터와 관련된 고민
  - 데이터를 어떻게 활용할 지?
  - 데이터를 다룰 수 있는 인재 확보가 됐는지?
  - 어떤 데이터를 어떻게 수집할 지?
  - 데이터를 제대로 활용하고 있는지?

### 25-1 데이터 활용 방법 생각하기

- 데이터를 활용해서 무슨 가능성이 있는지 모르겠다!
  - 문제 발견 : 추이와 구성 파악하기 / 경향과 특징 파악하기
    - 리포트 기반으로 문제를 찾고, 개선할 부분을 탐색
    - 수치화 및 시각화를 통해 개선 부분 찾고, 목표를 시각화하기
    - 최종적으로 매출 또는 사용자 수의 증가와 연결하기
  - 문제 해결 : 개선 제안하기 / 자동화, 정밀도 높이기
    - 인력의 한계를 넘어 자동화와 정밀도 향상
    - 매번 추천 쿼리를 짜고 다양한 사용자에 대응하는 것은 불가능
    - 단어 자동 등록이나 웹 출력 변경등을 자동화해서 빅데이터의 장점을 살리기
  - 가치 창조 : 추천 / 타킷팅
    - 태그 입력으로 자동으로 최적의 광고 출력, 추천 결과 API 제공등 시스템 제공으로 빠른 빅데이터 분석결과 제공
  - 빅데이터 : 데이터 축적 / 가공 / 집계 / 시각화

### 25-2 데이터와 관련한 등장 인물 이해하기

- 빅데이터 팀의 구성
  - 네트워크, 인프라 관리
    - 네트워크 구축 관리
    - 미들웨어 도입, 설정, 관리
  - 데이터 설계와 관리
    - 테이블, 로그 설계
    - 데이터 가공, 집약, 관리
  - 엔지니어
    - 로그 전송 라이브러리, API 만들기
    - 데이터 API(검색, 추천) 만들기
    - 도구 만들기
  - 분석 담당자
    - AdHoc등의 리포트 만들기
    - BI 도구에 출력할 쿼리 만들기
    - 알고리즘 검토
- 데이터와 환경 자체가 갖추어지지 않을 경우
  - 누구에게 무엇을 요구하고 데이터를 얻을 수 있을지 모를때, 협업을 누구와ㅣ하지 모를 때
  - 환경 자체를 위의 팀의 구성처럼 정비해보기
- 목적이 있지만, 해당 역할을 담당할 사람이 없을 경우
  - 인원에 충실하기보다는 목적에 충실하기
  - redshift, bigquery등으로 네트워크 엔지니어가 필요없을 수도 있고, bi도 라이브러리들이 많아짐

### 25-3 로그 형식 생각해보기

- 로그 형식을 만들 때 고려할 점
- 분석 내용 증가
  - `CUBE`함수를 사용하여 쿼리 하나로 처리
- 분석 대상 증가
  - `ROLLUP`함수를 사용하여 개별적인 서비스 단위와 전체 단위 집계
- 로그형식 예시
  - ![로그형식예시](https://user-images.githubusercontent.com/37397737/209812353-a603d629-98b5-44b8-9e0b-c5785bf57daf.png)
  - dt
    - 날짜만 저장하고 데이터의 분산키로 지정
    - 날짜와 시간을 함께 저장하는 문자열을 매번 가공할 필요 없음
  - service_code
    - 서비스를 유일하게 식별할 수 있는 임의의 문자열
  - action과 option
    - 행동 또는 시스템 제어 문자열 ex) view, search, clip 등
    - option에는 action을 보완 설명할 수 있는 문자열 설정 ex) like -> post, movie, commetn 등
  - short_session, long_session, user_id
    - 비회원 행동 분석을 위해, 회원 비회원 구분을 위해 user_id 필요
  - page_name
    - url과 상관없이 페이지를 식별할 수 있는 문자열
  - view_type
    - 피씨싸이트인지 스마트폰사이트 출력인지 
  - via
    - 페이지 이동하게 만든 계기
  - segment
    - AB 테스트 때 패턴 판별할 문자열 저장
  - detail, search, show, user, other
    - json으로 저장해서 추가정보를 넣기

### 25-4 데이터를 활용하기 쉽게 상태 조정하기

- 불필요 및 이상 값들을 제거해서 새로운 테이블을 만들어 사용하면 편리
- 데이터를 3개 계층으로 구분해서 다루기
  - 로그 계층
    - 로그 계층의 데이터가 적재됨. 데이터 집계는 하지 않음
  - 집약 계층
    - 1차 가공한 데이터로 분석에 주로 사용하고 BI에도 사용
  - 집계 계층
    - 높은 빈도로 확인하는 지표의 경우 미리 집계해서 저장하여 시스템의 부담 줄이기
- 로그 -> 집약
  - 데이터 검증, 불필요 데이터, 이상값 제외해서 분석 대상 데이터를 한정짓기
- 집얍 -> 집약
  - 로그 계층보단 정밀도와 신뢰도가 높은 데이터
  - 높은 빈도로 사용할 테이블의 경우 따로 추출해서 만들어 놓기. 집계는 아님
- 집약 -> 집계
  - 높은 빈도로 확인하는 데이터는 집계한 결과로 저장
  
### 25-5 데이터 분석 과정

- 4가지 과정
  - 서비스 이해하기
    - 리포트를 먼저 작성하기 보다는 서비스를 이해하고 문제 부분을 찾아나가기 
  - 그래프 만들기
    - 수치를 나열해도 어떤 부분이 상승하고 어떤 부분이 하락하는지 파악이 어려워 그래프를 만들어가며 보기
  - 여러 수치 비교하기
    - 관련 지표로 나누어 비교
    - 관련한 여러 지표를 한꺼번에 놓고 비교
    - 데이터를 세분화해서 확인
    - 이전 달, 작년 데이터와 함께 놓고 비교
    - 같은 지표를 다른 서비스와 비교
  - 가설 세우고 검증하기

### 25-6 분석을 위한 한 걸음 내딛기

- KGI와 KPI를 설정하고 관련 부분 분석하기
  - Key Gole Indicator: 중요 목표 달성 지표. 조직과 서비스의 최종적인 목표를 정량화한 지표 ex) 매출 0억, 사용자 수0명
  - Key Performance Indicator: 중요 업적 평가 지표. KGI를 달성하기 위한 중간 지표 ex) 매출 = 방문자 x cvr x 구매단가
  - 위와 같은 식을 만들었을 경우, KPI에 주목해서 다음과 같은 과정을 수행
    - KPI가 어떠한 추이를 가지며, 어떻게 구성되는지 확인
    - KPI를 달성한 사람과 달성하지 못한 사람을 비교
    - KPI를 더 분해하고, 추이와 구성을 살펴본다
- 숲을 보고 나무 보기
  - 팀 전체가 문제 공유

### 25-7 상대방에 맞는 리포트 만들기
- 임원/경영층
  - 세부적인 리포트 보다는 서비스의 규모, 매출 추이, 사용자 수 등을 기ㅐ
    - Z차트 업적 추이 확인
    - 방문 빈도를 기반으로 사용자 속성을 정의하고 집계
    - 방문 종류를 기반으로 성장지수 집계
- 사업부
  - KGI의 주목해야 하는 부분이나 전체적인 문제점을 찾아보기
    - ABC분석으로 잘 팔리는 상품 판별
    - 팬 차트로 상품의 매출 증가율 확인
    - RFM 분석으로 사용자를 3가지 관점으로 그룹으로 나누기
    - 사용자 행동 전체를 시각화 하기
    - 특정 아이템에 흥미가 있는 사람이 함꼐 찾아보는 아이템 검색
- 서비스 기획/개발
  - KPI 의식하여 구체적으로 무엇을 해야할 지에 관심
    - 지속과 정착에 영향을 주는 액션 집계
    - 액션 수에 따른 정착률 집계
    - 이탈률과 직귀율
    - 검색 조건들의 사용자 행동 가시화
    - 폴아웃 리포트
- 마케팅 담당
  - 광고의 효율, 효과를 얻기 위한 정보
    - 등록으로부터 매출 집계
    - 성과로 이어지는 페이지
    - 페이지 평가
- 리포트 열람자
  - plan, do, check, act중 어느단계에 있는지 확인하여 알맞은 레포트제공

### 25-8 빅데이터 시대의 데이터 분석자

- 데이터 분석자는 다양한 사람과 관련이 있고, 서로 협력해서 서비스와 조직의 가치를 극대화하는 중요한 포지션!!