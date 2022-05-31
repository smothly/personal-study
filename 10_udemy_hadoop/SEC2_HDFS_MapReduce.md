# SEC2 Hadoop 핵심 사용법: HDFS와 MapReduce

## HDFS 정의 및 작동 방식

---

- ![HDFS 아키텍처](https://blog.kakaocdn.net/dn/bNtyTE/btrc7O872Sa/iohONNse4K7WG4UNnNPrhk/img.png)
- 특징
  - 빅데이터를 전체 클러스터에 분산해 안정적으로 저장하여 애플리 케이션이 그 데이터를 신속하게 액세스하여 분석하게 해줌
  - 파일을 '데이터 블록' 으로 쪼갬. 블록의 기본값 128MB
  - 분산 처리를 위해 여러 컴퓨터에 걸쳐 저장되고 빠른 액세스를 보장함
  - 모든 블록마다 두개 이상의 복사본을 저장하여 Failover 가능
- 아키텍처
  - Name Node
    - 어느 블록이 어디에 있는지 추적함
    - edit log로 변경사항을 기록함
  - Data Node
    - 실제 파일의 블록을 저장함
    - 블록의 복사를 위해 각 노드끼리 통신하기도 함
- 동작법
  - 파일 읽기
    - clientnode가 namenode에게 파일 A가 필요하다고 함
    - namenode가 clientnode에게 파일 A가 저장된 datanode 블록의 위치들을 알려줌
    - clientnode가 해당 위치의 datanode에서 데이터를 가져옴
  - 파일 쓰기
    - clientnode가 namenode에게 파일 A 저장할 위치를 물어봄
    - namenode가 clientnode에게 파일 A 저장할 위치를 알려줌
    - clientnode가 datanode에 파일을 저장함
    - datanode는 주변 datanode에 복사본을 전달하고 저장함
    - datanode의 복사가 완료되면 clientnode에게 저장 완료를 알려줌
    - clientnode는 namenode에게 새로운 파일의 블록과 복사본의 위치를 전달함
- namenode는 하나만 존재함. 어떻게 고가용성을 유지하냐?
  - Metadata backup: NFS와 local disk 둘다 쓰면서 백업
  - Secondary name node: edit log의 복사본을 유지함
  - HDFS Federation
    - 각각의 namenode는 특정 namespace volume을 관리함
    - namenode하나가 다운되면 전체 클러스터 다운이 아닌 특정 데이터만 유실됨
  - **HDFS High Availibility**
    - edit log를 안전한 공유 저장소에 저장함
    - Zookeeper가 어떤 namenode와 소통해야하는지 알려줌
    - 정확히 하나의 namenode가 동작하도록 극단적인 조치를 취함: 사용하지 않는 namenode는 저원을 차단함
- HDFS 사용법
  - UI(Ambari)
  - CLI
  - Java Interface
  - HTTP로 직접 접근하거나 HDFS와 클라이언트 사이에 proxy 서버를 두어 접근 가능
  - NFS 게이트웨이: 파일 시스템을 마운트

## MovieLens 데이터 세트로 실습

---

- 세팅
  - HDP 실행
  - Ambari 127.0.0.1:8080 접속
  - 하나의 PC에 설치되어잇는 클러스터지만 가상의 여러개의 클러스터라고 가정하고 진행
- UI로 조작
  - HDFS에 들어가 ml-100k 폴더 생성
  - ml-100k에 u.data, u.item 데이터 업로드
  - 업로드한 데이터 삭제
- CLI
  
  ```bash
  ssh maria_dev@127.0.0.1 -p 2222 # SSH 접속
  hadoop fs -ls # 파일 리스트
  hadoop fs -mkdir ml-100k # 폴더 생성
  wget http://media.sundog-soft.com/hadoop/ml-100k/u.data # 데이터 가져오기
  hadoop fs -copyFromLocal u.data ml-100k/u.data # 로컬로부터 데이터 업로드
  hadoop fs -ls ml-100k # 데이터 확인
  hadoop fs -rm ml-100k/u.data # 데이터 삭제
  hadoop fs -rmdir ml-100k # 폴더 삭제
  ```

## MapReduce 정의 및 작동 방식

---

- MapReduce 과정
  - ![Mapreduce](https://t1.daumcdn.net/cfile/tistory/2136A84B59381A8428)
- 특징
  - 클러스터에 데이터를 분배
  - 데이터를 파티션으로 나눠서 클러스터에 걸쳐 병렬 처리되도록 함
  - 데이터를 매핑(transformed)하고 리듀싱(aggregate)함
  - Failover기능 제공
- 예시
  - 사용자가 얼마나 많은 영화를 평가했는지 살펴보기
    - key: 사용자ID
    - value: 집계할 영화
    - 핵심은 key가 같은것이 여러개 나올 수 있음. ex) 철수: 토르, 철수: 어벤저스
    - 순서
      1. mapper: key:value 쌍의 리스트를 만들어줌
      2. shuffle sort: key로 묶고 sort해줌
      3. reducer: 각 키에 값들에 len이라는 연산자로 집계해줌

## MapReduce 분산 처리 방법

---
- 아키텍처
  - ![아키텍처](https://i.stack.imgur.com/1NXUp.png)
- 내부적으로 MR이 어떻게 돌아가는지?
  - mapper: row를 나누어서 여러 노드에 전달하고 병렬적으로 mapper 처리
  - suffle and sort
    - 네트워크에 데이터를 주고보내는 방식이 아님
    - `merger sort`를 함
  - reducer: 주어진 key를 병렬적으로 집계합니다
  - 순서
    1. client node가 작업을 개시
    2. yarn 리소스 매니저와 대화하여 MR작업이 필요하다고 함
    3. HDFS에 데이터를 복사함
    4. node manager 아래에 있는 Map Reduce Application Master가 작동함
       - application master는 개별 매핑과 리듀싱 작업을 모니터링함, yarn과 협업해 작업을 클러스터에 배분
       - node manager는 MR이 동작하는 모든 것을 관리, 개별 PC를 관리
    5. 살제 작업은 각 node에서 MR작업이 수행되고, 이 때 HDFS와 통신하며 I/O를 진행함
  - 네트워크 작업을 최소화 하기 위해 MR작업을 최대한 데이터와 가까운 곳에서 실행. 웬만하면 데이터 블록을 가지고 있는 머신
- 깊게 살펴보기
  - MR과 Hadoop은 JAVA로 작성됨
  - STREAMING을 통해 표준 입춥력과 같은 PAI를 제공하여 다른 언어로도 접근 가능함
  - 실패시 처리 방법
    - worker가 실패하면? => 같은 노드 or 다른 노드에서 재시작
    - application master는 yarn이 restart 시켜줌 SPOF가 아님
    - 전체노드가 다운되면? yarn이 restart 시켜줌
    - yarn이 다운되면? zookeeper를 사용하여 stnad by yarn을 셋업함
  - Counter: 클러스터 전반에 걸쳐 공유된 총 수를 유지
  - Combiner: 매퍼 노드를 줄이고 최적화해서 간접비를 줄임
- 요즘은 MR을 직접 사용하기 보다는 Hive나 Spark 같은 고수준의 도구로 대체됨

## MapReduce 연습문제

---

- 평점이 몇개씩 있는지 살펴보기
  - key: 평점
  - value: 숫자1
  - map -> shuffle&sort -> reduce 과정은 동일함
  - python으로 프로그래밍
  
    ```python mapper
    from mrjob.job import MRJob
    from mrjob.step import MRStep
    
    class RatingsBreakdown(MRJob):
      def steps(self):
        return [
          MRStep(mapper=self.mapper_get_ratings, reducer=self.reducer_count_ratings)
        ]
      def mapper_get_ratings(self, _, line):
        (userID, movieID, rating, timestamp) = line.split('\t')
        yield rating, 1

      def reducer_count_ratings(self, key, values):
        yield key, sum(values)

    if __name__ == '__main__':
      RatingsBreakdown.run()
    ```

## MapReduce 실습

---

- 2.5버전 기준으로 작성
- 환경설정

  ```sh
  su root
  cd /etc/yum.repos.d
  cp sandbox.repo /tmp
  rm sandbox.repo
  cd ~
  
  yum install python-pip
  # python-pip 설치 안될 시 아래 커맨드 실행
  echo "https://vault.centos.org/6.10/os/x86_64/" > /var/cache/yum/x86_64/6/base/mirrorlist.txt
  echo "https://vault.centos.org/6.10/updates/x86_64/" > /var/cache/yum/x86_64/6/updates/mirrorlist.txt
  echo "https://vault.centos.org/6.10/extras/x86_64/" > /var/cache/yum/x86_64/6/extras/mirrorlist.txt
  echo "https://vault.centos.org/6.10/sclo/x86_64/rh/" > /var/cache/yum/x86_64/6/centos-sclo-rh/mirrorlist.txt
  echo "https://vault.centos.org/6.10/sclo/x86_64/sclo" > /var/cache/yum/x86_64/6/centos-sclo-sclo/mirrorlist.txt
  
  pip install google-api-python-client==1.6.4
  # 설치 안될 시 python2.7로 업데이트
  yum install scl-utils
  yum install centos-release-scl
  yum install python27
  scl enable python27 bash
  wget https://bootstrap.pypa.io/pip/2.7/get-pip.py
  python2.7 get-pip.py
  
  pip install mrjob==0.5.11
  
  yum install vim

  # 데이터랑 스크립트 다운로드
  wget http://media.sundog-soft.com/hadoop/ml-100k/u.data
  wget http://media.sundog-soft.com/hadoop/RatingsBreakdown.py  
  ```

- 실습1
  - 평점 각 개수 파악하기

  ```sh  
  python RatingsBreakdown.py u.data
  # 데이터가 작기 때문에 하나의 클러스터에서 모든걸 작업함.
  # 실제로는 hdfs와 여러개의 클러스터 사용
  python RatingsBreakdown.py -r hadoop --hadoop-streaming-jar /usr/hdp/current/hadoop-mapreduce-client/hadoop-streaming.jar u.data
  ```

- 실습2
  - 영화의 평점 개수 파악하고 정렬하여 출력하기
  - 실습1에서 사용했던 코드 수정
    - 정렬하는 법
      - 기존 리듀서의 결과를 다시 매퍼의 인풋으로 넣어주기
      - key를 Movie ID로 바꿔주기
      - 모든 데이터는 스트링으로 들어오기 때문에 zero padding을 해줌

  ```python
  from mrjob.job import MRJob
  from mrjob.step import MRStep

  class RatingsBreakdown(MRJob):
      def steps(self):
          return [
              MRStep(mapper=self.mapper_get_ratings,
                    reducer=self.reducer_count_ratings),
              MRStep(reducer=self.reducer_sorted_output)
          ]

      def mapper_get_ratings(self, _, line):
          (userID, movieID, rating, timestamp) = line.split('\t')
          yield movieID, 1

      #def reducer_count_ratings(self, key, values):
      #    yield key, sum(values)      
      def reducer_count_ratings(self, key, values):
          yield str(sum(values)).zfill(5), key
      
      def reducer_sorted_output(self, count, movies):
          for movie in movies:
              yield movie, count

  if __name__ == '__main__':
      RatingsBreakdown.run()
  ```

