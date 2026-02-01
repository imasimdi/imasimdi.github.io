---
layout: post
category: python
---

Celery는 작업을 요청하는 Producer와 작업을 수행하는 Worker가 분리되어 있습니다. 따라서, 이 둘 사이에서 데이터를 주고받기 위해서는 **직렬화를 하는 과정을 가집니다.**

### 데이터 흐름

1. **Producer:** `task.delay(status=BoxObjectStatus.PREPARING)`을 호출
2. **SerializeR:** Celery는 이 파라미터를 브로커로 보내기 위해 JSON으로 변환
    - JSON은 Python의 클래스나 Enum을 모릅니다. 따라서 `BoxObjectStatus.PREPARING`은 값인 문자열 `"PREPARING"`으로 변환되어 전송됩니다.
3. **Broker:** 메시지 큐에 `{"status": "PREPARING"}` 형태로 저장
4. **Worker:** 메시지를 꺼내서 역직렬화 수행
    - JSON `"PREPARING"`을 다시 Python 객체로 바꿈
    - 하지만 Celery는 이 문자열이 원래 `BoxObjectStatus`라는 Enum이었다는 사실을 알 방법이 없습니다. **그래서 그냥 `str` 타입인 `"PREPARING"`으로 함수에 전달합니다.**

> 결과: 함수 정의에 status: BoxObjectStatus라고 타입 힌트를 적어두었더라도, 실제 런타임에 들어오는 값은 str 타입이 되어버립니다 😭. 그대로 status.value 등을 호출하려 하면 AttributeError가 발생하게 됩니다.
>