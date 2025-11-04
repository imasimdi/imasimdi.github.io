---
layout: post
category: AWS
---


갑자기 배치 서버에서부터 SQS로의 요청이 가지 않는 일이 발생했습니다. 그래서 ecs exec를 사용해서 내부로 들어가서 curl을 날려보았습니다.

```python
aws ecs execute-command --cluster api --task  --container  --interactive --command "/bin/sh”
```

curl을 날려봤는데 public IP가 아닌 private IP가 오는 것을 확인했습니다.

```python
Trying 172.31.73.59:443...
```

뭔가 느낌이 '혹시 저번에 설정한 SQS 인터페이스 게이트웨이 때문인가?' 해서 SQS의 interface endpoint 설정에 들어가서 보안그룹을 살펴보았습니다. 아니나 다를까! endpoint의 보안 그룹에서 ECS 컨테이너들의 보안그룹이 빠져있었습니다.

그래서 잊지 말고자 조금 정리해보았습니다.

## VPC Endpoint를 설정하면 바뀌는 동작

### 프라이빗 IP로 연결되는 이유

ECS Fargate에서 SQS 엔드포인트에 접근할 때, IP가 172.31.x.x로 해석되어 연결이 되는 이유는 VPC 내에 SQS용 VPC Endpoint가 설정되어 있고, 이로 인해 DNS가 SQS 퍼블릭 도메인을 프라이빗 IP로 해석하기 때문입니다.

### VPC Endpoint와 프라이빗 DNS 동작 원리

Interface Endpoint는 ENI를 VPC 내부의 특정 서브넷에 붙고, 서비스 엔드포인트의 프라이빗 IP 주소를 할당해줍니다. “프라이빗 DNS 이름 활성화” 라는 옵션이 기본 값이고, 이 옵션이 활성화 되면 AWS 서비스의 퍼블릭 DNS 이름을 조회할 때 프라이빗 IP가 반환되도록 VPC 내부의 DNS가 자동으로 라우팅됩니다.

### 그래서 연결 실패 이유는?

그래서 SQS의 url로 요청을 보냈을 때, 이미 배포되어있던 SQS의 VPC endpoint의 프라이빗 IP로  자동으로 연결됩니다. 그런데 이때 VPC endpoint의 ENI에 연결된 보안 그룹이 Fargate 태스크가 속한 보안 그룹에 대해서 443 포트가 열어 있지 않으면 접근을 하지 못해 TimeoutError가 발생합니다.

출처:

[Internetwork traffic privacy in Amazon SQS - Amazon Simple Queue Service](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-internetwork-traffic-privacy.html)