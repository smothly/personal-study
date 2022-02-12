# SEC3 솔리디티(Solidity)와 Klatn SDK: caver-js

---

# CH1 스마트 컨트랙트, 솔리디티(Solidity)
- Smart Contracts
  - 특정 주소에 배포되어 있는 **Tx로 실행 가능한 코드**
    - 스마트 컨태랙트 소스코드는 함수와 상태를 표현
      - 함수는 상태를 변경하거나, 변경하지 않거나로 분류
    - 사용자가 스마트 컨트랙트 함수를 실행하거나 상태를 읽을 때 주소가 필요
  - 스마트 컨트랙트는 사용자가 실행
    - 상태를 변경하는 함수를 실행하려면 Tx를 생성하여 블록에 추가해야함
      - Tx체결 = 함수의 실행
    - 상태를 변경하지 않거나 상태를 읽는 행위는 Tx가 필요없고 노드에서 실행
  - Contract = Code + Data
    - code=함수, data=상태
- Solidity
  - Etherum/Klaytn에서 지원하는 스마트 컨트랙트 언어
  - 일반적인 프로그래밍 언어와 그 문법과 사용이 유사하나 몇가지 제약이 존재
    - 포인터의 개념이 없어서 recursive type의 선언이 불가능

---

# CH2 솔리디티 예제 1
# CH3 솔리디티 예제 2
#### 주석으로 설명
```java
pragma solidity ^0.5.6; // 솔리디티 버전을 지정


contract Coin{

  // address 타입은 Ethereum에서 사용하는 160-bit 주소
  address public minter;

  // balances는 mapping타입으로 address 타입을 key, uint 타입을 value로 가지는 key-value 매핑
  // 초기화되지 않은 값들은 모두 0으로 초기화되어 있음
  mapping(address => uint) public balances;
  
  // 클라이언트가 listening할 수 있는 데이터로, emit 키워드로 해당 타입의 객체를 생성하여 클라이언트에게 정보를 전달
  // 클라이언트 = application using a platform-specific SDK/Lib
  event Sent(address from, address to, uint amount);

  // 컨트랙트가 생성될 때 딱 한 번 실행되는 함수
  // 아래 함수는 msg.sender(함수를 실행한 사람의 주소)를 minter 상태변수에 대입
  constructor() public {
    minter = msg.sender;
  }

  // receiver 주소에 amount 만큼의 새로운 coin을 부여
  // require는 true일 때만 다음올 진행 ( assertdhk 비슷)
  function mint(address receiver, uint amount) public {
    require(msg.sender == minter); // 함수를 실행한 사람이 minter(contrac 소유자)일때만 진행, 
    require(amount < 1e60); // 새로 생성하는 Coin의 양 1 * 10^60개 미만일때만 진행
    balances[receiver] += amount; // receiver 주소에 amount만큼 더함
  }

  // msg.sender가 ceveiver에게 amount만큼 coin을 전송
  function send(address receiver, uint amount) public{
    require(amount <= balances[msg.sender], "Insufficient balance."); // sender의 잔고 체크 후 부족하면 에러 발생
    balances[msg.sender] -= amount; // 잔고 감소
    balances[receiver] += amount; // 잔고 증가
    emit Sent(msg.sender, receiver, amount); // 이벤트 생성
  }

}
```

- ![예시사진](https://www.researchgate.net/publication/342580319/figure/fig1/AS:908304075206658@1593567975522/EtherBank-an-example-smart-contract-written-in-Solidity.png)

---

# CH4 솔리디티 컴파일링
- EVM에서 실행 가능한 형태로 컴파일 되어야함
- solc = 솔리디티 컴파일러
  - npm으로 설치 가능
  - binary(apt, brew) 설치도 가능
- 컴파일 하면 Bytecode(.bin파일)와 ABI(.abi 파일)가 생성
  - Bytecode
    - 컨트랙트를 배포할 때 블록체인에 저장하는 정보
    - Bytecode는 Solidity 소스코드를 EVM이 이해할 수 있는 형태로 변환한 것
    - 컨트랙트 배포시 HEX로 ㅂ표현된 bytecode를 TX에 담아 노드에 전달
  - ABI(Application Binary Interface)
    - ABI는 컨트랙트 함수를 JSON형태로 표현한 정보로 EVM이 컨트랙트 함수를 실행할 때 필요
    - 헤더라고 생각하면 됨
    - 컨트랙트 함수를 실행하려는 사람은 ABI 정보를 노드에 제공
  
---

# CH5 Klatyn SDK: caver-js
- https://ko.docs.klaytn.com/dapp/sdk/caver-js
- 스마트 컨트랙트를 실행하는 프로그래밍
- Klaytn SDK(Software Development Kit)
  - BApp(Block chain) 개발을 위해 필요한 SDK 제공
  - 네트워크와 통신하는 방법, 트랜잭션 만드는 방법, 서명하는 방법, 보내는 방법 등을 만들어놓음
- 설명
  - [getBlockNumber](https://ko.docs.klaytn.com/dapp/sdk/caver-js/api-references/caver.rpc/klay#caver-rpc-klay-getblocknumber)
    - 파라미터를 callback function을 받음
      - 함수를 다 실행하고 나온 결과를 callback function으로 넘김
  - [wallet](https://ko.docs.klaytn.com/dapp/sdk/caver-js/api-references/caver.wallet)
    - create하면 비밀키/공개키가 자동으로 생성됨
    - 