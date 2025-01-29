---
layout: post
category: AWS
---

`requests` 라이브러리로 메시지 핸들러 람다에 POST 요청을 해야하는데, 현재 메시지 핸들러 람댜에는 별도의 api gateway를 붙이지 않고 아주 간편한 람다 function url을 사용하고 있습니다.  

문제는 이 function url은 람다가 삭제되지 않은 이상 변경되지는 않지만, 저는 SAM을 통해서 람다를 관리하고 있기 때문에  clourformation 스택제거를 통해서 언제든지 람다가 삭제될 수 있는 가능성이 있습니다.

그렇다고해서 굳이 API gateway를 달기에는 조금 과한 것 같아 람다에서 즐겨쓰고있던 **SSM Parameter Store**를 본서버에도 적용시켜보았습니다.

우선 매번 boto3를 생성해서 Parameter store를 부르기에는 코드 반복이 심해지니 이러한 객체를 만들어주었습니다.

```python
class ParameterStore:
    def __init__(self):
        self.ssm = boto3.client('ssm', region_name="ap-northeast-2")

    def get_parameter(self, parameter_name: str, with_decryption: bool = False) -> str:
        try:
            response = self.ssm.get_parameter(
                Name=parameter_name,
                WithDecryption=with_decryption
            )
            return response['Parameter']['Value']
        except Exception as e:
            print(f"파라미터 조회 실패: {e}")
            raise NetworkExceptions from e
```

이렇게 한번 감싸주면 밑처럼 쉽게 외부 파라미터를 **동적**으로 가져올 수 있습니다.

```python
import requests

parameter_store = ParameterStore()
fcm_handler_url: str = parameter_store.get_parameter(f"/fcm/{env}/fcm-handler-url")
fcm_api_key: str = parameter_store.get_parameter("/fcm/api-key")
headers: dict[str, str] = {'API_KEY': fcm_api_key}
requests.post(fcm_handler_url, json=message, headers=headers)
```

이러면 만약 람다의 function url이 달라지는 일이 생겨도, 운영서버는 재배포하지 않고 파라미터 스토어에서 값만 변경한다면 새 파라미터 값을 가져올 수 있습니다. 아주 간편하죠?

생각해보니 이것을 잘 활용하면 저번에 혁준이가 TIL에 올렸었던 것 같은 Feature Toggle 아키텍처를 쉽게 구성할 수 있을 것 같습니다.

Feature Toggle 아키텍처는 런타임에 기능을 동적으로 활성화/비활성화할 수 있게 해주는 소프트웨어 디자인 패턴입니다.  이는 마치 전등 스위치처럼 소프트웨어의 특정 기능을 켜고 끌 수 있게 해주는 패턴입니다. 코드를 변경하거나 재배포하지 않고도 기능의 활성화 상태를 제어할 수 있는게 큰 특징입니다.

그래서 주로 A/B 테스트를 하거나, 긴급 상황시 빠르게 모드를 바꾸거나 해줄 수 있습니다.