---
layout: post
category: python
---

## Prefork Worker Pool

- Celery의 기본 실행 워커 모델, 여러 개의 worker 프로세스를 미리 생성해서 작업을 처리
- Master Process가 여러 개의 Worker Process를 미리 생성
- 각 Worker는 독립적인 프로세스로 실행되어 메모리 격리를 보장
- 다중 프로세스 기반으로 CPU 바운드 작업에 적합하다.
- `--concurrency` 옵션으로 worker 프로세스의 수를 지정 가능
    - 보통 CPU 코어 수를 고려하여 prefork worker수 설정

```bash
celery -A your_app worker --concurrency=4 --prefetch-multiplier=1
```

## Eventlet & Gevent Worker Pool

- 둘다 경량 스레드인 greenlet 기반 비동기 처리
- 동기 함수를 비동기 이벤트 루프 위에 실행하기 위해 monkey patching 실행

```bash
celery worker --pool=eventlet or gevent
```

## 그 외..

- 단일 프로세스에서 실행하는 Solo Pool과 멀티 스레드 기반의 Thread Pool 이있는데 별로 쓸 일이 없을 것 같아서 스킵

## 선택 가이드

- I/O 집약적인 작업의 경우은 threads, gevent, eventlet 중에 선택할 수 있는데 그 중에서 Gevent가 제일 많이 사용해서 문서 및 사례도 많고, 경량 스레드로 최소한의 오버헤드를 얻을 수 있다고 한다.
- CPU 집약적인 작업의 경우 Prefork pool를 추천하는데, celery에서 지정한 기본 워커임과 동시에 멀티 프로세서를 활용해서 GIL을 우회할 수 있다 = CPU bound 작업에 용이

## Prefetch

- worker가 한 번에 가져올 수 있는 작업의 수를 제어하는 설정
- 각 Worker Process는 설정된 Prefetch 수만큼 작업을 미리 가져옵니다.
- 여기서 함정은 설정한 값 만큼 항상 채워있는게 아니라, 현재 배치의 모든 작업이 완료되어야만 다음 배치를 가져올 수 있다는 것이다.
- 따라서 한 워커에 실행 길이가 각기 다른 task들이 들어온다면, 실행 시간이 짧은 task는 불필요하게 오래 기다려야 될 수도 있다.
- 따라서 task의 성격이나 작업 길이마다 각기 다른 prefetch 설정의 워커에 배치해야 한다.

```bash
# 빠른 작업 전용 워커
celery -A tasks worker -Q fast_queue --prefetch-multiplier=4 -n fast_worker@%h

# 긴 작업 전용 워커
celery -A tasks worker -Q slow_queue --prefetch-multiplier=1 -n slow_worker@%h
```

- 설정 팁
    - 값이 크면 작업 분배가 균등하지 않을 수 있으나, 처리량이 증가
    - 값이 작으면 작업 분배가 더 균등해지나, 브로커와의 통신 부하가 증가
    - 따라서 작업 길이가 길다면 해당 워커는 1로 설정
    - 작업 길이가 짧다면 기본값인 4 또는 더 높은 값 사용

우선 Gevent는 Greenlet이라는 그린스레드를 사용한 라이브러리입니다. 그린 스레드가 뭐냐면 커널 공간이 아닌 사용자 공간에서 관리되고, 보통 하나의 네이티브 스레드에서 여러 그린 스레드를 실행하는 방식으로 작동합니다.

greenlet은 경량 스레드(그린 스레드)이기 때문에, 컨텍스트 스위칭 비용이 적습니다.

Gevent는 코루틴 기반의 Python 네트워킹 라이브러리입니다. Gevent는 libev 또는 libuv 이벤트 루프 위에서 동기적인 API를 제공하기 위해서 Greenlet을 사용하는데요, 그럼 Uvicorn과는 차이점이 뭘까요?

Uvicorn은 asyncio 위에서 돌아가기 때문에 네이티브한 async, await과 같은 비동기 함수를 사용할 수 있습니다. 하지만 비동기 함수에서 동기 함수를 실행하면 전체 이벤트 루프가 막히는 상황이 발생합니다.

하지만 Gunicorn과 Gevent의 조합은 조금 다릅니다. Gevent는 **monkey patch**라는 기능을 통해서 Python 표준 라이브러리들을 Gevent 라이브러리가 제공하는 구현체로 대체 할 수 있습니다. 즉, 몽키 패칭을 통해서 동기 코드를 비동기 코드로 변환해주고, gevent 이벤트 루프 위에서 도는거죠!

Gunicorn에서는 gevent worker를 사용하면 알아서 몽키패칭을 해줍니다.

```python
class GeventWorker(AsyncWorker):

    server_class = None
    wsgi_handler = None

    def patch(self):
        monkey.patch_all()

        # patch sockets
        sockets = []
        for s in self.sockets:
            sockets.append(socket.socket(s.FAMILY, socket.SOCK_STREAM,
                                 
```

그것이 아니라면 이렇게 직접 패치를 해줘야 합니다.

```python
from gevent import monkey
monkey.patch_all()  # 모든 모듈을 패치

# 또는 특정 모듈만 선택적으로 패치
# monkey.patch_socket()  # socket만 패치
# monkey.patch_ssl()    # ssl만 패치
# monkey.patch_time()   # time만 패치
```