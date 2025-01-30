---
layout: post
category: data-engineering
subcategory: learning-sql
tags: [DB]
---

## Repeatable Read 고찰

- 데이터베이스의 격리수준에서 가장 널리 사용되는 Repeatable Read에 대해서 더욱 고찰해보았다.
- Repeatable Read 격리 수준에서는 첫 번째 트랜잭션에서 읽은 데이터의 스냅샷을 유지한다.

```sql
CREATE TABLE deposits (
    id INT PRIMARY KEY AUTO_INCREMENT,
    hash VARCHAR(50),
    amount INT,
    status VARCHAR(20)
);

-- 테스트 데이터 삽입
INSERT INTO deposits (hash, amount, status) VALUES 
    ('user1', 1000, 'PENDING'),
    ('user2', 1000, 'PENDING')
```

- 하지만 자신의 트랜잭션 내 변경사항은 볼 수 있다.
- 이러한 상태에서 해당 SQL을 실행해보자.

```sql
START TRANSACTION;
    SELECT * FROM deposits WHERE status = 'PENDING';
    UPDATE deposits SET status = 'SUCCESS' WHERE hash = 'user1';
    SELECT * FROM deposits WHERE hash = 'user1';
COMMIT;
```

> '1', 'user1', '1000', 'SUCCESS'
'2', 'user2', '1000', 'PENDING'

- 마지막 SELECT 절에서 보인 결과는 다음과 같다.
- 즉, 같은 트랜잭션 내에서의 변경사항은 바로바로 적용이 되어 SELECT 절에 보이게 된다.
- Save Point도 마찬가지이다.

```sql
START TRANSACTION;
    SAVEPOINT sp1;
        SELECT * FROM deposits WHERE status = 'PENDING';
        UPDATE deposits SET status = 'SUCCESS' WHERE hash = 'user1';
        COMMIT;
    
    SAVEPOINT sp2;
		SELECT * FROM deposits WHERE hash = 'user1';
        COMMIT;
COMMIT;
```

- 이것도 위와 같은 결과를 보이게 된다.
- 하지만 다른 트랜잭션에서 UPDATE한 것은 어떻게 보일까?

```sql
-- 트랜잭션 1
START TRANSACTION;  -- 이 시점의 스냅샷을 사용
    SELECT * FROM deposits WHERE status = 'PENDING';
    -- 결과: user1, user2 두 건이 PENDING 상태로 조회

    -- 이 시점에 트랜잭션 2가 실행되어 변경을 수행
    
    SELECT * FROM deposits WHERE hash = 'user1' AND status = 'PENDING';
    -- 여전히 user1, user2 두 건이 PENDING 상태로 조회됨
    -- 트랜잭션 2의 변경사항은 보이지 않음
COMMIT;

-- 트랜잭션 2
START TRANSACTION;
    UPDATE deposits SET status = 'SUCCESS' hash = 'user1';
COMMIT;
```

- 하지만 이렇게 SQL을 작성하게 되면 메인 트랜잭션 속의 트랜잭션에서 실행된 것이라 ‘**자신의 트랜잭션 내 변경사항은 볼 수 있다**‘ 때문에 user1의 상태가 `SUCCESS`로 보인다. 제대로 된 테스트를 위해 Python code를 짜보자

