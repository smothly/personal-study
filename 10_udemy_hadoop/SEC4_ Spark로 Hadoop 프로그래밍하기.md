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

## Datasets & Spark2.0

---

- Spark는 RDD를 Dataframe으로 확장시킴
- Dataframe
  - 구조화된 행 객체
  - `SQL` 질의 가능
  - 스키마를 가지고 있고 json, hive, parquet으로 읽고 쓸 수 있음
  - JDBC/ODBC 등을 통해 DB와 통합가능
- SparkSQL in python
  
  ```python
  from pyspark import SparkContext, Row

  hiveContext = hiveContext(sc)
  inputData = spark.read.json(dataFile)
  inputData.createOrReplaceTempView('myStructuredStuff')
  myResultDataFrame = hiveContext.sql("""SELECT foo FROM bar ORDER BY foobar""") 

  ```

  - dataframe 문법을 사용할 수도 있음

  ```python
  myResultDataFrame.show()
  myResultDataFrame.select('fieldName')
  myResultDataFrame.filter(myResultDataFrame('fieldName' > 20))
  myResultDataFrame.groupBy(myResultDataFrame('fieldName')).mean()
  myResultDataFrame.rdd().map(mapperFunction)
  ```

- Datasets
  - spark2.0에서 도입된 개념으로, dataframe은 열 객체의 dataset
  - dataset이 더 큰 개념
  - Spark내 API를 사용하기 위한 공통의 객체
- UDF 지원

## Datasets 실습

---

- 실행
  - `export SPARK_MAJOR_VERSION=2`
  - `spark-submit LowestRatedMovieDataFrame.py` 

  ```python
  from pyspark.sql import SparkSession
  from pyspark.sql import Row
  from pyspark.sql import functions

  def loadMovieNames():
      movieNames = {}
      with open("ml-100k/u.item") as f:
          for line in f:
              fields = line.split('|')
              movieNames[int(fields[0])] = fields[1]
      return movieNames

  def parseInput(line):
      fields = line.split()
      return Row(movieID = int(fields[1]), rating = float(fields[2]))

  if __name__ == "__main__":
      # Create a SparkSession (the config bit is only for Windows!)
      spark = SparkSession.builder.appName("PopularMovies").getOrCreate()

      # Load up our movie ID -> name dictionary
      movieNames = loadMovieNames()

      # Get the raw data
      lines = spark.sparkContext.textFile("hdfs:///user/maria_dev/ml-100k/u.data")
      # Convert it to a RDD of Row objects with (movieID, rating)
      movies = lines.map(parseInput)
      # Convert that to a DataFrame
      movieDataset = spark.createDataFrame(movies)

      # Compute average rating for each movieID
      averageRatings = movieDataset.groupBy("movieID").avg("rating")

      # Compute count of ratings for each movieID
      counts = movieDataset.groupBy("movieID").count()

      # Join the two together (We now have movieID, avg(rating), and count columns)
      averagesAndCounts = counts.join(averageRatings, "movieID")

      # Pull the top 10 results
      topTen = averagesAndCounts.orderBy("avg(rating)").take(10)

      # Print them out, converting movie ID's to names as we go.
      for movie in topTen:
          print (movieNames[movie[0]], movie[1], movie[2])

      # Stop the session
      spark.stop()
  ```

## 실습: MLLib을 사용해 영화 추천하기

---

- 추천알고리즘은 ALS를 사용함
- 실습할 때 아래 데이터를 u.data에 추가
  
  ```text
  0 50  5 881250949
  0 172  5 881250949
  0 133  1 881250949
  ```

