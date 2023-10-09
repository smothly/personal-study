# SEC7 '유틸리티 컨테이너'로 작업하기 & 컨테이너에서 명령 실행하기

- `유틸리티 컨테이너`는 특정 환경만 포함하는 컨테이너를 의미

## 유틸리티 컨테이너

---

- `npm init`명령어를 수행하면 `package.json` 을 자동으로 생성해줌. 하지만, node를 설치해야 실행 가능함
- 호스트 머신에 설치 없이 유틸리티 컨테이너를 사용하기
- 컨테이너에서 명령을 실행하는 다양한 방법
  - `-it`옵션으로 시작
  - dettach 모드일 경우 `docker exec -it {container_name} npm init` 처럼 수행하면 됨
  - `docker run -it node npm init` 디폴트 명령이 override됨
- `docker run -it -v 현재 절대 경로:/app node-util npm init`
  - node이미지만 만들어놓고 명령어는 따로 수행
- `ENTRYPOINT`
  - `CMD`와 거의 유사하나 차이점은 ENTRYPOINT가 Prefix로 들어감
  - ex) ENTRYPOINT ["npm"] + (npm) install express --save
  - 컨테이너를 보호하기위한 수단으로도 활용할 수 있음
- `Docker Compose`를 사용하여 관리하기 더 쉽게 하기
  
  ```Dockerfile
  version: "3.8"
  services:
    npm:
      build: ./
      stdin_open: true
      tty: true
      volumes:
        - ./:/app
  ```

  - `docker-compose run --rm npm(서비스 이름) init`
- 권한
  - docker 명령으로 생성되는 것은 기본적으로 root 권한을 가짐
  - `USER node` + `RUN groupadd --gid 1000 node && useradd --uid 1000 --gid node --shell /bin/bash --create-home node`처럼 node사용자로 실행되게할 수 있음

  ```
  FROM node:14-slim

RUN userdel -r node

ARG USER_ID

ARG GROUP_ID

RUN addgroup --gid $GROUP_ID user

RUN adduser --disabled-password --gecos '' --uid $USER_ID --gid $GROUP_ID user

USER user

WORKDIR /app
  ```

