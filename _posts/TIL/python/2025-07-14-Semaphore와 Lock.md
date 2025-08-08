---
layout: post
category: python
---

## `threading.Semaphore()`

세마포어는 대학시절 운영체제에서 배워 익숙한 이름입니다. 파이썬에서 세마포어는 동시에 접근할 수 있는 스레드의 수를 제한하는 기능입니다. 파라미터에 초기값을 지정할 수 있으며, 이 값이 허용하는 만큼 여러 스레드가 동시에 임계 구역에 들어갈 수 있습니다.

코드는 아주 간단해요.

```python
import threading

sema = threading.Semaphore(3)

def limited_access():
    with sema:
        pass
```

초기 값에 N을 넣어준다면 최대 N개의 스레드에 동시 접근을 할 수 있게 됩니다.

## `threading.Lock()`

락은 DB 락과 비슷하게 한 번에 하나의 스레드만 임계 구역에 들어갈 수 있도록 제한해줍니다. 익히 알고 있듯이 한 스레드가 Lock을 획득하면, 다른 스레드는 Lock이 해제될 때까지 대기합니다. 따라서 데드락이 발생할 수 있고, 락을 획득했으면 반드시 이를 풀어줘야 합니다.

컨텍스트 매니저를 사용하면 아주 간단하게 Lock을 제어할 수 있어요.

```python
import threading

lock = threading.Lock()

def critical_section():
    with lock:
        pass

```

사실상 `Semaphore(1)` 은 `Lock()` 과 같습니다.