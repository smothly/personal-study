# SEC3 concurrency, cpu_boud vs io bound

## 3-1 what is concurrency

---

### Concurrency(동시성)
- CPU 가용성 극대화 위해 Parallelism의 단점 및 어려움을 구현 레벨에서 해결하기 위한 방법
- **싱글코어에 멀티스레드 패턴**으로 작업을 처리
- 동시 작업에 있어서 일정량 처리 후 다음 작업으로 넘기는 방식
- 즉, **제어권을 주고 받으며**하는 작업 처리 패턴, 병렬적은 아니나 유사한 처리 방식


### Concurrency(동시성) vs Parallelism(병렬성)
- 동시성
  - 논리적
  - 동시 실행 패턴(논리적)
  - 싱글코어, 멀티 코어에서도 실행 가능
  - 한 개의 작업을 공유 처리
  - 디버깅 매우 어려움
  - Mutex
- 병렬성
  - 물리적
  - 물리적으로 동시 실행
  - 멀티코어에서 구현 가능
  - 주로 별개의 작업 처리
  - 디버깅 어려움
  - OpenMP, CUDA 등에서 사용
- ![동시성 병렬성 비교](https://img1.daumcdn.net/thumb/R750x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FECnn0%2Fbtra4XET9VY%2FqxnReo4Pl32ED0I6zPnkX0%2Fimg.png)

## 3-2 Blocking IO vs Non-Blocking IO, Sync vs Async

- I/O는 파일 읽기 쓰기, 웹 페이지 로드 등이 있음

### Blocking IO
- system call 요청시 커널 IO 작업 완료시 까지 **응답 대기**
- 제어권: IO작업 -> 커널 소유 -> 응답(response) 전까지 대기(Block),다른 작업 수행 불가

### Non-Blocking IO
- system call 요청시 커널 IO 작업 완료 여부와 상관없이 **즉시 응답**
- 제어권: IO 작업 -> 유저 프로세스 -> 다른 작업 수행 가능(지속) -> 주기적으로 system call을 통해서 IO 작업 완료 여부 확인

### Sync vs Async
- 작업 완료여부를 누가 신경쓰느냐? 가 중요
- Async
  - IO 작업 완료 여부에 대한 noti는 커널(호출되는 함수) -> 유저 프로세스(호출하는 함수)
- Sync
  - IO 작업 완료 여부에 대한 noti는 유저프로세스(호출하는 함수) -> 커널(호출되는 함수)

### 추가 자료(정리) - [블로그](https://homoefficio.github.io/2017/02/19/Blocking-NonBlocking-Synchronous-Asynchronous/)
- Blocking/NonBlocking은 호출되는 함수가 바로 리턴하느냐 마느냐가 관심사
  - 바로 리턴하지 않으면 Blocking
  - 바로 리턴하면 NonBlocking
- Synchronous/Asynchronous는 호출되는 함수의 작업 완료 여부를 누가 신경쓰냐가 관심사
  - 호출되는 함수의 작업 완료를 호출한 함수가 신경쓰면 Synchronous
  - 호출되는 함수의 작업 완료를 호출된 함수가 신경쓰면 Asynchronous
- 성능과 자원의 효율적 사용 관점에서 가장 유리한 모델은 Async-NonBlocking 모델이다.
- ![정리 사진](https://images.velog.io/images/tess/post/7f1e8bff-032f-4709-a965-f1271d02d733/2021-03-07T20_37_39.png)
  - sync blocking io: 완료될 때 까지 대기함.(blocking) 완료됨도 호출하는 측에서 확인함(sync)
  - sync non blocking io: blocking과는 다르게 주기적으로 `system call`을 통해서 완료 여부 확인, 완료전까지 다른 작업 수행 가능(non-blocking) 완료됨은 호출하는 측에서 확인함(sync)
  - async blocking io: 완료될 때 까지 대기함.(blocking) 완료됨을 커널에서 알려줌(async)
  - async nonblocking io: blocking과는 다르게 주기적으로 `system call`을 통해서 완료 여부 확인, 완료전까지 다른 작업 수행 가능(non-blocking). 완료됨을 커널에서 알려줌(async). python async 라이브러리에서 사용하는 방법

## 3-3 Multiprocessing vs Multithreading vs Async I/O

---

### cpu bound
- CPU 연산 위주 작업
- `Multi processing`에 적합. 10개 부엌(프로세스)에 10명의 요리사(스레드)가 10개의 음식(작업) 만들기
- ex) 행렬 곱, 고속 연산, 압축, 집합 연산 등

### i/o bound
- 작업에 의해서 병목(수행시간)이 결정
- CPU 성능 지표가 수행시간 단축에 영향을 끼치지 않음
- `multi threading`에 적합. 1개 부엌(프로세스)에 10명의 요리사(스레드)가 10개의 음식(작업) 만들기
- ex) 파일 쓰기, 디스크 작업, 네트워크 통신, 시리얼 포트 송수신

