---
layout: post
category: data-engineering
subcategory: learning-sql
tags: [article]
---

![](https://velog.velcdn.com/images/leehjhjhj/post/be4dab74-27fc-491f-bf9f-3cb192d42ae0/image.png)
한번 생각해보는 시간을 가져봅시다. 현재 푸시 알림 서버를 따로 분리하고 있습니다. 특정 유저에게 푸시 알림을 보내기 위해서 `fcm_token` 이라는 테이블에서 한 유저가 가진 디바이스 토큰들을 모두 가져와야 합니다.

그런데 이유는 모르겠으나 같은 유저에 같은 디바이스 토큰이 여럿 존재하는 것을 발견했습니다. 굳이 같은 디바이스에 메시지를 보낼 이유가 없기 때문에, 중복된 토큰을 제거하고 가져오려고 마음을 먹습니다.

기존에는 어떻게 토큰을 뽑아오고 있었는지 레거시 코드를 열어보았습니다.

```sql
SELECT
    token
FROM
   (
	SELECT
		*
	FROM
		fcm_token
	WHERE
		user_id = :user_id
	ORDER BY crt_date desc
    ) AS fcm_token_device
GROUP BY token;
```

기존에는 한 유저의 토큰을 모두 가져온 뒤에 `GROUP BY` 를 통해서 중복을 제거해주고 있었습니다. 저는 뭔가 집계함수를 사용하지 않는데도 중복 제거만을 위해 `GROUP BY` 을 사용하는 것에 의구심이 들었습니다. 그래서 얼른 성능도 좋지 않을 것 같아 새롭게 쿼리를 짜봅니다.

```sql
SELECT DISTINCT
    token
FROM
    fcm_token
WHERE
    user_id = :user_id
ORDER BY crt_date desc;
```

이렇게  `DISTINCT`를 사용하면 간단하게 token 필드를 기준으로 중복을 제거한 결과 값을 얻을 수 있었습니다. 일단 쿼리는 엄청 간단하군요! 그렇다면 성능 차이는 어떤지 EXPLAIN을 통해서 살펴보았습니다. 참고로 저희 fcm_token 테이블의 총 row수는 중복 제외 **72,744**개 입니다.

### 실행 계획 비교

- `GROUP BY`를 사용한 기존 쿼리의 실행 계획

```sql
'1','SIMPLE','fcm_token',NULL,'ref','fcm_token_UN,fcm_token_user_id_IDX','fcm_token_UN','5','const','16','100.00','Using temporary'
```

- `DISTINCT`를 사용한 새로운 쿼리의 실행 계획

```sql
'1', 'SIMPLE', 'fcm_token', NULL, 'ref', 'fcm_token_UN,fcm_token_user_id_IDX', 'fcm_token_UN', '5', 'const', '16', '100.00', 'Using filesort'
```

맨 뒤에 있어서 잘 보이지는 않지만, `GROUP BY`는 `Using temporary`가, `DISTINCT`는 `Using filesort`가 발생하는 것을 알 수 있습니다.

`Using temporary` 는 임시 테이블을 생성해서 결과를 저장시키는 행동이고, 메모리나 디스크에 임시 테이블을 생성하므로 리소스 사용량이 많아집니다. `GROUP BY` 를 사용하면 그룹화를 위해 모든 데이터를 임시 테이블에 저장해야 하는 경우가 많기 때문에 `Using temporary` 가 발생할 확률이 커지게 됩니다. 하지만 `GROUP BY`가 항상 `Using temporary`를 발생시키는 것은 아닙니다. 인덱스 컬럼 순서와 GROUP BY 컬럼 순서가 일치하면 임시 테이블 없이 처리가 가능합니다.

### 실행 속도 비교

자, 이제 대망의 속도 차이입니다. `SHOW profiles` 를 통해서 기존 쿼리와 새로운 쿼리의 실행 속도를 비교해보았습니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/cf29be16-da85-4af3-9fa1-873921f28d97/image.png)


두 쿼리의 속도 차이는 있었지만, 0.0001초 차이라는 아주 근소한 차이 밖에 나지 않았습니다. 이마저도 `GROUP BY` 의 token에 인덱스가 걸려있었다면 `Using temporary` 도 발생하지 않아, 차이가 거의 없었을 것 같습니다.

### 결론

제가 내린 결론은 다음과 같습니다. 서로 비교했을 때 유의미한 속도 차이는 없으니, 가독성이 어떤 것이 더 좋은가와 집계함수의 사용 여부에 따라 결정하면 될 것 같습니다.