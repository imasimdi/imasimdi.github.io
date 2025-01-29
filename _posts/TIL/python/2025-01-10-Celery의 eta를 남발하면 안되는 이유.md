---
layout: post
category: python
---

최근 회사에 Celery를 도입하고 있는데, 우선적으로 적용한 곳이 FCM 발송 로직입니다.

행사 타임라인 서비스라고 행사 관련해서 공지사항, 변동사항이 생기면 팔로워들 푸시 알림을 보내는 기능이 있는데, 새벽 시간에 메시지가 전송되면 이것을 모아뒀다고 아침에 보내달라는 요구사항이 생겼습니다.

이것을 어떻게 구현할까 고민하다가 🤔 **음 그냥 ETA로 다 날려버릴까?** 라고 생각했습니다.

```python
from datetime import datetime, timedelta

future_time = datetime.utcnow() + timedelta(minutes=5)
my_task.apply_async(args=[data], eta=future_time)
```

celery의 eta는 요약하자면, `apply_async` 메서드를 사용할 때 celery task를 미래에 실행되도록 예약하는 기능입니다. 

**그런데 내부동작을 모르니 조금 찜찜해서** 공식문서를 뒤져봤습니다. 역시나 저 같은 사람이 많은지 새빨간색으로 경고하고 있네요. 

![https://docs.celeryq.dev/en/main/userguide/calling.html#eta-and-countdown](https://velog.velcdn.com/images/leehjhjhj/post/419cb217-fed8-4593-908a-3d5ebf468bb9/image.png)

https://docs.celeryq.dev/en/main/userguide/calling.html#eta-and-countdown

요약하자면 

- eta/countdown으로 예약된 태스크는 **즉시 워커가 가져가서 예약된 시간까지 메모리에 보관**한다.
- 먼 미래의 많은 태스크를 예약하면 워커의 RAM 사용량이 크게 증가할 수 있다.

결론은 **eta와 countdown은 먼 미래의 태스크 스케줄링에는 권장되지 않음** 이고, 워커 터지기 싫으면 테이블 하나 파서 celery beat로 주기적으로 긁던가 하라는 말이었습니다.

혹시 celery를 회사에 쓰시는 분들은 참고하시면 좋을 것 같아요! 저는 막연히 redis를 비롯한 메시지 브로커에 메시지를 넣어두지 않을까 싶었는데 워커의 메모리에 메시지를 담고 있네요. 모르고 쓰면 펑펑 터질 뻔 했습니다.