```python
import threading
import time
from sqlalchemy import create_engine, text

# DB 연결 설정
engine = create_engine('mysql+pymysql://root:0000@localhost/test')

def transaction1():
    with engine.connect() as conn:
        with conn.begin():
            # 첫 번째 SELECT
            result1 = conn.execute(text("SELECT * FROM deposits WHERE status = 'PENDING'"))
            print("Transaction 1 - First SELECT:", result1.mappings().all())
            
            # 트랜잭션2를 위한 잠시 대기
            time.sleep(2)
            
            # 두 번째 SELECT
            result2 = conn.execute(text("SELECT * FROM deposits WHERE status = 'PENDING'"))
            print("Transaction 1 - Second SELECT:", result2.mappings().all())

def transaction2():
		time.sleep(1)
    with engine.connect() as conn:
        with conn.begin():
            conn.execute(text("UPDATE deposits SET status = 'SUCCESS' WHERE hash = 'user1'"))
            print("Transaction 2 - UPDATE executed")

def run_test():
    # 테스트 데이터 초기화
    with engine.connect() as conn:
        with conn.begin():
            conn.execute(text("DELETE FROM deposits"))
            conn.execute(text("""
                INSERT INTO deposits (hash, amount, status) 
                VALUES ('user1', 1000, 'PENDING'), ('user2', 1000, 'PENDING')
            """))

    thread1 = threading.Thread(target=transaction1)
    thread2 = threading.Thread(target=transaction2)
    
    thread1.start()
    thread2.start()
    
    thread1.join()
    thread2.join()

if __name__ == "__main__":
    run_test()
```

- 코드 내용은 다음과 같다.
    - 두 개의 별도 스레드에서 각각 트랜잭션을 실행
    - 트랜잭션 1에서 SELECT 실행 후 잠시 대기
    - 그 동안 트랜잭션 2에서 UPDATE 실행
    - 트랜잭션 1에서 다시 SELECT 실행
- 이렇게 별도의 스레드에서 트랜잭션을 진행하게 되면 다음과 같은 결과를 얻을 수 있다.

```python
Transaction 1 - First SELECT: [{'id': 24, 'hash': 'user1', 'amount': 1000, 'status': 'PENDING'},
{'id': 25, 'hash': 'user2', 'amount': 1000, 'status': 'PENDING'}]
Transaction 2 - UPDATE executed
Transaction 1 - Second SELECT: [{'id': 24, 'hash': 'user1', 'amount': 1000, 'status': 'PENDING'},
{'id': 25, 'hash': 'user2', 'amount': 1000, 'status': 'PENDING'}]
```

- Repeatable Read 격리 수준에서는 트랜잭션 1의 두 번째 SELECT도 첫 번째와 동일한 결과를 얻게 된다.
- 트랜잭션 2의 변경사항은 트랜잭션 1이 커밋되고 **새로운 트랜잭션**이 시작될 때까지 반영되어 보이지 않는다.
- 하지만 두번째 SELECT에서 `FOR UDPATE` 를 붙인다면 다음과 같은 결과를 얻을 수 있다.

```python
Transaction 1 - First SELECT: [{'id': 26, 'hash': 'user1', 'amount': 1000, 'status': 'PENDING'},
{'id': 27, 'hash': 'user2', 'amount': 1000, 'status': 'PENDING'}]
Transaction 2 - UPDATE executed
Transaction 1 - Second SELECT: [{'id': 27, 'hash': 'user2', 'amount': 1000, 'status': 'PENDING'}]
```

- 이렇게 두번째 SELECT에도 반영된 결과과 보이는 이유는 `SELECT FOR UPDATE`는 Repeatable Read 격리 수준에서도 최신 데이터를 읽어오는 특별한 케이스이기 때문이다.
- 해당 이유는 다음과 같다.
    - SELECT FOR UPDATE는 해당 row에 대해 쓰기 잠금(write lock)을 획득하려고 시도
    - 쓰기 잠금을 획득하기 위해서는 해당 row의 최신 상태를 알아야 함
    - 따라서 트랜잭션 시작 시의 스냅샷이 아닌, 현재 커밋된 데이터를 읽음
    - 이러한 동작은 데이터 일관성과 동시성 제어를 위한 것
        - 데이터를 업데이트하기 전에 **최신 상태**를 확인하고 싶을 때
        - 여러 트랜잭션에서 **동시에 같은 데이터를 수정하는 것을 방지**하고 싶을 때

## 그렇다면 UPDATE의 WHERE절은 어떤 데이터를 볼까?

