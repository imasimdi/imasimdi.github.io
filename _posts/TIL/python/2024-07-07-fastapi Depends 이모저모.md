---
layout: post
category: python
---

## 1. Request를 비롯한 특정 개체는 자동으로 주입된다.

```python
class CustomRequest:
    def __init__(self, body) -> None:
        self.body = body

async def create_custom_request(request: Request) -> CustomRequest:
    body = await request.json()  # JSON 본문을 가져옵니다.
    return CustomRequest(body=body)

async def get_request(custom_request: Annotated[CustomRequest, Depends(create_custom_request)]):
    return custom_request

@app.get("/info/")
async def get_info(result = Depends(get_request)):
    return {"path": result}
```

- CustomRequest라는 객체는 request의 body만 담는다.
- 조금 꼬아 놓은 예시긴 한데, request와 같은 객체는 Depends 내부에서도 알아서 의존성이 주입된다. 순서는 다음과 같다.
    - 경로 함수 호출: 사용자가 /info/ 경로에 GET 요청
    - 의존성 주입: FastAPI는 `get_info` 함수가 `Depends(get_request)`를 통해 의존성을 주입받는 것을 확인
    - 의존성 해결: `get_request` 함수가 호출되기 전에 FastAPI는 `get_request` 함수의 매개변수인 `custom_request: Annotated[CustomRequest, Depends(create_custom_request)]`를 해결해야 함
    - 의존성 체인 해결: `create_custom_request` 함수가 Request 객체를 필요로 하기 때문에 FastAPI는 **자동으로** 현재 요청의` Request` 객체를 `create_custom_request` 함수에 전달
    - CustomRequest 생성: `create_custom_request` 함수는 `Request` 객체를 사용하여 JSON 본문을 읽고 `CustomRequest` 객체를 생성
    - 경로 함수 실행: 생성된 `CustomRequest` 객체가 `get_request` 함수에 전달되고, 최종적으로 `get_request` 함수의 반환 값이 `get_info` 함수에 전달
- Response, Cookie 등등의 객체도 마찬가지다. `Depends(get_request)` 에서 request에 대한 정보를 넘겨주지 않았는데 알아서 잘 받아와서 신기했다.

## 2. Annotated를 사용하라

- Depends를 명시하는데 여러가지 방법이 있다.

```python
custom_request: Annotated[CustomRequest, Depends(CustomRequest)]
```

```python
custom_request: CustomRequest = Depends(CustomRequest)
```

```python
custom_request: CustomRequest = Depends()
```

- 이 중 Annotated를 사용하면 의존성과 타입 힌트가 명확하게 구분되어 코드의 가독성이 향상된다.

## 3. 제네레이터를 적극 활용하라

```python
from typing import Annotated
from fastapi import FastAPI, Depends

app = FastAPI()

class DepC:
    def __init__(self, value: int):
        self.value = value

async def dependency_c():
    dep_c = DepC(value=42)
    try:
        yield dep_c
    finally:
        print("리소스 정리 중...")

@app.get("/items/")
async def read_items(dep_c: Annotated[DepC, Depends(dependency_c)]):
    return {"value": dep_c.value}
```

- 실행 순서는 다음과 같다.
    - `/items/` 엔드포인트에 대한 요청이 들어오면, FastAPI는 read_items 함수를 호출
    - read_items 함수의 매개변수에 주입될 의존성을 확인
    - 이 경우 dep_c는 `Depends(dependency_c)`에 의해 제공
    - dependency_c 비동기 제너레이터 함수가 호출되고, yield를 통해 DepC 객체를 생성하여 반환
    - read_items 함수가 dep_c 매개변수를 받아 실행
    - read_items 함수가 완료되고, 반환값이 클라이언트에게 응답으로 전송
    - read_items 함수가 완료된 후에 dependency_c의 finally 블록이 실행되어 리소스 정리
- 즉, 모든 요청이 종료되고 다시 제네레이터로 돌아오기 때문에 리소스를 정리해야하는 객체면 아주 유용하다.
- Depends로 `next()`나 `for`를 하지 않아도 yield가 가능하니까 편리하다.