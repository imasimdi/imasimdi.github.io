
```python
def find_hash_by_origin_url(self, origin_url: str) -> list[Optional[dict]]:
        response = self._table.query(
            IndexName='url-index',
            KeyConditionExpression='ou = :url',
            ExpressionAttributeValues={
                ':url': origin_url
            }
        )
        items: list[Optional[dict]] = response.get('Items', [])
        return items
```

- 최근에 제작 중인 URL 단축기에 사용한 코드다. 여기서 주목해야 할 것은 `KeyConditionExpression`과 `ExpressionAttributeValues` 이다.

## KeyConditionExpression

- 이것은 Query 작업에서만 사용하는 것인데 기본 형식은 다음과 같다.

```python
KeyConditionExpression='partition_key = :value'
```

- DynamoDB에서 저렇게 `=` 으로 바로 찾을 수 있는 것들은 파티션키, 파티션키 + 정렬 키 조합, GSI의 파티션키, GSI의 파티션 키와 정렬 키 조합, 파티션키와 LSI 정렬키 조합이 가능하다다.
- 일반 키는 단독으로 KeyConditionExpression을 통해서 `=` 조회가 불가능하고, `scan` 메서드의 `FilterExpression` 를 사용해야 한다. scan 메서드는 테이블의 모든 아이템을 읽어오기 때문에 매우 비효율적이다.
- 정렬키와의 조합으로는 이렇게 검색할 수 있다. SQL의 WHERE 절과 매우 닮아있다.

```python
KeyConditionExpression='partition_key = :pk AND sort_key = :sk'
```

- 이렇게 정렬키와 함께 검색할 때 `BETWEEN AND` 와 같은 질의도 가능하다.

```python
KeyConditionExpression='pk = :pk_val AND sk BETWEEN :start AND :end'
```

- GSI와 같은 인덱스를 사용한다면 다음과 같이 `IndexName` 을 명시해줘야 한다.

```python
response = table.query(
    IndexName='url-index',
    KeyConditionExpression='ou = :url'
)
```

- 기본적으로 `get_item` 메서드와 다르게 `query` 메서드는 반환 값이 `list[dict]` 인 것을 잊지 말아야 한다.

## UpdateExpression

- 업데이트 할 때 사용하는 표현식으로, SQL의 UPDATE문과 상당히 닮아있다.

```python
# SET: 값 설정/수정
UpdateExpression='SET #name = :name, age = :age'

# REMOVE: 속성 제거
UpdateExpression='REMOVE deletedAt'

# ADD: 숫자에 더하기 또는 세트에 요소 추가
UpdateExpression='ADD #count :increment'

# DELETE: 세트에서 요소 제거
UpdateExpression='DELETE categorySet :category'
```

- `UpdateExpression` 을 사용할 때는 `Key` 를 꼭 명시해줘야 한다.
- 해당 표현식은 `update_item` 메서드와 함께 사용된다.

## ExpressionAttributeValues

- 맨 처음 코드에서도 유추 가능하듯, 값을 바인딩 해주는 역할이다. 바인딩 한 값을 위의 `KeyConditionExpression` 나 `UpdateExpression` 에 사용할 수 있다.

## ExpressionAttributeNames

- DynamoDB에도 많은 예약어가 있는데, 해당 예약어를 바인딩할 때는 `ExpressionAttributeNames` 를 사용해주는 것이 좋다.
- `#`으로 시작하는 별칭을 만들어서 `KeyConditionExpression` 나 `UpdateExpression` 에 사용된다.
- 밑의 예시는 `ExpressionAttributeNames` 와 `ExpressionAttributeValues` 를 같이 쓴 예시다.

```python
response = table.update_item(
    Key={'id': '123'},
    UpdateExpression='SET #name = :name_val, #status = :status_val, #timestamp = :ts_val',
    ExpressionAttributeNames={
        '#name': 'name',
        '#status': 'status',
        '#timestamp': 'timestamp'
    },
    ExpressionAttributeValues={
        ':name_val': 'John Doe',
        ':status_val': 'active',
        ':ts_val': 1234567890
    }
)
```

## ProjectionExpression

- 반환한 속성을 지정할 수 있는데, SQL의 SELECT문과 같은 역할을 한다.

```python
response = table.query(
    IndexName='status-index',
    KeyConditionExpression='status = :status',
    ProjectionExpression='id, #name, email',
    ExpressionAttributeNames={
        '#name': 'name'
    },
    ExpressionAttributeValues={
        ':status': 'active'
    }
)
```

- 이렇게 DynamoDB의 Expression을 정리해보았다. 개인적으로 query에 저런 카멜형식의 파라미터들을 다는 것이 좋지 않아보여서 캡슐화가 필요해보인다.