---
layout: post
category: AWS
---

## samconfig.toml & templaye.yaml
- 우선 SAM의 samconfig.toml과 template.yaml을 통해서 개발 환경을 분리해보자
- samconfig.toml은 SAM CLI의 배포 설정을 저장하는 구성 파일이다.

### samconfig.toml

```toml
[dev]
[dev.global]
[dev.global.parameters]
stack_name = "dev-test"

[dev.deploy]
[dev.deploy.parameters]
s3_prefix = "dev-test"
region = "ap-northeast-2"
resolve_s3 = true
disable_rollback = true
image_repositories = []
capabilities = "CAPABILITY_IAM"
confirm_changeset = true
parameter_overrides = [
    "Environment=dev"
]

[prod]
[prod.global]
[prod.global.parameters]
stack_name = "prod-test"

[prod.deploy]
[prod.deploy.parameters]
s3_prefix = "prod-test"
region = "ap-northeast-2"
resolve_s3 = true
disable_rollback = true
image_repositories = []
capabilities = "CAPABILITY_IAM"
confirm_changeset = true
parameter_overrides = [
    "Environment=prod"
]
```

- `[dev]` 이나 `[prod]`와 같이 배포 설정을 분리할 수 있다.
- stack name을 통해서 cloudformation의 스택 이름을 정할 수 있다.
- `parameter_overrides`로 sam의 template.yaml 의 `Parameters`을 오버라이딩 할 수 있다.
- 예시에서는 Environment 파라미터를 오버라이딩해서 template.yaml에서 배포 환경에 따라 해당 값을 다르게 전달한다.

### template.yaml

- template.yaml에서는 cloudformation 템플릿을 사용해 서버리스 리소스(Fucntion, Scheduler) 을 정의한다.
- 기존 cloudformation 처럼 Globals, Parameters와 같은 설정이 있다.
- 여기에서 ssm parameter store에 운영환경 (dev, prod)별로 경로를 다르게 설정 한 뒤 변수들을 저장시켜둔다.
    - 예를 들면 `/example/prod/db-url` 과 `/example/dev/db-url`로 경로를 설정할 수 있다.
{% raw %}
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Parameters:
  Environment:
    Type: String

Globals:
  Function:
    Timeout: 30
    MemorySize: 256

    Tracing: Active
    LoggingConfig:
      LogFormat: JSON
  Api:
    TracingEnabled: true
Resources:
  TransferFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub "${Environment}-test"
      CodeUri: test/
      Handler: app.lambda_handler
      Runtime: python3.9
      Architectures:
      - x86_64
      Policies:
      - AmazonSQSFullAccess
      - SSMParameterReadPolicy:
            ParameterName: '/example/*'
      Events:
        SQSEvent:
          Type: SQS
          Properties:
            Queue: !Sub '{{resolve:ssm:/example/${Environment}/sqs-url}}'
            BatchSize: 5
            Enabled: true
      Environment:
        Variables:
          DB_URL: !Sub '{{resolve:ssm:/example/${Environment}/db-url}}'
          ENV: !Ref NowEnvironment
      ...
Outputs:
    ...
```

- samconfig.toml에서 설정한 parameter_overrides을 통해서 template.yaml의 `Parameters`을 환경별로 오버라이딩한다.
- sam의 환경변수나 기타 다른 설정에 ssm parameter store의 값을 가져오려면 정책에 `SSMParameterReadPolicy`를 추가해야 한다.
- 그리고 이것을 cloudformation의 `!Sub` 문법이나 `!Ref` 문법을 통해서 여러 곳에서 사용한다.
    - 예를 들면 `'{{resolve:ssm:/example/${NowEnvironment}/db-url}}'`은 cloudformation에서 ssm parameter store의 변수를 가져오는 문법인데, 여기에 `!Sub`를 통해서 Parameters의 `NowEnvironment` 값을 넣을 수 있다.
    - 트리거를 설정하는 Event에서도 SQS Queue url을 `Environment`로 나눌 수 있다.
- 참고로 아직 sam template 에서는 `SecretString` 유형의 파라미터를 불러올 수 없다.
{% endraw %}
## github action과의 통합

- 앞서 samconfig.toml에서 환경울 나눈 뒤, parameter overrides을 통해 template.yaml의 Parater로 값을 오버라이딩하여 개발/운영 환경에 따라서 필요한 환경변수 및 설정들을 나누는 것을 알아봤다.
- 이제 github action을 사용해 특정 branch에 따라서 해당 branch에 push시 다른 값의 파라미터들을 적용해 CI및 CD를 할 수 있다.

```yaml
name: Deploy SAM Application

on:
  push:
    branches:
      - develop
      - master

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v2

      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.9'

      - name: Install AWS SAM CLI
        run: |
          pip install aws-sam-cli

      - name: Set environment
        run: |
          if [ ${{ github.ref }} == 'refs/heads/master' ]; then
            echo "SAM_ENV=prod" >> $GITHUB_ENV
          else
            echo "SAM_ENV=dev" >> $GITHUB_ENV
          fi

      - name: Build SAM application
        run: |
          sam build --config-env ${{ env.SAM_ENV }}

      - name: Deploy SAM application
        env:
          AWS_ACCESS_KEY_ID: 
          AWS_SECRET_ACCESS_KEY: 
          AWS_REGION: ap-northeast-2
        run: |
          sam deploy --config-env ${{ env.SAM_ENV }} --no-confirm-changeset --capabilities CAPABILITY_IAM
```

- Set environment 단계에서 `${{ github.ref }}`를 통해, 해당 브랜치를 확인하고 $GITHUB_ENV에 SAM_ENV를 저장시킨다.
- 그리고 `sam build`를 할 때 `--config-env`를 통해 저장시킨 SAM_ENV를 불러와 samconfig.toml의 특정 환경으로 sam application을 buidl한다.
- 마찬가지로 `sam deploy`에서도 `--config-env`를 통해 samconfig.toml의 특정 환경 배포를 진행시킨다.

## 결론

- 여태까지 내가 설명한 과정을 그림으로 한눈에 그리면 이렇게 된다.

![alt text](/assets/images/AWS/image/3/image.png)

- 이렇게 sam을 통해서 lambda의 개발/운영 서버를 분리하면 보다 안전하게 애플리케이션을 테스트 및 운영할 수 있게 된다.