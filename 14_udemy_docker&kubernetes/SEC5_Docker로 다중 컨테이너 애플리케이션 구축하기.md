# SEC5 Docker로 다중 컨테이너 애플리케이션 구축하기

## 데모 프로젝트 소개

---

- 백앤드 웹 앱 - Nodes JS API
  - 로그 데이터가 삭제되면 안됨
  - 소스 코드 변경사항이 즉각반영 되어야 함
- DB - MongoDB
  - data가 삭제되면 안됨
  - 사용자를 제한할 수 있어야 함
- 프론트엔드 - React SPA
  - 소스 코드 변경사항이 즉각반영 되어야 함

## MongoDB

---

- 포트 하나 오픈해야 함

```bash
docker run --name mongodb --rm -d -p 27017:27017 mongo
docker logs mongodb
```