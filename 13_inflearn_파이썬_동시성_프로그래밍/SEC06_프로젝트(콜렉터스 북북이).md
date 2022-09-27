# SEC06 실전 프로젝트: "콜렉터스 북북이"(FastAPI)

---

## 6-1 프로젝트 소개

---

- 100개의 책정보를 수집해서 보여주는 홈페이지
- `FastAPI` + `Jinja Template` 사용

## 6-2 비동기 게이트웨이 ASGI, Uvicorn

---

- ASGI
  - `WSGI`를 개선한게 `ASGI`
  - Web Server Gateway Interface로 만들어졌지만 비동기를 위함
  - 서버와 어플리케이션을 연결해줌
- Uvicorn
  - ASGI 서버의 구현체

## 6-3 FastAPI Tutorial: Hello FastAPI!

---

### 예제
- 동적라우팅을 통해 url 구성 가능
- `Optional`은 말 그대로 선택항목

```bash
pip install fastapi
pip install uvicorn

uvicorn app.main:app --reload
```

```python
from typing import Optional
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def read_root():
    print("Hello World")
    return {"message": "Hello World"}


@app.get("/hello")
def read_fastapi_hello():
    print("Hello World")
    return {"Hello": "FastAPI"}


@app.get("/items/{item_id}/{xyz}")
def read_item(item_id: int, xyz: str, q: Optional[str] = None):

    return {"item_id": item_id, "q": q, "xyz": xyz}
```

## 6-4 FastAPI Tutorial: Jinja 템플릿 엔진

---

### 예제
- template 경로 설정
- `Jinja Template`으로 response를 보내기
- `TemplateResponse`에 `request` context는 필수적임

```bash
pip install jinja2
pip install aiofiles
```

```python
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse
from fastapi.templating import Jinja2Templates
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent

app = FastAPI()
templates = Jinja2Templates(directory=BASE_DIR / "templates")


@app.get("/items/{id}", response_class=HTMLResponse)
async def read_item(request: Request, id: str):
    return templates.TemplateResponse("item.html", {"request": request, "id": id, "data": "hello fastapi"})
```

## 6-5 템플릿 추가 설명, 프로젝트 셋업

---

- 이제 코드는 길어져서 최종 코드는 [강의 깃헙](https://github.com/amamov/teaching-async-python/tree/main/6-%EC%8B%A4%EC%A0%84-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%EC%BD%9C%EB%A0%89%ED%84%B0%EC%8A%A4)으로 대체
- css 적용
- `uvicorn`을 `server.py`에서 실행하도록 설정
- `secret.json` 가져오는 `config.py` 생성
- 요청 받은 파라미터들을 사이트에 보여지도록 변경


## 6-6 FastAPI + MongoDB: MongoDB ODM 셋업

---

- 시크릿 변수 설정
  - mongo db name
  - mongo db url
  - naver api id
  - naver api secret
  - .gitignore에 등록
- odmantic을 사용하여 fastapi와 연결
  - Object Document Mapper
  - 파이썬과 MongoDB를 연결하기 위함
  - RDB는 ORM을 사용
- `event`를 받아 mongodb와 연결을 자동화함
- models 디렉토리를 사용하여 추상화
  - `__init__.py`에서 mongo db 연결 코드 생성
- book 모델 개발
- db에 insert

## 6-7 책 데이터 수집 클래스 개발

---

- naver book search하는 로직 만들기
  - `book_scraper.py`를 만들기
  - async로 구현
  - return값을 하나의 리스트로 만듬

## 6-8 서비스 로직 개발

---

- 웹 페이지에서 search하면 book 정보를 가져와 DB에 저장
  - `saveall` 메소드로 네이버 API로 가져온 book 객체 한꺼번에 저장

## 6-9 프로젝트 마무리

---

- DB에 저장된 데이터를 보여주기
  - 템플릿 코드 생성
- 키워드의 중복현상 해결
  - 검색어가 없다면 입력해달라는 문구 띄우기
  - 이미 조회된 키워드면 기존 DB에 있는 항목들 보여주기

# SEC07 AWS에 프로젝트 올리기

## 배포
- vps인 lightsail 사용
- github에 코드 올리기
- 서버에서 코드 받기
- requirements.txt 라이브러리 설치
- server.py 실행