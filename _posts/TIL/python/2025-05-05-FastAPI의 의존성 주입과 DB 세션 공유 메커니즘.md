---
layout: post
category: python
---

## Fastapi의 의존성 주입

의존성 주입은 함수나 객체가 필요로 하는 의존성을 직접 생성하지 않고 외부로부터 제공 받는 방식을 말합니다. fastapi의 의존성 주입은 `Depends` 함수를 통해서 구현합니다.

흔히 DB session은 다음과 같이 주입받습니다.

```python
from fastapi import Depends

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/items/")
def read_items(db: Session = Depends(get_db)):
    items = db.query(Item).all()
    return items
```

## Depends를 활용한 DI 팩토리 함수

이를 더 응용해서, 저희 서비스에서는 fastapi의 `Depends` 를 사용해서 유사 DI Container 팩토리 함수를 만들어 의존성을 관리하고 있습니다. 예를 들어 service_b가 service_a를 의존할 때, 다음과 같이 팩토리 함수를 정의할 수 있습니다.

```python
def get_service_a(db: Session = Depends(get_db)):
    return ServiceA(db)

def get_service_b(db: Session = Depends(get_db), service_a: ServiceA = Depends(get_service_a)):
    return ServiceB(db, service_a)

@app.get("/endpoint")
def endpoint(service_b: ServiceB = Depends(get_service_b)):
    return service_b.do_something()
```

여기에서 드는 의문점은 다음과 같습니다.

> 팩토리 함수 각각이 `get_db` 를 호출하면 다른 DB Session을 갖는 것이 아닐까?
> 

하나의 로직에서 두 개의 DB Session을 갖고 있으면 데이터 정합성이 맞지 않을 수 있는 등의 불확실성이 높아집니다. 따라서 b가 a를 의존하고 있으면 같은 DB Session을 갖도록 강제하는 것이 좋습니다.

하지만 실제로는 serivice a와 service b는 같은 DB Session을 갖습니다. 

### DB 세션 공유의 작동 원리

FastAPI는 동일한 요청 내에서 동일한 의존성 함수를 여러 번 호출해도 같은 인스턴스를 반환합니다. 이는 FastAPI 공식 문서의 [서브 의존성 섹션](https://fastapi.tiangolo.com/tutorial/dependencies/sub-dependencies/)에서 확인할 수 있습니다.

> "하나의 경로 작업에 대해 같은 의존성이 여러 번 선언된 경우, FastAPI는 해당 의존성을 요청당 한 번만 호출하고 그 결과를 캐시에 저장하여 필요한 모든 곳에 전달합니다."
> 

FastAPI는 내부적으로 의존성 그래프를 구축하고 동일한 요청 처리 과정에서 동일한 의존성 함수에 대한 결과를 캐싱합니다. 이를 통해 DB 세션과 같은 객체를 효율적으로 관리할 수 있습니다.

따라서 위의 예시코드에서는 `get_db()` 함수는 한 번만 호출되고, 동일한 DB 세션 객체가 `get_service_a`와 `get_service_b` 모두에 전달됩니다. 왜나하면 FastAPI의 의존성 해결 메커니즘이 동일한 요청 내에서 동일한 의존성에 대해 같은 인스턴스를 재사용하기 때문입니다.

## 정말 같은 세션인지 테스트해보기

다음과 같은 두 가지 코드에서 정말 같은 DB Session을 공유하는지 알 수 있습니다.

우선 하나의 엔드포인트에서 동시에 DB 세션을 Depends했을 때, 같은 객체인지 확인합니다.

```python
@app.get("/check-session-objects/")
def check_session_objects(
    db1: Session = Depends(get_db),
    db2: Session = Depends(get_db)
):
    # 두 세션 객체가 동일한지 확인
    return {
        "same_session_instance": id(db1) == id(db2),
        "db1_id": id(db1),
        "db2_id": id(db2)
    }
```

결과는 다음과 같습니다.

```python
{
    "same_session_instance": true,
    "db1_id": 4424298800,
    "db2_id": 4424298800
}
```

이 엔드포인트를 호출하면 `same_session_instance`가 `true` 으로 반환하고, 이는 `get_db` 의존성이 동일한 요청 내에서 동일한 세션 객체를 반환한다는 것을 보여줍니다.

다음으로는 서비스가 다른 서비스에 의존하는 경우의 테스트 코드입니다.

```python
# 의존성 주입 컨테이너
def get_item_service(db: Session = Depends(get_db)):
    return ItemService(db)

def get_item_analysis_service(
    db: Session = Depends(get_db),
    item_service: ItemService = Depends(get_item_service)
):
    return ItemAnalysisService(db, item_service)
```

이렇게 ItemAnalysisService는 ItemService를 의존한다고 했을 때 예시 서비스 코드를 다음과 같이 정의합니다.

```python
class ItemService:
    def __init__(self, db: Session):
        self.db = db
        
    def get_db_id(self):
        return id(self.db)

class ItemAnalysisService:
    def __init__(self, db: Session, item_service: ItemService):
        self.db = db
        self.item_service = item_service
    
    def compare_db_id(self):
        return id(self.db) == self.item_service.get_db_id()
        
@app.get("/compare")
def analyze_items(service: ItemAnalysisService = Depends(get_item_analysis_service)):
    return service.compare_db_id()
```

그리고 해당 엔드포인트를 호출해보면 결과는 `true` 를 얻을 수 있습니다.