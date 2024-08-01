---
layout: post
category: AWS
---

## Resource와 API 정의

- Lambda를 배포할 때 바로 API로 배포해야하는 경우가 있는 경우 SAM을 활용하면 쉽게 API 배포가 가능하다.
- 우선 Resource에 API를 정의한다.

```yaml
Resources:
  DirectRegisterApi:
    Type: 'AWS::Serverless::Api'
    Properties:
      Name: 'Api'
      StageName: 'prod'
      Cors:
        AllowMethods: "'POST,OPTIONS'"
        AllowHeaders: "'content-type'"
        AllowOrigin: "'https://.com'"
```

- `AWS::Serverless::Api` 리소스를 사용하여 API Gateway를 생성
- 여기에서 Cors도 설정해줄 수 있다. 그리고 Function 부분에 Events를 작성해준다

```yaml
Events:
    GetResource:
        Type: Api
        Properties:
        RestApiId:
            Ref: Api
        Path: /post
        Method: post
```

- 여기에서 Path는 실제 API의 경로이고, stage의 이름에 합쳐져서 api gateway endpoint가 나온다.
- 또한 RestApiID는 상위의 `Ref`를 통해서 리소스의 ID를 참조한다.
- Method도 여기서 정의할 수 있다. 만약에 Post 로직이라면 Request Body를 받아야 하는데 이는 `AWS::ApiGateway::Model`를 활용해서 정의한다.

## Model 정의

```yaml
UserModel:
    Type: 'AWS::ApiGateway::Model'
    Properties:
      RestApiId: !Ref Api
      ContentType: application/json
      Name: UserModel
      Schema:
        type: object
        properties:
          user_id:
            type: integer
        required:
          - user_id
```

- ContentType은 데이터의 MIME 타입을 지정
- 해당 예시시는 `user_id` 라는 정수형 필드를 필요로 한다. 실제로 request body는 다음과 같다.

```
{
    "user_id": 0
}
```

- 만약의 nested request 처럼 복잡한 요청 본문을 정의 하라면 다음과 같이 중첩된 객체나 여러 필드를 포함한다.

```yaml
UserModel:
  Type: 'AWS::ApiGateway::Model'
  Properties:
    RestApiId: !Ref Api
    ContentType: application/json
    Name: UserModel
    Schema:
      type: object
      properties:
        user_id:
          type: integer
        name:
          type: string
        contact_info:
          type: object
          properties:
            email:
              type: string
              format: email
            phone:
              type: string
        addresses:
          type: array
          items:
            type: object
            properties:
              street:
                type: string
              city:
                type: string
              state:
                type: string
              postal_code:
                type: string
      required:
        - user_id
        - name
```

- 이런 경우의 실제 request_body는 다음과 같다

```
{
  "user_id": 123,
  "name": "John Doe",
  "contact_info": {
    "email": "john.doe@example.com",
    "phone": "123-456-7890"
  },
  "addresses": [
    {
      "street": "123 Main St",
      "city": "Anytown",
      "state": "Anystate",
      "postal_code": "12345"
    },
    {
      "street": "456 Elm St",
      "city": "Othertown",
      "state": "Otherstate",
      "postal_code": "67890"
    }
  ]
}

```

## URL/쿼리 파라미러 예시

- 우선 URL 파라미터는 Events에 정의된 API path에 `{}` 를 사용하여 파라미터를 받아준다

```yaml
Events:
    ApiEvent:
        Type: Api
        Properties:
        RestApiId: !Ref Api
        Path: /items/{item_id}
        Method: get
```

- 쿼리 파라미터는 별도의 설정없이 코드로 가져올 수 있다.
- 가령 `/item?page=1` 로 쿼리파라미터를 줬다면 밑의 방식으로 `1`을 가져올 수 있다.

```python
def lambda_handler(event, context):
    page = event.get('queryStringParameters', {}).get('page', None)
```

## Sam으로 API Gateway 배포와 Lambda Proxy

- AWS SAM를 사용해서 API Gateway와 Lambda를 배포할 때, 기본적으로 Lambda Proxy 통합을 사용한다.
    - Lambda Proxy 통합은?
    - API Gateway가 HTTP 요청을 그대로 Lambda 함수로 전달하고, Lambda 함수의 응답을 그대로 클라이언트에 반환하는 방식
- Lambda proxy가 아니라 기존의 api gateway의 lambda 통합도 사용할 수 있지만 매핑 템플릿을 명시적으로 정의해야하기 때문에 설정이 복잡하다.
- 따라서 람다 프록시를 사용한다면 모든 return 값에 statusCode와 함께 header를 직접 지정해줘야한다.

```python
return {
  "statusCode": 200,
  'headers': {
    'Access-Control-Allow-Headers': 'Content-Type',
    'Access-Control-Allow-Origin': 'https://.com',
    'Access-Control-Allow-Methods': 'POST, OPTIONS'
  },
  "body": json.dumps(response_body, default=json_default)
}
```
