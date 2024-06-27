---
layout: post
category: python
---

## sessionmaker의 autocommit
- `sessionmaker`에는 `autocommit`이라는 파라미터가 있다. 이름만 들으면 True일 때 자동으로 `commit()`을 호출해주는 것 같다.
- 하지만 알고보면 저 `autocommit`은 **Session의 트랜잭션 자동시작** 과 연관이 있다.
- `autocommit`가 **False** 면 세션의 트랜잭션, 즉 `session.begin()`이 자동으로 시작되고 *TRUE*면 `session.begin()`을 명시적으로 적어줘야 트랜잭션이 시작된다.
- 즉, `autocommit`가 False일 때 `session.begin()`을 하게되면 이미 시작된 트랜잭션을 또 시작하게 되기 때문에 해당 에러를 반환한다.

```
InvalidRequestError: A transaction is already begun. Use subtransactions=True to allow subtransactions
```

## autocommit이 False일 때 `begin_nested()`
- 현재 기존 코드에서는 repository 코드 내부에 `commit()`이 메서드 마다 호출하고 있다.
- 하지만 해당 메서드들에 의존하고 있는 다른 메서드들이 너무 많아, 쉽사리 변경했다가는 어떤 사이드 이펙트를 불러올 지 모르는 상황에서, 트랜잭션을 관리해줘야됐다.
- 앞선 deep dive에서 탐구했듯, 이럴 때 필요한 것이 `begin_nested()` 이다. 이미 autocommit이 False이기 때문에 트랜잭션은 시작되며 `session.begin()` 이후 `commit()` 을 만나면 바로 트랜잭션이 종료되기 때문에 `begin_nested()` 를 사용해서 savepoint를 만들어 준다.
- context manager를 사용해서 로직을 수행해보았다.

```python
def insertLogic1(session):
    query = """

    """
    session.execute(text(query))
    db.commit()

with session.begin_nested():
    insertLogic1(session)
    insertLogic2(session)
```

- 이렇게 수행하면 context manager가 트랜잭션의 시작과 끝을 자동으로 관리하기 때문에 `commit()`과 `rollback()`을 명시하지 않아도 된다.
- 하지만 정작 이를 수행하면 에러가 발생한다.

```
sqlalchemy.exc.InvalidRequestError: Can't operate on closed transaction inside context manager. Please complete the context manager before emitting further commands.
```

- 에러를 해석해보면 context manager의 내부에서 이미 닫힌 트랜잭션에서 작업을 수행하려고 할 때 문제가 발생한다는 것이다.
- with 구문을 사용하면 context manager가 트랜잭션의 시작과 끝을 자동으로 관리한다. insertLogic1의 내부에 `commit()`이 명시적으로 되어있는 것이 문제인데, 이 명시적인 호출이 트랜잭션을 종료시키고 이후의 작업은 다음 트랜잭션에서 처리된다.
- 즉, 자동으로 트랜잭션을 관리하는 context manager 안에서 명시적인 `commit()`의 발생은 트랜잭션을 종료시키는 예상치 못한 동작이다. 따라서 해결책은 다음과 같다.

```python
try:
    session.begin_nested()
    insertLogic1(session)
    insertLogic2(session)
    session.commit()
except:
    session.rollback()
```

- 이렇게 context manager를 사용하지 않고 직접 `commit()`과 `rollback`을 사용하면 save point의 역할을 제대로 수행한다.
- context manager에 트랜잭션 관리를 위임하지 않고, 직접 시작하여 `commit()`과 `rollback()` 을 명시하면, 오류가 발생하지 않는다.
-  insertLogic1와 insertLogic2의 내부에서 `commit()`이 일어나도 save point 역할을 하여 `session.commit()`가 되기 전까지는 `commit()`이 제대로 실행되지 않는다.

## 결론
- 이로써 기존의 레거시 코드를 건들지 않고 트랜잭션을 관리할 수 있게 되었다.