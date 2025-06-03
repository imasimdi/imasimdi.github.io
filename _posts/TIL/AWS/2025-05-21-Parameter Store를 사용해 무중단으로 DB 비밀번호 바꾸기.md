---
layout: post
category: AWS
category-show: 아티클
tags: [article]
---

지난 [환경변수 관리 개선기](https://imasimdi.dev/aws/AWS-Secrets-Manager%EC%99%80-Parameter-Store%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%9C-%ED%99%98%EA%B2%BD%EB%B3%80%EC%88%98-%EA%B4%80%EB%A6%AC-%EA%B0%9C%EC%84%A0)에 이어 이번에는 회사의 오래된 DB 비밀번호를 다운타임 없이 변경하기 위해 고민했던 경험을 정리해보았습니다.

## 고민점

DB의 오래된 비밀번호를 변경해야 하는 상황에서 몇 가지 큰 고민이 있었습니다.

**1. 다운타임 발생 우려**
사용자의 비밀번호를 변경하면 곧바로 다운타임이 발생하여 유저들의 불편이 예상되었습니다. 애플리케이션에서 DB 세션을 그리 오래 유지하지 않았고, 설령 길게 유지하더라도 그 사이에 변경하기에는 불확실성이 너무 큽니다.

**2. 대규모 재배포 필요**
DB 정보를 공유하는 곳이 운영 서버뿐만 아니라 약 20개가 넘는 Lambda까지 포함되어 있어 DB 비밀번호를 변경하면 모든 서비스를 재배포해야 합니다.
만약 이전 개선 작업으로 생성한 Secrets Manager에서 동적으로 DB 정보를 가져온다면, Lambda 호출 때마다 과금이 발생하여 비용 효율적이지 않는 문제도 존재합니다.

**3. Secrets Manager의 Aurora 비밀번호 자동 로테이션의 한계**
AWS Secrets Manager에서 제공하는 Aurora 비밀번호 자동 로테이션 기능도 고려해보았지만 이 또한 동일한 문제가 존재했습니다.
애플리케이션에서 DB 정보를 동적으로 가져오지 않는 이상, 비밀번호가 자동으로 변경되었을 때 모든 서비스의 재배포가 필요합니다. 만약 재배포를 즉시 하지 않는다면 서비스에 장애가 발생할 수 있어 결국 운영상의 리스크가 여전히 존재했습니다.
    
이러한 고민점을 어떻게 해결할 지 많은 고민을 하였는데요, 고민 하던 중에 **blue green 배포 전략**이 떠올라서 다음과 같은 아키텍처를 설계해보았습니다.

## 설계

![](https://velog.velcdn.com/images/leehjhjhj/post/d96f0917-b4ad-40f3-be64-053333922421/image.png)

작동 방식은 다음과 같습니다.

- 초기 상태: Parameter Store에 `blue_green_env` 값이 존재하며, 현재 blue로 설정되어 있고 DB_URL도 blue DB URL로 가동 중
- 전환 준비: Green DB URL을 수동으로 생성하고, `blue_green_env`를 green으로 변경
- 전환
	- Lambda: 공통된 Lambda Layer에서 `blue_green_env` 값에 따라 동적으로 Parameter Store에서 DB 정보를 가져옴
    - 운영 서버: API 호출마다 동적으로 DB 정보를 가져오면 지연시간이 늘어나므로 재배포 후 `blue_green_env`에 따라 DB URL을 가져와서 애플리케이션 실행
- 정리: Blue 커넥션이 충분히 끊긴 후 Blue 사용자 삭제

이 방식을 통해 DB 정보, 특히 비밀번호를 변경할 때 사용자의 다운타임을 최소화할 수 있습니다.

## 구현 과정

### 운영 서버
이전에 개선한 아키텍처에서 DB URL을 Secrets Manager 대신 `blue_green_env`가 포함된 변수(예: db/blue/password)를 Parameter Store에서 가져오도록 변경했습니다.

### 람다

람다에서는 애플리케이션 실행 시 동적으로 값을 가져오도록 Lambda Layer에 다음과 같은 코드를 추가했습니다.

```python
import os

from parameter_store import ParameterStore

env: str = os.environ.get("ENV", "dev")
parameter_store: ParameterStore = ParameterStore()

BLUE_GREEN_ENV: str = parameter_store.get_parameter(f"/db/{env}/blue-green-env")
WRITER_DB_URL: str = parameter_store.get_parameter(f"/db/{env}/{BLUE_GREEN_ENV}/writer-db-url")
READER_DB_URL: str = parameter_store.get_parameter(f"/db/{env}/{BLUE_GREEN_ENV}/reader-db-url")
```

이렇게 하면 Lambda는 parameter store에서 값을 동적으로 가져오기 때문에 비용 효율적이고, DB 비밀번호 변경시 수 많은 람다들을 재배포 할 필요도 없게 되었습니다.

입사 이후 하고 싶었던 숙원 작업 중 하나였는데, 큰 문제 없이 해냈어서 기분이 좋네요!



