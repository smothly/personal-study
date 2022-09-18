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

### 추가 자료(정리) - ![블로그](https://homoefficio.github.io/2017/02/19/Blocking-NonBlocking-Synchronous-Asynchronous/)
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

## 3-4 I/O bound, Synchronous

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

## 3-5 threading vs asyncio vs multiprocessing

---

### I/O-Bound Multithreading

### 예제
- 

```python

```