### asyncio
- single process, single thread
- cooperative multitasking, tasks coopetatibely decide switching
- slow I/O bound 작업에 유용
- 1개 부엌, 1명 요리사, 10개 요리

### 추가자료 - [스택오버플로우](https://stackoverflow.com/questions/27435284/multiprocessing-vs-multithreading-vs-asyncio-in-python-3)
- CPU Bound => Multi Processing
- I/O Bound, Fast I/O, Limited Number of Connections => Multi Threading
- I/O Bound, Slow I/O, Many connections => Asyncio

## 3-4 I/O bound(1) - Synchronous

---

### 예제
- `Sync Blocking` 예제로서 순차적으로 실행하는 로직임

```python
import requests
import time

# 실행함수1 (다운로드)
def request_site(url, session):
    # print(session)
    # print(session.headers)

    with session.get(url) as response:
        print(f'[Read Contents: {len(response.content)}, Status Code : {response.status_code} from {url}]')

# 실행함수2 (요청)
def request_all_sites(urls):
    with requests.Session() as session:
        for url in urls:
            request_site(url, session)

# 메인 함수
def main():
    # 테스트 URL
    urls = [
        "https://www.jython.org",
        "http://olympus.realpython.org/dice",
        "https://realpython.com"
    ] * 3

    # 실행 시간 측정
    start_time = time.time()

    request_all_sites(urls)

    # 실행 시간 종료
    duration = time.time()- start_time

    print()
    # 결과 출력
    print(f'Downloaded {len(urls)} sites in {duration} seconds')

# 메인함수 시작
if __name__ == '__main__':
    main()
```

## 3-5 I/O bound(2) -threading vs asyncio vs multiprocessing

---

- i/o bound 잡인 크롤링을 multithreading, multiprocessing, asyncio로 각각 구현해보고 차이를 보기

### 예제1 I/O Bound Multithreading
- thread는 스택을 제외하고는 공유하므로, 각각의 독립된 네임스페이스를 얻기위해 `threading.local()`을 사용함
- `GIL`제약이 있지만, I/O bound 태스크에서는 process 생성의 비용보다는 성능이 더 괜찮음

```python
import concurrent.futures
import threading
import requests
import time

# 각 스레드에 생성되는 객체(독립된 네임스페이스)
# 각 스레드마다 독립된 메모리를 얻는 것. 크롤링할 때 각 스레드가 다른 헤더를 가지게 할 때 쓰임
# 멀티스레딩으로 동일한 스레드 객체가 살아있으면 그대로 사용
thread_local = threading.local()

def get_session():
    if not hasattr(thread_local, "session"):
        thread_local.session = requests.session()
    return thread_local.session

# 실행함수1 (다운로드)
def request_site(url):
    # 세션획득
    session = get_session()        

    print(session) # session을 몇번 돌려 쓰는지 확인
    # print(session.headers)
    with session.get(url) as response:
        print(f'[Read Contents: {len(response.content)}, Status Code : {response.status_code} from {url}]')

# 실행함수2 (요청)
def request_all_sites(urls):
    # 멀티스레드 실행
    # 반드시 max_worker 개수 조절 후 session 객체 확인
    with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
        executor.map(request_site, urls)

# 메인 함수
def main():
    # 테스트 URL
    urls = [
        "https://www.jython.org",
        "http://olympus.realpython.org/dice",
        "https://realpython.com"
    ] * 5

    # 실행 시간 측정
    start_time = time.time()

    request_all_sites(urls)

    # 실행 시간 종료
    duration = time.time()- start_time

    print('=' * 15)
    # 결과 출력
    print(f'Downloaded {len(urls)} sites in {duration} seconds')

# 메인함수 시작
if __name__ == '__main__':
    main()
```

