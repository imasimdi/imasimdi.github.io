---
layout: post
category: etc
tags: [redis]
---

Redis의 WATCH는 트랜잭션의 일부로 사용되는 낙관적 락 메커니즘입니다. 데이터베이스 용어에서 **Compare and Set** 작업과 유사한 개념입니다. 데이터를 수정하기 전에 다른 클라이언트가 해당 데이터를 변경했는지 확인할 수 있게 해줍니다. `watch` 메서드는 다음과 같은 특징을 갖습니다.

- 특정 키의 변경을 감시
- 감시 중인 키가 변경되면 트랜잭션이 실패
- Check-and-Set 패턴을 구현할 때 유용

간단하게 코드로도 살펴보겠습니다.

```python
import redis

r = redis.Redis()

pipe = r.pipeline()

while True:
    try:
        # 키를 감시 시작
        pipe.watch('counter')
        
        # 현재 값 가져오기
        current_value = int(pipe.get('counter') or 0)
        
        # 트랜잭션 시작
        pipe.multi()
        
        # 값 업데이트
        pipe.set('counter', current_value + 1)
        
        # 트랜잭션 실행
        pipe.execute()
        
        # 성공하면 루프 종료
        break
        
    except redis.WatchError:
        # 다른 클라이언트가 값을 변경했다면 재시도
        continue
```

- `watch()` 명령으로 특정 키들을 감시하기 시작합니다. 이때부터 Redis는 해당 키들의 변경을 감지합니다.
- `get` 을 통해서 감시 중인 키들의 현재 값을 읽어옵니다.
- `multi()` 명령으로 트랜잭션을 시작합니다. 이후의 명령들은 즉시 실행되지 않고 큐에 쌓입니다.
- `execute()` 명령으로 큐에 쌓인 모든 명령을 실행합니다. 이때 감시 중인 키가 변경되지 않았다면 모든 명령이 성공적으로 실행됩니다.
- 만약 감시 중인 키가 변경되었다면 `WatchError`가 발생하고, 트랜잭션은 실패하여 실행되지 않습니다. `continue` 를 통해서 전체 과정을 재시도 합니다.

## multi()는 어떻게 동작하는가?

트랜잭션이 시작되는 과정을 내부 코드로 살펴보았습니다.

```python
def multi(self) -> None:
    """
    Start a transactional block of the pipeline after WATCH commands
    are issued. End the transactional block with `execute`.
    """
    if self.explicit_transaction:
        raise RedisError("Cannot issue nested calls to MULTI")
    if self.command_stack:
        raise RedisError(
            "Commands without an initial WATCH have already been issued"
        )
    self.explicit_transaction = True
```

- `multi()` 메서드가 시작되면 클라이언트의 `explicit_transaction` 를 True로 변경시킵니다.
- Redis 클래스의 메서드인 `pipeline()` 로 생성된 `Pipeline` 객체는 마지막에 오버라이딩 된`execute_command()` 을 실행시킵니다. `execute_command()`를 살펴보면 `explicit_transaction` 가 `True` 이면 `pipeline_execute_command()` 를 실행시킵니다.

```python
def execute_command(self, *args, **kwargs):
    if (self.watching or args[0] == "WATCH") and not self.explicit_transaction:
        return self.immediate_execute_command(*args, **kwargs)
    return self.pipeline_execute_command(*args, **kwargs)
```

- `pipeline_execute_command` 를 살펴보면 `command_stack` 이라는 리스트에 command를 append 시킵니다.

```python
def pipeline_execute_command(self, *args, **options) -> "Pipeline":
    """
    Stage a command to be executed when execute() is next called

    Returns the current Pipeline object back so commands can be
    chained together, such as:

    pipe = pipe.set('foo', 'bar').incr('baz').decr('bang')

    At some other point, you can then run: pipe.execute(),
    which will execute all commands queued in the pipe.
    """
    self.command_stack.append((args, options))
    return self
```

- 마지막에 `execute()`를 살펴보면 `command_stack` 에 쌓아두었던 command들을 실행시킵니다.