## Nosql
- RDBMS로 감당하기 어려울 때 사용하면 좋음
- 많은 경우 자주 사용하는 쿼리가 정해져있다. 대부분 지정된 키에 대한 정보를 `GET` 해오는 간단한 API 이다. 이런 경우 RDBMS가 아니라 수평적으로 확장 가능한 시스템이 필요하다.
- 그러나 항상 Nosql 이 필요한 것이 아니다. Mysql도 스케일아웃이 가능하다. 글로벌 서비스 정도의 크기면 문제가 됨
# HBase
- HDFS 위에 구축된 확장가능한 비관계형 데이터베이스
- API가 있어서 CRUD 작업이 가능. 트랜잭션을 제공한다, 매우 큰 데이터 작업 가능
- HMaster가 데이터가 어느 Region Server에 있는지 위치를 알고 있고, 지휘자 같은 역할을 한다.
- 예시
    - key: com.cnn.www
    - Contents column family
        Contents: <html> <head> CNN ~
    - Anchor column family
        Anchor:cnnsi.com "CNN"
        Anchor:my.look.ca "CNN.com"
- 간단한 사용은 Rest api, 성능은 Thrift나 Avro를 사용

## 실습
- 사용자 ID가 주어지면 그 사용자가 매긴 평점을 모두 찾아주는 서비스를 구축
    - UserID은 rating이라는 열 패밀리를 갖고, 모든 영화의 평점을 가지고 있다.
    - Rating:50 -> 열 패밀리 이름은 Rating, 50은 열 이름
    - Rating:50 = 1 이면 50번 영화에 평점 1점을 남겼다.
    ```python
    {
        "rating": {
            "50": "1",
            "751": "4",
            ...
            "295": "4"
        }
    }
    ```
- Python client -> Rest Service -> Hbase/HDFS
## PIG
- 파이썬으로는 한계가 있다.
- hbase에 파일을 올리고 hbase shell로 'user', 'userinfo' 를 create 한다.