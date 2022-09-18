# SEC2 Multiprocessing

## 2-1 Process vs Thread, Parallelism

---

![concurrent, parlallel](https://devopedia.org/images/article/339/6574.1623908671.jpg)
### Parrallelism
- 완전히 동일한 타이밍에 태스크 실행
- 다양한 파트로 나눠서 실행
- 멀티프로세싱에서 CPU가 1Core인 경우 만족하지 않음
- 딥러닝, 비트코인 채굴 등 python 에선 GIL 때문에 주로 활용

### Process vs Thread
- 독립된 메모리(프로세스) / 공유메모리(스레드)
- 많은 메모리 필요(프로세스) / 적은 메모리(스레드)
- 좀비 프로세스 생성 가능성 / 좀비 스레드 생성 쉽지 않음
- 오버헤드 큼(프로세스) / 오버헤드 작음(스레드)
- 생성, 소멸이 다소 느림(프로세스) / 빠름 (스레드)
- 코드 작성 쉬움, 디버깅 어려움(프로세스) / 코드 작성, 디버깅 어려움(스레드)

## 2-2 multiprocessing - Join, is_alive

---

### 예제
- thread와 비슷한 문법형태를 가짐
- thread와 다르게 process는 종료하는 `terminate`와 상태 확인하는 `is_alive`함수를 가지고 있음

```python
from multiprocessing import Process
import time
import logging

def prod_func(name):
    print('Sub-Process {}: starting'.format(name))
    time.sleep(3)
    print('Sub-Process {}: finishing'.format(name))


def main():
    # Logging format 설정
    format = "%(asctime)s: %(message)s"
    logging.basicConfig(format=format, level=logging.INFO, datefmt="%H:%M:%S")

    logging.info('Main-Process: before creating Process')
    p = Process(target=prod_func, args=('First',))

    # 프로세스
    p.start()

    logging.info('Main-Process: During Process')

    # logging.info('Main-Process: Terminated Process')
    # p.terminate()

    # logging.info('Main-Process: Joined Process')
    # p.join()

    # 프로세스 상태 확인
    print(f'Process p is alive: {p.is_alive()}')


if __name__ == '__main__':
    main()
    
```

## 2-3 multiprocessing - Naming, Parallel processing

---

### Naming
- 

### 예제
- process의 `id`와 `name`을 출력할 수 있음
- process를 여러개 생성하면 CPU가 peak침

```python
from multiprocessing import Process, current_process, parent_process
import time
import os
import random

# 실행
def square(n):
    # 랜덤 sleep
    time.sleep(random.randint(1,3))
    process_id = os.getpid()
    process_name = current_process().name
    # 제곱
    result = n * n
    print(f'Process ID: {process_id}, Process Name: {process_name}')
    print(f'Result of {n}, square: {result}')

# 메인
if __name__ == '__main__':
    # 부모 프로세스 아이디
    parent_process_id = os.getpid()
    # 출력
    print(f'Parent Process ID: {parent_process_id}')

    # 프로세스 리스트 선언
    processes = list()

    # 프로세스 생성 및 실행
    for i in range(1, 200):
        # 생성
        t = Process(name=str(i), target=square, args=(i,))

        # 배열에 담기
        processes.append(t)

        # 시작
        t.start()

    for process in processes:
        process.join()
    
    # 종료
    print('Main-Processing Done')
    
```

## 2-4 multiprocessing - ProcessPoolExecutor

---

### 예제
- `dict comprehension`을 사용하여 `future object: url` 형태의 dict를 만듬
- future는 아직 실행되지 않은 객체로 `as_completed` 함수를 사용하여 실행
- 실행한 결과는 `result` 함수를 사용하여 확인함
```python
from concurrent.futures import ProcessPoolExecutor, as_completed
import urllib.request

# 조회 URLS
URLS = [
    'http://www.daum.net',
    'http://www.cnn.com',
    'http://www.naver.com',
    'http://www.youtube.com',
    'http://www.some-made-up-domain.com',
]

# 실행함수
def load_url(url, timeout):
    with urllib.request.urlopen(url, timeout=timeout) as conn:
        # print(conn.read())
        return conn.read()

# 메인 함수
def main():
    # 프로세스풀 Context 영역
    with ProcessPoolExecutor(max_workers=5) as executor:
        # Future 로드(실행 x)
        future_to_url = {executor.submit(load_url, url, 1): url for url in URLS}

        # 중간 확인
        print(future_to_url)

        # 실행
        for future in as_completed(future_to_url): # timeout을 변경해가면서 테스트
            # Key 값이 Future 객체
            url = future_to_url[future]

            try:
                # 결과
                data = future.result()
            except Exception as exc:
                # 예외 처리
                print('%r generated an exception: %s' % (url, exc))
            else:
                # 결과 확인
                print('%r page is %d bytes' % (url, len(data)))

# 메인함수 시작
if __name__ == '__main__':
    main()
    
```

## 2-5 multiprocessing - Sharing state

---

### 예제
- 일반적인 변수를 사용할 경우 multiprocessing에서는 공유가 되지 않음
- `Value`, `Array`, `shared_memory`, `Manager` 등 공유자원 객체를 사용하여 구현하면 공유가 됨

``` python
from multiprocessing import Process, current_process, Value, Array
import os

# 프로세스 메모리 공유 예제(공유 O)

# 실행함수
def generate_update_number(v: int):
    for _ in range(50):
        v.value += 1
    print(current_process().name, "data", v.value)

# 메인 함수
def main():
    # 부모 프로세스 아이디
    parent_process_id = os.getpid()
    #출력
    print(f'Parent Process ID {parent_process_id}')
    
    # 프로세스 리스트 선언
    processes = list()

    # 프로세스 메모리 공유 변수
    # from multiprocess import shared_memory 사용 가능
    # from multiprocess import Manager 사용 가능

    # share_numbers = Array('i', range(50))
    share_value = Value('i', 0)

    for _ in range(1, 10):
        p = Process(target=generate_update_number, args=(share_value, ))
        processes.append(p)
        p.start() # 실행
    
    for p in processes:
        p.join()

    print('Final Data in parent process', share_value.value)

# 메인함수 시작
if __name__ == '__main__':
    main()
```

## 2-6 multiprocessing - Queue, Pipe

---

### 예제 1
- `queue`를 통해서 프로세스간 통신을 구현
- 2개이상의 진입점일 때 주로 사용함

```python
from concurrent.futures import process
from multiprocessing import Process, Queue, current_process
import time
import os

# 프로세스 메모리 공유 예제(공유 X)

# 실행함수
def worker(id, baseNum, q):
    process_id = os.getpid()
    process_name = current_process().name

    # 누적
    sub_total = 0

    # 계산
    for i in range(baseNum):
        sub_total += 1

    # Produce
    q.put(sub_total)

    # 정보 출력
    print(f'Process ID: {process_id}, Process Name: {process_name} ID: {id}')
    print(f'Result: {sub_total}')

# 메인 함수
def main():    
    # 부모 프로세스 아이디
    parent_process_id = os.getpid()
    #출력
    print(f'Parent Process ID {parent_process_id}')
    
    # 프로세스 리스트 선언
    processes = list()

    # 시작 시간
    start_time = time.time()

    q = Queue()

    for i in range(10):
        t = Process(name=str(i), target=worker, args=(1, 10000000, q))
        processes.append(t)
        t.start()

    for p in processes:
        p.join()

    # 순수 계산 시간
    print("--- %s seconds ---" % (time.time() - start_time))

    # 종료 플래그
    q.put('exit')

    # 대기
    total = 0
    while True:
        tmp = q.get()
        if tmp == 'exit':
            break
        else:
            total += tmp

    print('Main-Processing Total Count={}'.format(total))
    print('Main-Processing Done')


# 메인함수 시작
if __name__ == '__main__':
    main()
```

### 예제2
- `Pipe`를 통한 프로세스 간 통신
```python
from concurrent.futures import process
from multiprocessing import Process, Pipe, current_process
import time
import os

# 실행함수
def worker(id, baseNum, conn):
    process_id = os.getpid()
    process_name = current_process().name

    # 누적
    sub_total = 0

    # 계산
    for i in range(baseNum):
        sub_total += 1

    # Produce
    conn.send(sub_total)
    conn.close()

    # 정보 출력
    print(f'Process ID: {process_id}, Process Name: {process_name} ID: {id}')
    print(f'Result: {sub_total}')

# 메인 함수
def main():    
    # 부모 프로세스 아이디
    parent_process_id = os.getpid()
    #출력
    print(f'Parent Process ID {parent_process_id}')
    
    # 시작 시간
    start_time = time.time() 

    # Pipe 선언
    parent_conn, child_conn = Pipe()

    t = Process(name=str(1), target=worker, args=(1, 100000000, child_conn))
        
    t.start()
    t.join()

    # 순수 계산 시간
    print("--- %s seconds ---" % (time.time() - start_time))

    print('Main-Processing Total Count={}'.format(parent_conn.recv()))
    print('Main-Processing Done')


# 메인함수 시작
if __name__ == '__main__':
    main()
```
