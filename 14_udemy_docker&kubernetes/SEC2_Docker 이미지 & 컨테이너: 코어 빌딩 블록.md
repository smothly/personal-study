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
  - 최적화를 위해선, **`npm install` 명령어는 `package.json` 파일이 변경될 때만 다시 실행**하면 되므로, COPY구문보다 앞으로 옮기고, `package.json`만 먼저 복사해주면, 소스코드가 변경될 때 마다 매번 `npm install`을 실행하지 않아도 됨

- Docker 명령어
  - `docker --help` 옵션으로 도커 명령어의 사용법을 확인할 수 있음
  - `docker start {container name}` 컨테이너 시작
    - `docker start`는 detached mode가 기본값이며, `docker run`은 attached mode가 기본값임
      - `attach-mode`는 컨테이너의 표준 입력, 출력, 오류를 터미널에 연결하는 것을 의미
      - 옵션으로는 `-a`, -`d`로 설정하면 됨
      - `docker attach` 명령어로 컨테이너에 연결할 수 있음
  - `docker stop` 컨테이너 중지
  - `docker logs` 컨테이너의 로그 확인
    - `-f` 옵션으로 로그를 실시간으로 확인할 수 있음
  - `docker run -it {container id}`
    - `attach`만 할 경우 출력은 받을 수 있지만 입력을 따로 할 수 없음
    - `-i` interactive 모드
    - `-t` pseudo-tty 모드. 터미널 생성
  - `docker rm {container id or name}` 컨테이너 삭제
    - stop된 것들만 삭제 가능
    - container id or name은 여러개 받음
    - `docker container prune`으로 중지된 모든 컨테이너 삭제 가능
  - `docker rmi {image id}` 이미지의 모든 레이어 삭제
    - `docker image prune`으로 사용되지 않는 모든 이미지 제거
  - `--rm`옵션으로 컨테이너가 중지됐을 때 바로 제거하기
  - `docker image inspect {image id}` 이미지의 정보들을 알 수 있음
    - 생성 시간, OS, 레이어 구성 등
  - `docker cp {source path} {target path}`
    - ex) `docker cp dummy/. container_name:/test`
    - 컨테이너로부터 파일 및 데이터를 in/out할 수 있음.
    - 이미지를 직접 빌드하지 않고 컨테이너에 반영할 수 있음. 하지만 좋은 방법은 아님
  - 컨테이너와 이미지에 이름을 지정
    - `docker build -t name:tag`
    - `docker run -p 3000:80 -d --rm --name container_name image_name:tag`
  
- 이미지 공유하기
  - Dockerfile이 포함된 폴더를 `zip`파일로 공유
  - 빌드된 이미지를 공유하기
- Docker hub 활용하기
  - 공식 Docker Image Registry
  - `push` `pull`명령어로 이미지를 개시하고 다운받을 수 있음
  - Docker hub에서 레포지토리를 만들고, `docker push {repository_name/image_name}:tag`로 업로드
  - `docker pull {repository_name/image_name}:tag`로 다운받을 수 있음
    - 다운받아서 실행할려면 `docker run -p 8000:3000 {repository_name/image_name}`으로 실행하면 됨
    - 최신이미지를 최초에는 가져오나 로컬로 
    - 가져온 후에는 이미지를 따로 업데이트 하지않음. docker hub이미지의 최신사항을 반영할려면 Pull받아서 사용해야함
