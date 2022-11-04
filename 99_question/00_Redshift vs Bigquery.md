# Redshift vs Biguqery

---

사용 관점 보다는 **아키텍처 중심의 비교**입니다.

## 비교

| 항목      | Redshift                                                                  | Bigquery                                                                                                   |
| --------- | ------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| 플랫폼    | AWS                                                                       | GCP                                                                                                        |
| **구조**  | 프로비저닝 클러스터와 노드                                                | 서버리스                                                                                                   |
| **기반**  | PostgreSQL기반의 MPP DW                                                   | Borg(클러스터 관리) <br> + Dremel(쿼리엔진) <br> + Colossus(데이터 저장소) <br> + Juniper(데이터 네트워크) |
| 조회 성능 | Distkey와 Sortkey 설정                                                    | Partitioning과 ClusteringKey 설정                                                                          |
| 비용      | 클러스터 노드당                                                           | 저장비용과 쿼리 Scan당 비용                                                                                |
| 공통점    | OLAP에 적합,  각 클라우드의 서비스(s3, gcs, rds 등)들과의 통합, ANSI SQL, |

### Redshift 구조

---

- 리더 노드
  - 어플리케이션과 컴퓨팅노드에서 일어나는 모든 통신을 관리
  - 쿼리의 실행 계획 작성하여 컴퓨팅 노드로 할당
- 컴퓨팅 노드
  - 각 노드는 수직/수평 확장 둘 다 제공
  - 노드 슬라이스로 분할되고, 병렬적으로 작업을 수행함
- 쿼리가 빠른 이유
  - MPP: 분산키에 따른 병렬 방식 데이터 처리
  - 열 기반 데이터 스토리지: 디스크 I/O 줄어듬. 정렬키를 사용하면 블록의 데이터를 빠르게 필터링
    - ![열기반스토리지](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/images/03b-Rows-vs-Columns.png)
  - 데이터 압축: 디스크I/O 줄어듬
  - 쿼리 옵티마이저: MPP를 최적으로 해줌
  - 결과 캐싱: 특정 형식의 쿼리를 리더 노드 메모리에 캐싱함
  - 컴파일 코드: 쿼리를 컴파일 하여 모든 클러스터 노드에 분산

![Redshift 구조](https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/images/02-NodeRelationships.png)

### Bigquery 구조

---

- Borg(클러스터 관리)
  - 클러스터 관리시스템
  - 자원(CPU, RAM, disk, network)을 관리하고 할당하는 역할
- Dremel(쿼리엔진)
  - Dremel은 SQL 쿼리가 들어오면 이를 execution tree로 만들고, 분산된 노드에서 쿼리를 수행할 수 있고 쿼리를 쪼개 리프노드에 전달함
  - 리프노드에 전달된 쿼리들이 수행한 결과를 취합하여 부모노드쪽으로 보냄
  - ![Dremel](https://panoply.io/uploads/bigquery-architecture-2.png)
  - 실 사용할 때 포인트
    - Limit절로 스캔양을 줄일 수 없음. 쿼리 수행을 다 하고나서 마지막에 필터링 하기 때문임.
    - Join 최소화. 리프노드간의 셔플이 일어나기 때문에 조회 성능에 좋지 않음
- Colossus(데이터 저장소)
  - GFS(Google File System)의 후속 버전인 분산 파일 시스템
  - Columnar Storage
- Juniper(데이터 네트워크)
  - Jupiter 네트워크는 초당 1 페타바이트 규모의 데이터를 전송가능하여 거대한 작업량도 순식간에 분산

![Bigquery 구조](https://storage.googleapis.com/gweb-cloudblog-publish/images/BQ_Explained_3.max-500x500.jpg)

## 어떤걸 사용해야 할까?

(주관적인 생각)

---

- 현재 사용중인 환경에 적합한 것을 고려
- 전통적인 DW 사용 및 예측가능한 워크로드는 `Redshift` / DW 신규 도입 및 예측불가능한 워크로드는 `Bigquery` 라고 생각

### 참고 자료

---

- https://thewayitwas.tistory.com/457
- https://www.integrate.io/blog/redshift-vs-bigquery-comprehensive-guide/
- https://www.striim.com/blog/cloud-data-warehouse-comparison-redshift-vs-bigquery-vs-azure-vs-snowflake-for-real-time-data/
- https://panoply.io/data-warehouse-guide/bigquery-architecture/
- https://docs.aws.amazon.com/ko_kr/redshift/latest/dg/c_high_level_system_architecture.html