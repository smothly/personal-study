# SEC3 Pig로 Hadoop 프로그래밍 하기

## Ambari 소개

---

- ![ambari예시](https://docs.microsoft.com/ko-kr/azure/hdinsight/media/hdinsight-hadoop-manage-ambari/hdi-metrics-dashboard.png)
- Pig와는 연관 없음
- 대시보드 설명
- 마스터 노드에 Ambari 설치하고, 다양한 서비스를 클러스터에 설치할 수 있음
- YARN에서 클러스터의 개수/네트워크/실패 등 모니터링하고 설정할 수 있음
- service에 어떤 호스트가 쓰고 있는지 확인 가능
- Alert설정 가능
- admin에서는 각 서비스 버전과 관리를 할 수 있음
  - `ambari-admin-password-reset` 으로 변경

## Pig 소개

---

- ![Pig 위치](https://miro.medium.com/max/1400/1*0eSXK6JvX3A0z22FUPZpDg.png)
- MR의 가장 큰 문제는 개발 사이클 타임
- 매퍼/리듀서 없이 데이터를 빠르게 분석할 수 있음! SQL과 비슷함
- 내부적으로 MR을 사용하고, MR뿐만 아니라 TEZ도 사용함
- UDF도 생성 가능
- MR과 TEZ위에서 실행
  - `TEZ`는 MR과 달리 일련의 과정의 상호 의존성을 파악하고 가장 효율적인 경로로 실행하여 성능을 개선
- 사용법
  - CLI: Grunt
  - Script
  - Ambari / HUE

## Pig 실습

---

- 가장 오래된 5점 영화 찾기
- Pig view 들어가서 스크립트 생성 후 실행
  ```SQL
  ratings = LOAD '/user/maria_dev/ml-100k/u.data' AS (userID:int, movieID:int, rating:int, ratingTime:int);

  metadata = LOAD '/user/maria_dev/ml-100k/u.item' USING PigStorage('|') AS (movieID:int, movieTitle:chararray, releaseDate:chararray, videoRelease:chararray, imdbLink:chararray);

  nameLookup = FOREACH metadata GENERATE movieID, movieTitle, ToUnixTime(ToDate(releaseDate, 'dd-MMM-yyyy')) AS releaseTime;

  ratingsByMovie = GROUP ratings BY movieID;

  avgRatings = FOREACH ratingsByMovie GENERATE group AS movieID, AVG(ratings.rating) AS avgRating;

  fiveStarMovies = FILTER avgRatings BY avgRating > 4.0;

  fiveStarsWithData = JOIN fiveStarMovies BY movieID, nameLookup BY movieID;

  oldestFiveStarMovies = ORDER fiveStarsWithData BY nameLookup::releaseTime;
  DUMP oldestFiveStarMovies;
  ```

- Tez 옵션 클릭해서 실행 => 더 빠름!

## Pig