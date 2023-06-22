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
- 데이터가 저장되도록 `-v`를 활용해야 함
- 환경변수로 user/pw 추가하기
  - 노드앱에서 `mongodb://${process.env.MONGODB_USERNAME}:${process.env.MONGODB_PASSWORD}@mongodb:27017?authSource=admin`형식으로 바꿔줘야 함

```bash
docker logs mongodb
# docker run --name mongodb --rm -d -p 27017:27017 mongo
# docker run --name mongodb -v data:/data/db --rm -d -p 27017:27017 mongo
docker run --name mongodb -v data:/data/db \
-e MONGODB_USERNAME \
--rm -d -p 27017:27017 mongo
```

## Node app

---

- mongo ip는 `host.docker.ineternal`로 명시
- 프론트엔드에서 백엔드와 통신 실패하는 이유는, `EXPOSE 80`은 명시적인 거라 의미는 없음. 포트옵션(host.docker.internal)을 주거나 같은 네트워크(컨테이너명)에서 컨테이너를 실행해야 함
- 백엔드 폴더는 동기화(바인딩) 되도록 하고, 로그 저장 폴더랑 node_modules는 로컬이 덮어씌워지지 않도록 명시
- **소스코드가 변경되면 서버가 자동으로 재시작** 되도록 `packages.json`에 `nodemon: 2.0.4` extension을 추가하고 npm start로 바꿔야 함
- 복사하고 싶지 않은 파일은 `.dockerignore`에 추가하기 ex) node_moudles, Dockerfile, .git

```bash
docker build -t goals-node .
# docker run --name goals-backend --rm -d -p 80:80 goals-node
# docker run --name goals-backend --rm -d --network goals-net goals-node
docker run --name goals-backend \
-v {현재경로}:/app -v logs:/app/logs -v /app/node_modules \
--rm -d -p 80:80 --network goals-net goals-node \ 
```

## Frontend react spa

---

- node이미지 그대로 사용
- React 프로젝트 설정때문에 계속 상호작용하고 있지않으면 컨테이너가 자동으로 종료됨.
`-it`옵션을 통해 컨테이너를 종료되지 않도록 함
- 프론트엔드는 서버가 아닌 브라우저에서 실행됨. 그러므로 같은 네트워크를 사용해도 인식하지 못함
- 이미 react에 자동재시작 설정이 있어서 `nodemon` extenstion은 필요하지 않음
- `npm install`에서 많은 시간을 소모하고 설치된 파일을 다시 옮기는 작업이 오래걸림. .dockerignre에 추가하면 됨

```bash
docker build -t goals-react .
# docker run --name goals-frontend --rm -d -it -p 3000:3000 goals-react
docker run --name goals-frontend \
-v {절대경로}:/app/src
--rm -d -it -p 3000:3000 goals-react
```

## 데모 문제점

---

- 불편한 내용은 다음 강의에....
  - 각 컨테이너를 배포해야 하는 
  - 