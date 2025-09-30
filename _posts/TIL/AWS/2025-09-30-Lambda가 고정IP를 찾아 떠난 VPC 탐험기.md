---
layout: post
category: AWS
category-show: 아티클
tags: [article]
---

![](https://velog.velcdn.com/images/leehjhjhj/post/49db911d-67ea-499c-8a4a-1bba9c75fcc8/image.png)

해당 글은 [AWSKRUG 서버리스 소모임](https://www.youtube.com/watch?v=GHb1kywPOhk&t=6s) 발표와 동일한 주제입니다.

## 배경

제가 근무하고 플랫폼 서비스에서는 제가 작년에 개발한 '스마트 무통장'이라는 결제 수단이 존재합니다.

기존의 무통장 입금의 불편함을 해소하고자, 무통장 입금을 자동화시킨 결제 수단입니다. 현재 다양한 AWS 서버리스 서비스(Lambda, SQS, DynamoDB, EventBridge 등)을 사용해서 서버리스 아키텍처 위에서 운영되고 있습니다.

이 결제 시스템은 5분에 한번 입금건을 확인하고, 이를 주문서와 매칭시킨 뒤 판매자에게 돈을 보내주는 전체 사이클이 돌고 있습니다. 여기서 이번 아티클의 주무대가 되는 곳은 '송금' 부분 입니다.

현재 송금도 Lambda에서 이루어지고 있으며, FIFO SQS에서 메시지를 컨슘하고, DynamoDB와 통신하여 멱등성을 보장하고 있습니다.

하지만, 이번에 송금을 담당하는 서드파티를 변경해야 했는데, 해당 회사로부터 다음과 같은 이메일이 왔습니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/5af36627-aed3-4dbc-b18b-109dfa799645/image.png)

즉, 외부 서드파티에 접근하기 위해서는 **송금 람다에서 outbound 트래픽을 보낼 때, 고정된 IP로 서드파티에 접근**해야한다는 의미였습니다. 여기에서 저는 깊은 고민에 빠지게 되었습니다. 바로 현재 저희 서비스에서는 **고정 IP**가 존재하지 않기 때문이죠!

## 고정 IP가 없는 이유

현재 저희 서비스에 고정 IP가 없는 이유는 크게 두 가지 입니다.

- 메인서버는 ECS + Fargate로 돌아가고 있음
- 현재 스마트 무통장의 서버리스 아키텍처는 특정 VPC에 있지 않음

ECS는 '태스크'라는 단위로 ENI를 생성하고, 이 ENI에 public IP 및 private IP를 할당합니다. ECS 특성 상, 태스크 단위로 스케일 아웃이 되고 스케일 인이 되기 때문에 일정 범위의 IP는 얻을 수 있지만 고정된 IP를 얻기는 힘듭니다.

또한 현재 서버리스 아키텍쳐는 AWS의 퍼블릭 VPC인 Lambda Service VPC에 존재하기 때문에, Lambda가 외부로 요청을 보낼 때는 AWS가 랜덤 IP를 부여합니다. 그래서 저는 '송금 Lambda를 어떻게 해야 할까?' 라는 고민을 시작하였습니다.

## 송금 Lambda 마이그레이션

가장 먼저 떠오른 것은 'EC2를 사용하기' 입니다. EC2를 생성하면 자동으로 'eth0'라는 ENI가 만들어집니다. 이곳에 고정 IP인 EIP(elastic IP)를 붙여주면 아주 간단하게 고정 IP를 얻을 수 있습니다.

그래서 EC2로 송금 Lambda의 송금 로직을 옮기거나, 밑의 그림과 같이 프록시 서버를 생성하는 것을 생각해보았습니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/8a2cbf12-3874-44b2-87c5-0f04328aded0/image.png)

하지만, EC2를 사용하기엔 다음과 같은 우려가 있었습니다.

이를 이해하려면, 스마트 무통장 아키텍처가 왜 서버리스 아키텍처 위에 개발이 되었는지 알아야합니다.

스마트 무통장은 **24시간 돌아가는 준실시간성 결제수단**이기에, 중단이 되면 판매자들이 큰 불편을 겪습니다. 따라서 **죽지 않는 고가용성 서버**가 필요했습니다. 또한 이를 복구할 운영 비용과 더불어 부가적으로 서버 비용도 줄여야 했기에, 이에 최적인 서버리스 아키텍처를 선택하였습니다.

그렇기 때문에 EC2를 사용하게 되면, Lambda와 비교할 때 상대적으로 불안정하고 복구가 어렵다고 판단했습니다.

한마디로 이 결제 수단의 핵심 중 하나인 **송금**이 인프라적인 문제 때문에, 단일 장애 지점이 생기는 것을 막고자 하였습니다. 그래서 택한 것이 Lambda의 VPC mode를 활용한 방법입니다.

## Lambda VPC mode

Lambda VPC mode는 Lambda가 특정 VPC에 접근이 가능하도록 하는 옵션입니다. 이 VPC 모드를 사용하면 설정한 VPC 내 특정 서브넷과 보안 그룹 조합으로 **'Hyperplane ENI'** 라는 ENI가 생성됩니다.