- 테스트를 설계하자.
    - user1, user2는 amount가 각각 1000원이다.
    - 트랜잭션 1에서 user1의 amonut를 0원으로 update 한다.
    - 트랜잭션 1이 끝나기 전에 트랜잭션 2가 시작되고 UPDATE WHERE절에서 amount가 0이면 SUCCESS로 업데이트 해준다.

```python
def transaction1():
    with engine.connect() as conn:
        with conn.begin():
            print("Transaction 1 - Starting")
            
            # UPDATE 전 상태 확인
            result1 = conn.execute(text("SELECT * FROM deposits WHERE hash = 'user1'"))
            print("Transaction 1 before UPDATE:", result1.mappings().all())
            
            conn.execute(text("UPDATE deposits SET amount = amount - 1000 WHERE hash = 'user1'"))
            print("Transaction 1 - UPDATE executed")
            
            time.sleep(3)  # transaction2에게 기회를 줌
            
            print("Transaction 1 - Committing")

def transaction2():
    with engine.connect() as conn:
        with conn.begin():
            print("Transaction 2 - Starting")
            # amount = 0인 row의 status를 'SUCCESS'로 변경
            conn.execute(text("UPDATE deposits SET status = 'SUCCESS' WHERE amount = 0"))
            print("Transaction 2 - UPDATE executed")
            
            # 최종 상태 확인
            result = conn.execute(text("SELECT * FROM deposits"))
            print("Final state in Transaction 2:", result.mappings().all())
```

- 결과는 다음과 같다.

```python
Transaction 1 - Starting
Transaction 1 before UPDATE: [{'id': 62, 'hash': 'user1', 'amount': 1000, 'status': 'PENDING'}]
Transaction 1 - UPDATE executed
Transaction 2 - Starting
Transaction 1 - Committing
Transaction 2 - UPDATE executed
Final state in Transaction 2: [{'id': 62, 'hash': 'user1', 'amount': 0, 'status': 'SUCCESS'}, {'id': 63, 'hash': 'user2', 'amount': 1000, 'status': 'PENDING'}]
```

- 결과를 보면 트랜잭션 1이 끝나지 않고 중간에 트랜잭션 2를 시작했음에도 불구하고 트랜잭션 2의 UPDATE WHERE 절은 트랜잭션 1에서 amount가 0원이된 user1의 status를 `success` 로 잘 변경해줬다.
- 즉, UPDATE의 WHERE 절은 repeatable read 격리수준에서도 항상 최신의 데이터를 확인하며 업데이트를 진행한다.
- 이는 데이터 정합성 보장하기 위해서이며 현재 데이터를 기준으로 평가하고 lock을 획득함으로써 변경 충돌을 방지할 수 있기 때문이다.
- 참고로 트랜잭션 2의 UPDATE를 진행하기 전에 몇 초간 대기하게 되는데, 이는 UPDATE는 자동으로 락을 획득하기 때문이다.
    - UPDATE 문은 자동으로 행(row) 레벨의 배타적 락을 획득한다.
    - UPDATE문이 실행되면
        - WHERE 절에 매칭되는 행들에 대해 배타적 락을 획득
        - 이 lock은 트랜잭션이 `commit` 또는 `rollback`될 때까지 유지
    - Exclusive lock의 특징
        - 다른 트랜잭션은 이 행을 읽을 수도 쓸 수도 없다.
        - 즉, 다른 트랜잭션의 `SELECT FOR UPDATE`, `UPDATE`, `DELETE` 등이 차단됨
        - 일반 SELECT는 격리 수준에 따라 동작이 다르다. Repeatable Read에서는 select은 스냅샷 이전의 데이터를 읽게 된다.

## 결론

1. 일반 SELECT은 MVCC를 통해 트랜잭션 시작 시점의 스냅샷을 읽음
2. SELECT FOR UPDATE은 현재 실제 데이터를 읽고 row-level 잠금을 획득
3. UPDATE의 WHERE절은 격리 수준과 상관없이 항상 최신 데이터를 얻음
4. UPDATE는 자동으로 행레벨의 배타적 락을 얻음