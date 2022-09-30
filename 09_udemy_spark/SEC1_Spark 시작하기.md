# SEC1 Spark 시작하기

## 환경설정

---

- 나는 WSL을 사용하므로 따로 설치 진행. (강의는 windows환경에 설치함.)
- Java 설치

```bash
sudo apt-get install openjdk-8-jdk-headless
java -version
# openjdk version "1.8.0_342"
# OpenJDK Runtime Environment (build 1.8.0_342-8u342-b07-0ubuntu1~20.04-b07)
# OpenJDK 64-Bit Server VM (build 25.342-b07, mixed mode)
```

- Spark 설치
  - 강의에서는 2.1.x 버전을 사용하나, 현재 3.x 버전이 release 돼서 최신버전으로 설치해봄

```bash
wget https://archive.apache.org/dist/spark/spark-3.3.0/spark-3.3.0-bin-hadoop3.tgz
tar -xf spark-3.3.0-bin-hadoop3.tgz
cd spark-3.3.0-bin-hadoop3
bin/spark-shell # 확인

pip install pyspark
pyspark # pyspark 실행
```

```python
# 잘 되는지 테스트
rdd = sc.textFile("README.md")
rdd.count()
```

## 학습 데이터 다운로드

---

```bash
wget http://media.sundog-soft.com/es/ml-100k.zip
unzip m1-100k.zip 
```

## 예제코드 실행

---

- `spark-submit` alias 추가

```bash
alias spark-submit='/home/seungho/wsl_sandbox/spark_study/spark-3.3.0-bin-hadoop3/bin/spark-submit'
```

- `ratings-counter.py` 다운로드하여 data 경로만 변경
- 실행 `spark-submit ratings-counter.py`

```python
from pyspark import SparkConf, SparkContext
import collections

conf = SparkConf().setMaster("local").setAppName("RatingsHistogram")
sc = SparkContext(conf = conf)

lines = sc.textFile("file:///home/seungho/wsl_sandbox/spark_study/data/ml-100k/u.data")
ratings = lines.map(lambda x: x.split()[2])
result = ratings.countByValue()

sortedResults = collections.OrderedDict(sorted(result.items()))
for key, value in sortedResults.items():
    print("%s %i" % (key, value))
```