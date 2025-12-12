---
layout: post
category: etc
tags: [redis]
---

오늘 지급 정산 로직을 작성하면서 계좌 잔액 확인 시점에 동시성 제어가 필요했습니다. 잔액이 부족하면 Slack 알림을 보내고 Interactivity로 재시도하는 구조라서, 잔액 체크와 송금 사이에 race condition이 발생하면 안 됩니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/0f2cad72-1caa-4fca-96a3-8f821d33b814/image.png)

Redis Lock을 사용하기로 했는데 비슷해 보이는 파라미터가 두 개 있어서 정리했습니다.

## timeout vs block_timeout

### timeout
Lock을 획득한 후 유지되는 최대 시간입니다. 이 시간이 지나면 자동으로 해제됩니다.

```python
lock = redis.lock("payment:balance_check", timeout=10)
lock.acquire()
# 10초 후 자동 해제됨
```

데드락 방지용입니다. 작업이 너무 오래 걸리거나 예외가 발생해도 일정 시간 후엔 무조건 풀립니다.

### block_timeout
Lock 획득을 기다리는 최대 시간입니다. 다른 프로세스가 Lock을 잡고 있으면 이 시간 동안 대기합니다.

```python
lock = redis.lock("payment:balance_check", timeout=10)
acquired = lock.acquire(blocking=True, block_timeout=5)

if not acquired:
    # 5초 동안 기다렸는데도 못 잡으면 False
    raise LockAcquisitionError()
```

`blocking=True`면 Lock이 풀릴 때까지 계속 기다리고, `block_timeout`을 주면 그 시간만큼만 기다립니다.

## 정리
- `timeout`: Lock을 잡고 있을 수 있는 시간
- `block_timeout`: Lock을 기다릴 수 있는 시간

Redis의 Lock 객체를 쓰면 `SET NX`로 직접 구현할 필요 없어서 편합니다.