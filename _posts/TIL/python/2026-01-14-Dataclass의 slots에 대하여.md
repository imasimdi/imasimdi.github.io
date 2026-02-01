---
layout: post
category: python
---

## `slots=True`에 관하여

**`slots=True`**는 Python 3.10+에서 dataclass에 추가된 옵션으로, 이를 설정하면  **`__slots__`**를 자동 생성해 메모리 사용량과 속성 접근 속도가 크게 좋아집니다.

우선 `__dict__` 딕셔너리를 생성하지 않기 때문에 메모리를 절약해주고, 속성 접근을 할 때, 배열 인덱싱 방식을 사용합니다.

이 말이 무슨 말이냐면, slots=True를 주지 않으면 다음과 같이 dataclass의 속성을 조회합니다.

```python
obj.name  →  obj.__dict__['name'] 
```

이 조회도 O(1)로 상당히 빠르지면 slots=True를 주게 되면 배열을 사용해서 속성에 접근하게 되어서 속도가 더 빨라집니다.

```python
obj.name  →  obj._fields[0]
```

## Trade Off

하지만 당연히 Trade Off도 있는데요,

우선 동적으로 속성을 추가하는 것이 불가능해집니다.

예를 들어

```python
@dataclass(slots=True)
class UserDTO:
    id: int
    name: str
```

이런 dataclass가 있을 때,

```python
obj = UserDTO(1, "이현제")
obj.extra = "value" # <-- AttributeError
```

이렇게 동적으로 속성을 set을 해주게 되면 에러가 발생합니다. 하지만 저는 오히려 setter를 사용 못하게 강제해주기 때문에 오히려 좋다는 생각이 드네요

또, 앞서 말했듯이 `__dict__` 메서드를 사용하지 못하기 때문에, dictionary로 변경할 때 내부 메서드인 `asdict`만 사용할 수 있습니다.

또한 상속을 할 때도 부모와 자식 모두 slots=True 옵션을 줘야하기 때문에 매우 번거롭습니다.

제 생각에는 해당 옵션은 그냥 데이터 처리 배치나, 극단적으로 API Call 속도를 줄이고 싶을 때 사용할 것 같고, 그 외 DTO에는 사용하지 않을 것 같습니다.