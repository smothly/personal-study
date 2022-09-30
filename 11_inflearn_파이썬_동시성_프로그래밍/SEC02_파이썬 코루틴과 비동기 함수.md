# SEC02 파이썬 코루틴과 비동기 함수

---

## 2-1 I/O Bound & CPU Bound, Blocking

---

- CPU Bound
  - 프로그램이 실행될 때 실행 속도가 **CPU 속도**에 의해 제한됨
  - 정말 복잡한 수학 수식을 계산할 때 사용
- I/O Bound
  - 프로그램이 실행될 때 실행 속도가 **I/O**에 의해 제한됨
  - 사용자의 입력을 기다리는 것도 하나의 예시
  - 컴퓨터와 컴퓨터끼리 통신을 할 때에도 I/O Bound과 발생
- 블로킹
  - 바운드에 의해 코드가 멈추게 되는 현상이 일어나는 것



## 2-2 동기 vs 비동기

---

- 동기(Sync): 코드가 순차적 진행
- 비동기(Async): 코드의 순서가 보장되지 않음

### 예제  2개의 차이를 코드로 표현

```python
# 동기식 코드
import time

def delivery(name, mealtime):
    print(f"{name}에게 배달 완료!")
    time.sleep(mealtime)
    print(f"{name} 식사 완료, {mealtime}시간 소요...")
    print(f"{name} 그릇 수거 완료!")


def main():
    delivery("A", 5)
    delivery("B", 3)
    delivery("C", 4)


if __name__ == "__main__":
    main()
```


```python
# 비동기식 코드
import asyncio

# 코루틴 함수
async def delivery(name, mealtime):
    print(f"{name}에게 배달 완료!")
    await asyncio.sleep(mealtime)
    print(f"{name} 식사 완료, {mealtime}시간 소요...")
    print(f"{name} 그릇 수거 완료!")


async def main():
    await asyncio.gather(delivery("A", 10), delivery("B", 3), delivery("C", 4))


if __name__ == "__main__":
    asyncio.run(main())
```

## 2-3 파이썬 코루틴의 이해

---

- 메인루틴
  - 프로그램의 메인 코드의 흐름
- 서브루틴
  - 일반적인 함수나 메소드
  - 하나의 진입점과 하나의 탈출점이 있는 루틴
- 코루틴
  - 서브루틴의 일반화된 형태
  - 다양한 진입점과 다양한 탈출점이 있음
  - 파이썬 비동기 함수는 코루틴 함수로 만들 수 있음
### 예제
  - `awaitable 객체` = await 표현식에 사용할 수 있음 ex) 코루틴, 태스크, 퓨처
  - 직접 async task를 생생해서 동작시키는 것도 가능함
  - `asyncio.gather`는  awaitable객체를 한꺼번에 실행함

```python
import asyncio


async def hello_world():
    print("hello world")
    return 123


def sync_hello_world(a):
    print("hello world")
    return 123


async def sync_main():
    # async 객체 만둘기
    task1 = asyncio.create_task(sync_hello_world())
    # 실행
    await task1


if __name__ == "__main__":
    asyncio.run(hello_world())
```

## 2-4 파이썬 코루틴 활용

---

- async/await을 통한 비동기 함수 생성

### 예제 

- 동기/비동기 함수 차이
- `request`대신 `aiohttp`를 사용해야 함

```python
# 동기
import requests
import time


def fetcher(session, url):
    with session.get(url) as response:
        return response.text


def main():
    urls = ["https://naver.com", "https://google.com", "https://instagram.com"]

    with requests.Session() as session:
        result = [fetcher(session, url) for url in urls]
        print(result)


if __name__ == "__main__":
    start = time.time()
    main()
    end = time.time()
    print(end - start)

```

```python
# 비동기
import asyncio
import aiohttp
import time


async def fetcher(session, url):
    async with session.get(url) as response:
        return await response.text()


async def main():
    urls = ["https://google.com", "https://instagram.com", "https://naver.com"] * 10

    async with aiohttp.ClientSession() as session:
        result = await asyncio.gather(*[fetcher(session, url) for url in urls])
        print(result)


if __name__ == "__main__":
    start = time.time()
    asyncio.run(main())
    end = time.time()
    print(end - start)
```
- 