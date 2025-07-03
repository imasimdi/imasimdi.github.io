---
layout: post
category: python
---

`pytest-mock`는 `unittest.mock` 를 pytest와 통합하여 더 편하게 만들어준 라이브러리입니다.

사용 방법은 `unittest.mock` 보다 훨씬 직관적입니다. (context manager와 yield를 사용하지 않아도 됨)

만약 `requests.get` 메서드를 모킹하려면 다음과 같은 코드를 작성하면 됩니다.

```python
def test_api_call(mocker):
    # 외부 API 호출을 mock으로 대체
    mock_requests = mocker.patch('requests.get')
    mock_requests.return_value.json.return_value = {'status': 'success'}
    
    result = my_function_that_calls_api()
    assert result['status'] == 'success'
    mock_requests.assert_called_once()
```

주요 메서드는 다음과 같습니다.

### patch

객체나 함수를 mock으로 대체해줍니다.

```python
def test_database_call(mocker):
    mock_db = mocker.patch('myapp.database.get_user')
    mock_db.return_value = {'id': 1, 'name': 'John'}
```

### patch.object

특정 객체의 메서드를 mock으로 대체합니다.

```python
def test_method_call(mocker):
    user = User()
    mocker.patch.object(user, 'save', return_value=True)
```

실제로 pytest와 같이 쓴 코드를 보면 pytest의 fixture와 매우 잘 통합이 됩니다.

```python
# 테스트할 코드
import requests

def get_weather(city):
    response = requests.get(f'http://api.weather.com/{city}')
    return response.json()

# 테스트 코드
def test_get_weather(mocker):
    # requests.get을 mock으로 대체
    mock_get = mocker.patch('requests.get')
    mock_get.return_value.json.return_value = {
        'temperature': 25,
        'condition': 'sunny'
    }
    
    result = get_weather('seoul')
    
    assert result['temperature'] == 25
    assert result['condition'] == 'sunny'
    mock_get.assert_called_once_with('http://api.weather.com/seoul')
```

### return_value란?

`return_value`를 왜 저렇게 덕지 덕지 붙이는지 이해가 잘 안가서 좀 찾아봤습니다.

`return_value`는 Mock 객체가 **호출되었을 때** 무엇을 반환할지를 설정하는 것입니다.

가벼운 예를 들자면 다음과 같습니다.

```python
from unittest.mock import Mock

# Mock 객체 생성
mock_function = Mock()

# 이 함수가 호출되면 "hello"를 반환하도록 설정
mock_function.return_value = "hello"

# 실제 호출
result = mock_function()  # "hello"가 반환됨
print(result)  # "hello"
```

이제 boto3를 예시로 실제 코드를 보여드리자면

```python
import boto3

# 1. boto3.resource()를 호출하면 DynamoDB resource 객체를 반환
dynamodb = boto3.resource('dynamodb')

# 2. resource.Table()을 호출하면 Table 객체를 반환  
table = dynamodb.Table('my-table')

# 3. table.get_item()을 호출하면 딕셔너리를 반환
response = table.get_item(Key={'id': '123'})
```

이렇게 되는데, 이걸 모킹하면 다음과 같이 됩니다.

```python
# Mock으로 위의 동작을 재현
mock_boto_resource = mocker.patch("boto3.resource")

# 1. boto3.resource()가 호출되면 mock_resource를 반환하도록 설정
mock_resource = Mock()
mock_boto_resource.return_value = mock_resource

# 2. mock_resource.Table()이 호출되면 mock_table을 반환하도록 설정
mock_table = Mock()
mock_resource.Table.return_value = mock_table

# 3. mock_table.get_item()이 호출되면 딕셔너리를 반환하도록 설정
mock_table.get_item.return_value = {
    'ResponseMetadata': {...}
}
```

그냥 반환하고자 하는 객체를 `Mock()` 객체로 생성하고, 차근차근 `return_value` 를 붙여나가면 될 것 같습니다.
정리하자면 다음과 같습니다.

```python
# 실제 코드 작동
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('users')        
response = table.get_item(Key={'id': '123'})  

# Mock은 이렇게 작동
mock_boto_resource.return_value = mock_resource
mock_resource.Table.return_value = mock_table
mock_table.get_item.return_value = {...}
```