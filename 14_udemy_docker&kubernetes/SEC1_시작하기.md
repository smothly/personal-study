# SEC1 Spark 시작하기

## 시작하기

---

- 도커 다루는 방법
- 도커로 문제 해결
- 도커로 어플리케이션 배포
- 쿠버네티스
- 도커와 쿠버네티스의 연결점

## Docker란?

---

- 컨테이너 기반의 오픈소스 가상화 플랫폼
- 컨테이너
  - 표준화된 소프트웨어 유닛. **코드 패키지이며 해당 코드를 실행하는데 필요한 종속성과 도구를 포함**
  - NodeJS는 Javascript 코드를 실행하는데 사용할 수 있는 javascript 런타임 환경
  - 컨테이너에는 application code와 NodeJS 런타임 환경 도구들이 포함되며 컨테이너끼리는 동일한 환경임
  - 피크닉 박스로 비유
  - standalone으로 동작
- 도커는 컨테이너를 구축, 관리하기위한 도구

## Why Docker?

---

- why containers?
  - **독립적이고 표준화된 개발 환경**을 가짐
  - 예시
    - NodeJS에서 await이라는 키워드를 사용하려면 NodeJS 14.3 이상의 버전이 필요함
    - 팀이나 회사에서 각각의 개발환경
    - 혼자 작업할 때도 프로젝트마다 환경이 다름

## VM vs Docker Container

---

- ![vm vs container](https://images.contentstack.io/v3/assets/blt300387d93dabf50e/bltb6200bc085503718/5e1f209a63d1b6503160c6d5/containers-vs-virtual-machines.jpg)
- VM
  - 컴퓨터 안에 컴퓨터를 만드는 것
  - virtual OS가 가지는 오버헤드가 문제
  - 장점
    - 분리된 환경 생성할 수 있음
    - 환경별 구성을 가질 수 있고
    - 안정정적으로 공유 및 재생산 가능
  - 단점
    - 중복 복제로 인한 공간낭비
    - 성능 저하
    - VM을 구성하는 단일파일이 없어 재구성하기가 까다로움
- Container
  - 컨테이너 애뮬레이터를 지원하는 내장컨테이너 활용
  - Dokcer Engine이라는 도구를 실행함
  - OS 레이어가 존재하긴 하지만, VM만큼 모든 OS를 가지진 않음
  - 구성 파일을 사용하여 컨테이너를 구성하고 그를 설명할 수 있음
  - 장점
    - **OS와 시스템이 미치는 영향이 적음**
    - **공유, 재생산, 배포가 쉬움**
    - 불필요한 OS레이어들을 가지고 있지 않음

## Docker 환경설정

---

- Docker Desktop 설치
- IDE에서 Docker Plugin 설치
- NodeJS **로컬환경설정 다운없이 컨테이너**로 사용 가능
- 기본 예제 실핼
  - `Docker build .`
  - `Docker run -p 3000:3000 <image id>`

## 강의 개요

---

- 기초
  - Images & Containers
  - Data & Volumes
  - Containers & Networking
- 실전
  - Multi-Container Projects
  - Using Docker-compose
  - Utility Containers
  - Deploying Docker Containers
- k8s
  - k8s Introduction & Basics
  - k8s Data & Volumes
  - k8s Networking
  - Deploying k8s Clusters