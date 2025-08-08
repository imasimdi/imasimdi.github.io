---
layout: post
category: AWS
---

SQS로 메시지를 보낼 때 `MessageAttributes` 라는 것을 동봉해서 보낼 수 있습니다.

```go
_, err = p.client.SendMessage(
		&sqs.SendMessageInput{
			MessageBody:            aws.String(string(message)),
			QueueUrl:               aws.String(p.queueURL),
			MessageGroupId:         aws.String(groupId),
			MessageDeduplicationId: aws.String(duplicateMessageId),
			MessageAttributes:      messageAttributes, // << 여기
		},
	)
```

MessageAttributes에는 이러한 데이터를 보낼 수 있습니다.

```go
MessageAttributes: map[string]types.MessageAttributeValue{
    "Key": {
        DataType:    aws.String("String"),     // 또는 "Number", "Binary"
        StringValue: aws.String("value"),      // DataType이 String 또는 Number일 때
        BinaryValue: []byte{...},              // DataType이 Binary일 때
    },
}
```

이런 식으로 `DataType` 에 String이나 Number, Binary를 넣을 수 있습니다. 여기서 중요한 것은 Number의 경우에 `NumberValue`가 아니라 `StringValue` 를 사용해야 합니다. 이 값은 무조건 넣어야 합니다!

또 중요한 것은 한 메시지에 최대 10개의 MessageAttributes를 추가할 수 있습니다.

```go
attrs := map[string]types.MessageAttributeValue{
    "EventType": {
        DataType:    aws.String("String"),
        StringValue: aws.String("OrderCreated"),
    },
    "UserId": {
        DataType:    aws.String("Number"),
        StringValue: aws.String("12345"),
    },
}
```

그래서 대충 이런 식으로 메시지의 메타데이터를 넣을 수도 있고, 비지니스와 관련된 데이터도 넣는 경우가 많다고 합니다.

여기서 중요한 것이 컨슈머에서 MessageAttributes 값을 얻으려면 꼭 `MessageAttributeNames` 파라미터를 추가하고 메서드를 호출해야 합니다.

```python
# python boto3 receive_message 메서드

messages = self.sqs.receive_message(
                    QueueUrl=self.queue_url,
                    MaxNumberOfMessages=5,
                    WaitTimeSeconds=15,
                    MessageAttributeNames=['All']
                )
```

여기에서 특정 MessageAttribute의 키만 넣을 수 있고, `All` 을 넣으면 모든 MessageAttribute를 수신합니다.

이것을 넣지 않으면 메시지에는 MessageAttributes를 제외한 채로 가져옵니다. 매우 중요!!