---
layout: post
category: AWS
category-show: 아티클
tags: [article]
---

## Step Function 도입 배경

[meiru 뉴스 서비스](https://meiru.me/)를 만들면서 데이터 처리에 대한 문제가 발생했었습니다. 크롤링한 뉴스를 AI로 번역하는데, gemini 무료 plan을 쓰다보니 거의 매일 하나 이상의 필드가 null로 나왔습니다. 한일 뉴스를 번역해서 뉴스레터로 보내야 하는데, null 값이 그대로 표시되는 거죠.

![](https://velog.velcdn.com/images/leehjhjhj/post/67fc5c78-06b9-4967-8d61-9150ae31a84a/image.png)

그래서 이메일 발송 전에 매일 아침 수동으로 체크했습니다.
- DB에서 null 값이나 잘못된 값 확인
- 수동으로 AI 번역 재실행

사이드 프로젝트인 것 만큼, 저의 운영 리소스가 많이 들지 않았으면 했고, 이를 위해서 번역 데이터를 확실하기 처리하기 위해 Step Function을 도입했습니다.

## Step Function Workflow

![](https://velog.velcdn.com/images/leehjhjhj/post/50c9a0ab-45b7-410d-9dd9-6a3664e08b9f/image.png)

제가 구성한 워크플로우입니다. Step Function으로 상태 머신을 만들게 되면, 이렇게 UI로 workflow와 진행상황을 보여줍니다.

1. ProcessAI가 번역 실행
2. CheckNews Lambda가 댓글, 본문, 제목, 요약 등 null 체크
3. null이 있으면 `shouldRetry: true` 반환
4. 잠시 대기 후 `RetryCount` 증가
5. ProcessAI로 돌아가서 null인 필드만 재처리
6. 성공하면 종료

여기서 가장 중요한 건 ProcessAI를 멱등성 있게 만드는 것이었습니다.

## Step Function의 입력과 출력

워크플로우를 만들면서 가장 헷갈렸고, 중요한 것은 `$` 부호입니다. 이게 어디에 쓰이는지부터 알아야 합니다.

Step Function은 상태별로 입력을 받고 출력을 다음 단계로 전달합니다. 단계별 데이터 흐름은 다음과 같습니다.

| 단계 | 역할 | 예시 |
| --- | --- | --- |
| State Input | 이전 State에서 받은 데이터 | `{ "retryCount": 0 }` |
| Task Raw Output | Lambda 등의 원본 응답 | `{ "StatusCode": 200, "Payload": {...} }` |
| ResultSelector | Raw Output에서 필요한 것만 선택/가공 | `{ "retryCount.$": "$.Payload.retryCount", "shouldRetry.$": "$.Payload.shouldRetry" }` |
| Processed Output | ResultSelector 결과 | `{ "retryCount": 1, "shouldRetry": true }` |
| ResultPath | State Input과 Processed Output을 어떻게 합칠지 | `$` 또는 `$.checkResult` |
| Next State Input | 다음 State로 전달할 최종 데이터 | `{ "retryCount": 1, "checkResult": {...} }` |

정리하면:
- ResultSelector: Task 결과에서 필요한 것만 선택 → Processed Output 생성
- ResultPath: State Input과 Processed Output을 합침 → Next State Input 생성

## `$` 표현식의 의미

ResultSelector에서 `{ "retryCount.$": "$.Payload.retryCount" }`를 사용하면 Raw Output을 이렇게 가공합니다.

```json
{ "retryCount": 1 }
```

`retryCount.$`는 "retryCount 키를 만들되, 값은 JSONPath로 동적으로 가져온다"는 뜻입니다.

### `$.Payload.retryCount`는 뭘까?

Lambda를 호출하면 Step Functions가 return 값을 자동으로 Payload로 감쌉니다.

Lambda에서 `return { shouldRetry: true, retryCount: 1 }`을 하면
Step Functions는 `{ Payload: { shouldRetry: true, retryCount: 1 } }`로 받습니다.

`$.Payload.retryCount`는 JSONPath로 중첩된 JSON에서 값을 꺼내는 경로입니다. 

```json
// Lambda의 Raw Output
{
  "StatusCode": 200,
  "Payload": {
    "shouldRetry": true,
    "retryCount": 1
  }
}
```

- `$`: 루트
- `$.Payload`: `{ "shouldRetry": true, "retryCount": 1 }`
- `$.Payload.retryCount`: `1`

JavaScript의 `response.Payload.retryCount`나 Python의 `response["Payload"]["retryCount"]`랑 똑같습니다.

## ResultPath의 동작 방식

`ResultPath`가 좀 헷갈립니다. 다음 단계로 넘어가는 Input은 이렇게 만들어집니다.

```
Next State Input = Merge(State Input, Processed Output, ResultPath)
```

Processed Output `{ "retryCount": 1, "shouldRetry": true }`와 State Input을 합쳐서 다음 단계로 넘깁니다.
ResultPath 값에 따라 병합 방식이 달라집니다.

### `ResultPath: "$"`

```json
// State Input
{ "retryCount": 0 }

// Processed Output
{ "retryCount": 1, "shouldRetry": true }

// Next State Input (원본 완전히 대체)
{ "retryCount": 1, "shouldRetry": true }
```

### `ResultPath: "$.checkResult"`

```json
// State Input
{ "retryCount": 0 }

// Processed Output
{ "retryCount": 1, "shouldRetry": true }

// Next State Input (원본 보존하면서 추가)
{
  "retryCount": 0,
  "checkResult": {
    "retryCount": 1,
    "shouldRetry": true
  }
}
```

원본 데이터를 보존하면서 새로운 결과를 추가할 때 유용합니다. 저도 이 방식으로 워크플로우를 구성했습니다.

## 정리

제가 생각하는 사이드프로젝트에서 가장 중요한 점은 **지속 가능성** 입니다. 여기서 지속 가능성이란 서버비 및 금전적인 비용도 마찬가지지만, 운영 비용에 대한 지속 가능성도 포함됩니다.

만약 사이드프로젝트의 비용이 너무 크게되면 지속이 불가능해져, 금방 열정이 사라지곤 합니다. 이를 피하기 위해서 meiru 서비스는 full serverless 아키텍처를 택했고, 최소한의 비용으로 데이터를 처리하기 위해 Step Function을 도입했습니다. 그 결과, 아주 적은 비용으로 이제 `null` 값이 없는 뉴스레터를 볼 수 있게 되었습니다.