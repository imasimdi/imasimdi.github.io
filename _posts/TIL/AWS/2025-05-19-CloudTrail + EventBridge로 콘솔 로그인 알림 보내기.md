---
layout: post
category: AWS
category-show: 아티클
tags: [article]
---

![](https://velog.velcdn.com/images/leehjhjhj/post/4bcad03f-2897-4e2d-a100-3ae5b9ea791a/image.png)

인프라 개선 프로젝트에 일환으로 IAM 사용자가 콘솔로 로그인을 했을 때 슬랙으로 알림을 보내는 시스템을 만들었습니다.

## 구현 방식 선택

구현에는 두 가지 선택지가 있었는데 하나는 CloudTrail과 CloudWatch 알림을 사용하는 방법이었고
하나는 CloudTrail과 EventBridge를 사용하는 것이었습니다.

### CloudTrail + CloudWatch 알림
CloudWatch를 사용하는 방법은 메트릭 필터를 지정하고 다음과 같은 패턴을 정의합니다

```
{ ($.eventName = ConsoleLogin) && ($.additionalEventData.MFAUsed != Yes) }
```

해당 메트릭 지표에 임계값을 CloudWatch에서 설정하고 알림을 받을 곳을 연결하면 됩니다. 이렇게 한다면 정말 간단하게 구현할 수 있습니다.

### CloudTrail + EventBridge

하지만 향후 더 많은 조건을 필터링하거나 추가 동작을 수행하기에는 EventBridge 조합이 더 적합하다고 판단하여 EventBridge를 활용해 구현했습니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/544c8a75-5acf-4988-a231-e6b1b12d5f95/image.png)

그 전에 EventBridge Bus에 관해서 짚고 넘어 갑시다.

## EventBridge Bus

![](https://velog.velcdn.com/images/leehjhjhj/post/7e743ec9-8193-4245-bc9d-7a394ab49ceb/image.png)

### 개요

EventBridge에는 많은 서비스가 세분화 되어있으나, 이번 글에서는 EventBridge Bus만 다루겠습니다. EventBridge는 서버리스 이벤트 버스 서비스로, 거대한 우체국의 분류센터라고 생각하면 이해가 쉽습니다.
EventBridge에는 세 가지 유형의 이벤트 버스가 존재합니다

1. Default Event Bus
계정 생성 시 자동으로 생성되며 AWS 내부 서비스에서 발생하는 모든 이벤트를 자동 수신합니다. 이번 구현에 사용할 버스입니다.

2. Custom Event Bus
사용자가 직접 생성하는 전용 이벤트 버스이고, 특정 비즈니스 도메인이나 애플리케이션을 위한 용도로 사용됩니다. 따라서 마이크로서비스 간 통신, 비즈니스 이벤트 처리의 메시지 분류 및 전달 허브로 활용되는 경우가 많습니다.

3. SaaS Event Bus
AWS 파트너 SaaS 제공업체와 기본 통합되어서 API 연동 없이 SaaS 애플리케이션의 이벤트 수신 가능합니다.

### 핵심 구성 요소

EventBridge의 요소는 크게 source, detail-type, detail, event pattern으로 볼 수 있습니다.

- **Source와 Detail-Type**: 이벤트 자체의 속성으로, 이벤트를 분류해주는 기준으로 볼 수 있습니다.
	- source: "누가 이벤트를 발생시켰는가?"
    	- 예시: aws.ec2, com.mycompany.order, custom.payment
        - 이벤트의 출처를 식별하는데 사용
- **Detail-type**: "어떤 종류의 이벤트인가?"	
    	- 예시: "Order Created", "Payment Processed", "EC2 Instance State-change Notification"
        - 동일한 소스에서 발생한 여러 유형의 이벤트를 구분합니다.
- **Detail**
	- 이벤트와 관련된 실제 데이터를 포함하는 JSON 객체로, 이벤트 JSON에서 "detail" 필드에 해당합니다.
- **EventPattern**
	- 이벤트 패턴은 어떤 이벤트가 규칙과 일치하는지 결정하는 필터링 조건입니다.
    - 특정 조건을 만족하는 이벤트만 대상으로 라우팅하도록 도와줍니다.
    - 다른 요소와 다르게 이벤트 자체에 포함되는 것이 아니라, EventBridge 규칙에서 이벤트를 필터링 하는데 사용됩니다.
    - 이렇게 필터링 된 이벤트를 특정 컨슈머 (Target)으로 전송해 특정 로직을 수행합니다.
    - 예시:
      ```javascript
      {
        "source": ["com.mycompany.order"],
        "detail-type": ["Order Created"],
        "detail": {
          "totalAmount": [{"numeric": [">", 50]}]
        }
      }
      ```
	
    

### Kafka와 비교

이벤트를 중앙에서 수집 및 분류를 해준다는 면에서 Kafka와 유사한 면이 많은데요, 완전한 일대일 매칭은 불가능하지만 이해를 위해서 다음과 같이 나눌 수 있습니다.

- **Source + Detail-Type**은 Kafka의 토픽명과 유사 (이벤트 분류)
- **Detail**은 Kafka 메시지의 페이로드와 유사 (실제 데이터)
- **Event Pattern**은 Kafka의 컨슈머 필터링과 유사 (특정 메시지만 처리)

이제 EventBridge 버스에 대해서 대강 알았으니, 실제 구현에 들어가봅시다!

## 구현 과정

이제 본론으로 넘어가 EventBridge의 Default 버스를 활용하여 구현을 해보겠습니다.

### CloudTrail 설정

CloudTrail을 생성합니다. 어느 리전에 생성해도 상관없지만, Root 계정의 알림을 받기 위해 us-east-1에 생성했습니다.
중요한 것은 하나의 **CloudTrail도 존재하지 않으면 EventBridge는 CloudTrail 이벤트를 수신할 수 없습니다.**

만약 Console로 하나의 trail을 생성하면 자동으로 multi-region trail이 되어 모든 리전의 이벤트를 수집합니다. Root 계정 로그인과 같은 글로벌 서비스 이벤트는 US East리전에서 기록되지만, multi-region trail을 사용하면 어느 리전에서 trail을 생성해도 다른 이벤트들을 수신할 수 있습니다. [공식문서](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-trails.html)

### EventBridge 규칙 생성
그리고 EventBridge의 default 이벤트 버스에 규칙을 추가해줍니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/55fddf9c-4713-4251-9f58-d16d3607c41d/image.png)

