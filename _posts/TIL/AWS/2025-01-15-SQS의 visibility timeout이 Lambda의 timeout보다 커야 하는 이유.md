---
layout: post
category: AWS
---

람다 트리거에 SQS를 붙이고 SQS에 메시지가 들어가면 어떻게 람다는 메시지를 받을까요?

### 람다의 메시지 처리 프로세스

우선 첫번째로 SQS가 메시지를 수신하면 트리거를 추가할 때 설정한 다양한 설정(배치크기, 배치 기간)에 의해서 메시지를 일정 크기만큼 모으거나, 설정하지 않았다면 그대로 Lambda에 전달합니다.

이 때, SQS는 Lambda에 `ReceiptHandle`이라는 특별한 식별자를 함께 전달합니다. ReceiptHandle는 쉽게 말해서 도서관의 ‘대출 영수증’ 과 비슷한 역할을 합니다. 즉 이 영수증을 가지고 있어야 메시지를 처리하는 동안 `ReceiptHandle` 을 통해 해당 메시지에 대한 작업을 수행할 수 있는 권한을 가지게 됩니다. 후에 얘기 할 것 이지만 이 `ReceiptHandle`는 `Visibility Timeout` 동안 만 유지합니다.



> **Visibility Timeout**(가시성 제한 시간)은 SQS의 메시지 처리 신뢰성을 보장하는 핵심 메커니즘입니다. 또 도서관으로 비유하면 ‘대출 기간’ 으로 볼 수 있는데요, 이유는 Lambda에 전달된 메시지는 이 Visibility Timeout동안 SQS에서 숨겨져 다른 컨슈머(다른 Lambda)가 이 메시지를 볼 수 없게 해줍니다. 따라서 하나의 컨슈머에서 한 메시지가 처리되는 것을 보장시켜주는 것입니다.

SQS가 Lambda에 메시지를 `ReceiptHandle`와 함께 전달하는 순간 설정한 `Visibility Timeout` 는 시작됩니다.

두번째로 Lambda가 해당 메시지 처리를 성공하면 Lambda는 자동으로 DeleteMessage API를 호출합니다. 이때 앞서 받은 `ReceiptHandle`을 사용하여 해당 메시지를 식별합니다.

마지막으로, API를 통해서 삭제된 메시지는 영구적으로 큐에서 삭제됩니다. 이 시점에서 Visibility Timeout은 더 이상 의미가 없어지고, 따라서 삭제된 메시지는 다시 처리되지 않게 됩니다.

이를 그림으로 그리면 다음과 같이 됩니다.
![](https://velog.velcdn.com/images/leehjhjhj/post/a3a7df56-e000-4e7e-9f09-ae18f493ded9/image.png)
### visibility timeout이 Lambda의 timeout보다 크게 되면 생기는 문제

이제 Lambda가 SQS의 메시지를 처리하는 과정을 알았는데, 그래서 `Visibility Timeout` 가 timeout보다 크면 무슨 일이 생길까요?

이는 아주 치명적인데요, 만약 Lambda의 메시지 처리 시간이 길어지는 상황이 발생하면 다른 컨슈머에 의해 **중복 처리**가 될 가능성이 커집니다.

Lambda가 아직 메시지를 처리 중인데 `Visibility Timeout`이 먼저 만료되면 해당 메시지가 다시 큐에 보이게되고, 다른 Lambda 함수가 같은 메시지를 처리하게 되기 때문입니다.

Lambda Timeout이 15분이고, SQS의 `Visibility Timeout`이 5분라고 예를 들어봅시다. 만약에 해당 메시지를 처리시간이 10분이 걸리게 될 때, 처리 시작 5분 후 `Visibility Timeout`이 만료된다면 메시지가 다시 큐에 보이게 되어 다른 Lambda가 처리를 시작하게 됩니다. 이렇게 되면 동일한 메시지가 두 번 처리되는 거죠! 아주 끔찍합니다.

### 권장 스펙

그래서 대부분 `Visibility Timeout` 은 Lambda의 timeout보다 충분히 여유 있는 시간으로 지정합니다. 저는 2배정도로 지정하는 편입니다. 만약 재시도 정책을 지정했다면 재시도 진행시 다른 컨슈머가 또 메시지를 가져가는 일이 없어야 하기 때문에 이보다 훨씬 큰 시간을 지정해야 합니다.