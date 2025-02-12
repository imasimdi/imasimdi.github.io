---
layout: post
category: AWS
category-show: 아티클
tags: [article]
---

![](https://velog.velcdn.com/images/leehjhjhj/post/5ba89008-d0b4-4c92-923c-f70476f6cdf9/image.png)

소모임 출석체크 완료 후, 출석체크가 완료되었다는 이메일을 전송하고 싶었습니다. 굳이 메일을 보내는 이유는 메일함을 열 때 마다 소모임이 다시 생각날 것이고, 추후 굿즈 수령 시 출석 증빙 자료로도 활용할 수 있기 때문입니다.

그래서 어떻게 구현할까 고민을 했었습니다. 가장 단순한 방법은 출석체크 API에 이메일 전송 로직을 직접 넣는 것이지만, 이벤트 드리븐을 통해 느슨한 결합의 이점을 가져갈 수 있는 DynamoDB Streams를 활용하고자 합니다. 출석 데이터가 DynamoDB에 저장되면 Streams를 통해 이메일 전송 Lambda 함수가 트리거되는 방식으로 구현하려고 합니다.

## DynamoDB Streams란?

DynamoDB Streams란 DynamoDB의 테이블에 대한 변경 사항에 대한 정보를 실시간으로 추적하는 기능입니다. 즉, dynamodb에 변화가 발생하면 AWS가 관리하는 전용 스트림 스토리지에 해당 변경 사항을 기록해줍니다. 

간단한 예로 DynamoDB streams trigger를 설정한 Lambda에서는 event 파라미터로 다음과 같은 데이터들을 전송받습니다.

```javascript
{
    'eventID': '710f3696c288ae9203fff5e43e6aa2c5',
    'eventName': 'INSERT',
    'eventVersion': '1.1',
    'eventSource': 'aws:dynamodb',
    'awsRegion': 'ap-northeast-2',
    'dynamodb': {
        'ApproximateCreationDateTime': 1739242960.0,
        'Keys': {
            'phone': {
                'S': '01082593234'
            }
        },
        'NewImage': {
            'checked_at': {
                'S': '2025-02-11T12:02:40.394838'
            },
            'phone': {
                'S': '01082593234'
            }
            'email': {
                'S': 'hjforaws@gmail.com'
            }
        },
        'SequenceNumber': '2967500000000009611169822',
        'SizeBytes': 197,
        'StreamViewType': 'NEW_IMAGE'
    },
    'eventSourceARN': 'arn:aws:dynamodb:ap-northeast-2:038462794235:table/table/stream/2025-02-11T02:47:47.149'
}
```
`eventName`에는 (INSERT/MODIFY/DELETE)의 종류가 있고, `eventSource`나 `awsRegion` 등 이벤트의 기본적인 메타데이터들도 포함되어있습니다.
그 외의 정보들은 다음과 같습니다.

- `Keys` : 변경된 항목의 기본 키 정보
- `NewImage` : 변경 후의 전체 데이터
- `StreamViewType` : 스트림에서 어떤 데이터를 볼 수 있는지 지정
- `SequenceNumber` : 이벤트 순서 추적용

### Image

여기서 "Image"는 DynamoDB 항목의 전체 상태를 의미하며, 데이터베이스 용어에서 레코드의 스냅샷이라고 생각하면 됩니다. StreamViewType은 스트림에서 어떤 이미지를 캡처할지 결정합니다. 총 4가지 옵션을 제공하며, 이 옵션에 따라 스트림에서 캡처되는 데이터의 양과 세부 정보가 결정됩니다.

- `KEYS_ONLY`: 변경된 항목의 키 속성만 캡처하며, 테이블의 파티션 키와 정렬 키만 포함하도록 할 수 있습니다.

- `NEW_IMAGE`: 항목이 수정된 후의 새로운 상태 전체를 캡며, INSERT나 UPDATE 후의 최종 데이터를 가져올 때 쓰입니다.
```javascript
{
  "NewImage": {
    "user_id": {"S": "123"},
    "name": {"S": "John Doe"},
    "email": {"S": "john@example.com"}
  }
}
```
- `OLD_IMAGE`: 항목이 수정되기 전의 원래 상태 전체를 캡처하며, 변경 전 데이터 백업이 필요할 때와 같은 상황에 쓰일 수 있습니다.
```javascript
{
  "OldImage": {
    "user_id": {"S": "123"},
    "name": {"S": "John Doe"},
    "email": {"S": "old@example.com"}
  }
}
```
- `NEW_AND_OLD_IMAGES`: 변경 전과 후의 상태를 모두 캡처하며, 변경 전후를 비교해야 할 때 사용할 수 있습니다.
```javascript
{
  "OldImage": {
    "user_id": {"S": "123"},
    "email": {"S": "old@example.com"}
  },
  "NewImage": {
    "user_id": {"S": "123"},
    "email": {"S": "new@example.com"}
  }
}
```

### 스트림 데이터 구조

![](https://velog.velcdn.com/images/leehjhjhj/post/b911e160-2bce-4e3d-8365-3b7de0b30d1b/image.png)
[출처](https://docs.aws.amazon.com/ko_kr/amazondynamodb/latest/developerguide/Streams.html)

DynamoDB Streams는 계층적 구조로 데이터를 관리합니다. 최상위 레벨인 스트림은 테이블의 모든 변경 사항을 포함하며, 이 변경 사항들은 '샤드'라는 단위로 그룹화됩니다. 각 샤드는 여러 개의 스트림 레코드를 포함하고 있으며, 각 레코드는 테이블에서 발생한 하나의 데이터 수정을 나타냅니다.

스트림 레코드들은 시퀀스 번호를 통해 순서가 보장되며, 샤드 내에서 24시간 동안만 유지됩니다. 샤드는 테이블의 쓰기 활동량에 따라 동적으로 관리되는데, 처리량이 증가하면 하나의 샤드가 여러 개의 새로운 샤드로 자동 분할될 수 있습니다. 이때 **각 상위 샤드는 단 하나의 하위 샤드만** 가질 수 있습니다.

애플리케이션이 스트림 데이터를 처리할 때는 샤드의 계보(상위-하위 관계)를 고려해야 합니다. 데이터의 순서를 보장하기 위해 상위 샤드의 레코드를 하위 샤드보다 먼저 처리해야 하는데, DynamoDB Streams Kinesis 어댑터를 사용하면 이러한 처리 순서가 자동으로 보장된다고 합니다.

스트림을 비활성화하면 모든 열린 샤드가 닫히지만 기존의 데이터는 24시간 동안 계속 접근을 할 수 있습니다.


## SAM과 통합

SAM과의 통합 방법은 간단합니다. 우선 template.yaml에서 Function을 다음과 같이 수정합니다.

```yaml
EventCheckInTable:
    Type: AWS::DynamoDB::Table
    Properties:
        ...
      StreamSpecification:
        StreamViewType: NEW_IMAGE
EmailFunction:
    Type: AWS::Serverless::Function
    Properties:
            ...
      Events:
        StreamEvent:
          Type: DynamoDB
          Properties:
            Stream: !GetAtt EventCheckInTable.StreamArn
            StartingPosition: LATEST
            BatchSize: 5
```
- `Events` 하위에 `StreamEvent`를 추가하여 스트림을 생성하고, `StartingPosition`와 `BatchSize`를 설정합니다. 테이블에서는 위에서 설명한 `StreamViewType`를 지정해줘서 스트림을 활성화 시킵니다.

### 여기서 StartingPosition란?

StartingPosition 속성은 Lambda 함수가 스트림을 읽기 시작할 위치를 지정합니다.

- LATEST
    - 스트림의 가장 최신 레코드부터 읽기 시작
    - 함수가 활성화된 이후의 새로운 변경사항만 처리
    - 실시간 데이터 처리에 적합하며 가장 일반적으로 사용

- TRIM_HORIZON
    - 스트림에서 가장 오래된 레코드부터 읽기 시작
    - 24시간 이내의 모든 레코드를 처리
    - 데이터 복구나 초기 동기화에 유용

- AT_TIMESTAMP
    - 지정된 타임스탬프 이후의 레코드부터 읽기 시작
    - 특정 시점부터의 데이터 처리가 필요할 때 사용
    - 예를 들면 특정 날짜 이후의 출석 데이터만 처리할 수 있음

이렇게 설정을 하면 람다에서 스트림 데이터를 받은 뒤 이메일을 전송하는 로직을 구현하면 됩니다. 저는 출석 데이터가 삽입됐을 때만 이메일을 전송하도록 했여야 하기에 `eventName`이 `INSERT`인 것만 로직을 진행시켰습니다.

```python
def lambda_handler(event, context):
    try:
        yag = yagmail.SMTP(settings.smtp_username, settings.smtp_password)

        for record in event['Records']:
            if record['eventName'] != 'INSERT':
                continue
            new_image = record['dynamodb']['NewImage']
            recipient_email = new_image.get('email', {}).get('S')
            name = new_image.get('name', {}).get('S')
            event_code = new_image.get('event_code', {}).get('S')
            event_name = get_event_name(event_code)
```

처음에 보여드린 데이터 예시에서도 확인할 수 있듯이 `NewImage`에서 DynamoDB의 변경된 데이터들이 담겨있고있어 여기서 이메일을 전송할 데이터를 가져오면 됩니다.

이제 DynamoDB 중 스트림을 활성화한 `EventCheckInTable`에서 데이터가 `INSERT`되면 해당 람다가 발동해, 이렇게 이메일이 전송됩니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/9b1b4f2a-d516-4e8d-bc72-989f2fbb5366/image.png)