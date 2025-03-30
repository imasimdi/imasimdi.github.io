---
layout: post
category: data-engineering
subcategory: learning-sql
---

## 연산자 우선순위

SQL에서 가장 중요한 점은 **AND 연산자가 OR 연산자보다 우선순위가 높다**는 것입니다. 마치 수학에서 곱셈이 덧셈보다 먼저 계산되는 것과 유사한 성격입니다. 한번 예시를 살펴보겠습니다.

### 1. 우선순위 문제

다음 쿼리를 살펴봅시다

```sql
SELECT * FROM employees
WHERE department = 'IT' OR department = 'HR' AND salary > 50000;

```

이 쿼리는 어떤 직원들을 반환할까요? 많은 사람들이 "IT 또는 HR 부서에 있으면서 급여가 50,000 이상인 직원"을 반환한다고 생각하지만 실제로는 다음과 같이 작동합니다.

1. `department = 'HR' AND salary > 50000` 부분이 먼저 필터링됨 (AND 우선순위가 높음)
2. 그 다음 `department = 'IT' OR (위 결과)` 평가

따라서 실제 결과는 다음과 같은 직원들의 데이터를 가져오게 됩니다.

- 급여와 상관없이 IT 부서의 모든 직원
- HR 부서에서 급여가 50,000 이상인 직원

이를 의도한 대로 수정하려면 괄호를 사용해야 합니다.

```sql
SELECT * FROM employees
WHERE (department = 'IT' OR department = 'HR') AND salary > 50000;

```

### 2. 복잡한 조건 문제

```sql
SELECT * FROM products
WHERE category = 'Electronics' AND price < 500 OR category = 'Books' AND price < 30;

```

이 쿼리는 다음과 같은 데이터를 가져옵니다.

- `category = 'Electronics' AND price < 500` 평가
- `category = 'Books' AND price < 30` 평가
- 위 두 결과를 OR로 연결

앞선 예시로 `AND` 연산이 `OR` 보다 우선적으로 평가된다는 것을 알고 있지만, 다음과 같이 괄호를 통해서 명시적으로 작성해주면 조금 더 이해하기 쉬워집니다.

```sql
SELECT * FROM products
WHERE (category = 'Electronics' AND price < 500) OR (category = 'Books' AND price < 30);

```

## 성능 고려사항

OR 연산자는 쿼리 성능에 영향을 줄 수 있습니다. 데이터베이스 엔진은 OR로 연결된 조건들을 각각 평가하고 결과를 통합해야 하므로, 인덱스 활용이 어려워질 수 있습니다.

### IN 연산자 사용

많은 OR 조건이 있는 경우, 대신에 아래와 같이 IN 연산자를 고려할 수 있습니다.

```sql

SELECT * FROM customers
WHERE country = 'Korea' OR country = 'Japan' OR country = 'China';

SELECT * FROM customers
WHERE country IN ('Korea', 'Japan', 'China');

```

## NOT 연산자와의 상호작용

NOT 연산자가 추가되면 논리가 더 복잡해질 수 있습니다. 따라서 드모르간의 법칙을 잘 떠올려서 쿼리를 작성해줘야 합니다. 

- `NOT (A OR B)` = `NOT A AND NOT B`
- `NOT (A AND B)` = `NOT A OR NOT B`

### NOT과 AND/OR 조합

```sql
SELECT * FROM orders
WHERE NOT (status = 'Shipped' OR status = 'Delivered');

-- 위 쿼리는 다음과 동일함
SELECT * FROM orders
WHERE status != 'Shipped' AND status != 'Delivered';

```

## `AND`와 `IN` 연산자의 논리적 오류 방지하는 방법들

1. 복잡한 조건에서는 명확성을 위해 괄호를 사용하라
2. 여러 OR 조건 대신 IN 연산자를 사용할 수 있는지 검토하자
3. 매우 복잡한 조건은 더 작은 단위로 나누어 이해하기 쉽게 만들자
4. 드모간의 법칙을 잘 기억하고, 더 이해하기 쉬운 방향으로 쿼리를 작성하자