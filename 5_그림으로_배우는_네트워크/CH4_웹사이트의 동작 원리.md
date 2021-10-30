# 웹사이트의 동작 원리

---

## 4-1 웹사이트
- 웹사이트란 웹서버 어플리케이션이 공개하는 다양한 웹페이지의 집합
- 웹 페이지를 보여주는 과정
  1. 웹브라우저에서 웹서버 어플리케이션에 파일 전송 요청을 보냄
  2. 웹서버 어플리케이션은 요청받은 파일을 응답으로 돌려보냄
  3. 웹브라우저에서 수신한 파일을 표시
- 웹사이트에 접속에 이용하는 프로토콜의 조합읜 똑같다.
- ![응답예시](https://blog.kakaocdn.net/dn/mFgvh/btqMPiNuADf/8xS3pT40500B288GS9JvI1/img.png)

---

## 4-2 HTML
- 웹페이지는 HTML 파일로 만들어짐
  - Hyper Text Marup Launguage
  - 마크업 언어로서 문서의 구조를 명확히 표현하기 위한 언어
  - 문서의 제목, 헤드라인, 단락, 리스트 등의 구조를 명화히 함으로써, 컴퓨터로 문장 구조 분석을 간편하게 할 수 있음
- HTML 태그
  - 외관을 지정
  - 시작태그<>와 종료태그</>로 요소를 감싸줌. 이를 마크업이라고 표현
- ![HTML예시](https://upload.wikimedia.org/wikipedia/commons/thumb/8/84/HTML.svg/1200px-HTML.svg.png)

--- 

## 4-3 스타일시트
- 외관도 중요
  - HTML 태그내에서도 관리가 가능
  - but, 다수의 HTML 페이지를 매번 수작업하기는 어려워 스타일 시트로 따로 정의하여 관리
- 스타일 시트
  - 스타일 시트란 문서의 레이아웃, 폰트와 색, 디자인을 정의하는 방법
  - CSS(Cascading Syyle Sheets)
  - HTML과 별도로 관리하여 작성
- ![CSS예시](https://miro.medium.com/max/1838/0*du8LEJst4rlIqQ1u.png)

---

## 4-4 URL
- 웹사이트의 주소
  - 웹사이트는 HTML 파일로 작성한 웹페이지의 집합
  - 전송받고 싶은 웹페이지를 지정하는 것이 웹사이트의 주소
- URL의 의미
  - Uniform Resource Locator
  - http는 스킴으로 접속하기 위한 프로토콜
  - <스킴>://<호스트명>/<경로명>
- ![URL예시](https://lh3.googleusercontent.com/proxy/zvH8oIuA3Txg7LL97CTGypR6c19IE7pNBSY_k8Xfn1qHzzGdgKCZpUkciymuiOhJRSmoM7ksFJ2eLtpK)

---

## 4-5 HTTP
- HTTP 파일 전송
  - 파일 전송은 HTTP request & HTTP response로 이루어짐
  - HTTP 통신을 하기전에 TCP 커넥션을 맺음
- HTTP 리퀘스트
  - 리퀘스트 라인 / 메시지 헤더 /엔티티 바디 세부분으로 나뉨
    - 리퀘스트 라인: 실제 처리요청을 전달 메소드(GET/POST 등), URI, 버전으로 구성
    - 메시지 헤더: 웹브라우저의 종류와 버전 대응하는 데이터 형식등의 정보를 기술
    - 엔티티 바디: POST메소드로 데이터를 보낼때 사용