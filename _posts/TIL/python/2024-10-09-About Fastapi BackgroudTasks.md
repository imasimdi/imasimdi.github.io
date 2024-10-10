---
layout: post
category: python
---

## 개요

- FastAPI는 별도의 복잡한 설정 없이도 백그라운드 작업을 예약할 수 있다. `BackgroundTasks`의 `background_tasks.add_task` 을 사용하면 된다.
- FastAPI의 ASGI(Asynchronous Server Gateway Interface) 서버 + Starlette 기반이기 때문
- 이것의 장점은 기존의 동기 함수에서 비동기 함수를 **메인 이벤트 루프**에 얹을 수 있다는 것
- 레거시 코드에 새로운 I/O 로직을 추가해야할 때 유용하다.

## 코드 예시

```python
@app.post("/register-product")
def register_product(product: Product, background_tasks: BackgroundTasks):
    product_id = save_product_to_database(product)
    background_tasks.add_task(register_to_blockchain, product)
    
    return {"message": "Product registered successfully", "product_id": product_id}
```

- 여기서 `register_product`는 동기 함수이고 `register_to_blockchain`는 비동기 함수이다.
- `BackgroundTasks`의 `add_task`를 사용해서 작업을 등록해준다.
- 여기서 중요한 것은 `add_task`에서 받는 함수가 비동기 작업이냐, 비동기 작업이냐에 따라서 `BackgroundTasks`가 하는 작업을 달라진다.
- 만약 비동기 함수라면 fastapi의 메인 이벤트 루프에서 직접 실행된다. 즉 멀티스레딩이 아니라 코루틴 방식으로 실행된다.
    - 메인 이벤트 루프란?
        - ASGI 서버가 생성하는 이벤트 루프(ex: `uvicorn run`)
        - 비동기 작업 스케줄링 및 실행
        - I/O 작업 (네트워크 요청, 데이터베이스 쿼리 등) 관리
        - 코루틴 간 전환 조정
- 만약 동기 함수라면 `starlette.concurrency`의 `run_in_threadpool`를 통해서 별도의 스레드에서 실행된다.
- 실제 소스를 분석하면 더 쉽게 이해가 된다.

## 소스 코드

### starlette/background.py

```python
class BackgroundTasks(BackgroundTask):
    def __init__(self, tasks: typing.Optional[typing.Sequence[BackgroundTask]] = None):
        self.tasks = list(tasks) if tasks else []

    def add_task(
        self, func: typing.Callable[P, typing.Any], *args: P.args, **kwargs: P.kwargs
    ) -> None:
        task = BackgroundTask(func, *args, **kwargs)
        self.tasks.append(task)

    async def __call__(self) -> None:
        for task in self.tasks:
            await task()
```

- `BackgroundTasks`는 테스크들을 `BackgroundTask`로 객체화 시킨 후, `self.tasks`로 태스크를 자체 리스트에서 관리한다.

```python
class BackgroundTask:
    def __init__(
        self, func: typing.Callable[P, typing.Any], *args: P.args, **kwargs: P.kwargs
    ) -> None:
        self.func = func
        self.args = args
        self.kwargs = kwargs
        self.is_async = asyncio.iscoroutinefunction(func)

    async def __call__(self) -> None:
        if self.is_async:
            await self.func(*self.args, **self.kwargs)
        else:
            await run_in_threadpool(self.func, *self.args, **self.kwargs)
```

- `BackgroundTask` 에서는 `asyncio.iscoroutinefunction`를 통해서 해당 함수가 비동기냐, 동기냐 판단 한 후 비동기 함수면 그대로 `await`을 통해 메인 이벤트 루프에 등록한다.
- 만약 동기 함수라면 `run_in_threadpool`를 통해서 별도의 스레드에서 이 함수를 실행한다.

### starlette/concurrency.py

```python
async def run_in_threadpool(
    func: typing.Callable[P, T], *args: P.args, **kwargs: P.kwargs
) -> T:
    if kwargs:  # pragma: no cover
        # run_sync doesn't accept 'kwargs', so bind them in here
        func = functools.partial(func, **kwargs)
    return await anyio.to_thread.run_sync(func, *args)
```

- `run_in_threadpool`에서는 `functools.partial`을 통해서 키워드 인자가 포함된 복잡한 함수 호출을 키워드 인자 없이 호출 가능한 형태로 변환
    - 이미 키워드 인자를 바인딩 시켜두고 추가로 남은 위치의 인자들만 받게 만들어 준다.
- `anyio.to_thread.run_sync`로 동기 함수를 별도의 스레드로 실행하여 메인 이벤트 루프의 블로킹을 막는다.

## 주의점

- 내부 소스 코드를 보고나서 고려해야하는 점이 있다.
- 만약에 `BackgroundTask`를 통해서 **비동기 함수**를 백그라운드 작업을 스케줄링할 때, 내부에 블로킹 함수가 있다면 메인 이벤트 루프 전체가 **블로킹** 된다.
- 이는 일반적인 `async def`에서도 같은 상황이 발생하니 꼭 주의하자.

## 추가 지식

- 동기 함수를 별도의 쓰레드로 실행시켜주는 방법은 `asyncio.to_thread()`과 `ThreadPoolExecutor`가 있다.
- 파이썬 3.9부터는 더욱 고수준 API인 `asyncio.to_thread()`를 사용하자.
    - 내부적으로 스레드 풀을 자동으로 관리
    - 내부적으로 컨텍스트 관리
    - 간단한 비동기 작업이나 일회성 블로킹 작업에 적합
- 다만 더욱 세심하거나 복잡한 멀티스레딩 프로그래밍이 필요하면 저수준 API인 `ThreadPoolExecutor`을 사용하자.
    - 스레드 풀 관리 가능