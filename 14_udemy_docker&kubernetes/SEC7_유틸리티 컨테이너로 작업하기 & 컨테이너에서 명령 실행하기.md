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

---

- 