---
layout: post
category: python
---

## **back_populates**

`back_populates`는 두 `relationship()`이 서로를 가리키고 있음을 SQLAlchemy에 알려줍니다. 이를 통해 **자동 동기화**가 가능해집니다.

```python
class User(Base):
    addresses = relationship("Address", back_populates="user")

class Address(Base):
    user_id = Column(ForeignKey("user.id"))
    user = relationship("User", back_populates="addresses")
```

이렇게 하면 `user.address` 로 접근이 가능하고, `address.user` 로도 서로 접근이 가능합니다.

중요한 것은 양쪽에 명시적으로 선언해야하고, 테이블 이름이 아니라 반대쪽 relationship 속성과 이름이 같아야 합니다.

## primaryjoin와 foreign_keys

Base 모델에 FK관련 코드가 없거나, 실제로 FK가 걸려있지 않으면 이 두 개는 반드시 지정해야합니다.

```python
class Order(Base):
    __tablename__ = 'orders'
    id = Column(Integer, primary_key=True)
    user_email = Column(String)  # ❌ ForeignKey 없음!

    # ❌ 이렇게 하면 에러: NoForeignKeysError
    # customer = relationship("User", back_populates="orders")

    customer = relationship("User", 
        foreign_keys="Order.user_email",
        primaryjoin="User.email == Order.user_email",
        back_populates="orders")
```

또한 동일 테이블에 다중 FK가 있을 때도 명시적으로 적어줘야 해요. 

## lazy, uselist, overlaps

**lazy**는 로딩 전략인데, 어떻게 연관된 데이터를 가져오냐를 명시해줍니다. 근데 중요한 것은 이걸 설정해줘도 `selectinload()`나 `joinedload()` 가 더 우선됩니다.

| 값 | 동작 | 언제 쓰나 |
| --- | --- | --- |
| `'select'` | 접근 시 별도 SELECT (기본값) | 대부분의 경우 |
| `'joined'` | JOIN으로 한 번에 로드 | 1:1, 자식 적을 때 |
| `'selectin'` | IN 절로 한 번에 로드 | 1:N 컬렉션 |
| `'dynamic'` | 쿼리 객체 반환 (`.all()` 해야 로드) | 관계에서 필터링 필요 |
| `'raise'` | 접근 시 에러 발생 (디버깅용) | N+1 탐지 |
| `'noload'` | 로드 안함 (성능 최적화) | 확실히 안 쓸 때 |

따라서 lazy를 따로 설정해두지 않으면 기본이 `select`라 1:N 관계에서 for문을 돌리면 N+1 문제가 발생합니다. 조심

**uselist**는 True로 하면 list[Child]를 반환하고, 아니면 단일 객체를 바노한합니다. 즉, 1:N이나 N:1 관계에서는 True, 1:1 때는 False를 주면 됩니다.

**overlaps**는 중복 로딩 방지입니다. 기본 값은 False이고, 중첩된 eager 로딩에서만 의미가 있는 파라미터에요.

예를 들어서 이런 상황이에요.

```python
# ❌ 중복 로딩 시도 → 성능 저하 또는 경고
stmt = select(User).options(
    joinedload(User.posts).joinedload(Post.comments),     # 1. JOIN으로 posts
    selectinload(User.posts).selectinload(Post.comments)  # 2. IN으로 posts (중복!)
)
```

여기에서 overlaps=True를 주먼, 첫 번째 로딩 결과를 재사용해서 이를 반환합니다. 아주 굿

<aside>
💡

1. User + posts (JOIN) ← 첫 번째
2. posts → comments (SELECT)
3. User.posts 이미 로딩됨 → posts ids 재사용
4. comments만 추가 로딩 (posts 생략!)
</aside>