### 예제2 I/O Bound Multiprocessing
- process 생성의 비용이 크므로 `initializer` 파라미터를 통해 session을 초기에 생성하고 공용으로 사용한다.


```python
import multiprocessing
import requests
import time

# 각 프로세스 메모리 영역에 생성되는 객체(독립적)
# 함수 실행 할 때 마다 객체 생성은 좋지 않으므로 각 프로세스마다 할당

session = None

def set_global_session():
    global session
    if not session:
        session = requests.session()

# 실행함수1 (다운로드)
def request_site(url):    
    # session 확인
    print(session) 
    # print(session.headers)
    with session.get(url) as response:
        name = multiprocessing.current_process().name
        print(f'[{name} Read Contents: {len(response.content)}, Status Code : {response.status_code} from {url}]')

# 실행함수2 (요청)
def request_all_sites(urls):
    # 멀티프로세싱  실행
    # processes  개수 조절 후 session 객체 및 실행 시간 확인
    with multiprocessing.Pool(initializer=set_global_session, processes=4) as pool:
        pool.map(request_site, urls)

# 메인 함수
def main():
    # 테스트 URL
    urls = [
        "https://www.jython.org",
        "http://olympus.realpython.org/dice",
        "https://realpython.com"
    ] * 5

    # 실행 시간 측정
    start_time = time.time()

    request_all_sites(urls)

    # 실행 시간 종료
    duration = time.time()- start_time

    print('=' * 15)
    # 결과 출력
    print(f'Downloaded {len(urls)} sites in {duration} seconds')

# 메인함수 시작
if __name__ == '__main__':
    main()
```

### 예제3 Asyncio Basic
- 3.4부터 비동기(asyncio) 표준라이브러리 등장
- 비동기 함수는 `async`를 붙이고, 비동기 함수 내에서 비동기 함수를 호출할 때는 `await`을 붙임
- 비동기 함수내에 동기 함수가 있으면 동기적으로 실행됨

```python
import asyncio
import requests
import time

def exe_calculate_sync(name, n):
    for i in range(1, n+1):
        print(f'{name} -> {i} of {n} is calculating...')
        time.sleep(1)
    print(f'{name} - {n} working done!')

def process_sync():
    start = time.time()

    exe_calculate_sync('One', 3)
    exe_calculate_sync('Two', 2)
    exe_calculate_sync('Three', 1)

    end = time.time()

    print(f'>>> total seconds : {end - start}')

async def exe_calculate_async(name, n):
    for i in range(1, n+1):
        print(f'{name} -> {i} of {n} is calculating...')
        # time.sleep(1) # 기존 time.sleep 함수는 동기임. await을 지원하지 않음. 동기함수를 사용하게되면 전체함수가 동기적으로 변하게 됨
        await asyncio.sleep(1)
    print(f'{name} - {n} working done!')

async def process_async():
    start = time.time()

    await asyncio.wait([
        exe_calculate_async('One', 3),
        exe_calculate_async('Two', 2),
        exe_calculate_async('Three', 1),
    ])
    
    end = time.time()

    print(f'>>> total seconds : {end - start}')
if __name__ == '__main__':
    # Sync 실행
    process_sync()
    
    # Async 실행
    # 3.7이상에서는 아래와 같이 사용
    asyncio.run(process_async())
    # 이전 버전
    # asyncio.get_event_loop().run_until_complete(process_async())
```


