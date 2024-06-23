---
layout: post
category: python
---
- `Architecture patterns with python` 에서의 UoW는 이렇게 정의된다.
    - 작업 단위의 패턴, 즉, 원자적 연산이라는 개념에 대한 추상화이다.
    - 작업의 트랜잭션 단위를 정하고, 서비스 계층의 직접적인 데이터 베이스 요청으로 부터 분리시킬 수 있다.
- 책에서 나온 UoW를 조금 변형에서, 레포지토리 클래스를 초기화 때 받아 여러 곳에서 재사용 가능하게 만들었다.

## 최종본
```python
class AbstractUnitOfWork(metaclass=ABCMeta):
    @abstractmethod
    def __enter__(self):
        pass

    @abstractmethod
    def __exit__(self):
        pass

    @abstractmethod
    def commit(self):
        pass

    @abstractmethod
    def rollback(self):
        pass

class UnitOfWork(AbstractUnitOfWork):
    def __init__(self, db: Session, repository_cls):
        self._db = db
        self._repository_cls = repository_cls

    def __enter__(self):
        self.repo = self._repository_cls(self._db)
        return self

    def __exit__(self, *args):
        self._db.rollback()
        self._db.close()

    def commit(self):
        try:
            self._db.commit()
        except SQLAlchemyError:
            self.rollback()
            raise

    def rollback(self):
        self._db.rollback()

class UnitOfWorkForRegacy(AbstractUnitOfWork):
    def __init__(self, db: Session):
        self._db = db

    def __enter__(self):
        return self

    def __exit__(self, *args):
        self._db.rollback()
        self._db.close()

    def commit(self):
        try:
            self._db.commit()
        except SQLAlchemyError:
            self.rollback()
            raise

    def rollback(self):
        self._db.rollback() 
```

- 추상 클래스를 만들고, 두 개의 uow가 상속했다.
- uow에 레포지토리를 초기화해 서비스 계층에서는 더 이상 레포지토리에 직접 의존하지 않아도 된다.
- 다만, db를 repository 메서드에 직접 넣는 코드 같은 경우, `db.commit()` 만 레포지토리 계층에서 서비스 걔층으로 빼서 하나의 트랜잭션으로 만들어주는 UnitOfWorkForRegacy도 만들어 주었다.