# SEC04 동시성 프로그래밍으로 데이터 수집

---

## 4-1 서버와 클라이언트, HTTP, API 이해

---

- ![웹 동작 원리](https://velog.velcdn.com/images%2Fwonhee010%2Fpost%2F12a999e9-21bd-4b10-834e-6c8a2b553851%2Fimage.png)
- HTTP: 프로토콜 get, post 메소드 사용
- API: 코드를 몰라도 언터페이스를 만들어 통신



## 4-2 웹 크롤링, 스크래핑과 법적 주의사항

---

- `robots.txt` 파일을 확인해서 크롤링 가능한지 보기
- `bs4` 기본적인 사용법

## 4-3 동시성 프로그래밍으로 웹 크롤링, 스크래핑 성능 극대화

---

### 예제
- bs4로 `title`만 뽑아 출력하기
- `async`를 활용해서 더 빠른 데이터 수집

```python
from bs4 import BeautifulSoup
import aiohttp
import asyncio


async def fetch(session, url):
    async with session.get(url) as response:
        html = await response.text()
        soup = BeautifulSoup(html, "html.parser")
        cont_thumb = soup.find_all("div", "cont_thumb")
        print(cont_thumb)
        for cont in cont_thumb:
            title = cont.find("p", "txt_thumb")
            if title is not None:
                print(title.text)


async def main():
    BASE_URL = "https://bjpublic.tistory.com/category/%EC%A0%84%EC%B2%B4%20%EC%B6%9C%EA%B0%84%20%EB%8F%84%EC%84%9C"
    urls = [f"{BASE_URL}?page={i}" for i in range(1, 10)]
    async with aiohttp.ClientSession() as session:
        await asyncio.gather(*[fetch(session, url) for url in urls])


if __name__ == "__main__":
    asyncio.run(main())

```
## 4-4 포스트맨 셋업, 네이버, 카카오 오픈 API 사용하기

- API 도구 포스트맨 설치
- 네이버/카카오 API 인증 및 쿼리 파라미터 요청 확인

---

## 4-5 오픈 API를 활용한 이미지 데이터 수집

---

### 예제
- `secrets.json`에 API KEY같은 것들을 관리
- start 파라미터는 page number가 아닌 object number여서 20개씩 조회하기위해 수정함

```python
import aiohttp
import asyncio
from config import get_secret


async def fetch(session, url, i):
    print(i + 1)
    headers = {
        "X-Naver-Client-Id": get_secret("NAVER_API_ID"),
        "X-Naver-Client-Secret": get_secret("NAVER_API_SECRET"),
    }
    async with session.get(url, headers=headers) as response:
        result = await response.json()
        items = result["items"]
        images = [item["link"] for item in items]
        print(images)


async def main():
    BASE_URL = "https://openapi.naver.com/v1/search/image"
    keyword = "cat"
    urls = [f"{BASE_URL}?query={keyword}&display=20&start={1+ i*20}" for i in range(10)]
    async with aiohttp.ClientSession() as session:
        await asyncio.gather(*[fetch(session, url, i) for i, url in enumerate(urls)])


if __name__ == "__main__":
    asyncio.run(main())
```

## 4-6 동시성 프로그래밍으로 이미지 다운로더 개발(feat.aiofiles)

---

### 예제
- `aiofiles`를 통해서 파일스기도 비동기로 구현
- 

```python
import os
import aiohttp
import asyncio
from config import get_secret
import aiofiles

# pip install aiofiles==0.7.0


async def img_downloader(session, img):
    img_name = img.split("/")[-1].split("?")[0]

    try:
        os.mkdir("./images")
    except FileExistsError:
        pass

    async with session.get(img) as response:
        if response.status == 200:
            async with aiofiles.open(f"./images/{img_name}", mode="wb") as file:
                img_data = await response.read()
                await file.write(img_data)


async def fetch(session, url, i):
    print(i + 1)
    headers = {
        "X-Naver-Client-Id": get_secret("NAVER_API_ID"),
        "X-Naver-Client-Secret": get_secret("NAVER_API_SECRET"),
    }
    async with session.get(url, headers=headers) as response:
        result = await response.json()
        items = result["items"]
        images = [item["link"] for item in items]
        await asyncio.gather(*[img_downloader(session, img) for img in images])


async def main():
    BASE_URL = "https://openapi.naver.com/v1/search/image"
    keyword = "cat"
    urls = [f"{BASE_URL}?query={keyword}&display=20&start={1+ i*20}" for i in range(10)]
    async with aiohttp.ClientSession() as session:
        await asyncio.gather(*[fetch(session, url, i) for i, url in enumerate(urls)])


if __name__ == "__main__":
    asyncio.run(main())
```