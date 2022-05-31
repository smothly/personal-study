# SEC54 Spark로 Hadoop 프로그래밍 하기

## What is Spark?

---

- ![Spark](https://images.velog.io/images/king3456/post/c1f7ba13-b240-4f3a-b7cc-af41734ca897/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-04-19%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%209.17.10.png)
- 프로그래밍언어(java, python, scala)를 사용하여 스크립트 조작
- Pig와 다른점은 Spark위에 머신 러닝, 데이터 마이닝, 데이터 스트리밍등 다양한 생태계가 존재
- 확장성도 가지고 있어 Spark는 꼭 Hadoop위에서 실행될 필요는 없음
- 데이터를 병렬로 처리함 exceutor라는 단위를 통해 진행하고 내부에는 cache와 task를 가짐
  - ![spark 아키텍처](https://spark.apache.org/docs/latest/img/cluster-overview.png)
- 메모리 기반 MR이라 빠름
- `DAG(Directed Acyclic Graph)`를 사용하여 워크플로를 최적화 시킴. Tez와 비슷함
- `RDD(Resilient Distributed Dataset)`객체 기반으로 작업함
- `SparkSQL`, `SparkStreaming등의` 기능을 제공하며 간단하게 어려운 작업을 할 수 있음
- scala는 java 바이트 코드로 바로 컴파일 하기 때문에 python 보다 오버헤드가 적어 빠름

## RDD(Resilient Distributed Dataset)

---

- 작업을 고르케 클러스터에 **분산**하고 실패가 생겨도 **회복**할 수 있음
- 프로그래밍 관점에서는 그냥 데이터 세트임
- 드라이버 프로그램이 SparkContext라는 것을 만들어주는데 이것이 RDD가 실행되는 환경임과 동시에 RDD를 만들어줌
- RDD 생성코드
  - textfile로 s3, hdfs
  - Hivecontext 사용
  - JDBC, Cassandra, Hbase, ES, JSON 등 다양한 소스로부터 생성할 수 있음
- Transforming RDD
  - map: 입력1개에 출력1개
  - flatmap: 입력 출력개수가 다양하게 나누어져 있음
  - filter: RDD에서 행을 유지할지 말지 
  - distinct: unique 값 찾기
  - sample
  - union, intersection, subtract, cartesian
- map Example
  
  ```python
  rdd = sc.parallelize([1,2,3,4])
  squaredRDD = rdd.map(lambda x: x*x)

  => 1, 4, 9, 16
  ```

- RDD actions
  - 매핑에 해당
  - collect
  - count
  - countByValue
  - take
  - top
  - reduce ...
- **Lazy Evaluation**
  - 명령어가 호출되기 전까지는 아무일이 일어나지 않음
  - 거꾸로 돌아가 최적의 경로를 찾기 위함

## RDD 실습

---

- 평점이 가장 낮은 영화 찾기
  - 설정 변경: admin으로 ambari 로그인하여 spark1, 2 둘다 작업 => spark 들어가기 => loglevel ERROR로 변경 => restart 
  - 코드 실행
  
  ```shell
  wget http://media.sundog-soft.com/hadoop/ml-100k/u.item
  wget http://media.sundog-soft.com/hadoop/Spark.zip
  unzip Spark.zip
  less LowestRatedMovieSpark.py
  spark-submit LowestRatedMovieSpark.py
  ```

  - python code

  ```python
  from pyspark import SparkConf, SparkContext

  # This function just creates a Python "dictionary" we can later
  # use to convert movie ID's to movie names while printing out
  # the final results.
  def loadMovieNames():
      movieNames = {}
      with open("ml-100k/u.item") as f:
          for line in f:
              fields = line.split('|')
              movieNames[int(fields[0])] = fields[1]
      return movieNames

  # Take each line of u.data and convert it to (movieID, (rating, 1.0))
  # This way we can then add up all the ratings for each movie, and
  # the total number of ratings for each movie (which lets us compute the average)
  def parseInput(line):
      fields = line.split()
      return (int(fields[1]), (float(fields[2]), 1.0))

  if __name__ == "__main__":
      # The main script - create our SparkContext
      conf = SparkConf().setAppName("WorstMovies")
      sc = SparkContext(conf = conf)

      # Load up our movie ID -> movie name lookup table
      movieNames = loadMovieNames()

      # Load up the raw u.data file
      lines = sc.textFile("hdfs:///user/maria_dev/ml-100k/u.data")

      # Convert to (movieID, (rating, 1.0))
      movieRatings = lines.map(parseInput)

      # Reduce to (movieID, (sumOfRatings, totalRatings))
      ratingTotalsAndCount = movieRatings.reduceByKey(lambda movie1, movie2: ( movie1[0] + movie2[0], movie1[1] + movie2[1] ) )

      # Map to (rating, averageRating)
      averageRatings = ratingTotalsAndCount.mapValues(lambda totalAndCount : totalAndCount[0] / totalAndCount[1])

      # Sort by average rating
      sortedMovies = averageRatings.sortBy(lambda x: x[1])

      # Take the top 10 results
      results = sortedMovies.take(10)

      # Print them out:
      for result in results:
          print(movieNames[result[0]], result[1])
  ```
