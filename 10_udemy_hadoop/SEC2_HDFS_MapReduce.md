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
