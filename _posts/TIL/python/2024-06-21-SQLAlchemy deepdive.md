---
layout: post
category: python
---
- 막연하게 그냥 사용하기 보다는 engine, session, transaction에 대해서 제대로 알고 사용하자.

## engine
- engine은 SQLAlchemy에서 데이터베이스와 연결을 관리하는 핵심 객체이다.
- engine은 데이터베이스와의 연결을 설정하고 연결 풀을 유지하고, 연결을 생성 및 재활용

```python
from sqlalchemy import create_engine
engine = create_engine(
    url,
    encoding = 'utf-8',
    pool_size = int,
    max_overflow = int,
    pool_recycle = int
)
```

### parameter
- pool_size
    - 연결 풀의 크기를 지정
    - 연결 풀은 데이터베이스와의 연결을 재사용하여 성능을 향상
    - ex) pool_size=10 (최대 10개의 연결을 유지)
- max_overflow
    - 기본 연결 풀 크기 외에 추가로 생성할 수 있는 연결의 최대 개수를 지정. 기본값은 10
    - ex) max_overflow=20 (기본 풀 크기 외에 최대 20개의 추가 연결 허용)
- pool_recycle
    - 연결이 재활용되기 전에 대기할 최대 시간(초)을 지정합니다. 기본값은 -1(무제한)입니다. 이는 특정 시간 이후에 연결을 재활용하여 오래된 연결을 닫고 새로 여는 데 사용됩니다.
    - ex) pool_recycle=3600 (연결이 1시간 후에 재활용됨)

### `connection()`
- `engine.connection() as connection`을 통해서 `connection.excute()`를 실행할 수 있다.

```python
from sqlalchemy import create_engine, text

# 데이터베이스 연결 생성
engine = create_engine('sqlite:///example.db')

# 연결 객체 생성
with engine.connect() as connection:
    # SQL 쿼리 실행
    result = connection.execute(text("SELECT * FROM user"))
    for row in result:
        print(row)

```

## Session
- 데이터베이스와의 대화와 관련된 모든 작업을 관리하며, 트랜잭션을 관리하고, ORM 객체를 추적하며, 변경 사항을 데이터베이스에 반영한다.

### sessionmaker
- 세션 객체 (Session)를 생성하는 팩토리 함수
    - bind: Session 객체가 사용할 데이터베이스 엔진
    - autocommit: 자동으로 커밋을 할 것인가?
    - autoflush: 세션이 자동으로 플러시(변경 사항을 데이터베이스에 반영)할지 여부, 기본값 True
    - expire_on_commit: 세션이 커밋 후 객체를 만료시킬지 여부
- sessionmake에서 만들어 지는 것은 sessionmaker 객체(예시: SessionFactory)이고, Session 객체를 만들기 위해서는 `SessionFactory()`를 해야한다.

### `session.begin()`
- SQLAlchemy에서 트랜잭션을 시작하는 방법 중 하나
- 이 메서드를 호출하면 새로운 트랜잭션이 생성되고, 이후의 DB 작업들은 이 트랜잭션의 일부로 간주
- 트랜잭션이 종료되기 전까지는 DB에 대한 변경 사항이 커밋되거나 롤백되지 않는다.
- 트랜잭션 종료 조건은 `session.commit()`이나 `session.rollback()`을 해야한다
- `session.begin()`은 `transaction` 객체를 반환한다. 이는 많은 시사점을 야기한다.

### `session.begin()` VS `transaction = session.begin()`
1. `session.begin()`만 호출하기

```python
session.begin()

try:
    # 데이터베이스 작업 수행
    session.commit()
except:
    session.rollback()
    raise
finally:
    session.close()
```

- 트랜잭션 객체를 명시적으로 다루지 않아도 된다. 코드가 간결해진다.
- 일반적인 경우에는 이 방법을 사용한다.

2. `transaction` 객체를 반환하고, 이를 다루기

```python
transaction = session.begin()

try:
    # 데이터베이스 작업 수행
    transaction.commit()
except:
    transaction.rollback()
    raise
finally:
    session.close()

```

- 이 경우 transaction 객체를 통해서 트랜잭션 상태를 더 세밀하게 제어할 수 있다.
- 예를 들어 트랜잭션 상태를 확인하거나 특정 시점에서 커밋 또는 롤백을 수행한다.
- 코드가 조금 더 명확하게 트랜잭션의 범위를 나타낼 수 있다.

