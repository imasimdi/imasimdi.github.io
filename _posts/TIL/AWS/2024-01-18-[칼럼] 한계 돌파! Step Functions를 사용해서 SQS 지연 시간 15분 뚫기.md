---
layout: post
category: AWS
---

![](https://velog.velcdn.com/images/leehjhjhj/post/545c291c-b7bd-4e1b-956a-817735485063/image.png)


[전의 포스트](https://imasimdi.dev/aws/%EC%B9%BC%EB%9F%BC-%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4%EB%A1%9C-%EA%B7%80%EC%B0%AE%EC%9D%8C-%EB%8D%9C%EA%B8%B0) 에서 SQS 지연 시간에 대해서 짤막하게 소개를 하였습니다. SQS 지연 시간은 최대 15분 동안 메시지 전송을 지연시켜줄 수 있는 좋은 기능이지만 15분이라는 제한 시간이 있습니다. 만약 15분 이상으로 메시지 전송을 지연시키고 싶다면, Consumer에서 SQS의 `MessageAttributes` 를 활용하여 count를 증가시켜주는 방식으로 구현이 가능합니다.

```python
def handle_message(message):
    current_hop = int(message.get('MessageAttributes', {}).get('hop', {}).get('StringValue', '0'))
    total_hops_needed = 4  # 1시간 = 4 x 15분
    
    if current_hop < total_hops_needed:
        # 다음 큐로 메시지 전달
        next_queue.send_message(
            MessageBody=message['Body'],
            DelaySeconds=900,  # 15분
            MessageAttributes={
                'hop': {
                    'StringValue': str(current_hop + 1),
                    'DataType': 'Number'
                }
            }
        )
    else:
        # 최종 처리 로직
        process_final_message(message)
```

이런 식으로 `MessageAttributes`를 사용해서 간단하게 지연 시간을 늘릴 수 있지만 다음과 같은 치명적인 단점이 있었습니다. 바로 **비용 문제**인데요. 예시는 지연 시간이 1시간이라 3번의 추가적인 람다와 SQS 요청이 들었지만, 그 이상으로 올라가면 기하급수적으로 추가 호출 빈도가 늘어납니다. 하루만 지연시킨다고해도 양쪽에 **95번의 추가 요청**이 필요합니다.

사용 빈도가 적은 애플리케이션이라면 이 정도의 추가 호출은 괜찮지만, 만일 요청량이 많은 애플리케이션이라면 조금 곤란해지겠죠?

또 다른 문제점으로는 바로 **대기 상태를 관리하기 힘들다**는 점입니다. 중간에 대기를 삭제하고 싶거나 변경하고 싶어도 메시지는 루프에 갇혀 관리를 하기에 매우 까다로워집니다. 멈출 수 없는 대기라고 생각하면 조금 곤란하네요.

이러한 문제점들을 한번에 해결시켜주는 방법이 있습니다. 바로 **`AWS StepFunctions`**를 사용하는 방법입니다.

## Step Functions 사용하기

**AWS Step Functions**는 서버리스 워크플로우 서비스입니다. Json이나 시각적 워크플로우로 Lambda 함수, ECS 태스크, API 호출 등 다양한 AWS 서비스들을 순차적 또는 병렬로 호출 할 수 있는 상태 머신을 만들어줍니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/acbaf902-acd4-454f-b2ee-6a03afc44844/image.png)

![](https://velog.velcdn.com/images/leehjhjhj/post/e828c0e6-f5a5-452c-9742-73c5b4c52ed9/image.png)


예를 들어서 S3 -> Lambda -> DynamoDB에 저장하는 간단한 이미지 처리 파이프라인이 존재할 때, Step Functions를 사용하면 람다에 따로 트리거를 설정하지 않아도 Step Functions 상태머신이 알아서 리소스들을 호출해줍니다.

만약 복잡한 워크 플로우가 있을 때, 해당 워크플로우의 상태 머신을 만들어두면 필요할 때 상태 머신 하나만 실행하면 됩니다. 이를 통해서 재사용성 증가화 더불어 일종의 추상화 효과도 제공해줍니다.

서버리스 서비스이기 때문에 가격도 아주 저렴한데 **1,000개의 상태 전환당 $0.025원**이 부과됩니다. 사용한 만큼 돈이 나가는 진짜 서버리스 서비스네요.

## Step Functions으로 SQS 지연시간 늘리기

### 어떻게 가능하죠

자, 그렇다면 이 Step Functions로 어떻게 SQS의 지연시간을 늘릴 수 있을까요? Step Functions의 상태 중 `Wait`라는 것이 있는데, 이를 사용하면 다음 단계로 전환 전에 대기를 할 수 있습니다. `Wait` 상태는 말 그대로 워크플로우의 실행을 일정 시간 동안 일시 중지하는 기능입니다.

제일 중요한 것이 대기 가능한 시간인데, Step Functions의 `Wait` 상태는 무려 **최대 1년**까지 가능합니다! 이는 Step Functions의 실행 자체의 제한 시간도 1년이기 때문입니다.

Wait은 상태 머신 생성시 다음과 같이 지정해줄 수 있습니다.

```json
{
  "Type": "Wait",
  "Seconds": 10,
  "Next": "NextState"
}
```

이렇게 `Seconds`로 직접 설정해줄 수도 있고 `SecondsPath`를 사용해서 동적으로도 할당이 가능합니다.

```json
{
  "Type": "Wait",
  "SecondsPath": "$.waitSeconds",
  "Next": "NextState"
}
```

즉, 상태 머신 시작 후 `Wait` 상태로 대기를 한 뒤에, `Next` 단계에 SQS의 `SendMessage` 이벤트를 달아주면 SQS 요청을 15분 이상으로 대기를 시켜줄 수 있습니다.

### 어떻게 만들어요

우선 Step Functions 상태 머신을 만들어 줍니다.

```json
{
  "StartAt": "Wait",
  "States": {
    "Wait": {
      "Type": "Wait",
      "SecondsPath": "$.waitTime",
      "Next": "SendMessage"
    },
    "SendMessage": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage",
      "Parameters": {
        "QueueUrl": "url",
        "MessageBody.$": "$"
      },
      "End": true
    }
  }
}
```

`SecondsPath`로 대기 시간을 동적으로 할당시키고 `Next`로 `SendMessage`를 지정해줍니다. 이렇게 만들어주면 간단한 워크플로우가 생성됩니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/06cdf4f7-ccb2-4d95-9491-0f047fe4e68d/image.png)

이후에 이 상태 머신으로 메시지를 보내는 프로듀서 람다를 만들어 줍니다. 굳이 람다가 아니어도 해당 상태 머신을 부를 수 있는 권한만 있다면 어디서든 부를 수 있습니다.

```python
stepfunctions = boto3.client('stepfunctions')

def lambda_handler(event, context):
    try:
        body = json.loads(event['body'])
        request = EventRequest(**body)
        state_machine_arn: str = os.environ['STATE_MACHINE_ARN']
        
        # wait_time이 없을 경우 기본 값 2시간 후 처리하도록 설정
        wait_time: int | None = request.wait_time
        if not request.wait_time:
            wait_time = 7200
        
        # Step Function 입력 데이터 구성
        input_data = {
            "waitTime": wait_time,  # Wait 상태에서 사용할 대기 시간
            "message": request.message.model_dump()
        }
        
        # Step Function 실행
        response = stepfunctions.start_execution(
            stateMachineArn=state_machine_arn,
            input=json.dumps(input_data)
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Step Function execution started successfully',
            })
        }
        
    except Exception as e:
		...
```

코드도 엄청 간단합니다. wait time과 메시지를 받고 이를 `start_execution`를 호출해 상태머신으로 보내줍니다. 이제 해당 람다에 POST 요청을 보내봅니다.

```json
{
    "wait_time": 60,
    "message": {
        "title": "title",
        "body": "body"
    }
}
```

이렇게 요청을 보내면 상태 머신이 작동하고, 콘솔에서도 이를 확인할 수 있습니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/0162b942-3255-410a-9db7-2d22ee979cde/image.png)

진행 상황도 콘솔에서 자세히 볼 수도 있습니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/df1d7108-baae-48b5-9a57-85163e4e373f/image.png)

해당 화면에서 다음 상태 변화까지 남은 시간을 알 수도 있고, 실행 취소도 가능해 관리에 용이합니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/fd3cadaa-1d9a-4ae0-aea6-fd7fbe9a3cb7/image.png)

`Wait`이 성공적으로 끝나면 다음 상태인 `SendMessage`로 잘 넘어갔고, 비로소 SQS 요청이 날아간 것을 확인할 수 있습니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/06f1f1c3-130f-42b0-965c-b97c995e9a5d/image.png)

메시지도 잘 도착했습니다. 이렇게 Step Functions를 사용하면 간단하게 SQS 지연 시간을 대폭으로 늘려줄 수 있습니다. 잘 활용하면 푸시 메시지 전송 예약와 같은 다양한 곳에 사용이 가능할 것 같네요!


