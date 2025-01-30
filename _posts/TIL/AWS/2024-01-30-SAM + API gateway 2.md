---
layout: post
category: AWS
---

1편에 이어서 2편에서는 더욱 간편하게 함수에 API gateway를 연동시키는 방법에 대해서 소개하겠습니다.

```yaml
ApiGatewayApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: Prod
      EndpointConfiguration: 
        Type: REGIONAL
```

우선 이런 식으로 sam의 template.yaml에 API gateway를 정의해줍니다.

`EndpointConfiguration` 은 API Gateway의 엔트 포인트 타입을 지정해줍니다. `REGIONAL` 로 설정하면 API가 배포된 AWS 리전에서 요청을 처리하며, 같은 리전의 다른 AWS 서비스나 리소스와 통신할 때 최적의 성능을 제공해줍니다. `PRIVATE` 는 VPC 내부에서만 접근 가능한 API를 만들어주고, `EDGE` 는  CloudFront의 전 세계 엣지 로케이션을 활용하여 사용자와 가장 가까운 위치에서 요청을 처리해줍니다.

연결할 함수 리소스 하단에 다음과 같이 Url을 기준으로 작성해줍니다.

```yaml
Events:
	RedirectUrl:
	  Type: Api
	  Properties:
	    RestApiId: !Ref ApiGatewayApi
	    Path: /{hash}
	    Method: GET
	RedirectUrlMain:
	  Type: Api
	  Properties:
	    RestApiId: !Ref ApiGatewayApi
	    Path: /
	    Method: GET
	RedirectUrlWithUniqueId:
	  Type: Api
	  Properties:
	    RestApiId: !Ref ApiGatewayApi
	    Path: /{hash}/{unique_id}
	    Method: GET
```

이렇게 생성하면 프록시 통합된 API gateway 리소스들이 생성이 됩니다.

![image.png](https://velog.velcdn.com/images/leehjhjhj/post/03b03f76-adeb-495d-9fa2-1fb61531d974/image.png)

여기서  `/{}` 는 무엇일까요?

만약 `/{hash}/{unique}` 로 지정하게 되면 람다의 `event` 변수에 다음과 같은 데이터 형식으로 값이 들어옵니다.

```yaml
{
    'resource': '/{hash}/{unique_id}',
    'path': '/p/12345',
    'httpMethod': 'GET',
    'headers': {
        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7',
        ...
        'X-Forwarded-Proto': 'https'
    },
    'multiValueHeaders': {
        ...
    },
    'queryStringParameters': None,
    'multiValueQueryStringParameters': None,
    'pathParameters': { # 여기 주목 !! 
        'hash': 'p',
        'unique_id': '12345'
    },
    ...
}
```

즉, url에 지정해둔 파라미터 그대로 `pathParameters` 안에 키 값의 형태로 요청이 들어옵니다. 이것을 지정해주지 않고 `/p/12345` 와 같은 요청을 보내면 API Gateway는 500 에러를 반환합니다. 한마디로, 선택 사항이 아니라 **꼭** 지정해줘야 합니다!

여담으로 이렇게 Domain 이름도 붙여주면 간단하게 api gateway에 도메인도 붙일 수 있습니다.

```yaml
ApiGatewayDomainName:
    Type: AWS::ApiGateway::DomainName
    Properties:
      DomainName: !Ref DomainName
      RegionalCertificateArn: !Ref CertificateArn
      EndpointConfiguration:
        Types: 
          - REGIONAL
      SecurityPolicy: TLS_1_2
```

도메인에 대한 인증서가 있는 Certificate의 arn을 `RegionalCertificateArn`에 넣어주게 되면 간단하게 api gateway에 도메인을 사용할 수 있게 됩니다.