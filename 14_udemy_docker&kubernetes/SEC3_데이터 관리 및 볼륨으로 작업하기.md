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

## 요약

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
  - host file system위치함
  - 재사용 및 공유 가능
