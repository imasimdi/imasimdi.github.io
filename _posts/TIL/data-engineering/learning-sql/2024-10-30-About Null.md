---
layout: post
category: data-engineering
subcategory: learning-sql
tags: [SQL]
---

## Where 조건

```sql
WHERE column = NULL
WHERE column IS NULL
WHERE column != 'A'
```

- `= Null` 은 작동하지 않는다. `IS NULL` 이 올바름
- 이 부분이 중요한데 `!=`를 한다고 해도 Null이 포함되지 않는다. Null은 제외된다.

## 집계 함수에서의 NULL

```sql
COUNT(*)
COUNT(column)        -- NULL 제외하고 카운트
SUM(column)          -- NULL 무시
AVG(column)          -- NULL 무시
MAX(column)          -- NULL 무시
MIN(column)          -- NULL 무시
```

- `COUNT(*)`를 제외한 모든 집계 함수는 NULL을 무시

## Group By 에서의 NULL

```sql
INSERT INTO orders VALUES
(1, 100, 'COMPLETED'),
(2, NULL, 'PENDING'),
(3, 200, NULL),
(4, 300, 'COMPLETED');

SELECT status, COUNT(*)
FROM orders
GROUP BY status;
```

- 이렇게 했을 때 결과는 다음과 같다.
    - COMPLETED 2
    - PENDING 1
    - NULL 1
- 즉, group by에서 NULL은 하나의 그룹으로 취급된다.

## Null 처리를 해보자

```sql
SELECT COALESCE(amount, 0) FROM orders;
SELECT IFNULL(amount, 0) FROM orders;
SELECT NULLIF(amount, 0) FROM orders;
```

- `COALESCE`: 첫 번째로 NULL이 아닌 값 반환
- `IFNULL` (MySQL): NULL을 다른 값으로 대체
- `NULLIF`: 두 값이 같으면 NULL 반환

## 그 외 주의할 점은 무엇일까?

- NULL과의 연산 결과는 무조권 `NULL` 이다.
- NULL과의 비교는 `IS NULL` 또는 `IS NOT NULL` 사용하라.
- WHERE 절에서 NULL 처리 시 명시적으로 `IS NULL` 조건 추가하라.