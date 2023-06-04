# SEC3 데이터 관리 및 볼륨으로 작업하기

## 데이터 종류

---

- Application(Core + Environment)
  - 사용자가 작성한 파일
  - 이미지가 빌드된 이후에는 수정하지 못함
- Temporary App Data
  - 컨테이너 내에서 생산된 임시 데이터
  - ex) 입력폼에 입력한 임시 데이터
- Permanent App Data
  - 컨테이너가 삭제되더라고 유지되어야 하는 데이터
  - `Volumes`에 저장할 수 있음
  - ex) 사용자 계정

## 앱으로 3가지 데이터 종류 구현

---

- ![docker layer](https://www.partech.nl/publication-image/%7BF7AA82CF-A27D-4031-9695-0464A7642295%7D)
- 컨테이너 중지(docker stop) -> 재시작하면 파일 존재
- 컨테이너 제거(--rm 옵션) -> 파일 사라짐
- 컨테이너가 사라지면 데이터가 사라지는 문제가 발생함

## Docker volume

---

- Volume: 호스트 머신 폴더
  - ![docker volume](https://blog.kakaocdn.net/dn/FxUIW/btruWkhEWvW/nAasmW0lkZFtUM0CjqX5pk/img.png)
  - docker container 안쪽으로 마운트 됨
  - COPY와 비슷하지만, copy는 일회성 스냅샷이고 **volume은 지속적인 연결**임
- 2가지 외부 데이터 저장
  - volume
    - annonymous volumes
      - 도커가 관리하므로 **컨테이너가 살아있을 때만 존재**
      - `Dockerfile`에 `VOLUME ["/app/feedback"]` 추가하기: annonumous volumes여서 컨테이너가 삭제되면 볼륨도 지워짐
    - **named volumes**
      - **컨테이너가 종료되어도 볼륨 유지**
      - `docker run -d -p 3000:80 --rm --name feedback-app -v feedback:/app/feedback feedback-node:volumes`
  - bind mounts
    - 볼륨과 비슷하지만, 호스트 머신상에 매핑될 컨테이너의 경로를 설정함
    - **편집 가능한 데이터에 적합**함
    - `docker run -d -p 3000:80 --rm --name feedback-app -v feedback:/app/feedback -v {로컬 절대경로}:/app feedback-node:volumes`
      - 이렇게 하면 /app폴더를 덮어쓰기 때문에 `npm install`을 실행하지 못함. 
      - 아래 명령어처럼 **annonymous volume추가해서 해결**하기
        - `docker run -d -p 3000:80 --rm --name feedback-app -v feedback:/app/feedback -v {로컬 절대경로}:/app -v /app/node_modules feedback-node:volumes`
    - 만약 마운트가 되지 않는다면, `preference > resources > file sharing`에서 허용된지 확인
- 제거
  - 컨테이너가 시작될때마다 새 익명볼륨이 생성되기 때문에 정리가 필요함
  - `docker volume rm {vol_name}`
  - `docker volume prune`
- nodemon 패키지 추가
  - 기존에는 `console.log`를 찍어도, 컨테이너를 재시작하기 전까지 나오지 않음
  - 파일이 변경될 때마다 노드 서버를 다시 시작
  - `package.json`에 아래 추가

  ```json
  "scripts": {
    "start": "nodemon server.js"
  },
  "dependencies": {
    "body-parser": "^1.19.0",
    "express": "^4.17.1"
  },
  "devDependencies": {
    "nodemon": "2.0.4"
  }
  ```

## 볼륨 요약

---

- annonymous volume
  - 컨테이너가 내부에 모든 데이터를 저장과 관리할 필요가 없으므로 성능과 효율성에 도움이 됨
  - 컨테이너끼리 공유할 수 없음
  - 재사용할 수 없음
- named volume
  - `:` 앞에 볼륨명을 명시
  - 컨테이너가 종료해도 살아 있음
  - 다른 컨테이너와 새 컨테이너에 attach/detach할 수 있음
  - **도커 영역안에서 관리**
- bind mounts
  - host file system에 위치함
  - 컨테이너에서 항상 최신의 데이터를 사용할 수 있음
  - 재사용 및 공유 가능

## 읽기 전용 볼륨

---

- 변경이 되면 안되는 볼륨이 있을 때 유용
- `ro`옵션을 추가하면 됨
  - `docker run -d -p 3000:80 --rm --name feedback-app -v feedback:/app/feedback feedback-node:volumes:ro`

## Docker 볼륨 관리하기

---

- `docker volume ls`
  - 바인드 마운트는 표시되지 않음
  - named volume은 볼륨은 볼륨이 없을경우 자동으로 생성됨
- `docker volume inspect {volume}`
  - 볼륨생성 위치, 날짜, 옵션 등 메타정보를 알 수 있음
- `docker volume rm {volume}`
  - 컨테이너에서 사용중이지 않은 볼륨만 제거해야 함

## Copy vs Bind mount

---

- 코드의 스냅샷을 유지하고 싶으면 `copy` 사용
  - `.dockerignore`파일로 copy 제외항목 정할 수 있음
  - node_modules는 npm install로 설치가 되니, COPY할때 덮어씌우지 않도록 추가하면 됨
- 개발환경을 지속적으로 반영하고 싶으면 `bind`하여 사용

## Argument & Envirionment

---

- arg
  - `--build-arg` 옵션
  - `Dockerfile`내에 명시
    - Dockerfile내에서만 사용 가능
    - 컨테이너가 시작될 때 실행되는 런타임 명령이기 때문에 `CMD`에서 사용할 수 없음
  
  ```Dockerfile
  ARG DEFAULT_PORT 80
  EXPOSE $DEFAULT_PORT
  ```

- env
  - `--env` 옵션
    - `--e PORT=8000`
  - `Dockerfile`내에 명시
  
  ```Dockerfile
  ENV PORT 80
  EXPOSE $PORT
  ```

  - `.env`파일을 만들어서 `--env-file ./.env` 옵션으로도 명시 가능
  - 보안 데이터는 파일로 관리하고 `.gitignore`에 추가되도록 하기

- arg와 env를 배치할 때 레이어가 추가되지 않도록 `npm install` 다음에 추가하기
