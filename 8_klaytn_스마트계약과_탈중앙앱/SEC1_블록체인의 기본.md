# SEC1_블록체인의 기본

---

# CH1 블록체인의 정의, 해시함수
- 블록체인 
  - 링크드리스트와 같은 블록끼리의 연결체
- 해시함수를 사용
  - 하나의 데이터에서 오직 단 하나의 값이 나옴
  - 비슷한 값이여도 해시 값은 크게 다름
  - ex) sha-256: hex 64글자(1byte = hex 2자리) = 32bytes = 32 x 8 = 256bit

---

# CH2 블록체인의 구조, 주요 용어
- 블록, 블록헤더, 해시포인터
  - 블록은 헤더와 바디로 구분
  - 헤더는 블록을 설명하는 정보와 **이전 블록의 해시**를 포함
  - 해시포인터는 이전 블록의 해시를 가지고 있다고 생각하면 됨
- 블록높이, 블록생성주기
  - 블록이 생성됨에 따라 체인의 높이가 늘어남
  - 첫번째 블록의 높이는 0
  - 블록생성주기 = 다음 블록을 생성하기까지 걸리는 시간
  - 블록주기 = 블록 생성사이에 주기
    - 비트코인은 10분, 이더리움은 15초
    - 클레이튼은 1초
    - 체결까지 걸리는 시간이 중요.
    - 클레이튼은 서비스에서 사용될 블록체인을 지향하기에 짧은 시간으로 생성

---

## CH3 블록체인 네트워크
- 모든 노드들이 같은 블록을 가지고 있다.
- 합의(Consensus)
  - 자격이 있는 참여자는 블록은 지안(propsoe)할 수 있음
  - PoW(Proof of Work)
    - 제안자가 올바른지, 블록이 올바른지에 대한 검증과정
- 블록체인의 성질
  - 탈중앙화
  - 모두가 같은 데이터(같은 순서의 블록)를 가짐
- 블록 제안 자격에 따른 볼록 제안
- 합의에 따른 체인에 블록 추가
- 무결성, 투명성

---

## CH4 합의 알고리즘
- ![합의 알고리즘](https://mblogthumb-phinf.pstatic.net/MjAyMDA4MjlfMjk2/MDAxNTk4NjkwOTcxNDIx.jB0-ShcACqkK_n8dE_8FobJ5iplY5Uro3J0cP6sCWl4g.W11xK7jSopk5qAf3nq5DBkQ5IHsER6KTdw8vlk0xA58g.PNG.s_usie/%ED%95%A9%EC%9D%98%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98.PNG?type=w800)
- 표가 설명이 너무 잘 되어 있음

---

## CH5 블록체인의 비교
- Public vs Private
  - 기록된 정보(블록)을 자유롭게 읽을 수 있는가?
  - 자격없이 정보를 블록체인 네트워크에 기록할 수 있는가?
- Permissionless vs Permissioned
  - 네트워크 참여의 제한 여부
- public/private는 **정보의 접근성(Access)**, Permissionless/Permissioned는 **정보의 제어(Control)**
- ![비교](https://www.researchgate.net/profile/Blesson-Varghese/publication/329295642/figure/fig2/AS:699768577720330@1543849238962/Permissioned-versus-permissionless-blockchains-across-trust-and-anonymity-axes-This.jpg)

---

## CH6 공개키 암호화와 전자서명
- 대칭키/비대칭키 암호
  - 암호화에 사용한 (공개키 Public Key)키와 복호화에 사용한 키(비밀키 Private Key)가 동일한 경우 대칭키. 다를경우 비대칭키.
- 전자서명
  - 누가 정보를 보냈는지 알기 위하 사용
  - 서명은 비밀키로만 생성 가능
- 메시지와 전자서명을 같이보냄
  - 전자서명은 누가 보냈는지
  - 메시지는 암호화된 문서
- ![안전한 통신 최종](https://media.vlpt.us/images/dnjscksdn98/post/32c57d4f-1008-4a5e-99b6-19010a01b36a/image.png)

---

## CH7 블록체인에서 사용되는 암호화 기법
- 원장은 address와 balance로 기록
  - 원장은 어느 주소에 BTC가 있는지 기록, 그 주소가 누구에게 속한지는 기록하지 않음
- 공개키암호화를 사용한 소유권 증명
- 전자서명에 무언가를 하면 전자서명의 공개키가 나옴
- 구현방법
  - UTXO(Unspent Transaction Output) 기반 블록체인
    - 돈에다가 주소를 줌
    - 자격검증방법은 UTXO의 정보와 일치하는 공개키로 검증가능한 전자서명을 제출
    - 비트코인에 해당
  - Account-based Blockchain
    - 어카운트 공개키로 검증가능한 전자서명을 생성
    - 상태를 기록할 수 있기 때문에 스마트 컨트랙트를 구현하기에 용이
    - 이더리움, 클레이튼에 해당
    - 비밀키 -> 공개키 -> 주소
- 트랜잭션
  - 어카운트의 행동임
  - 트랜잭션의 순서는 중요
  - 블록체인 참여자들은 트랜잭션의 순서가 올바른지 검증
  - 각각의 트랜잭션들은 **어카운트에 연결된 공개키로 검증가능한 서명**을 포함
- Confirmation vs Finality
  - Confirmation은 트랜잭션이 블록에 포함된 이후 생성된 블록의 숫자
    - T가 높이가 100이고, 현재 블록높이가 105라면 T의 confirmation 숫자는 6
  - Finality는 블록의 완결성(사라질 수 있음)
  - PoW는 finality가 없기 때문에 confirmation 숫자가 중요
    - 동시에 블록생성하면 하나는 사라짐
    - 블록이 확률적 완결성(99.9%)을 갖기까지 일정 갯수 이상의 블록이 생성되기까지를 기다려야함
- 6 Confirmations Rules
  - 퍼즐을 빠르게 풀 수 있는 악의적인 참여자가 있을 경우 confirmation 숫자를 조정
    - 6개는 25%의 해시능력을 가졌을때를 가정
  - longest chain법칙 때문에 해시능력이 높은 사람들이 독식할 수 있음
  - 따라서 임의로 블록체인을 변경하지 못할 정도로 충분히 많은 블록이 생성되기를 기다려야함.
- BFT 기반 블록체인
  - 블록의 완결성이 보장
    - 네트워크가 동기화되어 있기 때문
    - 네트워크 참여자 구성이 고정되어 있어야 합의가 가능
    - 네트워크 통기화를 가정하기 때문에 네트워크 사용량이 높음