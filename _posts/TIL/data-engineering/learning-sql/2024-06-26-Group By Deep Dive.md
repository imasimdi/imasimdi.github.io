---
layout: post
category: data-engineering
subcategory: learning-sql
---
- 24년 6월 24일에 열린 AUSG 빅챗 발표 중 이태현님의 발표를 듣고, 더 찾아보며 정리한 내용입니다.

## `GROUP BY`은 어떤 조건에서 실행되는가

- `GROUP BY` 절은 특정 열을 기준으로 데이터를 그룹화하여 집계 함수(예: `MAX`, `SUM`, `COUNT`, `AVG`)를 사용할 때 유용
- 하지만 MySQL에서는 `sql_mode` 설정에 따라 `GROUP BY` 절의 동작 방식이 달라질 수 있다.

### 예시 쿼리
- `artwork-view-log` 테이블에서는 artwork와, 그것을 본 user_id 그리고 본 시각이 적혀있는 log가 존재한다.
- `artwork-view-log` 테이블에서 사용자별로 가장 마지막에 본 작품을 찾는 쿼리는 다음과 같다

```sql
SELECT 
    user_id,
    MAX(view_datetime) AS last_view_datetime,
    artwork_id
FROM 
    artwork_view_log
GROUP BY 
    user_id;
```
- 하지만 위 쿼리는 실행되지 않는다.  이유는 `artwork_id`가 `GROUP BY` 절에 포함되지 않았고, 집계 함수로 처리되지 않은 컬럼이기 때문이다.
- MySQL의 `sql_mode` 설정에 따라 이런 경우 오류가 발생할 수 있다.

### sql_mode = only_full_group_by

- `sql_mode`에 `only_full_group_by`가 설정되어 있는 경우, `GROUP BY`에 포함되지 않은 필드를 `SELECT` 절에 포함하면 오류가 발생
- 하지만 함수적 종속성을 고려하면 `only_full_group_by` 설정을 바꾸지 않아도 그룹바이가 실행된다.
- RDS, AURORA에는 `ONLY_FULL_GROUP_BY` 를 직접적으로 바꿀 수 없다.

### 함수적 종속성 (Functional Dependency)

- 함수적 종속성은 데이터베이스 이론에서 특정 속성 집합이 다른 속성 집합을 **유일하게 식별**할 수 있는 관계를 의미한다.
    - 예를 들어, 테이블에서 `A -> B`가 성립하면, `A` 값이 주어졌을 때 `B` 값이 유일하게 결정된다는 뜻
- 데이터 무결성과 관련이 깊음

| user_id | user_email       | user_type |
|---------|------------------|-----------|
| 1       | user1@example.com| admin     |
| 2       | user2@example.com| user      |
| 3       | user3@example.com| user      |

- `user_id`는 기본 키이며, `user_email`과 `user_type`은 `user_id`에 함수적으로 종속
- 즉, `user_id`가 주어지면 `user_email`과 `user_type`은 유일하게 식별 가능하다.
- 만약에 집계되지 않음 컬럼이 select되더라도, Group By 구의 대상이 되는 컬럼과 **함수적 종속성이 있다면 집계가 가능**하다.

#### 함수적 종속성 예시

```sql
SELECT 
    user_id,
    MAX(user_email) AS max_email,
    user_type
FROM 
    users
GROUP BY 
    user_id;
```

- `user_type` 필드가 `GROUP BY` 절에 포함되지 않았고, 집계 함수로 처리되지 않았기 때문에 원래대로라면 쿼리는 위의 논리대로라면 실행되면 안된다.
- 그러나 `user_id`가 기본 키이므로, `user_id`가 주어지면 `user_email`과 `user_type`이 유일하게 식별된다. 따라서 쿼리 실행이 가능하다.
#### `sql_mode` 설정 변경

`sql_mode` 설정을 변경하여 `ONLY_FULL_GROUP_BY`를 비활성화하면 쿼리를 실행할 수 있습니다:

```sql
SET sql_mode = '';

SELECT 
    user_id,
    MAX(user_email) AS max_email,
    user_type
FROM 
    users
GROUP BY 
    user_id;
```

이 경우 MySQL은 `user_id`가 `user_email`과 `user_type`을 함수적으로 결정한다고 가정하고 쿼리를 실행합니다.

### 요약

- **함수적 종속성**: 특정 속성 집합이 다른 속성 집합을 유일하게 결정할 수 있는 관계.
- 함수적 종속성이 있다면`GROUP BY` 절에 집계되지 않은 필드를 `SELECT` 절에 포함할 수 있음.
- **`sql_mode` 설정**: `ONLY_FULL_GROUP_BY` 설정이 활성화된 경우, `GROUP BY` 절에 포함되지 않은 필드를 `SELECT` 절에 포함할 수 없음.

## 효율적으로 `GROUP BY` 사용하기
### 복합 인덱스와 GROUP BY 필드 순서 맞추기 
- **인덱스 구성**: `UNIQUE(user_id, artwork_id, view_datetime)`와 같은 복합 인덱스를 사용하면 쿼리 성능이 향상
- **정렬 순서**: `GROUP BY` 절의 필드 순서가 인덱스의 필드 순서와 일치하면 인덱스를 효과적으로 사용 가능
- **EXPLAIN**: 쿼리 실행 계획을 확인하여 인덱스가 제대로 사용되는지 확인 가능
    - 순서를 맞추지 않으면 `Using temporary` 발생
    - 인덱스 정렬 순서가 탐색에 반영되지 못해서, 임시 테이블을 만들고 거기서 임시로 정렬을 한다. -> 비효율적
    - 반면에 순서를 맞추면 index full search가 일어남. -> 적용이 됨
- 이미 충분히 효율적이지만 COUNT(DISTINCT)를 통해서 더 최적화 가능

### DISTINCT를 사용하여 범위 검색
```sql
SELECT 
    user_id, 
    artwork_id, 
    COUNT(DISTINCT view_datetime)
FROM 
    views
GROUP BY 
    user_id, artwork_id;

```
- 각 (user_id, artwork_id) 조합에 대해 고유한 view_datetime 값의 개수를 세는 쿼리이다.

| id | select_type | table | type  | possible_keys       | key     | key_len | ref  | rows | Extra                    |
|----|-------------|-------|-------|---------------------|---------|---------|------|------|--------------------------|
|  1 | SIMPLE      | views | range | PRIMARY             | PRIMARY | 12      | NULL | 1000 | Using index for group-by |

- explain을 실행하면 다음과 같이 나오는데, `type: range`에 주목하자
- 만약 DISTINCT 키워드가 사용된 쿼리에서 해당 열에 인덱스가 존재한다면, 데이터베이스는 이 인덱스를 사용하여 효율적으로 범위 탐색 가능
- 인덱스를 사용하여 중복된 값을 건너뛸 수 있는데 이러한 범위 탐색은 인덱스가 **정렬된 상태**로 저장되어 있기 때문에 가능