# SEC4 네트워크: 컨테이너 통신

## 통신 종류

---

- WWW통신
  - 컨테이너 내부에서 WWW로 보내는 것은 아무 작업없이 잘 동작함
- 컨테이너 <-> 호스트머신 통신
  - 로컬에 설치된 DB에 접근하기
  - `host.docker.inertnal` 키워드로 로컬 호스트 머신의 IP주소를 알 수 있음
  - `mongodb://host.docker.inertnal:27017/swfavorites`
- 컨테이너간 통신
  - IP 주소로 하드코딩
    - mongodb 컨테이너 띄우기

      ```bash
      docker run -d --name mongodb mongo
      docker container inspect # IP 찾기
      
      ```

    - `mongodb://{mongo ip}:27017/swfavorites` 사용
  - **컨테이너 네트워크**
    - `docker network create favorites-net` 
    - `docker run -d --name mongodb --network favorites-net`
    - **컨테이너 이름을 주소로 사용** ex) `mongodb://{mongodb container name}:27017/swfavorites` 사용
    - 같은 네트워크를 쓰면 따로 포트를 오픈할 필요 없음. 외부로 오픈할 때만 포트를 오픈하도록해야함



