---
layout: post
category: AWS
---

## 문제 상황

- IAM 사용자에 한정된 권한을 주기 위해서 제한된 정책을 만들 때, 예기치 못한 permission 오류가 발생한다.
- 가령 EventBridge의 특정 scheduler만 관리할 수 있는 사용자를 생성한다고 하자.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "scheduler:ListSchedules",
                "scheduler:ListScheduleGroups"
            ],
            "Resource": "*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "scheduler:ListTagsForResource",
                "scheduler:GetSchedule",
                "scheduler:UpdateSchedule",
                "scheduler:GetScheduleGroup",
                "scheduler:CreateScheduleGroup",
            ],
            "Resource": [
                "arn:aws:scheduler:ap-northeast-2::schedule/default/someFunction1",
                "arn:aws:scheduler:ap-northeast-2::schedule/default/someFunction2"
            ]
        }
    ]
}
```

- 이렇게 정책을 정의했을 때, 해당 유저로 들어가서 변경을 하려고 하면 다음과 같은 에러가 발생한다.

> User: arn:aws:iam::123:user/eventbridge-console is not authorized to perform: iam:PassRole on resource: arn:aws:iam::123:role/eventbridge-lambda because no identity-based policy allows the iam:PassRole action

- 여기서 자세히 읽어보면 arn:aws:iam::123:role/eventbridge-lambda 라는 역할에 대해서 `iam:PassRole` 권한이 없다고 한다.

## PassRole과 해당 에러 해결법

- `PassRole`이란 **특정 역할을 다른 AWS 서비스에 전달할 수 있는 권한** 이다.
- 즉, 이 권한은 기본적으로 **한 AWS 리소스**가 **다른 리소스에 접근할 수 있도록 역할**을 위임하는 데 사용된다.
- 만약에 Lambda 함수가 특정 IAM 역할을 사용하여 S3 버킷에 접근해야 할 때, Lambda 함수의 실행 역할이 다른 역할을 **전달**할 수 있어야 한다.
- 즉, 위와 같은 상황에서는 EventBridge Scheduler가 Lambda 함수를 호출할 때 필요한 역할을 사용자가 전달 받지 못해서 접근이 거부된 것이다.
    - 참고로 PassRole에 의해서 전달된 역할은 임시 자격 증명을 생성해서 사용되고, 특정 시간동안만 유효하다.
- 해당 권한 거부의 해결법은 이렇게 `iam:PassRole`의 권한을 생성하고 사용자에게 연결해주면 된다.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::123:role/eventbridge-lambda"
        }
    ]
}
```