### `session.begin()`과 `with session.begin():` 의 차이점
- `with session.begin():`를 사용하면 context manager를 통해서 트랜잭션을 관리
- 트랜잭션의 시작과 종료를 자동으로 처리해주며, 블록을 벗어날 때 자동으로 커밋, 예외 발생시 자동으로 롤백한다.
- 하지만 `session.close()`는 자동으로 되지 않기 때문에 명시적으로 해줘야한다.
- `session.begin()`는 명시적으로 commit()과 rollback()을 제시한다.

### `session.begin_nested()`
- 저장점이나 서브트랜잭션을 생성하여 트랜잭션 내에서 부분적으로 롤백할 수 있는기능

```python
session = Session()

try:
    # 메인 트랜잭션 시작
    session.begin()
    
    # 첫 번째 작업
    user1 = User(name='User1')
    session.add(user1)
    
    # 서브트랜잭션 시작 (저장점 설정)
    session.begin_nested()
    
    try:
        # 두 번째 작업
        user2 = User(name='User2')
        session.add(user2)
        
        # 일부러 예외 발생
        raise ValueError("Something went wrong in the nested transaction")
        
        # 서브트랜잭션 커밋
        session.commit()
    except Exception as e:
        # 서브트랜잭션 롤백
        session.rollback()
        print(f"Nested transaction rolled back: {e}")
    
    # 메인 트랜잭션 커밋
    session.commit()
    print("Main transaction committed successfully.")
except Exception as e:
    # 메인 트랜잭션 롤백
    session.rollback()
    print(f"Main transaction rolled back: {e}")
finally:
    # 세션 닫기
    session.close()
```

- begin_nested를 통해서 트랜잭션 안의 트랜잭션을 만들 수 있다.
- 가령 테스트 중, 테스트 이후 모든 DB는 롤백이 되어야 할 때, 테스트 내부의 `commit()`이후에도 마지막에 `rollback()`이 되어야 할 때 사용할 수 있다.
- 밑의 예시처럼 `session.begin()`에는 SessionTransaction 객체를 반환하고, `session.begin_nested()`는 savepoint를 반환하여(이 또한 SessionTransaction이다) 명시적으로 관리하는 것도 좋은 방법이다.

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# 데이터베이스 엔진 생성
engine = create_engine('sqlite:///example.db')

# 세션 생성기
Session = sessionmaker(bind=engine)

# 세션 생성
session = Session()

try:
    # 트랜잭션 시작
    transaction = session.begin()
    
    # 데이터베이스 작업 수행
    # 예: session.add(new_record)
    
    # Savepoint 생성
    savepoint = session.begin_nested()

    # 내부 로직에서 commit이 필요한 경우
    try:
        # 일부 작업 수행
        # 예: session.add(some_record)
        
        # 커밋
        savepoint.commit()
        
        # 커밋 이후의 상황을 테스트
        # 예: some_record = session.query(SomeModel).filter_by(id=some_id).one()
        
    except Exception as e:
        # Savepoint 롤백
        savepoint.rollback()
        print(f"An error occurred during the inner transaction: {e}")
    
    # 최종적으로 모든 변경 사항을 롤백
    transaction.rollback()
except Exception as e:
    print(f"An error occurred during the outer transaction: {e}")
finally:
    # 세션 닫기
    session.close()
```

### `session.close()`
- 필요성
    - 데이터베이스 연결은 자원을 소모한다.
    - 커넥션 풀을 반환해서 다른 작업에 연결을 재사용할 수 있다.
    - 메모리 누수를 방지하고 메모리 사용량을 줄인다.
- session의 생명주기
    - 이건 애플리케이션의 요구사항에 따라 다르다.
    - 단일 트랜잭션 범위
    - 단일 요청 범위 등등
- `transaction = session.begin()`을 했을 때 `transaction.close()`는 필요한가?
    - 엄연히 말하자면 `session.close()`와 `transaction.close()`는 다르다.
    - `transaction.close()`는 해당 트랜잭션 객체를 닫는거고 `session.close()`는 세션을 닫는 것이다.
    - `transaction.close()`도 명시적으로 적어주는 것이 좋다.