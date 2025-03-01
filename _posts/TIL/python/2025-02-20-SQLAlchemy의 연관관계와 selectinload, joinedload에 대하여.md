---
layout: post
category: python
---

## SQLAlchemy에서의 양방향 관계

sqlalchemy에서는`back_populates` 을 사용해서 연관관계를 설정해줍니다. 실제 DB 관계까지 정의하는 스프링 엔티티와는 다르게 파이썬의 `relationship()`은 실제 데이터베이스 컬럼이 아닌 Python 객체 레벨의 참조입니다. 즉, 파이썬 내에서 참조를 할 수 있도록 도와주는 헬퍼 정도 역할을 해줍니다.

```python
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    # posts를 통해 User -> Post 접근
    posts = relationship("Post", back_populates="user")

class Post(Base):
    __tablename__ = "posts"
    id = Column(Integer, primary_key=True)
    title = Column(String)
    user_id = Column(Integer, ForeignKey("users.id"))
    # user를 통해 Post -> User 접근
    user = relationship("User", back_populates="posts")

```

일반적으로 User와 Post가 1:N 관계를 맺고있다면, 아래와 같이 데이터에 접근을 할 수 있습니다.

```python
# User -> Post 접근
user = db.query(User).first()
user.posts  # 해당 유저의 모든 게시글

# Post -> User 접근
post = db.query(Post).first()
post.user  # 해당 게시글의 작성자

```

`back_populates` 은 양쪽 모델에 명시적으로 관계를 정의해야합니다. 하지만 `backref` 을 사용하면 한쪽에만 정의하면 자동으로 반대쪽 관계가 생성되나, 명시으로 관계를 표현해주는 `back_populates` 를 더 많이 사용한다고 합니다.
## SQLAlchemy의 selectinload 과 joinedload
각각의 로딩 방식은 다음과 같은 차이점이 있습니다. 우선 간단한 예시로 다음과 같은 모델이 있다고 가정해보겠습니다

```python
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class Author(Base):
    __tablename__ = 'authors'
    
    id = Column(Integer, primary_key=True)
    name = Column(String)
    books = relationship("Book", back_populates="author")

class Book(Base):
    __tablename__ = 'books'
    
    id = Column(Integer, primary_key=True)
    title = Column(String)
    author_id = Column(Integer, ForeignKey('authors.id'))
    author = relationship("Author", back_populates="books")
```

Author는 여러 개의 책을 가질 수 있기 때문에 Book과 1 대 N 관계이고, 반대로 Book의 저자는 Author 한 명이기 때문에(공동 저자 없다는 가정 하에) N대 1 관계입니다.

Author에서는 `books` 라는 필드로 Book과 연관관계를 생성하였고, Book에서는 `author`  라는 필드로 Author와 연관관계를 설정하였습니다.

이런 상황에서 Author와 Book을 Join을 해서 가져와야할 때, N + 1 조회문제를 회피하기 위해 `selectinload`와 `joinedload`를 다음과 같이 사용할 수 있습니다.

### selectinload

우선 Author의 모든 Book을 가져오고 싶을 때는

```python
authors = session.query(Author).options(selectinload(Author.books)).all()
```

를 사용하고, 실제 쿼리는 다음과 같이 나갑니다.

```sql
-- 첫 번째 쿼리: 모든 저자 조회
SELECT authors.id, authors.name 
FROM authors;

-- 두 번째 쿼리: 조회된 저자들의 책 목록을 한 번에 가져옴
SELECT books.id, books.title, books.author_id 
FROM books 
WHERE books.author_id IN (
    /* 첫 번째 쿼리에서 조회된 저자 id들 */
    1, 2, 3, ...
);
```

우선 첫 번째 커리에서 모든 저자를 조회하여 authors.id를 가져옵니다. 이후에 두 번째 쿼리에서는 조회된 저자들의 책 목록을 IN절로 불러옵니다.

즉, 두 개의 분리된 쿼리로 실행되며, IN 절을 사용해 필요한 데이터만 효율적으로 가져옵니다. 이후에 다음과 같은 코드에서 추가 쿼리가 나가지 않고 book의 속성에 접근할 수 있습니다.

```python
book_title_list: list[str] = []

for author in authors:
		for book in author.books:
				book_title_list.append(book.title)
```

### joinedload

반대로 Book을 가져올 때 Author의 정보 또한 가져오고 싶을 때는 

```python
books = session.query(Book).options(joinedload(Book.author)).all()
```

를 사용하고, 실제 쿼리는 다음과 같이 나갑니다.

```sql
SELECT 
    books.id AS books_id,
    books.title AS books_title,
    books.author_id AS books_author_id,
    authors_1.id AS authors_1_id,
    authors_1.name AS authors_1_name
FROM books 
LEFT OUTER JOIN authors AS authors_1 
    ON authors_1.id = books.author_id;
```

우리가 아는 Outer join을 사용해서 Book과 연관관계에 있는 author의 정보까지 가져옵니다. 이렇게 books를 가져오면 다음과 같은 코드에서 추가 쿼리가 나가지 않게 됩니다.

```python
author_name_list: list[str] = []

for book in books:
		author_name_list.append(book.author.name)
```

### 각각 최적으로 사용하기

실제 쿼리를 보면 알 수 있듯이 각각이 쓰이는 상황이 다릅니다.

`joinedload` 의 경우에는 1:1 또는 1:N 관계에서 데이터가 적을 때 사용하기에 편리합니다. 우선 join을 통한 단일 쿼리이기 때문에, 조인으로 인한 불필요하게 큰 테이블에서의 중복 데이터가 발생할 수 있기 때문입니다.

