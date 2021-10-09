# 6-8 Route 53 DNS Resolver

## private hosted zones
- VPC내에 생성한 도메인들에 대한 DNS 쿼리 응답 정보가 담긴 일종의 컨테이너

### VPC DNS 옵션
- DNS resolution: AWS 제공 DNS 해석기
- DNS hostname: AWS 제공 Public과 Private Hostname을 사용

### VPC DHCP Option Sets
- 도메인 네임과 도메인 네임 서버의 정보를 제공
- DHCP 옵션을 생성하여 VPC에 적용 가능

### 프라이빗 호스팅 영역 DNS 쿼리 과정
- 서브넷 -> DNS 해석기 -> 프라이빗 호스팅 영역

### 다수 VPC가 1개의 프라이빗 호스팅 영역에 연결

### VPC DNS 고려사항
- 퍼블릭 보다 프라이빗 호스팅 영역 DNS 쿼리를 우선시함
- EC2는 resolover(해석기)에 초당 최대 1,024개 요청을 보낼 수 있음.


---

## Routre 53 Resolver
- 온프레미스와 VPC간에 DNS쿼리를 할 때, Route 53 해석기와 전달 규칙을 이용하여 서로 DNS 쿼리 가능

### resolvoer 관련 용어
- resolver = 해석기
- Forwarding Rule: 전달 규칙, 다른 네트워크 환경에 도메인 쿼리를 하기 위한 정보
- Inbound Endpoint: AWS VPC에서 DNS 쿼리를 받을 수 있는 네트워크 인터페이스
- Outbound Endpoint: 전달 규칙을 다른 네트워크로 쿼르를 할 수 있는 네트워크 인터페이스

### 하이브리드 환경에서 resolver 동작
1. 온프레미스에서 AWS로 DNS 쿼리를 할 경우
    - 인바운드 엔드포인트만 생성
2. AWS에서 온프레미스로 DNS 쿼리를 할 경우
    - 아웃바운드 엔드포인트 생성하여 전달 규칙을 VPC에 연결하면 가능
    - 다른 계정의 VPC의 경우에는 AWS RAM(Resource Access Manager)를 이용하여 연결이 가능


### resolver 규칙 유형
- 전달 규칙(=Conditional Forwarding Ruel)
    - 특정 도메인에 대한 쿼리를 지정한 IP(DNS 서버)로 전달
- 시스템 규칙(=System rules)
    - 전달 규칙보다 우선
    - 종류
        - 프라이빗 호스팅 영역
        - Auto defined VPC DNS
        - Amazon 제공 외부 DNS 서버

---

## Route 53 Resolver - Query Logs
- 모든 VPC내 서비스들에서 DNS 쿼리 날린 것을 cloudwatch에서 확인 가능 

