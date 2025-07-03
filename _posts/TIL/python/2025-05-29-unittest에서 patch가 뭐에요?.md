---
layout: post
category: python
---

파이썬의 unittest.mock 모듈에는 `patch` 라는 놈이 존재하는데, 쉽게 말하자면 테스트 중에 특정 함수나 메서드를 가짜 객체로 대체하는, 즉 모킹하는 기능입니다.

사용 방법은 데코레이터, 컨텍스트 매니저 둘 다 사용 가능합니다.

- 컨텍스트 매니저로 사용할 때
    
    ```python
    with patch('Controller.Alarm.c_email.sendQna') as mock_send_qna:
        # 이 블록 안에서만 sendQna가 mock으로 대체됨
        ai_callback_handler.handle_ai_callback(callback_request, "customer_inquiry")
        
        # mock 객체의 호출 여부 확인
        mock_send_qna.assert_not_called()  # 호출되지 않았는지 확인
        # 또는
        mock_send_qna.assert_called_once()  # 한 번 호출되었는지 확인
    ```
    
- 데코레이터로 사용 될 때
    
    ```python
    @patch('Controller.Alarm.c_email.sendQna')
    def test_example(self, mock_send_qna):
        # 테스트 로직
        mock_send_qna.assert_called_once()
    ```
    

### 사용처

이제 이것을 사용해서 해당 객체가 호출이 되었는지 알 수 있습니다.

```python
with patch('Controller.Alarm.c_email.sendQna') as mock_send_qna:
    # 테스트 실행
    some_function_that_calls_sendQna()
    
    # 호출 여부 확인
    mock_send_qna.assert_called()           # 호출되었는지
    mock_send_qna.assert_not_called()       # 호출되지 않았는지  
    mock_send_qna.assert_called_once()      # 정확히 한 번 호출되었는지
    mock_send_qna.assert_called_with(arg1, arg2)  # 특정 인자로 호출되었는지
    
    # 호출 횟수 확인
    assert mock_send_qna.call_count == 2
```

또한 이렇게 객체의 return value를 넣어줘서 외부 API를 모킹할 수도 있습니다.

```python
@pytest.fixture(scope="function")
def mock_lambda_client(self):
    with patch("boto3.client") as mock_boto_client:
        mock_client = Mock()
        
        # 성공 응답 기본 설정
        mock_client.invoke.return_value = {
            'StatusCode': 202,
            'ResponseMetadata': {'HTTPStatusCode': 202},
            'Payload': Mock()
        }
        
        mock_boto_client.return_value = mock_client
        yield mock_client
```

mock_boto_client는 boto3.client 함수를 모킹합니다.

그리고 `mock_client = Mock()` 를 통해서 실제 Lambda client를 흉내낼 객체를 생성합니다. 이후에 `invoke.return_value` 로  `invoke` 메서드의 반환 값을 만들어 주고, 아래와 같이 mock_boto_client가 해당 return_value를 갖도록 지정해줍니다.

```python
mock_boto_client.return_value = mock_client
```

이렇게 설정하면 `mock_boto_client`를 호출했을 때, `mock_client`가 반환되고, `mock_client` 의 invoke 메서드는 지정한 값을 반환하게 해줍니다.

그러면 실제 코드에서 Lambda 클라이언트의 호출 과정은 이렇게 됩니다.

```python
import boto3
lambda_client = boto3.client('lambda')  # mock_client가 반환됨
response = lambda_client.invoke(...)     # mock_client.invoke() 호출
```