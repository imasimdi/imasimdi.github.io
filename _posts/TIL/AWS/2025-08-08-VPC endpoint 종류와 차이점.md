---
layout: post
category: AWS
---

자격증 공부 때 추상적으로만 알았던  interface endpoint와 gateway endpoint를 실제로 사용해야돼서 위치 차이와 작동 방식에 대해서 정리해보았습니다.

AWS의 VPC 엔드포인트는 크게 **Interface Endpoint**와 **Gateway Endpoint**로 나뉘며, 두 엔드포인트는 위치와 작동 방식에 차이가 있습니다.

## 위치 차이

**Interface Endpoint**는 VPC 내 특정 서브넷에 하나 이상의 ENI를 생성하여 배치됩니다. 즉, 프라이빗 IP를 가진 네트워크 인터페이스가 VPC 내 서브넷에 직접 존재하며, 엔드포인트를 통해 AWS 서비스와 통신합니다.

**Gateway Endpoint**는 VPC 자체에 위치하며, 서브넷 내부가 아닌 라우팅 테이블에서 특정 서비스 대상 주소, 예를 들면 S3나 DynamoDB를 가리키도록 설정하여 작동합니다. ENI가 생성되지 않습니다.

### 작동 차이

**Interface Endpoint**는 ENI를 통해 **프라이빗 IP로 AWS 서비스에 접근하는 방식**입니다. 보안 그룹과 연결되어 트래픽 제어와 보안 정책 설정이 가능하며, AWS PrivateLink 기술을 기반으로 동작합니다.

같은 리전뿐만 아니라 다른 리전이나 **온프레미스 환경**에서도 사용할 수 있어 활용도가 높습니다. **VPC 내 서브넷에 직접 배치**되며, **DNS**를 통해 엔드포인트의 프라이빗 IP를 조회합니다.

사용 시 비용이 발생하고, 가용성을 위해 각 가용 영역에 ENI를 배치하는 것을 AWS에서 권장하고 있습니다. SNS, SQS, Kinesis 등 다양한 AWS 서비스를 지원하며, 라우팅 테이블 설정 없이 ENI를 통해 트래픽이 직접 전송됩니다.

**Gateway Endpoint**는 **라우팅 테이블**을 통해 동작하는 방식입니다. 라우팅 테이블에서 엔드포인트가 대상 서비스의 프리픽스 리스트를 가리키도록 설정하면, 해당 서비스로 트래픽이 자동으로 전달됩니다. 즉, ENI를 사용하지 않습니다.

Amazon S3와 DynamoDB에만 사용 가능하다는 제약이 있지만, 비용이 발생하지 않는다는 어마어마한 장점이 있습니다. 다만 온프레미스 환경에서는 직접 접근할 수 없으며, VPC 내부에서만 동작합니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/d97bc705-5fb4-41c0-8924-245f0084cd8b/image.png)

실제로 살펴보면 이렇게 첫번째 줄이 interface endpoint고 밑이 Gateway Interface일 때

![](https://velog.velcdn.com/images/leehjhjhj/post/7af11824-d4b7-4096-9d50-09df3671e344/image.png)


interface endpoint만 서브넷 내부에 ENI가 생성된 것을 볼 수 있습니다.

### Interface Endpoint의 보안그룹 설정

보안 그룹은 Interface Endpoint ENI에 연결되어, 엔드포인트를 통하는 트래픽을 인바운드 및 아웃바운드 규칙으로 제어합니다.

인바인드 규칙은 보통 Interface Endpoint를 호출하는 VPC 내 리소스의 IP 또는 보안 그룹을 소스로 지정해 허용하는 것이 일반적입니다. 또 필요한 포트 또한 443 포트만 여는 경우가 많습니다.

아웃바운드 규칙은 기본적으로 모든 아웃바운드 트래픽이 허용되어야 하고, Interface Endpoint ENI에서 AWS 서비스 쪽으로 아웃바운드 연결이 원활히 이뤄지도록 해야합니다.