# SEC5 블록체인 응용사례

---

# CH1 블록체인 응용사례 개요
- Blockchain for businesses
  - supply chain
  - Cross-border settlement
- Blockchain for End-Users
  - Electronic Medium of Exchange
  - Unique Electronic Assets

---

# CH2 공급망(Supply Chain) 관리
- Supply Chain Management
  - 부품 제공, 생산, 배포, 고객 까지 이르는 물류의 흐름을 하나의 가치사슬 관점에서 파악
  - 필요한 정보가 원할히 흐르도록 지원하는 시스템
  - 고품질, 저가격, 적기 납기의 중요성이 증대함으로써 SCM의 중요성도 증대
  - 제품가치의 60~70퍼가 제조 이외의 부분에서 발생하므로 **전체 공급 라인의 관리**가 필요
  - 예시
    - 월마트 Food Trust Chain
      - 병충해 발생시 빠르게 근원지 파악
        - 투명성 강조와 유퉁경로 격리를 통한 피해 최소하
        - 2건의 PoC 실행
          - Hypreledger Fabric으로 구현
          - 망고와 돈육 유통 추적하는 시스템을 구축
            - 망고 원산지 추적 시간 감소: 7일 -> 2.2초
            - 돈육 인증서 블록체인 기록: 신뢰도 향상
      - 5개 공급 채널에서 25개 제품을 블록체인으로 추적관리 중

---

# CH3 해외 송금/정산
- 은행을 거친 지급 결제
  - 환거래은행을 거침
  - 복잡 + 고비용 + 즉시성 떨어짐
- 송금전문업체의 등장
  - 웨스턴유니언, 머니그램
  - 비교적 빠른 송금 가능
- 결제 전문 서비스들의 등장
  - 환거래은행 배제
  - 송금 불가능
- BoC & MAS cross-border Settlement
  - 두개의 다른 네트워크 간 통신
    - BoC=R3 Corda, MAS=JPM Quorum
    - Payment versus Payment(PvP)지급건마다 atomic swap 실행 (Hashed time-locker contract)
    - 즉시 송금 가능 & 수수료 최소화
    - 큰 액수의 결제에는 아직 사용되지 않음

---

# CH4 지역화폐
- Electronic Medium of Exchange
  - 실물화폐에서 전자화폐로 변화하는 사회
    - 중앙은행, 기업들이 발행 및 원장 관리를 수행 -> 불투명한 운영
    - 위변조 위험이 있음
  - 블록체인 기반 전자화폐
    - 분산원장(=블록체인)에 기록함으로써 투명성 및 추적성 확보
    - ex) 김포페이
      - KT의 블록체인 기반 지역화폐 플램폼 '착한페이'를 통해 발행
      - 김포시의 세금이 투입됨으로 투명성이 필수
  
---

# CH5 NFT(Non-Fungible Token)
- Unique Electronic Assets
  - 블록체인의 불변성을 사용하여 복제불가능한 디지털 데이터를 생성
    - 기존 데이터(ex, mp3)는 복제 가능.
    - 블록체인의 불변성과 스마트 컨트랙트를 사용하면 **한번 생성된 데이터와 동일한 데이터를 생성할 수 없도록 제한**(희소성)
  - NFT(Non-Fungible Token)
    - 유일한 한정수량 상품 개발 가능
    - 상품권, 증명서, 게임아이템 제작 같은것들이 가능
    - CryptoKitties 게임
      - Ethereum 기반의 가상 펫 육성 게임
      - 고양이 캐릭터를 수집하교 교배, Ether를 사용하여 캐릭터 거래가능
      - 각각의 캐릭터는 NFT로 발급되어 희소성을 가집
      - 게임성이 강조된 수집형 DApp
        - 교배를 통해 새로운 성질의 Kitty를 생성