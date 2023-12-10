# SEC8_더 복잡한 설정: Laraverl&PHP 도커화 프로젝트

---

## 프로젝트 설명

- Laravel과 PHP는 로컬에서할 경우 복잡한 설정을 하기 때문에 예제로 가져왔음
  - NodeJS는 javascript런타임으로서 js로 모든 어플리케이션 코드를 작성하고 NodeJS만 설치하면 됨
  - 3개의 앱 컨테이너가 필요
    - PHP Interpreter
    - Nginx Web server
    - MySQL database
  - 3개의 유틸리티 컨테이너가 필요
    - Composer - 패키지설치
    - Laravel Artisian - DB초기설정같은걸 해주는 도구
    - npm - js코드가 필요한 경우

## nginx 웹서버 추가

---


```dockerfile
version: "3.8"

services: 
  server:
    image: 'nginx:stable-alpine'
    ports: 
      - '8000:80'
    volumes: 
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
```

- 컨테이너의 경로는 공식문서보고 설정해야 함
- `ro`는 readonly 옵션임

## PHP 컨테이너 추가

---

- `/var/www/html` 경로가 이 프로젝트의 기본 폴더경로임
- `delegated`는 컨테이너에서 파일을 수정할 경우 배치처리로 호스트머신에 반영하는 형태로 성능이 약간 더 나아짐

```Dockerfile
  php:
    build:
      context: .
      dockerfile: dockerfiles/php.dockerfile
    volumes: 
      - ./src:/var/www/html:delegated
```

- Docker service명 php를 nginx conf에 사용할 수 있음
- Php의 기본 포트가 9000이므로 변경
- local host통신이 아니라 docker network를 통해 통신함

```conf
server {
    listen 80;
    index index.php index.html;
    server_name localhost;
    root /var/www/html/public;
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

## mysql 컨테이너 추가

---

- 컨테이너끼리 docker-network에 있으므로 따로 네트워크 설정해줄 필요가 없음
- `env`에 db설정을 넣어줌

```Dockerfile
  mysql:
    image: mysql:5.7
    env_file: 
      - ./env/mysql.env
```

## composer 컨테이너 추가

---

- 유틸리티 컨테이너로 어플리케이션 컨테이너들을 실행시키기 위한 환경임
- composer를 따로 Dockerfile로 구성한 이유는 entrypoint를 설정하기 위함임
  - 파일명: composer.dockerfile
  - `--ignore-platform-reqs`는 일부 종속성이 누락되더라도 경고나 오류없이 실행하는 옵션임

```Dockerfile
FROM composer:latest

WORKDIR /var/www/html

ENTRYPOINT [ "composer", "--ignore-platform-reqs" ]
```

```Dockerfile
  composer:
    build:
      context: ./dockerfiles
      dockerfile: composer.dockerfile
    volumes: 
      - ./src:/var/www/html      
```

## 실행해보기

---

- `docker-compose run --rm composer create-propject --prefer-dist laravel/laravel .`로 실행
- 서버 - PHP와 연결을 위해 volume(.src:/var/www/html)을 추가해줌
- `depends_on`옵션을 통해 의존하는 서비스들도 같이 불로오도록 설정
- `docker-compose up -d --build server`로 실행 --build옵션은 Dockerfile변경이 있는 경우 이미지를 재생성하도록 하는 것임

## 다른 유틸 컨테이너 추가하기

---

- `artisan` 컨테이너는 PHP로 빌드된 Laravel 명령임

```Dockerfile
  artisan:
    build:
      context: .
      dockerfile: dockerfiles/php.dockerfile
    volumes: 
      - ./src:/var/www/html
    entrypoint: ["php", "/var/www/html/artisan"]
  npm:
    image: node:14
    working_dir: /var/www/html
    entrypoint: ["npm"]
    volumes: 
      - ./src:/var/www/html
```

## docker compose에서 Dockerfile을 사용하는 이유

---

- Dockerfile을 사용하면 docker-compose.yml파일을 더 간결하게 유지할 수 있음
- docker-compose내에서는 `COPY`, `RUN`등의 명령어를 사용할 수 없음
- **마운트 바인드를 하는 것은 로컬에서는 유용할 수 있지만 배포할 때는 제외하는게 좋음**
  - 소스코드와 설정들을 직접 이미지로 복사해서 배포
  - COPY명령어와 네이밍 변경등을 통해 설정들을 직접 이미지에 복사함
  - `ENTRYPOINT`나 `CMD`를 지정할 필요가 없어짐

  ```Dockerfile
  FROM nginx:stable-alpine

  WORKDIR /etc/nginx/conf.d

  COPY nginx/nginx.conf .

  RUN mv nginx.conf default.conf

  WORKDIR /var/www/html

  COPY src .
  ```

  - 아래와 같이 context를 설정하여 dockerfile을 직접 지정하여 사용하면 됨

  ```docker-compose
  server:
    # image: 'nginx:stable-alpine'
    build:
      context: .
      dockerfile: dockerfiles/nginx.dockerfile
    ports: 
      - '8000:80'
    volumes: 
      - ./src:/var/www/html
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on: 
      - php
      - mysql
  ```