- `pip install numpy==1.16`
- `spark-submit MovieRecommendationsALS.py`

  ```python
  from pyspark.sql import SparkSession
  from pyspark.ml.recommendation import ALS
  from pyspark.sql import Row
  from pyspark.sql.functions import lit

  # Load up movie ID -> movie name dictionary
  def loadMovieNames():
      movieNames = {}
      with open("ml-100k/u.item") as f:
          for line in f:
              fields = line.split('|')
              movieNames[int(fields[0])] = fields[1].decode('ascii', 'ignore')
      return movieNames

  # Convert u.data lines into (userID, movieID, rating) rows
  def parseInput(line):
      fields = line.value.split()
      return Row(userID = int(fields[0]), movieID = int(fields[1]), rating = float(fields[2]))


  if __name__ == "__main__":
      # Create a SparkSession (the config bit is only for Windows!)
      spark = SparkSession.builder.appName("MovieRecs").getOrCreate()

      # This line is necessary on HDP 2.6.5:
      spark.conf.set("spark.sql.crossJoin.enabled", "true")

      # Load up our movie ID -> name dictionary
      movieNames = loadMovieNames()

      # Get the raw data
      lines = spark.read.text("hdfs:///user/maria_dev/ml-100k/u.data").rdd

      # Convert it to a RDD of Row objects with (userID, movieID, rating)
      ratingsRDD = lines.map(parseInput)

      # Convert to a DataFrame and cache it
      ratings = spark.createDataFrame(ratingsRDD).cache()

      # Create an ALS collaborative filtering model from the complete data set
      als = ALS(maxIter=5, regParam=0.01, userCol="userID", itemCol="movieID", ratingCol="rating")
      model = als.fit(ratings)

      # Print out ratings from user 0:
      print("\nRatings for user ID 0:")
      userRatings = ratings.filter("userID = 0")
      for rating in userRatings.collect():
          print movieNames[rating['movieID']], rating['rating']

      print("\nTop 20 recommendations:")
      # Find movies rated more than 100 times
      ratingCounts = ratings.groupBy("movieID").count().filter("count > 100")
      # Construct a "test" dataframe for user 0 with every movie rated more than 100 times
      popularMovies = ratingCounts.select("movieID").withColumn('userID', lit(0))

      # Run our model on that list of popular movies for user ID 0
      recommendations = model.transform(popularMovies)

      # Get the top 20 movies with the highest predicted rating for this user
      topRecommendations = recommendations.sort(recommendations.prediction.desc()).take(20)

      for recommendation in topRecommendations:
          print (movieNames[recommendation['movieID']], recommendation['prediction'])

      spark.stop()
  ```

## 실습: 평균 평점이 가장 낮은 영화를 평점의 개수로 걸러내기

---

- `Dataframe`을 사용해서 해보기

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

      # Filter out movies rated 10 or fewer times
      popularTotalsAndCount = ratingTotalsAndCount.filter(lambda x: x[1][1] > 10)

      # Map to (rating, averageRating)
      averageRatings = popularTotalsAndCount.mapValues(lambda totalAndCount : totalAndCount[0] / totalAndCount[1])

      # Sort by average rating
      sortedMovies = averageRatings.sortBy(lambda x: x[1])

      # Take the top 10 results
      results = sortedMovies.take(10)

      # Print them out:
      for result in results:
          print(movieNames[result[0]], result[1])
  ```

- `Datasets` 사용하기
  
  ```python
  from pyspark.sql import SparkSession
  from pyspark.sql import Row
  from pyspark.sql import functions

  def loadMovieNames():
      movieNames = {}
      with open("ml-100k/u.item") as f:
          for line in f:
              fields = line.split('|')
              movieNames[int(fields[0])] = fields[1]
      return movieNames

  def parseInput(line):
      fields = line.split()
      return Row(movieID = int(fields[1]), rating = float(fields[2]))

  if __name__ == "__main__":
      # Create a SparkSession (the config bit is only for Windows!)
      spark = SparkSession.builder.appName("PopularMovies").getOrCreate()

      # Load up our movie ID -> name dictionary
      movieNames = loadMovieNames()

      # Get the raw data
      lines = spark.sparkContext.textFile("hdfs:///user/maria_dev/ml-100k/u.data")
      # Convert it to a RDD of Row objects with (movieID, rating)
      movies = lines.map(parseInput)
      # Convert that to a DataFrame
      movieDataset = spark.createDataFrame(movies)

      # Compute average rating for each movieID
      averageRatings = movieDataset.groupBy("movieID").avg("rating")

      # Compute count of ratings for each movieID
      counts = movieDataset.groupBy("movieID").count()

      # Join the two together (We now have movieID, avg(rating), and count columns)
      averagesAndCounts = counts.join(averageRatings, "movieID")

      # Filter movies rated 10 or fewer times
      popularAveragesAndCounts = averagesAndCounts.filter("count > 10")

      # Pull the top 10 results
      topTen = popularAveragesAndCounts.orderBy("avg(rating)").take(10)

      # Print them out, converting movie ID's to names as we go.
      for movie in topTen:
          print (movieNames[movie[0]], movie[1], movie[2])

      # Stop the session
      spark.stop()
  ```
