---
layout: post
category: data-engineering
subcategory: learning-sql
tags: [article]
---

![](https://velog.velcdn.com/images/leehjhjhj/post/cae565da-eea6-413c-bc7b-2d9d77f83048/image.png)

## 문제상황

최근 행사를 진행하는 부스의 상품에서 재고 이슈가 발생했습니다. 모든 상품 옵션을 20개씩으로 제한했는데, 판매 이후 확인해보니 총 판매 개수가 22개, 21개인 상품이 다수 발생했습니다.

이러한 일이 발생한 이유는 다음과 같습니다.

1. 상품 판매 도중에 관리자가 상품 수정하기 페이지 진입
2. 수정 페이지가 열려있는 동안 상품이 판매됨
3. 상품 수정 시 해당 옵션을 수정하지 않았어도 최초 로드된 옵션 수량이 다시 저장되어 재고가 잘못 증가

### 실제 사례

1. 2025-04-12 23:20 경 관리자가 상품 수정하기 페이지에 진입
2. 옵션 ID 29XXXX(첫 번째 옵션)은 페이지 진입 당시 재고(remain)가 12개, 즉 8개가 팔린 상태였음
3. 2025-04-12 23:35 경 주문 ID 14XXXXX이 제출되면서 해당 옵션이 1개 판매됨 (재고가 11개로 감소)
4. 2025-04-12 23:41 경 관리자가 열어두었던 상품 수정 폼을 저장(POST)함. 이때 첫 번째 옵션의 재고가 원래 값인 12개로 저장됨
5. 결과적으로 실제 재고는 11개여야 하는데, 시스템상 12개로 기록되어 총 수량(available)은 21개로 잘못 증가

## 고민

이런 동시성 문제를 해결하기 위해 여러 방법을 고려했습니다.

- Redis를 사용한 분산 락(lock) 구현
- 데이터베이스 수준의 비관적 락(Pessimistic Lock)
- UPDATE 쿼리의 WHERE 조건을 활용한 방법

하지만 이런 복잡한 기술 스택을 도입하기보다 SQL UPDATE 문의 특성을 활용해 간단하고 우아하게 해결하는 방법을 선택했습니다.

## 해결법

### 예시 타임라인

1. 수정하기 POST 를 할 때, 수정하기 페이지 처음에 GET 에서 불러온 재고 값들을 같이 보내줍니다.
	- 예시
    ```python
      {
          "product_option_id": 123,
          "name": "수정을 위한 상품",
          "price": 18000,
          "before_remain": 11,  # 폼 로드 시점의 재고
          "remain": 12          # 수정하려는 재고
      }
      ```
      - 예시에서는 `before_remain`라는 필드를 사용했습니다.
      
2. 수정하기 `POST` 요청을 하면 `before_remain`과 `remain` 두 값을 비교해서 업데이트를 해줄지 여부를 결정합니다.
    - 만약 `before_remain`과 `remain`(사용자가 넣은 재고)이 동일하다면 사용자가 재고를 수정하지 않은 것이기에 UPDATE를 하지 않습니다.
    - 만약 `before_remain`과 `remain`(사용자가 넣은 재고)이 동일하지 않다면 사용자가 직접 재고를 수정한 것입니다. 이 때는 현재 재고와 `before_remain`(수정하기 들어왔을 당시 재고)을 비교해서 `POST` 요청 시 처음과 재고가 달라졌는지 검사합니다.
3. 만약 두 값이 다르다면, 재고 수정이 불가능하기 때문에 모달을 띄우고 수정을 하지 못하게 합니다.
	- 예시
    	- 수정 중에 재고가 변동되었습니다. 변경된 재고를 확인해주세요. 모달 발생
    	- 프론트에서 다시 수정하기 `GET` API를 fetching해서 재고만 새롭게 refresh 해줍니다.

### 현재 재고와 수정하기 들어왔을 때 당시 재고를 비교한 법

본 글의 제목인 UPDATE 문으로 간단하게 해결하였습니다. 예시 코드는 다음과 같습니다.

```python
def update_option(data: OptionData, before_remain: int) -> bool:
    query = text("""
        UPDATE
            product_option
        SET
            available = :new_available,
            remain = :remain
        WHERE
            product_option_id = :product_option_id
            AND remain = :before_remain
    """).bindparams(**data, before_remain=before_remain)
    result = db.execute(query)
    if result.rowcount == 0:
        return False
    return True
```

UPDATE의 WHERE절은 트랜잭션 격리수준과 관계없이 항상 최신의 데이터를 기반으로 동작합니다.

WHERE 절의 `remain = :before_remain` 조건은 현재 시점에서 데이터베이스의 최신 재고 값이 폼 로드 시점의 재고 값과 일치할 때만 업데이트가 수행됩니다. 이는 비관적 락을 명시적으로 걸지 않아도 동시성 제어가 가능하게 합니다.

1. 만약 수정 중에 재고가 변동되지 않았다면 WHERE 조건이 만족되어 업데이트가 성공합니다.
2. 수정 중에 재고가 변동되었다면 WHERE 조건이 만족되지 않아 행이 선택되지 않고 업데이트가 실패합니다(rowcount == 0).

이는 낙관적 락의 원리와 유사하지만, 별도의 버전 관리 컬럼 없이 실제 업무 재고 값를 직접 비교하는 방식입니다.

```python
result: bool = self._prod_repo.update_option(option_data, product_option.before_remain)
if not result:
    uow.rollback()
    raise UpdateFailException
```

이 후 업데이트의 결과가 False, 즉 수정하기 들어왔을 때 당시 재고와 업데이트 당시 재고가 다르다면 외부에서 재고가 달라졌다는 의미기에 트랜잭션을 롤백시키고 예외를 발생시킵니다.

프론트는 400에러를 받으면 모달을 띄우고, 수정하기 GET 패칭을 한번 더해, 수정하기 페이지에서 재고만 refresh 해줍니다.

## 결론

분산 락이나 복잡한 동시성 제어 메커니즘 없이도 SQL UPDATE 문의 특성을 활용해 재고 관리의 동시성 문제를 간단하게 해결할 수 있었습니다. "가장 단순한 해결책이 종종 가장 좋은 해결책"이라는 엔지니어링 원칙을 잘 지킨 것 같습니다.
