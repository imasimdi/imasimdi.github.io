---
layout: post
category: data-engineering
subcategory: learning-sql
---

만약 인증을 한 사용자만 쿼리해오고 싶으면 어떻게 할까요?

가장 간단한 방법은 JOIN을 하는 것이 생각이 납니다.

```sql
SELECT
	*
FROM
	users u JOIN verify v ON u.id = v.user_id
```

인증 정보와 같이 1:1 상황이라면 간단하게 INNER JOIN을 사용하면 되겠지만 만약 글이 여러 개 있는 사용자만 불러오고 싶을 때도 JOIN을 써야 할까요?

```sql
SELECT
	*
FROM
	users u JOIN articles a ON u.id = a.user_id
GROUP BY
	u.id
```

이렇게하면 글이 있는 user들을 불러올 수 있지만, 조인을 하는 과정에서 불필요한 JOIN이 정말 많이 일어날 것 같습니다. 실제로 해당 쿼리의 실행 속도는 다음과 같습니다.

- `(actual time=1116.316..1116.344 rows=63 loops=1)`

해당 쿼리는 EXISTS 서브 쿼리로 성능 향상을 시킬 수 있습니다.

```sql
SELECT
	*
FROM
	users u
WHERE
    EXISTS (SELECT 1 FROM article a WHERE u.id = a.user_id)
GROUP BY
	u.id
```

해당 쿼리는 다음과 같이 동작합니다.

- 메인 쿼리에서 각 행을 하나씩 처리합니다.
- 각 행에 대해 EXISTS 내부의 서브쿼리를 실행합니다.
- 서브쿼리가 하나 이상의 결과를 반환하면 TRUE, 아무 결과도 반환하지 않으면 FALSE입니다.
- TRUE인 경우만 메인 쿼리의 결과에 포함됩니다.

해당 쿼리의 실행 속도는 다음과 같았습니다.

- `(actual time=164.810..177.526 rows=63 loops=1)`

EXISTS 서브쿼리가 JOIN보다 약 6배 빠른 결과를 보여주었는데, 왜 이러한 차이가 발생하는 걸까요?

## EXISTS 서브 쿼리가 더 빠른 이유

### 데이터 처리량 감소

JOIN은 모든 매칭 레코드를 중간 결과로 생성합니다. 사용자 한 명당 여러 글이 있다면, 중간 결과는 매우 커질 수 있습니다.

- 사용자 100명, 각 사용자당 평균 50개 글
- JOIN: 최대 5,000개 행의 중간 결과 생성
- EXISTS: 사용자 100명에 대해서만 처리

### 조기 중단(Short-circuit) 효과

위에서 쿼리 동작에서 봤듯이, EXISTS는 첫 번째 일치하는 레코드를 찾으면 즉시 TRUE를 반환하고 더 이상 검색하지 않습니다. 사용자가 글을 많이 작성했더라도 하나만 찾으면 됩니다.

## 마무리

1:N 관계에서 N의 수가 많을수록 EXISTS의 성능 이점이 더 커집니다. 불필요하게 JOIN을 했던 쿼리들이 있으면 최적화를 해보는 건 어떨까요?