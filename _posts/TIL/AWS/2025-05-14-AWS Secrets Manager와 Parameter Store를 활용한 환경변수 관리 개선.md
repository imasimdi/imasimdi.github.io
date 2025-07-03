---
layout: post
category: AWS
category-show: 아티클
tags: [article]
---

![](https://velog.velcdn.com/images/leehjhjhj/post/5d97a3ea-53f1-446d-92cb-904d126d7504/image.png)

## 현재 문제점

저희 회사의 환경변수 관리에는 몇 가지 문제점이 있었습니다. 로컬에 환경변수 추가 시 팀원 전체가 각자 개별적으로 추가해야 하는 번거로움과 운영 ECS 환경으로 배포 시 여러 task 정의를 수동으로 업데이트하고 서비스를 재배포해야 하는 불편함이 존재했습니다.

이전 환경변수 관리 프로세스는 다음과 같았습니다

1. 개발 환경에 추가 시 모든 팀원들에게 고지
2. 각 개발 환경 스크립트에 환경변수 추가
3. 운영 환경 배포 시 모든 ECS 태스크 정의에 추가, 서비스 업데이트 작업

이렇게 환경 변수를 관리하면 특정 task 정의에 환경변수를 넣지 않고 배포하는 휴먼 에러 발생 가능성이 컸고, ECS 서비스 개수가 늘어날수록 환경변수 업데이트에 소요되는 시간이 길어졌습니다. 추가로 팀원 간 환경 변수 차이로 인한 문제와 실수로 운영 DB에 로컬 환경에서 접근하는 위험한 상황도 발생했습니다.

이번에 회사 인프라를 개선하는 작업을 하는 김에, 환경변수 관리도 조금 더 편하게 하기 위해 다음과 같은 방법으로 개선을 진행했습니다.

## 개선 1: AWS Secrets Manager로 고정 환경변수 통합

고정적인 환경변수들은 AWS Secrets Manager로 통합했습니다. 해당 변수들을 Parameter Store에 저장하는 것도 고려하였으나 다음과 같은 이유로 Secrets Manager를 선택하였습니다.

### 저장할 수 있는 글자 수 차이

AWS Secret Manager는 최대 64KB까지 저장가능하지만, Parameter Store는 4KB에 불가합니다. 따라서 Parameter Store에는 많은 환경 변수를 한꺼번에 저장하기에는 부족하기에 개별로 저장할 수 밖에 없습니다. AWS Secret Manager에서는 ASCII 문자 사용시 65,536자까지 저장이 가능하기 때문에, 전체 환경변수를 저장할 수 있었습니다.

### 자동 로테이션

AWS Secret Manager에서는 자동 로테이션이라는 기능이 존재하는데, 주기적으로 특정 Lambda를 실행해서 작업을 수행할 수 있습니다. 더욱 장점인 것은 EventBridge Scheduler를 사용하지 않아도 로테이션 일정을 설정할 수 있다는 것 입니다. 참고로 Amazon RDS, Aurora, DocumentDB 등 일부 AWS 서비스는 자동으로 관리되는 로테이션 제공을 제공합니다.

운영 환경에 따라서 보안 암호를 각각 생성하고, 애플리케이션 구동 시 다음 코드로 Secrets Manager에서 값을 가져와 Pydantic 객체에 주입시킵니다.

```python
def load_settings():
    """환경에 따라 설정을 로드하는 함수"""
    secret_name = f"env/{secret_manager_env}"

    session = boto3.session.Session()
    client = session.client(
        service_name='secretsmanager',
        region_name="ap-northeast-2"
    )
    try:
        get_secret_value_response = client.get_secret_value(SecretId=secret_name)
        secret = get_secret_value_response['SecretString']
        secrets_data = json.loads(secret)
        lowercase_secrets = {k.lower(): v for k, v in secrets_data.items()}
        return Settings(**lowercase_secrets)
    except Exception as e:
        raise ValueError("설정을 로드할 수 없습니다.")

try:
    settings = load_settings()
except ValueError as e:
    print("에러:", e)

```

## 개선 2: Parameter Store로 가변 환경변수 관리

다양한 환경변수에 따라 별도로 관리되어야 하는 환경변수들(예를 들어 client_url, server_url 등)은 별도 관리가 필요했습니다. 브랜치 서버마다 다른 값을 가져야 하는 변수들을 위해 매번 Secert Manager의 새 보안 암호를 생성하는 것은 중복이 많이 발생할 뿐더러 비용 효율적이지 않았습니다.

그래서 SSM Parameter Store를 함께 활용하는 하이브리드 접근법을 채택했습니다.

```python
parameters = {
    "server_domain": parameter_store.get_parameter(f"{env}/server_domain"),
    "client_url": parameter_store.get_parameter(f"{env}/client_url")
}

class Settings(BaseModel):
    server_domain: str = Field(default=parameters['server_domain'])
    client_url: str = Field(default=parameters['client_url'])
    # ... 기타 고정 변수들

```

환경 식별자(여기서는 `env`)를 기반으로 Parameter Store에서 해당 환경에 맞는 값을 가져오는 방식입니다. 로컬 환경이면 'local', 개발 환경이면 'dev', 브랜치 환경이면 'branch-1' 같은 접두어로 파라미터를 구성했습니다.

정리하자면 다음과 같습니다.

- **AWS Secrets Manager**: 전체 개발 환경이 공유하는 고정적인 환경변수(DB 접속 정보, API 키 등)
- **Parameter Store**: 환경별로 달라져야 하는 가변적인 환경변수(URL, 도메인 등)

## 개선 효과

가장 두드러지는 것은 개발 생산성을 비롯한 관리 효율성이 높아졌습니다. 모든 팀원이 동일한 환경 변수를 사용하도록 강제함과 더불어, 환경 변수의 중앙 집중화로 한 곳에서만 환경 변수를 추가 / 삭제 / 변경을 하면 모든 ECS 서비스 및, 개발 서버에서도 자동으로 적용을 시킬 수 있어 매우 편리해졌습니다.

또한 일관성 유지 + 배포 간소화 + 관리 효율성을 비롯하여 더 이상 로컬에 환경 변수들을 저장하지 않아도 된다는 측면에서 보완성까지 챙길 수 있었습니다.

AWS 관리형 서비스로 좋은 개선을 이루어낸 사례인 것 같아 아주 뿌듯한 경험이었습니다 😁