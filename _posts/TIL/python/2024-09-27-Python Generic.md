---
layout: post
category: python
---

## `Generic[T]`

```python
from typing import TypeVar

T = TypeVar('T')
```

- `TypeVar`는 타입 변수를 생성, 실제 타입을 나중에 지정할 수 있는 플레이스 홀더 역할
- `T`는 관례적인 이름
- 함수나 클래스가 여러 타입에 대해 동작할 수 있게 만들 때 사용
- 입력 타입과 출력 타입 사이의 관계를 표현 가능

```python
from typing import TypeVar, List

T = TypeVar('T')

def first(l: List[T]) -> T:
    return l[0]
int_result = first([1, 2, 3])
str_result = first(["a", "b", "c"])
```

- 제네릭을 통해서 파이썬에서도 타입 안정성을 가져갈 수 있다.

```python
from typing import TypeVar, Union

Number = TypeVar('Number', int, float)
```

- 이렇게 `int`, `float` 등 TypeVar을 생성할 때 제약조건을 추가할 수 있다.

## 클래스에서의 Generic

- `Generic`은 **클래스가** 하나 이상의 타입 변수를 사용하는 제네릭 클래스임을 나타낸다.
- 이를 통해 클래스 내부에서 타입 변수를 사용 가능하게 함

```python
from typing import TypeVar, Generic

T = TypeVar('T')

class Box(Generic[T]):
    def __init__(self, content: T):
        self.content = content

    def get_content(self) -> T:
        return self.content

int_box = Box[int](5)
str_box = Box[str]("Hello")

print(int_box.get_content())  # 5
print(str_box.get_content())  # Hello

# 타입 체크도 가능하다
reveal_type(int_box.get_content())  # Revealed type is "int"
reveal_type(str_box.get_content())
```