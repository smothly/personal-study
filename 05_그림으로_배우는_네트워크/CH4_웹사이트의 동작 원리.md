# 웹사이트의 동작 원리

---

## 4-1 웹사이트

- 웹사이트란 웹서버 어플리케이션이 공개하는 다양한 웹페이지의 집합
- 웹 페이지를 보여주는 과정
  1. 웹브라우저에서 웹서버 어플리케이션에 파일 전송 요청을 보냄
  2. 웹서버 어플리케이션은 요청받은 파일을 응답으로 돌려보냄
  3. 웹브라우저에서 수신한 파일을 표시
- 웹사이트에 접속에 이용하는 프로토콜의 조합은 똑같다.
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
- ![URL예시](https://www.beusable.net/blog/wp-content/uploads/2021/02/image-7.png)

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

---

## 4-6 HTTP 리스폰스

- HTTP 리스폰스
  - 리스폰스 라인, 메시지 헤더, 엔티티 바디로 구성
  - 리스폰스 라인은 다시 버전 / 상태코드 / 설명문으로 나뉨
  - ![response 구조](https://media.vlpt.us/post-images/rosewwross/6fc65770-4b39-11ea-abce-67c155f8f58a/image.png)
  - 상태코드는 200 / 404 가 주로 보임.
  - 웹브라우저에 돌려내는 파일은 주로 HTML

---

## 4-7 HTTP 쿠키

- 웹페이지의 내용을 커스터마이징하고 싶을 때는 HTTP 쿠키를 이용
- 쿠키
  - 웹서버 어플리케이션이 웹브라우저에서 특정 정보를 저장해 두는 기술
  - 리스폰스에 쿠키를 HTTP 헤더에 포함하여 전달함
  - 웹브라우저는 수신한 쿠키를 저장하며 같은 웹사이트에 접속할 때 HTTP 리퀘스트에 쿠키를 포함
    - 로그인 정보나 사이트 내 웹페이지 열람 이력을 관리할 수 있음
    - 웹페이지를 개인화할 수 있음

---

## 4-8 프록시 서버

- 웹 접속을 대신하는 서버
  - 웹 브라우저와 웹서버 어플리케이션이 서로 통신할 때 **프록시 서버**를 거처가는 경우가 있음
  - 클라이언트 측에서는 프록시 서버를 사용하는지 알 수 없음
  - ![프록시 서버](https://marvel-b1-cdn.bc0a.com/f00000000216283/www.fortinet.com/content/fortinet-com/en_us/resources/cyberglossary/proxy-server/_jcr_content/par/c05_container_copy_c/par/c28_image_copy_copy_.img.jpg/1625683502431.jpg)

---

## 4-9 프록시 서버의 목적

- 목적
  - 기업 네트워크 관리자 입장
    - 클라이언트의 웹브라우저에서 접속하는 웹사이트를 확인
    - 부정한 웹사이트를 URL 필터링을 통하여 접속을 제한

---

## 4-10 웹 어플리케이션

- 웹브라우저는 단순히 웹사이트만 보는 것이 아니라 유저 인터페이스로도 널리 이용
- 웹브라우저를 유저 인터페이스로 이용하는 어플리케이션을 **웹 어플리케이션**이라고 함
  - 클라이언트 전용 어플리케이션을 설치할 필요가 없어짐
  - ![WAS](https://s3-ap-northeast-2.amazonaws.com/opentutorials-user-file/module/4074/11299.png)

---

## 4-11 웹 접속 시의 어플리케이션과 프로토콜

- 어플리케이션
  - 웹브라우저는 chrome, edge, firefox 등이 있음
  - 프록시 서버를 이용할 때는 프록시 서버의 IP 주소와 포트 번호를 설정해야함
  - 웹 서버는 **웹서버 어플리케이션**이 필요, 주로 Apache나 IIS nginx
    - WAS는 공개할 웹사이트의 파일이 있는 디렉터리 등을 설정
- 프로토콜
  - HTTP/TCP/IP/이더넷 를 사용
    - ![인터넷 접속에 사용하는 프로토콜](https://cdn.kastatic.org/ka-perseus-images/6a0cd3a5b7e709c2f637c959ba98705ad21e4e3c.svg)
  - DNS(IP주소 알아냄)와 ARP(MAC주소 알아냄)를 통해 이름을 자동으로 해석함

---

## 4-12 DNS 이름해석, HTTP 리퀘스트와 HTTP 리스폰스

- 웹사이트 볼 때의 동작
  - request/response 전에 **DNS 이름해석과 ARP의 주소해석 기능**이 동작
  - 그 전에 TCP 커넥션도 맺음
  - 흐름
    1. URL을 입력
    2. URL을 DNS서버에 질의해 웹서버의 IP 주소를 해석
         - 이 과정에서 ARP도 동작하여 MAC주소를 구함
    3. TCP/IP에서 응답받은 IP주소를 지정하여 TCP 커넥션을 맺음
    4. HTTP request/response를 주고 받음
    5. TCP에서 복수로 분할된 웹페이지의 파일을 조립하여 웹브라우저에서 내용을 표시
