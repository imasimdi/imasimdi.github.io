---
layout: post
category: data-engineering
subcategory: hadoop
tags: [hadoop]
---

- spark와 조금이라도 더 친해지기 위해서 kaggle의 데이터를 받아와서 가벼운 실습을 진행하였다.
- https://www.kaggle.com/datasets/blastchar/telco-customer-churn 에서 csv파일로 된 어떤 통신회사의 정보를 받아왔다.

## 실습

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("Telco 회원 분석") \
    .getOrCreate()
```

- `builder`를 통해서 세션을 만든다.

```python
data_file_path = "Telco-Customer-Churn.csv"
data = spark.read.csv(data_file_path, header=True, inferSchema=True)
```
- `read.csv` 을 통해서 csv 파일을 불러온다.
- `header`: CSV 파일의 첫 번째 줄이 컬럼 이름(헤더)으로 사용된다는 것을 의미한다.
    
    ```
    name,age,city
    Alice,30,New York
    Bob,25,Los Angeles
    ```

    - 이렇게 되어있을 때 name, age, city가 칼럼 이름으로 사용된다.

- `inferSchema`: PySpark가 각 컬럼의 데이터 타입을 자동으로 추론하도록 한다.
    - 예를 들어, 숫자 형식의 데이터가 포함된 컬럼은 IntegerType이나 DoubleType으로, 날짜 형식의 데이터가 포함된 컬럼은 DateType으로 추론한다.
    - 만약 `inferSchema=False`로 설정하면, 모든 컬럼의 데이터 타입이 기본적으로 StringType으로 설정된다.

```python
data = data.na.drop()
```

- 컬럼별 Null 값을 제거한다.

```python
selected_data = data.select("gender", "SeniorCitizen", "Partner", "Dependents", "tenure", 
                            "PhoneService", "InternetService", "MonthlyCharges", "TotalCharges", "Churn", "Constract")
```

- `select`을 통해서 특정 컬럼을 선택한다.

```python
churn_rate = selected_data.groupBy("Churn").count()
churn_rate.show()
"""
+-----+-----+
|Churn|count|
+-----+-----+
|   No| 5174|
|  Yes| 1869|
+-----+-----+
"""
```

- 고객 이탈 여부를 나타낸다. No는 이탈한 고객, Yes는 이탈한 고객이다.

```python
avg_monthly_charges = selected_data.groupBy("InternetService").avg("MonthlyCharges")
avg_monthly_charges.show()

"""
+---------------+-------------------+
|InternetService|avg(MonthlyCharges)|
+---------------+-------------------+
|    Fiber optic|  91.50012919896615|
|             No| 21.079193971166454|
|            DSL|  58.10216852540261|
+---------------+-------------------+
"""
```

- InternetService: 인터넷 서비스 유형
- avg(MonthlyCharges): 해당 서비스 유형의 평균 월 요금

```python
age_distribution = selected_data.groupBy("SeniorCitizen").count()
age_distribution.show()

"""
+-------------+-----+
|SeniorCitizen|count|
+-------------+-----+
|            1| 1142|
|            0| 5901|
+-------------+-----+
"""
```

- 고령자 여부를 나타낸다. 1은 고령자, 0은 고령자가 아니다.


```python
contract_distribution = selected_data.groupBy("Contract").count()
contract_distribution.show()
"""
+--------------+-----+
|      Contract|count|
+--------------+-----+
|Month-to-month| 3875|
|      One year| 1473|
|      Two year| 1695|
+--------------+-----+ 
"""
```

- Contract: 계약 유형
- count: 각 계약 유형에 속한 고객의 수

```python
# 고객 기본 정보 데이터프레임
customer_info = data.select('customerID', 'gender', 'SeniorCitizen', 'Partner', 'Dependents')

# 서비스 정보 데이터프레임
service_info = data.select('customerID', 'InternetService', 'MonthlyCharges', 'TotalCharges')

merged_data = customer_info.join(service_info, on='customerID', how='inner')
filtered_data = merged_data.where(col('MonthlyCharges') > 50)
```

- 기존 데이터 셋을 고객의 기본 정보와, 서비스 정보 데이터프레임으로 나눈다.
- customerID를 기준으로 조인을 실행하고, 서비스 정보 중에서 MonthlyCharges가 50 이상인 데이터만 추출한다.


## 스터디 회고
- 8주에 걸쳐서 Hadoop 에코시스템에 대한 전반적인 이해를 하는 시간
- 분산 시스템에 대해서 조금 더 이해하게 됨
    - ex) pandas vs pyspark
- 하둡에서의 수 많은 데이터 처리 방법에 대해서 알게됨
    - HIVE, HBase, Pig, Spark, Scoop 등등
- 제일 재밌었던 것은 직접 클라우드에 하둡을 설치한 것
- 뿐만 아니라 다양한 데이터베이스와 알맞은 선택 기준에 대해서 알게됨
    - 가장 기억에 남는 것: CAP(일관성, 가용성, 파티션 저항성)
- 중간에 발표인 디스코드의 DB 변천사도 준비하면서, 훗날 나도 DB 마이그레이션을 해보면 좋은 경험이 되겠다고 생각했음