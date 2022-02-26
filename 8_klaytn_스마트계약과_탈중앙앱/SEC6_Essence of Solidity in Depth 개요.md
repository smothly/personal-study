# SEC6 Essence of Solidity in Depth

---

# CH1 Essence of Solidity in Depth
- 섹션 개요
  - Structure of a Contaract
  - Data Types
  - Special Variables and Functions
  - Expressions and Control Structure
  - Contracts

---

# CH2 컨트랙트 구조
- Structure of a Contaract
  - State Variables
    - 블록체인에 영구히 저장할 값들을 State Variable로 선언
    - public 키워드는 변수를 외부에 노출
      - public은 자동으로 getter함수 생성
  - Functions
    - 실행 가능한 코드를 정의
    - external, public, internal, private 중 하나로 visibility를 설정 가능
    - payable, view, pure등 함수의 유형을 정의 가능
  - Function Modifiers
    - 함수의 실행 전, 후의 성격을 정의. 실행조건을 정의하는데 사용됨
    - ex)
    ```java
    // 투표 앱 예시
    contract Ballot{
        constructor() public { chairperson = msg.sender; }
        addeess chairperson;
        modifier onlyChair{
            require(msg.sender == chairperson, "Only the chairperson can call this function.");
            _;
        }
    }
    function giveRightToVote(address to) public onlyChair{
        // 'onlyChair' modifier ensures that this function is called by the chairperson
    }
    ```
  - Events  
    - EVM 로깅을 활용한 시스템
    - 이벤트가 실행될 때마다 트랜잭션 로그에 저장
    - 저장된 로그는 컨트랙트 주소와 연동되어 클라이언트가 RPC로 조회 가능
    ```java
    // Contract
    contract Ballot{
        event Voted (address voter, uint proposal);
        function vote(uint proposal) public {
            ...
            emit Voted(msg.sender, proposal)
        }
    }
    ```
    ```javascript
    // Client using caver-js
    const BallotContract = new caver.klay.Contract(abi, address);
    BallotContract.events.voted{
        { fromBlock: 0 },
        function(error, event){
            console.log(event);
        }
    }.on('error', console.error)
    ```
  - Stuct Types
    - Solidity에서 제공하지 않는 새로운 형식의 자료를 만들 때 사용.
    ```java
    contract Ballot{
        struct Voter{
           uint weight;
           bool voted;
           address delegate;
           uint vote; 
        }
    }
    ``` 
  - Enum Types
    - Enum은 임의의 상수를 정의하기에 적합
      - 상태: activae, inactive
      - 요일: monday, tuesday ...
    ```java
    contract Ballot{
        enum Status{
           Open,
           Closed
        }
    }
    ``` 

---

# CH3 자료형
- Data types
  - Booleans
  - Integers
  - **Address**
    - klaytn 주소의 길이는 20바이트: address -> bytes2.0으로 형변환 가능
    - address vs address payable
      - 두개가 할 수 있는 것이 다르다.
      - address payable => address 는 가능
      - address => address payable 는 불가능. uint 160을 거쳐서만 가능
  - **Fixed-size byte arrays**
  - **Reference Types**
    - structs, arrays, mapping과 같이 크기가 정해지지 않은 데이터를 위해 사용
    - 변수끼리 대입할 경우 같은 값을 참조 = Call By Reference
    - 저장되는 위치를 반드시 명시
      - memory: 함수 내에서 유효한 영역에 저장. 함수와 함께 생겼다가 사라짐
      - storage: state variables와 같이 영속적으로 저장되는 영역에 저장. 
      - calldata: external 함수 인자에 사용되는 공간
      - 서로 다른 영역을 참조하는 변수 간 대입(storage -> memory)이 발생할 시 데이터를 참조가 아니라 복사함
  - **Arrays**
    - 선언 유형
      - T[k] x; => k개의 T를 가진 배열 x를 선언
      - T[] x; => x는 T를 담을 수 있는 배열. x의 키기는 가변적
      - T[][k] x => k개의 T를 담을 수 있는 dynamic size 배영 x를 선언
    - 모든 유형의 데이터를 배열에 담을 수 있음. ex) mapping, struct
    - `.push`, `.length`, `delete`=값들을 지워 초기화해줌
    - 런타임에 만드는 memory 배열은 항상 **new키워드와 size를 지정해주어야함**
      ```java
      contract C{
          function f(uint len) public pure {
              bytes memory b = new bytes(len);
              assert(b.length == len);
          }
      }
      ```
    - bytesN vs bytes/string vs byte[]
      - 가능하면 bytes를 사용
        - byte[]는 배열 아이템간 31바이트 패딩이 추가됨
      - 기본 룰
        - 임의의 길이의 바이트 데이터를 담을 때는 bytes
        - 임의의 길이의 데이터가 UTF-8과 같이 문자로 인코딩 될 수 있을 때는 string
        - 바이트 데이터의 길이가 정해져잇을 때는 value type의 bytes1, bytes2, ... bytes32를 사용
        - byte[]는 지양
  - **Mapping Types**
    - 해시테이블과 유사, 배열처럼 사용
      - storage 영역에만 저장 가능
      - 함수 인자, public 리턴 값으로 사용할 수 없음
    ```java
    contract MappingExample {
        mapping(address => uint) public balances;

        function update(uint newBalance) public {
            balances[msg.sender] = newBalance
        }
    }
    ```
  - **Contract Types**