### 예제4 I/O Bound Asyncio
- `requests`는 동기 라이브러리여서 `aiohttp` 비동기 라이브러리를 통해 구현
- `async with` 문안에서 `await`을 실행해서 비동기적으로 한꺼번에 실행

```python
import asyncio
import aiohttp
# import requests 동기 라이브러리라 사용하지 못함
import time

# 스레딩보다 높은 코드 복잡도

# 실행함수1 (다운로드)
async def request_site(session, url):    
    # session 확인
    print(session) 
    # print(session.headers)

    async with session.get(url) as response:
        print('Read Contents {0}, from {1}'.format(response.content_length, url))

# 실행함수2 (요청)
async def request_all_sites(urls):
    async with aiohttp.ClientSession() as session:
        # 작업 목록
        tasks = []
        for url in urls:
            # 태스크 목록 생성
            task = asyncio.ensure_future(request_site(session, url))
            tasks.append(task)
        
        # 태스크 확인
        print(*tasks)
        print(tasks)
        await asyncio.gather(*tasks, return_exceptions=True)

# 메인 함수
def main():
    # 테스트 URL
    urls = [
        "https://www.jython.org",
        "http://olympus.realpython.org/dice",
        "https://realpython.com"
    ] * 5

    # 실행 시간 측정
    start_time = time.time()

    # asyncio.run(request_all_sites(urls)) # 버그가 있어 이전 버전으로 실행
    asyncio.get_event_loop().run_until_complete(request_all_sites(urls))

    # 실행 시간 종료
    duration = time.time()- start_time

    print('=' * 15)
    # 결과 출력
    print(f'Downloaded {len(urls)} sites in {duration} seconds')

# 메인함수 시작
if __name__ == '__main__':
    main()
```

## 3-6 CPU Bound(1) - Synchronous

### 예제
- 300만번 loop돌면서 곱하기 하는 예제
- Sync하게 돌아가고 multiprocessing으로 바꾸기 위해 만든 예제

```python
import time

# 실행 함수1(계산)
def cpu_bound(number):
    return sum(i * i for i in range(number))

# 실행 함수2
def find_sums(numbers):
    result = []
    for number in numbers:
        result.append(cpu_bound(number))

    return result

def main():
    numbers = [3_000_000 + x for x in range(30)]

    print(numbers)

    start_time = time.time()
    # 실행
    total = find_sums(numbers)

    # 결과 출력
    print()
    print(f'Toral list: {total}')
    print(f'Sum : {sum(total)}')

    duration = time.time() - start_time

    print()
    print(f'Duraion: {duration} seconds')

if __name__ == "__main__":
    main()
```

## 3-7 CPU Bound(2) - Multiprocessing

### 예제
- 이전 예제와 다르게 시간이 1/5 가량 줄어들음
- `manager.list()` 를 통해 프로세스간 공유 리스트 생성

```python
import os
import time
from multiprocessing import current_process, Array, Manager, Process, freeze_support

# 실행 함수1(계산)
def cpu_bound(number, total_list):

    process_id = os.getpid()
    process_name = current_process().name

    # Process 정보 출력
    print(f'Process ID : {process_id}, Process Name : {process_name}')
    total_list.append(sum(i * i for i in range(number)))

def main():
    numbers = [3_000_000 + x for x in range(30)]

    print(numbers)

    start_time = time.time()
    
    # 프로세스 리스트 선언
    processes = list()
    
    # 프로세스 공유 매니저
    manager = Manager()

    # 리스트 획득(프로세스 공유)
    total_list = manager.list()

    # 프로세스 생성 및 실행
    for i in numbers: # 1 ~ 100적절히 조절
        # 생성
        t = Process(name=str(i), target=cpu_bound, args=(i, total_list,))
        # 배열에 담기
        processes.append(t)
        # 시작
        t.start()

    # Join
    for process in processes:
        process.join()
    
    # 결과 출력
    print()
    print(f'Total list: {total_list}')
    print(f'Sum : {sum(total_list)}')

    duration = time.time() - start_time

    print()
    print(f'Duraion: {duration} seconds')

if __name__ == "__main__":
    main()
```