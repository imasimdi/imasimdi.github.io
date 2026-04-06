---
layout: post
category: python
---

`pool_pre_ping`은 SQLAlchemy의 **커넥션 풀에서 꺼낸 연결이 아직 살아 있는지 먼저 확인하는 옵션**입니다. 기본적으로 `create_engine(..., pool_pre_ping=True)`처럼 켜며, 보통 `SELECT 1` 같은 가벼운 쿼리로 연결 상태를 검사한 뒤 문제가 있으면 새 연결로 다시 잡습니다.

## 왜 필요한가

데이터베이스 연결은 오래 쉬어 있으면 서버 쪽 설정이나 네트워크 문제로 끊길 수 있습니다. 이때 풀에 남아 있던 “죽은 연결”을 그대로 쓰면 애플리케이션에서 `MySQL server has gone away` 같은 오류가 날 수 있어서, `pool_pre_ping`이 그런 상황을 줄여 줍니다. 이걸 제가 찾아본 이유가 저 에러가 떠서 일단 해결하기 위해 적용하기 위함입니다.

## 동작 방식

연결을 **빌려주는 시점** 에 먼저 짧게 ping을 날려서 살아 있는지 확인합니다. 검사에 실패하면 SQLAlchemy가 그 연결을 버리고 새 연결을 만들도록 처리합니다.

```python
from sqlalchemy import create_engine

engine = create_engine(
    "mysql+pymysql://user:password@host:3306/dbname",
    pool_pre_ping=True
)
```

이처럼 엔진 생성 시 옵션으로 넣는 방식이 일반적입니다.

## 주의할 점

`pool_pre_ping`은 **풀에서 꺼낼 때의 연결 유효성**을 확인해 주지만, 이미 쿼리를 실행하던 도중에 갑자기 끊기는 문제까지 완전히 막아 주지는 않습니다. 또한 체크가 한 번 더 들어가므로 아주 미세한 오버헤드는 있지만, 보통은 끊긴 연결로 인한 장애를 줄이는 이점이 더 큽니다.