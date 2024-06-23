---
layout: post
category: AWS
---
- AWS에서 제공하는 Infra as Code
- AWS 리소스를 코드로 정의하고 관리할 수 있는 서비스

## 구성 요소
- Template: Json 또는 Yaml로 AWS소스를 정의, 리소스, 파라미터, outfut 등이 포함
- stack: 스택은 템플릿 기반으로 생성된 AWS 리소스 집합, 해당 스택을 업데이트하거나 삭제 가능
- parameter: 템플릿에 값을 동적으로 전달
- output: 스택이 생성한 리소스의 중요한 정보를 반환
- 그외 조건, 매핑이 존재

## 실전 예시
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 예시 서버 템플릿

Parameters:
    EC2InstanceType:
        Type: String
        Default: t2.micro
        Description: EC2 Instance Type
    ...

Resources:
  EC2Instance:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: !Ref EC2InstanceType
      ImageId: !Ref AMIId
      IamInstanceProfile: !Ref IAMInstanceProfile
      SecurityGroupIds:
        - !Ref SecurityGroup
      SubnetId: !Ref SubnetId1
      Tags:
        - Key: Name
          Value: !Ref InstanceName
    TargetGroup:
    Type: 'AWS::ElasticLoadBalancingV2::TargetGroup'
        Properties:
        Name: !Ref InstanceName
        TargetType: instance
        Protocol: HTTP
        Port: 80
        VpcId: !Ref VPCId
        HealthCheckProtocol: HTTP
        HealthCheckPort: 80
        HealthCheckPath: /
        Matcher:
            HttpCode: 200
        Targets:
            - Id: !Ref EC2Instance
            Port: 80
        DependsOn: EC2Instance
    LoadBalancer:
        Type: 'AWS::ElasticLoadBalancingV2::LoadBalancer'
        Properties:
        Name: !Ref InstanceName
        Scheme: internet-facing
        Subnets:
            - !Ref SubnetId1
            - !Ref SubnetId2
            - !Ref SubnetId3
            - !Ref SubnetId4
        SecurityGroups:
            - !Ref SecurityGroup
        DependsOn: TargetGroup

  HttpListener:
    Type: 'AWS::ElasticLoadBalancingV2::Listener'
    Properties:
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup
      LoadBalancerArn: !Ref LoadBalancer
      Port: 80
      Protocol: HTTP
    DependsOn: LoadBalancer
```
- Parameters에 변수를 담는다. 한 칸 들여써서 변수를 할당한다.
- Resource에 실제 리소스를 담는다. Type, Properties 순이다.
- `!Ref`로 파라미터의 값이나 템플릿을 통해 만들어진 리소스를 변수처럼 사용할 수 있다.
    - !Ref는 간편한 점이 리소스의 기본 식별자(주로 리소스의 ID)를 반환한다.
    - 하지만 lister의 경우 `LoadBalancerArn: !Ref myLoadBalancer`의 경우 Arn을 반환한다. SSM같은 경우는 직접 파라미터 이름을 반환하거나 SNS의 경우에도 주제의 ARN을 반환한다. 만들 때 공식문서를 잘 참고해야겠다.
- `!Sub`는 템플릿에서 문자열 내에서 변수를 대체하거나, 복잡한 문자열 조합을 만들 때 사용
    - 단순 문자열 대체에서는 `${}` 구문을 사용
    - `!Sub |` 을 통해 json을 만들거나 복잡한 매개변수를 대체한다.
- `Fn::GetAtt`는 특정 리소스의 속성 값을 가져오는데 사용된다.
- `Fn::Base64`는 함수의 입력 문자열의 Base64 표시를 반환한다.

    ```bash
    UserData: !Base64 |
                #! /bin/bash
                yum update
                yum install git
                yum install ansible
    ```