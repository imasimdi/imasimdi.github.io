---
layout: post
category: python
---

세션 문제가 발생해서, 잊지 않기 위해 다시 한번 정리했습니다.

특정 데이터베이스 행(Row)이 단일 `Session` 내에서 오직 하나의 파이썬 객체로만 존재하도록 보장하는 메커니즘이다.

쉽게 말하자면 DB의 특정 데이터베이스 행을  파이썬 메모리 상의 단일 객체와 1:1로 매핑해주는 내부 저장소가 되겠다.

SQLAlchemy에서는 보통 DB와 상호작용을 할 때 `Session` 객체를 사용한다. 이 `Session` 객체 내부에 Identity Map를 갖고 있고, PK를 기준으로 객체를 관리한다. 따라서 다음과 같은 상황에서 딱히 변경된 값을 반환하지 않아도, 알아서 유지된다.

예시코드다.

```python

def sync_box_tracking(session, box_id: int, injected_box_objects: list[BoxObject]) -> None:
    """
    내부 로직: DB에서 특정 박스들을 새로 조회하여 상태를 업데이트합니다.
    """
    box_object_to_update = session.query(BoxObject).get(box_id) 
    box_object_to_update.status = 'DELIVERED'

def caller_function(session):
    my_boxes = session.query(BoxObject).filter(BoxObject.id == 1).all()
    target_box = my_boxes[0]
    print(f"함수 호출 전 상태: {target_box.status}" # 출력: 'SHIPPED'
    sync_box_tracking(session, target_box.id, my_boxes)

    print(f"함수 호출 후 상태: {target_box.status}")  # 출력: 'DELIVERED'
```

Identity Map는 영속상태에서만 존재하는데, 이참에 SQLAlchemy의 생명주기에 대해서도 정리했다.

## SQLAlchemy 객체의 생명주기

### Transient

```python
new_user = User(name="이현제", age=29)
```

파이썬 코드 레벨에서 인스턴스로 생성된 상태

### Pending

```python
session.add(new_user)
```

Session에 추가되었지만, 아직 `Flush`나 `Commit` 이 되지 않고, 대기열에 서 있는 상태

### Persistent

Pending인 객체가 `Flush`나 `Commit` 되어서 실제 데이터베이스에 존재하는 상태이거나, 쿼리를 통해 조회한 객체들의 상태이다.

```python
session.commit() 
# commit(내부적으로 flush 포함)이 발생하여 DB에 INSERT

# 쿼리로 가져온 객체도 Persistent 상태입
existing_user = session.get(User, 1)
```

위의 Identity Map에서 관리되고 있는 객체들이다. 이 상태의 객체 속성을 변경하면 (setter를 통해서 바꾸거나.. 직접 속성 접근을 해서 바꾸거나..) 세션이 이를 추적하고 다음 `Flush`나 `Commit` 에서 이를 UPDATE 쿼리로 알아서 변경해준다.

### Detached

Session이 닫히거나, 객체를 명시적으로 Session에서 제거한 상태이다. 이 상태에서 값을 수정해도 변경되지 않고, 지연로딩 속성에 접근하려고 하면 `DetachedInstanceError`가 발생한다.