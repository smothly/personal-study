# SEC03 파이썬 멀티스레딩과 멀티프로세싱

---

## 3-1 컴퓨터 구조와 운영체제 기반

---

- 컴퓨터의 구성 요소
  - CPU: 명령어 해석하여 실행하는 장치
  - 메모리
    - 주메모리: 작업에 필요한 프로그램과 데이터를 저장하는 장소
    - 보조메모리: 저장장치라고 불리며 데이터를 일시적 또는 영구적으로 저장하는 장소
  - 입출력장치
  - 시스템버스
- OS
  - 컴퓨터 시스템을 운영하고 관리하는 소프트웨어
  - 운영체제가 있는 컴퓨터를 프로그램이 가능한 기계라고 함.
  - ex) windowOS, MacOS, Ubuntu
- 프로세스
  - 프로그램: 어떤 문제를 해결하기 위해 컴퓨터에게 주어지는 처리 방법과 순서를 기술한 일련의 명령문의 집합체
  - 프로그램은 HDD,SDD와 같은 저장장치에 보관되어이 있고 그걸 실행
  - 프로그램 실행
    - 프로그램의 코드들이 주메모리에 올라옴 작업 실행 => 프로세스 생성 => CPU가 프로세스가 해야할 작업 수행
- 스레드
  - 프로세스의 작업을 할 때 CPU가 처리하는 작업의 단위
  - 스레드가 여러개로 동작하면 **멀티스레딩**
  - 멀티스레딩에는 메모리 공유와 통신이 가능해서 자원의 낭비를 막고 효율성을 향상시키나, 문제가 생기면 전체 프로세스에 영향을 끼침
  - 스레드 종류는 사용자/커널로 나뉨

## 3-2 동시성 vs 병렬성

--- 

- 동시성
  - 한 번에 여러 작업을 `동시에 다루는 것
    - 작업을 스위칭 하며 작업
  - 멀티스레드, 싱글 스레드에서 가용됨
  - 논리적 개념
- 병렬성
  - 합 번에 여러 작업을 병렬적으로 처리하는 것
    - 동시에 = at the same time
  - 멀티코어 작업에서 여러 작업을 한꺼번에 수행 가능
  - 물리적 개념

## 3-3 파이썬 멀티 스레딩

---

### 예제
- max_workers를 늘릴수록 빨라짐
- 코루틴 방식의 `asyncio`가 threading보다는 통신 I/O에서 효율이 좋음. threading 만드는과정이 있기 때문

```python
import requests
import time
import os
import threading
from concurrent.futures import ThreadPoolExecutor


def fetcher(params):
    print(params)
    session = params[0]
    url = params[1]
    print(f"{os.getpid()} process | {threading.get_ident()} url: {url}")

    with session.get(url) as response:
        return response.text


def main():
    urls = ["https://google.com", "https://instagram.com", "https://naver.com"] * 10

    executor = ThreadPoolExecutor(max_workers=5)

    with requests.Session() as session:
        params = [(session, url) for url in urls]
        results = list(executor.map(fetcher, params))
        # print(results)


if __name__ == "__main__":
    start = time.time()
    main()
    end = time.time()
    print(end - start)
```

## 3-4 파이썬 멀티 프로세싱, GIL

---

- 멀티스레드는 스레드끼리 자원을 공유함.
- 자원을 공유할 때 충돌이 발생하거나 문제가 전파될 수 있음 => `GIL`로 이러한 문제를 막음
- GIL(Global Interpreter Lock)
  - 한 번에 1개의 스레드만 유지하는 락
  - 다른 스레드를 차단해 제어를 얻는 것을 막아줌
  - 파이썬에서는 스레드로 병렬성 연산 수행 불가 => i/o bound에선 유용하나, cpu bound에선 유용하지 않음
- 멀티 프로세싱은 이러한 GIL 문제를 막아줌. 하지만, 프로세스를 복제하고 프로세스끼리 통신해야하는데 이 비용이 큼

### 예제 
- cpu bound 작업은 multi threading으로 할 때 오래걸림