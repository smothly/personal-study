# 6-10 AWS Direct Connect

![direct connect](https://imgnew.megazone.com/2020/01/20200128-1.png)

- co location 환경에서 AWS와 전용 네트워크 연결
- VPN은 인터넷 환경에 따라 연결 품질이 좌우되지만 실제 전용선을 연결하는 것이라 일관성 있는 품질

### 특징

- AWS Direct Connect Location
  - 각 리전마다 Direct Connect 로케이션이 존재
  - 실제 AWS 백본 네트워크와 연결되는 엣지 라우터가 위치함
  - 서울리전은 가산과 평촌 두 곳
- Dedicated Connection
  - 1Gbps 또는 10Gbps 대역폭 제공
  - 파트너사 네트워크장비로도 연결 가능
- Hosted Connection
  - 파트너를 통해 Hosted Connection 구성도 가능
- 가상 인터페이스
  - Direct Connect 연결에 대한 가상 인터페이스를 생성해야함
  - 프라이빗, 퍼블릭, 전송 유형으로 생성 가능
    - 프라이빗은 AWS VPC의 연결을 위해 사용
    - 퍼블릭은 AWS 백본 네트워크와 직접 연결(s3 같은 퍼블릭 서비스)
    - 전송은 라우팅 서비스를 제공하는 AWS 전송 게이트웨이와 연결

### 기능

- BGP 라우팅 프로토콜 사용
- 네트워크 경로를 서로 교환할 뿐만 아니라 BGP 피어링을 구성하여 AWS Direct Connect와 연결
- Multiple Connections
  - 여러 연결을 통해 복원력과 대역폭 향상 가능
  - 유지보수가 있어 이중화를 필수로 함
    - 로케이션을 한곳만 사용(Dev)
    - 로케이션을 두곳 사용하여 이중화 구성(Live)
    - DX는 하나만 사용하고 VPN으로 백업구성하여 이중화 구성
    - 로케이션 + 라우터로 둘다 이중화
- Link Aggregation Group(LAG)
  - LACP(Link Aggregation Control Protocol) 표준 프로토콜 지원
  - 다수의 연결을 하나의 논리적인 연결로 구성 가능 = 더 높은 대역폭 및 장애에 대한 내결함성을 높임
- Bidirectional Forwarding Detection(BFD)
  - BGP에서 Peer Keepalive를 통해서 감지하기엔 시간이 많이 소요
  - BFD는 지정된 임계치 이후에 빠르게 BGP연결을 끊어 다음 경로로 Routing 가능

### 구성

- 구성 순서: Direct connect connection 요청 -> LOA-CFA 다운로드 -> LOA-CFA 전달 -> 가상 인터페이스 생성 -> 구성 다운로드 -> 가상 인터페이스 상태 확인
  - Direct connect connection 요청
    - 로케이션과 대역폭 선택 파트너도 선택
    - 대역폭을 변경할려면 CONNECT연결과 가상 인터페이스를 새로 생성해야함
  - LOA-CFA 다운로드
    - Letter of Authorization and Connecting Facility Assignment
    - 연결일 선택해서 세부 정보 보기하면 다운로드 가능
  - LOA-CFA 전달
    - 파트너나 회선 사업자에게 전달
    - Hosted의 경우에는 콘솔을 통해 확인 가능
  - 가상 인터페이스 생성
    - 용도에 맞는 가상 인터페이스 선택하여 생성
    - BGP 피어링에 필요한 정보 입력
  - 구성 다운로드
    - 네트워크 장치에 설정할 설정값을 다운로드 가능
    - 다운로드 받은 정보로 온프레미스 네트워크 장치에 설정
  - 가상 인터페이스 상태 확인
    - up이면 사용 가능
    - AS 번호와 공인 IP 대역을 입력하면 검증함. 약 72시간

### Direct Connect Gateway

- 동일 리전만 가능
- 10개의 VPC 까지 가능
