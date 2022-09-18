# SEC1 Multithreading

## 1-1 Process와 Thread의 차이점

---

![process vs thread](https://blog.kakaocdn.net/dn/cllUdv/btq0LPK8Czt/rx4QEXCBpUKhMz7qOcnpiK/img.jpg)
### Process
  - 운영체제에서 할당 받는 자원 단위 = **실행 중인 프로그램**
  - CPU 동작시간, 주소공간이 다 **독립적**
  - code, data, stack, heap이 전부 독립적
  - 최소 1개의 메인 스레드를 가지고 있음
  - 파이프, 파일, 소켓 등을 사용해서 프로세스간 통신 -> context switching 비용이 큼
### Thread
  - 프로세스 내에 실행 흐름 단위
  - 프로세스 자원 사용
  - **Stack만 별도 할당**하고 나머지는 공유(code, data, heap)
  - 한 스레드의 결과가 다른 스레드의 영향 끼침 => 동기화 문제를 주의해야함
### Multi Thread
  - 단일 어플리케이션에서 여러 스레드로 구성 후 작업 처리
  - 시스템 자원 소모가 감소됨
  - 통신 부담은 감소, 디버깅 어려움, 단일 프로세스에는 효과 미약, 자원 공유 문제(데드락), 프로세스 영향을 줌
### Multi Process
  - 단일 어플리케이션에서 여러 프로세스로 구성 후 작업 처리
  - 프로세스 문제 발생은 확산이 없음. 프로세스를 kill 하면 됨
  - cache change, 복잡한 통신 방식으로 인해 cost가 높음(context switching으로 인한 오버헤드)

## 1-2 GIL(Global Interpreter Lock)

---

### Python GIL
- ![GIL 사진](https://velog.velcdn.com/images/chaduri7913/post/a50fddb4-2c86-4013-ac39-808f66c65fcd/image.png)
- 단일 스레드만이 python object에 접근 하게 제한하는 **mutex**
- cpython이 메모리 관리가 취약함. **thread-safe하게 하기 위해 GIL 사용**
- 단일 스레드로 충분히 빠름
- 프로세스 사용이 가능하기 때문에 GIL이 큰 제약이 되지 않음
- 병렬 처리는 multi processing, asyncio가 있음
- thread 동시성 완벽 처리를 위해 jython, ironpython, stackless python 등이 존재하긴 함

## 1-3 Thread - baic

---

### 예제
- 로깅 함수 사용
- 자식스레드여도 자신이 맡은 일을 끝까지 하고 종료함
- join을 추가하면 자식스레드 일을 끝날때 까지 기다리기
```python
import logging
import threading
import time

# 스레드 실행 함수
def thread_func(name):
    logging.info("Sub-Thread: %s: starting", name)
    time.sleep(3)
    logging.info("Sub-Thread: %s: finishing", name)

# 메인 영역
if __name__ == "__main__":
    # Logging format 설정
    format = "%(asctime)s: %(message)s"
    logging.basicConfig(format=format, level=logging.INFO, datefmt="%H:%M:%S")
    logging.info("Main-Thread: before creating thread")

    # 함수 인자 확인
    x = threading.Thread(target=thread_func, args=('First',))
    logging.info("Main-Thread: before running thread")

    # 서브 스레드 시작
    x.start()

    # 주석 전후 결과 확인
    x.join()

    logging.info("Main-Thread: wait for the thread to finish")

    logging.info("Main-Thread: all done")

```

## 1-4 Thread - Daemon, Join

---

### Daemon Thread
- 백그라운드에서 실행됨
- 일반 스레드는 작업 종료시 까지 실행되나 데몬스레드는 메인스레드 종료시 **즉시 종료** 됨
- 주로 백그라운드 무한 대기 이벤트 발생 실행하는 부분 담당 ex) JVM(가비지 컬렉션), 문서 자동저장

### 예제
- `daemon` 파라미터 T/F에 따른 차이를 보는 것이 핵심
```python
import logging
import threading
import time

# 스레드 실행 함수
def thread_func(name, d):
    logging.info("Sub-Thread: %s: starting", name)
    
    for i in d:
        print(i)
    
    logging.info("Sub-Thread: %s: finishing", name)

# 메인 영역
if __name__ == "__main__":
    # Logging format 설정
    format = "%(asctime)s: %(message)s"
    logging.basicConfig(format=format, level=logging.INFO, datefmt="%H:%M:%S")
    logging.info("Main-Thread: before creating thread")

    # 함수 인자 확인    
    x = threading.Thread(target=thread_func, args=('First', range(20000)), daemon=True) # Daemon: default False
    y = threading.Thread(target=thread_func, args=('Two', range(10000)), daemon=True)
    logging.info("Main-Thread: before running thread")

    # 서브 스레드 시작
    x.start()
    y.start()

    # 데몬스레드 확인
    # print(x.isDaemon())

    # 주석 전후 결과 확인
    # x.join()
    # y.join()

    logging.info("Main-Thread: wait for the thread to finish")

    logging.info("Main-Thread: all done")
```


## 1-5 Thread - ThreadPoolExecutor

---

### 그룹스레드
- python 3.2 이상 표준 라이브러리
- `concurrent.futures`: threading 관련 함수들을 사용하기 쉽게 wrapping해놓은 라이브러리
- `with` 사용으로 생성, 라이프사이클 관리 용이
- 디버깅하기 난해함
- 대기중인 작업 -> **Queue** -> 완료 상태 조사 -> 결과 또는 예외 -> 단일화

- 방법 1은 thread를 매번 선언해줘야하는 문제가 있지만, 방법 2 `with` 문과 `map`을 같이 사용하면 코드가 간단해짐
```python
import logging
from concurrent.futures import ThreadPoolExecutor
import time

def task(name):
    logging.info('Sub-Thread %s: starting', name)

    result = 0
    for i in range(10001):
        result += i

    logging.info('Sub-Thread %s: finishing result: %d', name, result)

    return result

def main():
    # Logging format 설정
    format = "%(asctime)s: %(message)s"
    logging.basicConfig(format=format, level=logging.INFO, datefmt="%H:%M:%S")
    logging.info("Main-Thread: before creating and running thread")

    # 실행 방법 1    
    # max_workers: 작업을 실행할 갯수
    executor = ThreadPoolExecutor(max_workers=3)
    
    task1 = executor.submit(task, ('First',))
    task2 = executor.submit(task, ('Second',))
    
    # 결과 값 있을 경우 출력가능
    print('\n')
    print(task1.result())
    print(task2.result())

    # 실행 방법 2
    with ThreadPoolExecutor(max_workers=3) as executor:
        tasks = executor.map(task, ['First', 'Second'])

        # 결과 확인
        print(list(tasks))
if __name__ == "__main__":
    main()

```


## 1-6 Thread - Lock, Deadlock

---

### 세마포어(Semaphore)
- **프로세스간 공유 된 자원에 접근** 시 문제 발생 가능성 => 한 개의 프로세스만 접근하도록 함(경쟁 상태 예방)
### 뮤텍스(Mutex)
- **공유된 자원의 데이터를 여러 스레드가 접근**하는 것을 막는 것 => 경쟁 상태 예방
### 세마포어 뮤텍스 차이점
- 모두 병렬 프로그래밍에서 **상호배제**를 위해 사용
- 뮤텍스 개체는 단일 스레드가 리소스 또는 중요 섹션을 소비 허용
- 세마포어는 리소스에 대한 제한된 수의 동시 액세스를 허용
- 세마포어는 뮤텍스가 될 수 있으나 뮤텍스는 세마포어가 될 수 없다. 단일 프로세스의 세마포어일 경우 뮤텍스이다
### Lock
- 상호 배제를 위한 잠금 처리 => 데이터 경쟁
### 데드락(DeadLock)
- ![데드락](https://images.velog.io/images/jess29/post/2e105766-20d0-458a-a145-cca047e6d6f9/Deadlock-in-Java.png)
- 프로세스가 자원을 획득하지 못해 다음 처리를 못하는 무한 대기 상황(교착 상태)
### Thread Synchronization
- 스레드 동기화를 통해서 안정적으로 동작하게 처리함. (동기화 메소드, 동기화 블럭)

### 예제
- 스레드 동기화를 `Lock`을 통해 구현
- `with` 절을 사용한 두번째 방법이 더 유용할 것으로 판단
```python
import logging
from concurrent.futures import ThreadPoolExecutor
import time
import threading

class FakeDataStore:
    # 공유 변수(value)
    def __init__(self):
        self.value = 0 # 데이터 영역에 저장
        self._lock = threading.Lock()

    # 변수 업데이트 함수
    def update(self, n):
        logging.info('Thread %s: starting update', n)

        # Lock 획득 (방법 1)
        self._lock.acquire()
        logging.info('Thread %s has lock', n)

        # 뮤텍스 & Lock 등 동기화 (Thread Synchronization 필요)
        local_copy = self.value # 스택 영역에 저장. 스레드는 스택만 별도. 값이 업데이트기되기전에 참조해서 원하는 값이 나오지 않음
        local_copy += 1
        time.sleep(0.1)
        self.value = local_copy

        logging.info('Thread %s about to releas lock', n)
        # Lock 반환
        self._lock.release()

        # Lock 획득(방법2)
        with self._lock:
            logging.info('Thread %s has lock', n)

            local_copy = self.value
            local_copy += 1
            time.sleep(0.1)
            self.value = local_copy

            logging.info('Thread %s about to releas lock', n)

        logging.info('Thread %s: finishing update', n)


def task(name):
    logging.info('Sub-Thread %s: starting', name)

    result = 0
    for i in range(10001):
        result += i

    logging.info('Sub-Thread %s: finishing result: %d', name, result)

    return result


if __name__ == "__main__":
    # Logging format 설정
    format = "%(asctime)s: %(message)s"
    logging.basicConfig(format=format, level=logging.INFO, datefmt="%H:%M:%S")

    store = FakeDataStore()
    logging.info('Testing update. Starting Value is %d', store.value)

    # With context 시작
    with ThreadPoolExecutor(max_workers=2) as executor:
        for n in ['First', 'Second', 'Third', 'Four']:
            executor.submit(store.update, n)
    
    logging.info('Testing update. Ending Value is %d', store.value)

```


## 1-7 Thread - Prod and Cons Using Queue

---

### 생산자 소비자 패턴
- ![패턴](https://jenkov.com/images/java-concurrency/producer-consumer-1.png)
- 멀티스레드 디자인 패턴의 정석
- 서버측 프로그래밍의 핵심
- 주로 허리역할

### Python Event 객체 설명
- 변수
  - Flag 초기값 0
- 함수
  - Set() -> 1
  - Clear() -> 0
  - Wait -> 1 리턴, 0 -> 대기
  - isSet -> 현 플래그 상태

### 예제
- `event` 객체를 사용해서 producer, consumer를 멀티스레드로 동작하게 함
```python
from concurrent.futures import ThreadPoolExecutor
import logging
import time
import queue
import random
import threading

# 생산자
def producer(queue, event):
    """네트워크 대기 상태라 가정(서버)"""

    while not event.is_set(): # event가 0일 경우 중단
        message = random.randint(1, 11)
        logging.info('Producer got message: %s', message)
        queue.put(message)
        
    logging.info('Producer sended  event Exiting')


# 소비자
def consumer(queue, event):
    """응답 받고 소비하는 것으로 가정 or DB 저장"""
    while not event.is_set() or not queue.empty(): # event가 0일 경우 중단
        message = queue.get()
        logging.info('Consumer storing message: %s (size=%d)', message, queue.qsize())

    logging.info('Consumer received event Exiting')


if __name__ == '__main__':
    # Logging format 설정
    format = "%(asctime)s: %(message)s"
    logging.basicConfig(format=format, level=logging.INFO, datefmt="%H:%M:%S")

    # 사이즈 중요. 무작정 키우는 것만이 정답은 아님
    pipeline = queue.Queue(maxsize=10)

    # 이벤트 플래그 초기 값 0 
    event = threading.Event()
    
    # with context 시작
    with ThreadPoolExecutor(max_workers=2) as executor:
        executor.submit(producer, pipeline, event)
        executor.submit(consumer, pipeline, event)

        # 실행 시간 조정
        time.sleep(1)
        logging.info('Main: about to set event')

        # 프로그램 종료
        event.set()
```
