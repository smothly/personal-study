# SEC2 Docker 이미지 & 컨테이너: 코어 빌딩 블록

## Images vs Containers

---

- 컨테이너
  - 어플리케이션, 웹, 노드 서버, 어플리케이션 환경 등 무엇이든 **포함하는 작은 패키지**
  - Unit of Software
  - 실행 애플리케이션
- 이미지
  - 컨테이너를 만들기 위한 **템플릿**으로, 코드와 코드를 실행하는 도구들을 포함
  - 블루프린트로서 코드와 애플리케이션을 포함
  - 이미지를 기반으로 컨테이너를 실행

## 외부 이미지 사용

---

- Docker Hub 활용
  - Docker run node하면 Docker hub에서 최신의 공식 이미지를 다운로드함
- 컨테이너 확인
  - `docker ps -a`
  - `docker run -it node`

## NodeJS 앱

---

- 로컬에서 NodeJS 어플리케이션 실행
- Dockerfile 구축

  ```Docker
  FROM node # 이미지 이름. 로컬 머신에 이미지가 없으면 Docker Hub에서 다운로드

  WORKDIR /app # 작업 디렉토리 설정

  COPY . /app # 현재 디렉토리의 모든 파일을 작업 디렉토리로 복사

  RUN npm install # 노드 어플리케이션의 모든 종속성 설치

  EXPOSE 80 # 컨테이너가 외부에 노출할 포트로, 도커는 로컬과 격리되어 있어 자체 네트워크를 가지고 있음. 명시적인 옵션으로 실제 실행할때는 -p 옵션으로 포트를 지정해야 함

  CMD ["node", "server.js"] # 컨테이너가 시작되면 실행할 명령어
  # RUN node server.js는 이미지가 빌드할 때 마다 실행되고, CMD는 컨테이너가 시작될 때 마다 실행됨
  ```

- 이미지 빌드 및 컨테이너 실행

  ```bash
  docker build .
  docker run -p 3000:80 {docker image id}
  docker ps
  docker stop {container name}
  ```

- 코드 변경사항 반영
  - 컨테이너를 다시시작해도 코드 변경사항이 반영되지 않음
  - 업데이트된 코드를 반영하기 위해서는 **이미지를 다시 빌드**해야 함
  - 중요한 것은, **이미지가 읽기 전용**이라는 정보

- 이미지 레이어
  - `Using Cache` 메시지 처럼 도커는 **모든 명령 결과를 캐시**하고 이미지를 다시 빌드할 때 **명령을 다시 실행할 필요가 없으면 이러한 캐시된 결과**를 사용함
  - 파일이 변경됨을 감지하면, 그 파일을 포함한 레이어 이후의 모든 레이어를 다시 빌드함
  - **`npm install` 명령어는 `package.json` 파일이 변경될 때만 다시 실행**하면 되므로, COPY구문보다 앞으로 옮기고, `package.json`만 먼저 복사해주면, 소스코드가 변경될 때 마다 매번 `npm install`을 실행하지 않아도 됨