이 룰에서는 다음과 같은 이벤트 패턴을 생성해주는데요, source를 `aws.signin`에서 받고, `Detail-Type` 을 Root와 IAMUser로 넣어주면 모든 유저의 콘솔 로그인에 대해서 이벤트를 수신하게 됩니다.
![](https://velog.velcdn.com/images/leehjhjhj/post/5abb3d3d-37d1-4cd1-8c90-702de4122c55/image.png)

### 대상 설정

이벤트를 받은 후 대상을 지정하는데, 저는 특정 채널에 슬랙 메시지를 보내는 Chatbot(현재는 Q Developer)에 연결된 SNS를 대상으로 지정했습니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/9e573272-f879-49bf-90be-fb55d622e9e2/image.png)

### 결과

설정을 완료한 후 콘솔에 로그인하면 다음과 같이 슬랙 메시지가 오게 됩니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/09b41287-c1a4-484c-b213-9d649e65322a/image.png)

이렇게 EventBridge와 CloudTrail를 사용해서 간단하게 콘솔 로그인 알림을 설정하였습니다. EventBridge 기반 아키텍처를 사용하면 다양한 보안 이벤트로 확장을 할 수 있습니다.

예를 들면 IAM 생성/삭제 관리 모니터링이라던지, 보안그룹이 변경되거나 S3 버킷 정책 변경시 알림을 보낸다거나, 혹은 RDS나 EC2등 중요 리소스들의 생성/삭제/수정/중지와 같은 이벤트를 수신하여 다양한 타겟에 넣을 수 있을 것 같습니다.

이렇게 EventBridge는 full managed로 강력한 이벤트 버스 역할을 해주기 때문에 default 버스 뿐만 아니라 custom 버스를 사용해서 애플리케이션을 구축해보고 싶다는 생각이 들었습니다.