---

# CH4 컨트랙트 빌트인 값과 함수
- Special Variables and Functions
  - -
    - blockhash(uint blockNumber) returns (bytes32): 블록 해시 (최근 256 블록까지만 조회가능)
    - block.number (uint): 현재 블록 번호
    - block.timestamp (uint): 현재 블록 타임스탬프
    - gasleft() returns (uint256): 남은 가스량
    - msg.data (bytes calldata): 메세지(현재 트랜잭션)에 포함된 실행 데이터 (input)
    - msg.sender (address payable): 현재 함수 실행 주체의 주소
    - msg.sig (bytes4): calldata의 첫 4 바이트 (함수 해시)
    - msg.value (uint): 메세지와 전달된 KLAY (peb 단위) 양
    - now (uint): block.timestamp와 동일
    - tx.gasprice (uint): 트랜잭션 gas price (25 ston으로 항상 동일)
    - tx.origin (address payable): 트랜잭션 주체 (sender)
  - Error Handling
    - assert(bool condition): condition이 false일 경우 실행 중인 함수가 변경한 내역을 모두 이전 상태로 되돌림 (로직 체크)
    - require(bool condition): condition이 false일 경우 실행 중인 함수가 변경한 내역을 모두 이전 상태로 되돌림 (외부 변수 검증), 웬만하면 require쓰는 게 좋음
    - require(bool condition, string memory message): require(bool)과 동일. 추가로 메시지 전달
  - Cryptographic Functions
    - 가스비를 먹는 3대장이여서 가급적 안쓰는게 좋음
    - keccak256(bytes memory) returns (bytes32): 주어진 값으로 Keccak-256 해시를 생성
    - sha256(bytes memory) returns (bytes32): 주어진 값으로 SHA-256 해시를 생성
    - ecrecover(bytes32 hash, uint8 v, bytes32 r, bytes32 s) returns (address): 서명(v, r, s)로부터 어카운트 주소를 도출(서명 => 공개키 => 주소)

---

# CH5 제어구조, 컨트랙트 자료형
- Expressions and Control Structure
  - 제어 구문
    - if, els, while, do, for, break, contunue, return
  - 예외처리 기능 없음(try-catch)
  ```java
  // for문
  function loop(uint repeat) public pure returns (uint) {
    uint sum = 0;
    for (uint i = 0; i < repeat; i++) {
    sum += i;
    }
    return sum;
  }
  ```
  ```java
  // while문
  function whileloop(uint repeat)
    public pure
    returns (uint)
    {
        uint sum = 0; uint i = 0;
        while (i < repeat) {
        sum += i;
        i++;
    }
    return sum;
  }
  ```
- Contracts
  - 일반적인 컨트랙트 생성 => 배포
  - 컨트랙트를 클래스처럼 사용
    - new 키워드르 사용
  ```java
  contract A {
    B b;
    constructor() public { b = new B(10); }
    function bar(uint x) public view
        returns (uint)
    {
        return b.foo(x);
    }
  }
  ```
  - Visibility and Getters
    - external
      - 다른 컨트랙트 & 트랜잭션을 통해 호출 가능
      - internal 호출 불가능
    - public
      - 트랜잭션을 통해 호출 가능, internal 호출 가능
    - internal
      - 외부에서 호출 불가능, internal 호출 가능, 상속받은 컨트랙트에서 호출 가능
    - private
      - internal 호출 가능
  - Function Declartions
    - pure
      - State Variable 접근 불가 = read 불가 write 불가
    - view
      - State Variable 변경 불가 = read 가능 write 불가
    - none
      - 제약없음
  - Fallback function
    - 컨트랙트에 일치하는 함수가 없을 경우 실행
      - 단 하나만 정의 가능 & 함수명, 함수인자, 반환값 없음
      - 반드시 external로 선언
    - 컨트랙트가 KLAY를 받으려면 payable fallback function이 필요
      - payble fallback이 없는 컨트랙트가 KLAY를 전송받으면 오류 발생
    ```java
    contract Escrow {
        event Deposited(address sender, uint amount);
        function() external payable {
            emit Deposited(msg.sender, msg.value);
        }
    }
    ```