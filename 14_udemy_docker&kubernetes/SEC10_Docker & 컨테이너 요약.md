# SEC10 Docker & 컨테이너 요약

---

## 요약

---

- 도커 핵심 개념
  - 이미지  
    - Blueprints for Containers
    - Code + Environment
    - Read-only 파일로 이미지 자체가 실행되지 않음
  - 컨테이너
    - 코드와 그 코드를 실행하는 런타임 환경을 포함하는 박스 ex) 웹 서버, 프론트엔드, DB
    - isolated
    - single task focused
    - sahreable, reproducible
    - stateless
  - 볼륨
    - 바인드마운트 or 볼륨추가
    - named volume으로 볼륨 데이터 유지 가능
    - anonymous volume은 컨테이너가 삭제되면 데이터도 삭제됨
  - 네트워크
    - 컨테이너간 통신을 위해서는 Docker 네트워크를 사용하는 것을 권장
- 도커 명령어
  
  ```bash
  docker build -t NAME[:TAG] .
  docker run --name NAME --rm -d IMAGE
  docker push/pull REPOSITORY/NAME[:TAG]
  ```

- Docker Compose
  - 여러 컨테이너를 하나의 서비스로 정의하여 관리
  - `docker-compose.yml` 파일을 사용
  - `docker-compose up`으로 실행
- 도커 활용
  - Localhost에서 개발로 사용하는 것만으로도 활용가치가 있음
    - isolated, encapsulated, reproducible environment
    - no dependency or software clashes
  - Production에서는 localhsot에 있는 장점 뿐만아니라 update하기 쉬운 장점도 있음
- 배포시 고려사항
  - bind mounts를 사용하지 않고 COPY를 사용
  - 멀티 컨테이너는 여러 호스트를 필요로 함. 같은 호스트에서 실행시킬 수도 있음
  - 멀티 스테이지 빌드도 있음
