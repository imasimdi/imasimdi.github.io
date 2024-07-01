---
layout: post
category: python
---

- 기존의 레포지토리 레이어의 로직에는 각 SQL문에 한번씩 `commit()`을 수행했다.
- 이렇게 되면 `session.begin()`이나 `session.begin_nested()`을 수행하더라도 `commit()`이 savepoint를 빠져 나가서 의미가 없어진다.
- 그래서 생각한 것이 commit을 flag의 True False로 관리하면 되지 않을까? 였다.

``` python
class CustomSession(Session):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.skip_commit = False

    def commit(self):
        if self.skip_commit:
            print("Commit skipped")
        else:
            super().commit()
```

- Session을 오버라이딩하여서 skip_commit이라는 플래그를 생성해준다.
- 그리고 skip_commit이 True일 때 커밋을 스킵하고 False이면 commit을 해준다.
- 그리고 Session을 만들 때 session maker에 해당 class를 지정해준다.

```python
SessionLocal = sessionmaker(... , class_=CustomSession)
```

- skip을 위한 Unit of Work를 만들어준다.

```python
class UnitOfWorkForSkip(AbstractUnitOfWork):
    def __init__(self, db: Session):
        self._db = db

    def __enter__(self):
        self._db.skip_commit = True
        return self

    def __exit__(self, *args):
        print('rollback')
        self._db.rollback()
        self._db.close()

    def commit(self):
        try:
            self._db.skip_commit = False
            self._db.commit()
            print('commit')
        except SQLAlchemyError:
            self.rollback()
            raise TransactionFailException

    def rollback(self):
        self._db.rollback()
```

- 해당 UoW는 `__enter__`시 skip_commit을 True로 하여서 컨텍스트 내부에서 일어나는 commit을 전부 스킵해준다.
- 그리고 `commit()`을 할 때 skip_commit 플래그를 다시 False로 바꿔줘서 실제 commit이 가능하게 만들어주고, commit을 진행한다.
- 커밋도중 오류가 나면 rollback을 시키고 예외를 발생시킨다. 마찬가지로 컨텍스트에서 빠져나갈 때 `commit()`을 명시하지 않았다면 rollback을 진행하고 세션을 종료시킨다.
- 사용 예시는 다음과 같다

```python
with UnitOfWorkForSkip(session) as uow
    insertLogic1(session)
    insertLogic2(session)
    uow.commit()
```

- 이렇게 기존 코드에 적용하면 해당 skip uow를 사용하지 않은 코드들도 영향이 가지 않은 채로 레거시를 현명하게 대처할 수 있다.