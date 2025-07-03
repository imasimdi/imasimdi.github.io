---
layout: post
category: data-engineering
subcategory: learning-sql
---


`REPLACE INTO`는 INSERT와 UPDATE의 기능을 결합한 문법인데요, 사용법은 기존 INSERT INTO와 동일합니다.

```sql
REPLACE INTO 테이블명 (컬럼1, 컬럼2, ...) 
VALUES (값1, 값2, ...);

-- 또는
REPLACE INTO 테이블명 
SET 컬럼1 = 값1, 컬럼2 = 값2, ...;
```

이렇게 동일한 문법으로 UPSERT를 쳐주다니, 정말 간편한 친구입니다. 동작 방식은 이렇습니다.

1. PRIMARY KEY나 UNIQUE 제약조건이 중복되지 않는 경우에는 일반적인 INSERT처럼 새 행을 삽입합니다.
2. PRIMARY KEY나 UNIQUE 제약조건이 중복되는 경우에는 기존 행을 삭제하고 새 행을 삽입합니다.

users 테이블로 간단한 예시를 들어보겠습니다.

```sql
-- 사용자 테이블 예시
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    email VARCHAR(100) UNIQUE
);

-- 새 사용자 삽입 (id=1이 존재하지 않는 경우)
REPLACE INTO users (id, name, email) 
VALUES (1, 'hyunje', 'hyunje@email.com');

-- 기존 사용자 교체 (id=1이 이미 존재하는 경우)
REPLACE INTO users (id, name, email) 
VALUES (1, 'hyunje Lee', 'hyunje.lee@email.com');
```

이렇게 간단하지만 주의해야할 점이 있는데요, 바로 데이터 손실 위험입니다.

당연하게도 `REPLACE INTO`는 기존 행을 완전히 삭제하고 새 행을 삽입하므로, 명시하지 않은 컬럼의 데이터가 손실될 수 있습니다.

따라서 데이터의 중요도가 높지 않은 곳에서 사용해야합니다. 저는 fcm token 테이블에서 간단한게 REPLACE INTO를 사용하였습니다. 토큰은 사라져도 되니까요!

또 주의해야하는 점이 `REPLACE INTO`는 기존 레코드를 삭제한 후 새로 삽입하므로, **AUTO_INCREMENT 컬럼의 값은 반드시 증가합니다.**

그래서 대안이 `INSERT ... ON DUPLICATE KEY UPDATE` 가 있다고 합니다. 

```sql
INSERT INTO users (id, name, email) 
VALUES (1, 'Hyunje lee', 'hyunje.lee@email.com')
ON DUPLICATE KEY UPDATE 
    name = VALUES(name), 
    email = VALUES(email);
```

이런 식으로 하면 기존 행을 삭제하지 않고, 필요한 컬럼만 업데이트를 시켜준다고 하는데, 이쯤 되면 그냥 애플리케이션 수준에서 변경하는 게 맞겠네요