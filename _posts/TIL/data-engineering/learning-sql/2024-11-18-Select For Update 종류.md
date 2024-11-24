## **일반 SELECT**

- 단순 읽기 작업
- 다른 트랜잭션의 읽기/쓰기를 막지 않음
- 동시에 여러 트랜잭션이 같은 데이터를 읽을 수 있음

## **SELECT FOR UPDATE**

```sql
BEGIN;
SELECT * FROM product_option WHERE id = 1 FOR UPDATE;
COMMIT;
```

- 배타적 잠금(쓰기 잠금)을 획득
- 다른 트랜잭션이 잠금을 이미 보유중이면 해제될 때까지 대기
- 데드락 위험이 있음

## **SELECT FOR UPDATE NOWAIT**

```sql
BEGIN;
SELECT * FROM product_option WHERE id = 1 FOR UPDATE NOWAIT;
COMMIT;
```

- 즉시 잠금을 획득하려고 시도
- 잠금을 획득할 수 없으면 즉시 에러 발생 → 대기하지 않음

## **SELECT FOR UPDATE SKIP LOCKED**

```sql
BEGIN;
SELECT * FROM product_option WHERE id IN (1,2,3) FOR UPDATE SKIP LOCKED;
COMMIT;
```

- 이미 잠겨있는 행을 건너뛰고 잠금이 가능한 행만 선택
- 큐 시스템이나 작업 분배에 유용

## **SELECT FOR SHARE**

```sql
BEGIN;
SELECT * FROM product_option WHERE id = 1 FOR SHARE;
COMMIT;
```

- 공유 잠금(읽기 잠금)을 획득
- 다른 트랜잭션의 읽기는 허용하지만 쓰기는 차단
- 오히려 많은 트랜잭션이 공유락을 들고 있으면 쓰기가 오래 차단될 수 있기 때문에 락 시간을 최소화하거나 타임아웃을 잘 정해두어야 한다.
- 정리하면 다음과 같다.

| 특징 | FOR UPDATE | NOWAIT | SKIP LOCKED | FOR SHARE |
| --- | --- | --- | --- | --- |
| 락 타입 | 배타적 | 배타적 | 배타적 | 공유 |
| 대기 여부 | 대기함 | 대기안함 | 건너뜀 | 대기함 |
| 동시성 | 낮음 | 중간 | 높음 | 높음 |
| 데드락 위험 | 높음 | 낮음 | 낮음 | 중간 |
| 용도 | 단일 레코드 업데이트 | 빠른 실패 필요 시 | 작업 분배 시스템 | 읽기 정합성 보장 |

## **SELECT FOR UPDATE SKIP LOCKED 예시**

- 정확히 어떻게 작동하는지 알기 위해서 또 파이썬 코드를 가져왔습니다.
    
    ```python
    def transaction1():
        with engine.connect() as conn:
            with conn.begin():
                print("Transaction 1 - Starting")
    
                result = conn.execute(
                    text("SELECT * FROM deposits WHERE status = 'PENDING' FOR UPDATE")
                )
                print("Transaction 1 - Selected rows:", result.mappings().all())
    
                time.sleep(3)
    
                conn.execute(
                    text("UPDATE deposits SET amount = amount - 500 WHERE status = 'PENDING'")
                )
                print("Transaction 1 - UPDATE executed")
    
                print("Transaction 1 - Committing")
    
    def transaction2():
        with engine.connect() as conn:
            with conn.begin():
                print("Transaction 2 - Starting")
    
                result = conn.execute(
                    text("SELECT * FROM deposits WHERE status = 'PENDING' FOR UPDATE SKIP LOCKED")
                )
                rows = result.mappings().all()
                print("Transaction 2 - Selected rows (SKIP LOCKED):", rows)
    
                if rows:
                    conn.execute(
                        text("UPDATE deposits SET status = 'SUCCESS' WHERE status = 'PENDING' FOR UPDATE SKIP LOCKED")
                    )
                    print("Transaction 2 - UPDATE executed")
    
                result = conn.execute(text("SELECT * FROM deposits"))
                print("Final state in Transaction 2:", result.mappings().all())
    
    def run_test():
        with engine.connect() as conn:
            with conn.begin():
                conn.execute(text("DELETE FROM deposits"))
                conn.execute(text("""
                    INSERT INTO deposits (hash, amount, status)
                    VALUES
                        ('user1', 1000, 'PENDING'),
                        ('user2', 1000, 'PENDING'),
                        ('user3', 1000, 'PENDING')
                """))
    
        with engine.connect() as conn:
            result = conn.execute(text("SELECT * FROM deposits"))
            print("Initial state:", result.mappings().all())
    
        thread1 = threading.Thread(target=transaction1)
        thread2 = threading.Thread(target=transaction2)
    
        thread1.start()
        time.sleep(1)
        thread2.start()
    
        thread1.join()
        thread2.join()
    
        with engine.connect() as conn:
            result = conn.execute(text("SELECT * FROM deposits"))
            print("\\nFinal state after both transactions:", result.mappings().all())
    
    if __name__ == "__main__":
        run_test()
    
    ```
    
- 시나리오
    - deposits 테이블에 3개의 PENDING 상태 row를 생성
    - transaction1이 먼저 시작되어 모든 PENDING 상태의 row에 대해 FOR UPDATE로 lock을 건다.
    - transaction2는 FOR UPDATE SKIP LOCKED를 사용하여 잠기지 않은 row만 선택하려 시도
- 결과
    
    ```python
    Initial state: [{'id': 68, 'hash': 'user1', 'amount': 1000, 'status': 'PENDING'}, {'id': 69, 'hash': 'user2', 'amount': 1000, 'status': 'PENDING'}, {'id': 70, 'hash': 'user3', 'amount': 1000, 'status': 'PENDING'}]
    Transaction 1 - Starting
    Transaction 1 - Selected rows: [{'id': 68, 'hash': 'user1', 'amount': 1000, 'status': 'PENDING'}, {'id': 69, 'hash': 'user2', 'amount': 1000, 'status': 'PENDING'}, {'id': 70, 'hash': 'user3', 'amount': 1000, 'status': 'PENDING'}]
    Transaction 2 - Starting
    Transaction 2 - Selected rows (SKIP LOCKED): []
    Final state in Transaction 2: [{'id': 68, 'hash': 'user1', 'amount': 1000, 'status': 'PENDING'}, {'id': 69, 'hash': 'user2', 'amount': 1000, 'status': 'PENDING'}, {'id': 70, 'hash': 'user3', 'amount': 1000, 'status': 'PENDING'}]
    Transaction 1 - UPDATE executed
    Transaction 1 - Committing
    Final state after both transactions: [{'id': 68, 'hash': 'user1', 'amount': 500, 'status': 'PENDING'}, {'id': 69, 'hash': 'user2', 'amount': 500, 'status': 'PENDING'}, {'id': 70, 'hash': 'user3', 'amount': 500, 'status': 'PENDING'}]
    ```
    
    - `Transaction 2 - Selected rows (SKIP LOCKED): []` 에서 보면 알 수 있듯이, 트랜잭션1에서 lock을 걸고 있는 동안에는 트랜잭션 2에서는 아무 것도 select 해오지 못한다.
    - 데드락은 피할 수 있을 것 같은데, 재고 문제를 해결하기에는 조금 애매한 것 같다. 이유는 다음과 같다.
        - 먼저 주문한 사람보다 나중에 주문한 사람이 구매할 수 있음
        - 대기 시간과 무관하게 우연히 락이 해제된 시점의 요청이 성공
    - 이 얼마나 불합리한가! 역시 빠른 놈 위에 운 좋은 놈이다.