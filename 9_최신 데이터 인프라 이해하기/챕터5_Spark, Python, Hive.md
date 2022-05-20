# 5장 Spark, Python, Hive

- 아래 다이어그램 중 **Spark Platform**  +  **Python Libs** + **Batch Query Engine**부분에 대한 설명
![최종도표](https://img1.daumcdn.net/thumb/R1280x0.fjpg/?fname=http://t1.daumcdn.net/brunch/service/user/3hD/image/Pooto4-Wi0R5dsKZCrFkh5mCSEM)
- 워크플로내에서 태스크 단위로 실행하는데, 내부적으로 스파크를 통해서 태스크를 수행하게 된다.

---

## Python Libs

### pandas

- 패널 데이터(엑셀 같은 row/col 데이터)를 다룸
- dataframe이라는 단위로 조작, 조인, 서브셋, 그래프 등으로 데이터를 조작함
- SQL 과 다르게 코드로 데이터를 조작한다 보면됨

### boto3

- python에서 amazon 리소스 사용

### Dask

- python을 native하게 병렬로 처리

### Ray

- Dask는 중앙에서 스케줄링 Ray는 분산 bottom-up 스케줄링
- task latency를 줄이고 throughput
- Dask는 데이트 프레임 분산처리  Ray는 여러대의 서버에서 머신러닝 돌릴 때

---

## Spark Platform

- 대규모 데이터를 위한 분산처리 클러스터 컴퓨팅 프레임워크
- 하둡과 많이 비교됨
  - ![비교](https://www.xpertup.com/wp-content/uploads/2020/07/Hadoop-MapReduce-vs-Apache-Spark-1024x536.jpg)
- 스파크가 왜 빠른가?
  - [발표자료](https://www.slideshare.net/yongho/rdd-paper-review)
  - RDD(Resilient Distributed Datasets): 쉽게 복원이 되는 데이터셋, 계보(lineage)를 통해 데이터를 파악함
  - in-memory 기반
  - lazy execuetion 마지막에 실행만 하므로 최적의 코스로 실행할 수 있음
- 스파크 구성
  - 여러가지 일(SQL, ML등) RDD기반으로 spark core에서 처리함.
  - ![스파크 구성](https://images.velog.io/images/king3456/post/c1f7ba13-b240-4f3a-b7cc-af41734ca897/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-04-19%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%209.17.10.png)

---

## Data bricks

- Saprk에 managed services를 추가함
- 델타레이크, MLflow 등을 붙여 하나의 솔루션

---

## EMR

- AWS에서 만든 Spark 사용 플랫폼
- 대규모에서는 비용절약이 됨

---

## Hive

- hive는 hadoop ecosystem과 연결
- meta데이터스토어를 가지고 있는데, spark와 연결해서 쓸 수 있음
- 다양한 데이터소스를 연결할 수 있음