`selectinload` 의 경우에는 1:N 상황에서 주로 사용합니다. 1:N 관계에서 `joinedload`를 사용하면 다음과 같은 데이터 중복이 발생합니다.

```python
# joinedload 사용 시
authors = session.query(Author).options(joinedload(Author.books)).all()
# SQL 실행 결과:
# Author1, Book1
# Author1, Book2  <- Author1 데이터 중복
# Author1, Book3  <- Author1 데이터 중복
# Author2, Book4
# Author2, Book5  <- Author2 데이터 중복
```

이렇게 N이 크고, 연관시켜야하는 테이블의 개수가 많을 수록 많은 데이터가 중복되고 그만큼 데이터가 커지게 되어서 메모리가 낭비 됩니다.

```python
# selectinload 사용 시
authors = session.query(Author).options(selectinload(Author.books)).all()
# 첫 번째 쿼리 결과:
# Author1
# Author2
# 두 번째 쿼리 결과:
# Book1, Book2, Book3 (Author1의 책들)
# Book4, Book5 (Author2의 책들)
```

반면에 `selectinlaod`를 사용하게 되면 `WHERE IN` 절을 사용하기 때문에 필요한 데이터만 가져와 효율적입니다.

## 연관관계를 사용하지 않을 때

만약에 연관관계가 걸려있지 않을 때, `Author.books` 과 같은 코드를 사용하고 싶을 때 어떻게 해야할까요?

우선은 `Author.books` 는 사용하지 못하더라도, `joinedload`와 유사하게 다음과 같이 직접 조인을 하여 데이터에 접근할 수 있습니다.

```python
# 직접 join을 사용한 방식
def get_author_with_books(author_id: int):
    return session.query(
        Author,
        Book
    ).outerjoin(
        Book,
        and_(
            Author.id == Book.author_id,
            Book.is_deleted == False
        )
    ).filter(
        Author.id == author_id,
        Author.is_deleted == False
    ).first()
```

단, 이렇게되면 실제 반환 값은 튜플 형식이며 다음과 같이 데이터에 접근해야합니다.

```python
# get_author_with_books의 반환값
result = (Author_instance, Book_instance)

# 이 경우 접근 방식
author, book = result
```

그렇다면 `selectinload` 를 직접 구현하면 어떻게 될까요?

```python
def get_author_with_books_manual_selectin(author_id: int):
    # 1. 먼저 Author 정보를 가져옴
    author = session.query(Author).filter(
        Author.id == author_id,
        Author.is_deleted == False
    ).first()
    
    # 2. 해당 author_id에 해당하는 모든 books를 별도 쿼리로 가져옴
    if author:
        books = session.query(Book).filter(
            Book.author_id == author.id,
            Book.is_deleted == False
        ).all()
        
        # 3. author 인스턴스의 books 컬렉션에 수동으로 할당
        author.books = books
    
    return author
```

이런 식으로 구현할 수 있지만, 실제로 `selectinload` 는 추가적으로 캐싱, 세션 관리, 트랜잭션 관리까지 하기 때문에 `selectinload`을 쓰는 것이 더 유리합니다.

그렇다면 `joinedload` 를 사용하고 싶을 때, 추가적인 조인 조건을 얻고 싶으면 어떻게 해야할까요? 정답은 `contains_eager` 를 사용하는 것입니다.

```python
result = session.query(Author).outerjoin(
    Book,
    and_(
        Author.id == Book.author_id,
        Book.rating >= 4.0  # 추가 필터링
    )
).options(
    contains_eager(Author.books)
).filter(
    Author.id == author_id
).first()
```

만약 조인을 해올 때, `Book.rating` 와 같이 추가적인 필터링을 사용하기 위해서는 `outerjoin` 과 `contains_eager` 를 사용할 수 있습니다. 이렇게 되면 `author.books` 에서 for 문을 돌아도 추가적인 쿼리가 발생하지 않아 `joinedload` 을 했을 때와 같이 객체를 사용할 수 있습니다.

또한 다음과 같이 JOIN된 테이블의 컬럼으로 정렬이 필요한 경우에도 사용합니다. 

```python
result = session.query(Author).outerjoin(
    Book
).options(
    contains_eager(Author.books)
).order_by(
    Book.published_date.desc()
).first()
```

하지만 SQLAlchemy에서 연관관계를 맺을 때, 미리 join 조건을 걸어 줄 수 있습니다. 코드 재사용성을 높이고 조인 조건을 빼먹는 휴먼 에러를 줄이기 위해서라도 연관관계에서 조건을 걸어주는 것이 좋습니다.

다음과 같이 연관관계에서 조건을 걸어줄 수 있습니다.

```python
class Author(Base):
    __tablename__ = 'authors'
    
    id = Column(Integer, primary_key=True)
    name = Column(String)
    
    # 문자열 대신 직접 표현식 사용
    fantasy_books = relationship(
        "Book",
        primaryjoin=lambda: and_(
            Author.id == Book.author_id,
            Book.genre == 'fantasy'
        )
    )
    
    popular_recent_books = relationship(
        "Book",
        primaryjoin=lambda: and_(
            Author.id == Book.author_id,
            Book.rating >= 4.0,
            Book.published_year >= 2020
        )
    )
```

lambda를 사용하지 않고 “”로 단순 문자열로도 접근이 가능합니다.