이 ENI는 Lambda와 VPC간 '네트워크 터널' 역할을 해줘서 Lambda가 VPC 내부에 접근이 가능하게 해줍니다. 특이한 것은 ec2나 ecs의 eni처럼 각 인스턴스나 태스크마다 ENI가 생성되는 것이 아니라, **여러 Lambda가 하나의 ENI를 공유한다는 것입니다.**

![](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2019/08/28/v2n-architecture-1024x613.png)

[출처](https://aws.amazon.com/ko/blogs/compute/announcing-improved-vpc-networking-for-aws-lambda-functions/)

이에 대한 자세한 내용은 [예전 포스트](https://imasimdi.dev/aws/%EB%82%98%EC%9D%98-%EB%9E%8C%EB%8B%A4%EB%8A%94-%EC%99%9C-%EB%82%98%EC%9D%98-%EB%9E%8C%EB%8B%A4%EB%A5%BC-%EB%AA%BB-%EB%B6%88%EB%A0%80%EB%8A%94%EA%B0%80)에 있습니다.

이번 아티클에서 눈여겨보아야 할 Hyperplane ENI의 특징은 바로 **'임시성'** 입니다.

이 ENI는 특이하게 65,000개의 연결이 넘어가거나, 서브넷이나 보안그룹이 달라지거나, 장기간 사용이 되지 않으면 자동으로 생성되거나 삭제됩니다. 따라서 EC2의 'eth0 ENI'처럼 ENI에 직접 EIP를 붙이는 것은 안정성 측면에서 떨어집니다. 이에 대해서는 후에 조금 더 자세히 다룰 예정입니다.

## Lambda VPC mode의 장점

Lambda VPC mode를 사용하면 비교적 쉽게 Lambda에 EIP를 붙일 수 있는 옵션들이 존재합니다. 제일 좋은 장점은 기존의 아키텍처를 바꾸지 않아도 된다는 것입니다.

현재 아키텍처는 대부분의 Lambda앞에 SQS가 붙어있습니다. 만약 송금 로직을 EC2로 마이그레이션하게 되면 SQS를 컨슈밍하는 로직을 손수 만들어줘야 한다는 것을 의미합니다.

하지만 VPC mode를 사용하면 Lambda가 제공하는 편리한 기능을 그대로 사용할 수 있을 뿐더러, VPC 내부에서만 사용이 가능한 AWS 서비스도 사용이 가능해집니다.

예를 들면 ElastiCache을 Lambda에서 사용할 수 있게 됩니다. 기존 송금 Lambda에서는 ElastiCache가 있는 VPC와 통신이 불가능 했기 때문에 캐시 사용이 불가능했습니다. 하지만 이제 해당 VPC와 통신이 가능해지기에, 캐싱을 비롯한 INCR, NX 등 활용성이 높은 명령어를 사용할 수 있게 됩니다.

## Lambda VPC mode에서 고정 IP를 붙이는 방법들

이제 본격적으로 고정 IP를 붙이는 방법에 대해서 소개하갰습니다. Lambda VPC mode만으로는 고정 IP를 얻을 수 없습니다. 기초 공사라고 생각하시는 게 좋을 것 같아요. 방법은 크게 두 가지 입니다.

- Hyperplane ENI에 직접 EIP 붙이기
- NAT gateway 사용하기

### Hyperplane ENI에 직접 EIP 붙이기

앞서 VPC mode를 설명드릴 때 언급한 내용입니다. 말 그대로 VPC mode를 설정하면 생성되는 ENI에 EIP를 붙이는 것입니다. 장점으로는 매우 간단하다는 것입니다. 콘솔에서 네트워크 인터페이스를 들어간 후, 해당 ENI에 EIP를 설정하면 끝입니다.

하지만 hyperplane ENI의 특징인 '임시성' 때문에 저는 해당 방법을 택하지 않았습니다.


앞서 언급했듯, 해당 결제 수단은 24시간 돌아가는 시스템입니다. 하나의 EIP는 하나의 ENI에만 붙일 수 있기 때문에, ENI가 자동으로 생성되거나 삭제될 때마다 수동으로 EIP를 다시 설정해줘야 합니다. 이는 고정 IP를 유지하기 위한 운영 비용 증가로 이어집니다.

그래서 제가 택한 방법은, 정석적인 NAT gateway를 사용하는 것입니다.

### NAT gateway 사용하기

NAT란 내부 IP 주소를 가진 장치들이 외부 인터넷에 접속할 수 있도록 네트워크 주소를 변환해주는 기능을 말합니다.

AWS의 NAT gateway도 비슷합니다. NAT gateway는 퍼블릭 서브넷에 생성되고, 프라이빗 서브넷의 요청이 라우팅 테이블에 의해 NAT gateway로 전달되면, 사설 IP를 공인 IP로 변환하여 VPC 외부로 나가게 됩니다.

특징으로는 AWS가 완전관리를 해주기 때문에 매우 안정적입니다.

하지만 단점도 존재합니다. 우선 가격이 꽤 비쌉니다. 프로비저닝을 하게 되면 기본 요금이 존재하고, 데이터양에 따라서 GB당 요금이 부과됩니다.

또, private 서브넷에서 AWS의 서비스에 접근할 때도 외부 인터넷을 사용하기 때문에 보안적인 문제도 존재합니다.

이 두 가지를 모두 해소해줄 수 있는 것이 **'VPC Endpoint' 입니다.**

VPC Endpoint를 사용하면 VPC 내부에서 인터넷을 거치지 않고 NAT gateway 보다 저렴하게 AWS 내부 서비스와 통신이 가능하게 됩니다. 종류는 Interface endpoint와 Gateway endpoint가 존재합니다.
자세한 내용은 [이전 포스트](https://imasimdi.dev/aws/VPC-endpoint-%EC%A2%85%EB%A5%98%EC%99%80-%EC%B0%A8%EC%9D%B4%EC%A0%90)에서 설명하고 있으니 참고 부탁 드립니다.

## 최종 아키텍처

이렇게 해서 최종 아키텍처(송금 부분만 해당)은 다음과 같습니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/fc687c92-8081-4a53-9b96-37166af6fd2a/image.png)

불안정한 ENI에 직접 EIP를 붙이는 것 보다는, 약간의 비용상승을 고려하더라도 안전하게 송금이 될 수 있도록 NAT gateway를 사용하였습니다. 이 NAT Gateway에 EIP를 붙여서 완전히 고정된 IP로 외부 API 서버에 접근을 하도록 하였습니다.

NAT Gateway는 AWS가 완전 관리하기 때문에 Hyperplane ENI의 생명주기와 무관하게 안정적으로 고정 IP를 유지할 수 있습니다.

다만 NAT Gateway의 비용 부담을 줄이기 위해, 앞서 소개드린 VPC Endpoint를 함께 사용하였습니다. SQS, DynamoDB 등 AWS 서비스는 VPC Endpoint를 통해 AWS 내부 네트워크로 직접 통신하여 NAT Gateway를 거치지 않도록 했습니다. 외부 서드파티 API 호출 시에만 NAT Gateway를 사용하여 안정성과 비용 효율성의 균형을 맞췄습니다. 


## 트러블 슈팅

이렇게 송금 Lambda와 관련된 부분들을 VPC 내부에 넣은 이후, 이상한 일이 간헐적으로 발생했습니다. 그것은 메인 서버에서 SQS로의 호출이 되지 않는 것입니다.

많은 테스트로 SQS만 호출이 되지 않는 것을 확인했고, Fargate 내부로 들어가서 SQS Queue URL로 Curl 요청을 날려보았습니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/b6faf243-dcb1-47b9-bf00-c567a0b2f131/image.png)

이상하게도 private IP로 요청을 보내는 것을 보고, 메인 서버에서 SQS에 접근하지 못한 이유가 **Interface Endpoint** 때문이라고 눈치를 챘습니다.

VPC 내부에서 SQS용 interface endpoint가 존재하면 SQS 퍼블릭 도메인을 DNS가 프라이빗 IP로 해석합니다. 즉, **모든 SQS의 요청은 무조건 interface endpoint로 모이게 되는** 것입니다.

interface endpoint는 ENI가 생성되는 것이기 때문에 **'보안 그룹'** 을 설정해줘야 합니다. 전에 interface endpoint를 생성할 때 Lambda가 속한 보안 그룹만 접근을 허용하도록 설정해두었기 때문에, 다른 리소스에서는 SQS에 접근이 불가능한 상황이었습니다.

배포한 SQS의 interface endpoint에 메인 서버에 속한 보안 그룹을 443 포트로 추가를 하니, 정상적으로 통신이 잘 되었습니다.

## 정리

외부 서드파티의 요구사항 변화로, 많은 고민을 하면서 문제 해결을 하였습니다. 비록 고정 IP를 얻기 위해 송금 로직을 Private VPC에 넣었지만, 의도치 않게 보안도 향상되는 이점을 얻었습니다. 또 ElastiCache 사용이 가능해져서 성능적인 향상도 가져왔습니다.

은행과 통신을 하기 위해서는 '고유한 번호'가 필요했는데, 이를 DB 수준의 락을 사용해서 얻어오기 보다는 Redis 기반 `INCR`을 사용해서 가져오도록 변경했습니다. 여기에서도 많은 고민이 있었는데, [이전 포스트](https://imasimdi.dev/etc/%EB%8F%99%EC%8B%9C-%EC%9A%94%EC%B2%AD%EC%97%90%EC%84%9C-%EC%A0%95%ED%99%95%ED%95%9C-%EC%88%9C%EC%B0%A8%EB%B2%88%ED%98%B8-%EC%83%9D%EC%84%B1%ED%95%98%EA%B8%B0)에서 정리해두었었습니다. 덕분에 더 많은 동시 요청에도 안정적이게 송금이 가능해졌습니다.

덕분에 VPC에 대해서 다시 한번 복습하였고, Lambda 만의 특별한 ENI인 하이퍼 플레인 ENI에 대해서도 완벽하게 정리되었네요. 긴 글 읽어주셔서 감사합니다!