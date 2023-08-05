# SEC6 Docker Compse: 우아한 다중 컨테이너 오케스트레이션

## 지난 모듈 문제점

---

- 설정을 모드 도커 명령어로 실행했어야 함
- 네트워크를 만들어야 함
- 제거할려면 네트워크나 볼륨들도 별도로 삭제해야 함

## Docker Compose

---

- **다중 컨테이너를 더 쉽게 관리해줌**
- 여러개의 `docker build` `docker run` 을 **하나의 구성파일**로 가짐
- 다수의 호스트에서 다중 컨테이너를 관리하는데 적합하지 않음. **하나의 동일한 호스트에서 다중 컨테이너를 관리**할 때 좋음
- 서비스는 여러개의 컨테이너로 구성되고 터미널에서 수행했던 환경설정들을 하나의 파일에서 할 수 있음
- Dockerfile자체를 대체하지 않음
- 요약
  - 복잡한 프로젝트를 긴 명령어 없이 쉽게 구성할 수 있음
  - conf파일 하나로 관리각 용이하기 때문에 단일 컨테이너 설정에도 사용 됨
  - docker file을 대체하지않고 run과 build를 부분적으로만 대체함

## Compose 실습

- `docker-compose.yml` 파일 생성
- 속성
  - version: 도커컴포즈 버전 (내가 느끼기엔 태그와 약간 비슷)
  - servieces: 컨테이너를 하위요소로 가짐
- 최종 변환
  - 네트워크는 선언하지 않을 경우 동일한 네트워크로 묶임
- 

```yaml
version: "3.8"
services:
  mongodb:
    image: 'mongo'
    volumes: 
      - data:/data/db
    # environment: 
    #   MONGO_INITDB_ROOT_USERNAME: max
    #   MONGO_INITDB_ROOT_PASSWORD: secret
      # - MONGO_INITDB_ROOT_USERNAME=max
    env_file: 
      - ./env/mongo.env
  backend:
    build: ./backend
    # build: 위의 구문과 같은 문자
    #   context: ./backend
    #   dockerfile: Dockerfile
    #   args:
    #     some-arg: 1
    ports:
      - '80:80'
    volumes: 
      - logs:/app/logs
      - ./backend:/app # 절대 경로 사용하지 않아도 됨
      - /app/node_modules
    env_file: 
      - ./env/backend.env
    depends_on:
      - mongodb
  frontend:
    build: ./frontend
    ports: 
      - '3000:3000'
    volumes: 
      - ./frontend/src:/app/src
    stdin_open: true
    tty: true # -it 옵션과 같음
    depends_on: 
      - backend

volumes: 
  data:
  logs:
```

```bash
docker-compose up -d
docker-compose down
```

- `--build`옵션을 추가하면 항상 이미지를 리빌드 함
- `-v`옵션을 하면 볼륨도 같이 삭제됨. 하지만 추천하지 않음
- 컨테이너 이름을 자동으로 생성해줌. `container_name`통해 지정도 가능

