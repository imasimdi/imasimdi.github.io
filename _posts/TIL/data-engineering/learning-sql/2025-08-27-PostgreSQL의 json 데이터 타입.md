---
layout: post
category: data-engineering
subcategory: learning-sql
---

## PostgreSQL의 json 데이터 타입

PostgreSQL은 일반 JSON 타입과 이진 저장 및 인덱싱이 가능한 jsonb 타입을 제공합니다. 이 jsonb 타입으로 데이터를 저장하면 쿼리 성능이 훨씬 뛰어나고 저장 공간 최적화도 가능합니다.

쿼리 성능이 뛰어난 이유는 바로 `JIN` 인덱스 덕분인데, 이 인덱스를 사용하면 JSONB 문서 내 복수의 키와 값에 대한 부분 인덱싱이 가능해, 전체 문서를 스캔하지 않고도 빠른 검색 가능하다고 합니다.

## GIN 인덱스 작동 방식

- jsonb 컬럼의 모든 키와 값 쌍을 추출해 인덱스 구조에 저장
- 각 키와 값은 별도의 인덱스 엔트리로 관리
- 키 존재 여부나 특정 키-값 포함 여부 검사에 매우 빠른 쿼리 성능을 제공

### 연산자 종류

- **containment 연산자 (@>)**
    - 왼쪽 jsonb 값이 오른쪽 jsonb 값을 포함하는지 검사
        
        ```sql
        SELECT * FROM test WHERE data @> '{"field": "value"}';
        ```
        
    - data 컬럼의 JSON 값이 key "field"에 "value"를 포함하고 있으면 true를 반환합니다. 즉, 키-값 쌍 포함 여부를 체크합니다.
- **key existence 연산자 (?)**
    - JSON 개체에서 특정 키의 존재 여부를 검사합니다.
        
        ```sql
        SELECT * FROM test WHERE data ? 'field';
        ```
        
    - ata JSON 객체의 최상위에 key "field"가 있으면 true를 반환합니다.

### jsonb_path_ops란?

PostgreSQL jsonb에 대해 GIN 인덱스를 생성할 때 선택할 수 있는 operator class 중 하나로, 기본 값인 jsonb_ops에 비해  인덱스 크기가 훨씬 작고, containment 쿼리(@>)에 대해 더 빠른 성능을 제공합니다.

containment 쿼리에서 더 빠른 성능을 낼 수 있는 건 내부적으로 key와 값을 합쳐 해시한 단일 인덱스 엔트리를 만들기 때문입니다.

그런데 key existence 연산자 및 다른 연산자에는 적용되지 않기 때문에 키 존재 여부 검사를 자주 한다면 **기본 값인 jsonb_ops이 적합!**

## GIN 인덱스를 실제로 사용하지 못하는 경우

많은 경우가 GIN 인덱스를 사용하지 못하는데, 대표적인 경우는 밑과 같습니다.

- JSONB 내부의 중첩된 필드나 특정 경로 위치의 값을 직접 비교하는 쿼리
- JSONB 내부 값의 범위 비교(>, <, >=, <= 등)나 타입 변환 후 비교
- 정규 표현식이나 와일드카드를 사용하는 텍스트 매칭
- JSON 값을 가져와서 함수나 연산을 수행하는 경우

## 그러면 단점은?

GIN 인덱스는 쓰기 작업 시 오버헤드가 크므로 자주 갱신되는 큰 JSONB 컬럼은 비효율적입니다.

또, 위와 같이 키-값 쌍의 포함 여부를 빠르게 판단하는데 최적화 되어있어, 세밀한 경로나 값의 범위, 또 특정 패턴을 판단하는 쿼리에는 제한적입니다.

그래서 **단순히 JSON 데이터를 저장하고 검색을 하지 않는 경우**에는 GIN 인덱스를 생성하는 걸 심사숙고 해봐야겠네요..!