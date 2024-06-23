# MongoDB
- 빅데이터 + 문서 기반 데이터, 기업 전문
- 일관성과 분할 불할 내성을 이은 선상에 위치
- 스키마가 없어서 어떤 데이터도 넣을 수 있다. _id를 자동으로 만들어준다.
    - 그러나 빠르게 데이터를 찾고 싶다면 스키마를 설정하는 것을 추천
- 검색이나 쿼리를 위해 데이터를 유연하게 색인할 수 있다.
- 컬렉션과 문서가 존재한다.
    - Database
    - Collections: 거의 모든 것이 들어 수 있으나 다른 데이터베이스의 컬렉션으로 옮길 수 없다.
    - Documents
## 아키텍처
- Replication Sets
    - 단일 마스터 아키텍처, 가용성 보다 일관성을 우선
    - 프라이머리 write -> 세컨더리 노드로 복사
    - 동기화는 오직 프라이머리랑, 그리고 프라이머리가 죽으면 빠르게 세컨더리에서 대체
    - 과반수로 정해지기 때문에 짝수의 서버를 가질 수 없다.
    - 그래서 결정권자 노드를 가질 수 있다.
    - 지연 세컨더리: 실수할 때를 대비해 딜레이를 둔다. 복제본 사이에 한 시간의 딜레이를 설정하면 한 시간 전의 데이터를 신속하게 살릴 수 있다.
- 샤딩
    - 다수의 레플리카 세트를 가지고 각 레플리카 세트가 데이터베이스 내에서 색인화된 일정한 값을 가진다.
    - 색인은 레플리카 세트의 정보의 양의 균형을 유지
    - 자동 샤딩은 시간이 지남에 따라 자동 분배를 하는데 가끔 실패야 스플릿 스톰을 일으킨다.

## 장점
- 자체 분산 파일 시스템을 가져 스파크 DataSet을 사용할 수 있다.
- javascript 연동 가능 .. ?
- Spark와의 연동: `MongoDBIntegration` 을 a

## 실습

```python
[root@sandbox-hdp ~]# cat MongoSpark.py
from pyspark.sql import SparkSession
from pyspark.sql import Row
from pyspark.sql import functions

def parseInput(line):
    fields = line.split('|')
    return Row(user_id = int(fields[0]), age = int(fields[1]), gender = fields[2], occupation = fields[3], zip = fields[4])

if __name__ == "__main__":
    # Create a SparkSession
    spark = SparkSession.builder.appName("MongoDBIntegration").getOrCreate()

    # Get the raw data
    lines = spark.sparkContext.textFile("hdfs:///user/maria_dev/ml-100k/u.user")
    # Convert it to a RDD of Row objects with (userID, age, gender, occupation, zip)
    users = lines.map(parseInput)
    # Convert that to a DataFrame
    usersDataset = spark.createDataFrame(users)

    # Write it into MongoDB
    usersDataset.write\
        .format("com.mongodb.spark.sql.DefaultSource")\
        .option("uri","mongodb://127.0.0.1/movielens.users")\
        .mode('append')\
        .save()

    # Read it back from MongoDB into a new Dataframe
    readUsers = spark.read\
    .format("com.mongodb.spark.sql.DefaultSource")\
    .option("uri","mongodb://127.0.0.1/movielens.users")\
    .load()

    readUsers.createOrReplaceTempView("users")

    sqlDF = spark.sql("SELECT * FROM users WHERE age < 20")
    sqlDF.show()

    # Stop the session
    spark.stop()
```

## 몽고 쉘
- `db.users.find( {user_id: 100} )` 자바 스크립트 같이 사용 가능
- 인덱스를 걸어저야 한다. `db.users.createIndex({user_id: 1})`
- `db.users.count()`