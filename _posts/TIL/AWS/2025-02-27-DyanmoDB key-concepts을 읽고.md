---
layout: post
category: AWS
---

## 번역
[DyanmoDB key-concepts](https://docs.aws.amazon.com/whitepapers/latest/best-practices-for-migrating-from-rdbms-to-dynamodb/key-concepts.html)

이전 섹션에서 설명한 바와 같이, DynamoDB는 데이터를 아이템으로 구성된 테이블로 구성합니다. DynamoDB 테이블의 각 아이템은 임의의 속성 집합을 정의할 수 있지만, 테이블의 모든 아이템은 해당 아이템을 고유하게 식별하는 기본 키를 정의해야 합니다. 이 키는 파티션 키로 알려진 속성과 선택적으로 정렬 키라고 하는 속성을 포함해야 합니다. 다음 그림은 파티션 키와 정렬 키를 모두 정의하는 DynamoDB 테이블의 구조를 보여줍니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/ab796448-ef62-46e6-a8e2-e688f843510a/image.png)


만약 하나의 아이템이 단일 속성 값으로 고유하게 식별될 수 있다면, 이 속성이 파티션 키로 기능할 수 있습니다. 다른 경우에는 아이템이 두 개의 값으로 고유하게 식별될 수 있습니다. 이 경우 기본 키는 파티션 키와 정렬 키의 복합으로 정의됩니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/9b2433d0-0812-455f-9982-6f4d297da2d0/image.png)


미디어 파일과 이를 트랜스코딩하는 데 사용된 코덱을 연결하는 RDBMS 테이블은 DynamoDB에서 파티션 키와 정렬 키로 구성된 기본 키를 사용하여 단일 테이블로 모델링될 수 있습니다. DynamoDB 테이블에서 데이터가 비정규화되어 있음을 주목하세요. 이는 RDBMS에서 NoSQL 데이터베이스로 데이터를 마이그레이션할 때 일반적인 관행이며, 이 백서의 후반부에서 자세히 다룰 것입니다.

이상적인 파티션 키는 테이블의 아이템들에 걸쳐 균일하게 분포된 많은 수의 고유한 값을 포함할 것입니다. 사용자 ID는 테이블의 아이템들에 걸쳐 균일하게 분포되는 경향이 있는 속성의 좋은 예입니다. RDBMS에서 조회 값이나 열거형으로 모델링될 수 있는 속성들은 일반적으로 좋지 않은 파티션 키가 됩니다. 그 이유는 특정 값들이 다른 값들보다 훨씬 더 자주 발생할 수 있기 때문입니다.

이러한 개념들은 표 2에서 보여집니다. user_id의 수는 균일한 반면 status_code의 수는 그렇지 않다는 점에 주목하세요. status_code가 DynamoDB 테이블에서 파티션 키로 사용되는 경우, 가장 자주 발생하는 값이 단일 파티션에 저장되게 되며, 이는 대부분의 읽기와 쓰기가 해당 단일 파티션에 집중됨을 의미합니다. 이를 핫 파티션이라고 하며 성능에 부정적인 영향을 미칩니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/f2999600-5553-4422-b6e3-92d09f44fd70/image.png)


아이템은 기본 키를 사용하여 테이블에서 가져올 수 있습니다. 종종 파티션 키와 정렬 키가 아닌 다른 값들을 사용하여 아이템을 가져오는 것이 유용할 수 있습니다. DynamoDB는 로컬 및 글로벌 보조 인덱스를 통해 이러한 작업을 지원합니다. 로컬 보조 인덱스는 테이블에 정의된 것과 동일한 파티션 키를 사용하지만, 정렬 키로는 다른 속성을 사용합니다. 글로벌 보조 인덱스는 어떤 스칼라 속성이든 파티션 키나 정렬 키로 사용할 수 있습니다. 두 인덱스 유형의 중요한 차이점은 로컬 보조 인덱스는 테이블 생성 시에만 만들 수 있고 테이블이 삭제될 때까지 존재하는 반면, 글로벌 보조 인덱스는 언제든지 생성하고 삭제할 수 있다는 것입니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/1fca762b-1de7-4144-bab9-f01b8d52315e/image.png)


보조 인덱스를 추가하면 추가적인 저장 공간과 쓰기 용량이 소비되므로, 다른 데이터베이스와 마찬가지로 테이블에 정의하는 인덱스의 수를 제한하는 것이 중요합니다. 이를 위해서는 DynamoDB를 영구 저장소로 사용하는 모든 애플리케이션의 데이터 접근 요구사항을 이해해야 합니다. 또한, 글로벌 보조 인덱스는 속성 값들이 인덱스에 프로젝션되어야 합니다. 이는 인덱스가 생성될 때, 부모 테이블의 속성 중 일부를 선택하여 인덱스에 포함시켜야 한다는 것을 의미합니다. 글로벌 보조 인덱스를 사용하여 아이템을 조회할 때, 반환되는 아이템에는 인덱스에 프로젝션된 속성들만 포함됩니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/aa06b12d-0342-4a29-809c-21ee38065b0b/image.png)


원래의 파티션 키와 정렬 키 속성들은 자동으로 글로벌 보조 인덱스에 프로젝션됩니다. 글로벌 보조 인덱스에 대한 읽기는 항상 최종적 일관성을 가지는 반면, 로컬 보조 인덱스는 최종적 일관성 또는 강력한 일관성을 지원합니다. 

마지막으로, 로컬과 글로벌 보조 인덱스 모두 인덱스에 대한 읽기와 쓰기에 용량을 소비합니다. 이는 메인 테이블에 아이템이 삽입되거나 업데이트될 때, 보조 인덱스들이 인덱스를 업데이트하기 위해 용량을 소비한다는 것을 의미합니다. 유일한 예외는 인덱스의 기본 키의 일부인 속성들이 아이템에 존재하지 않아 아이템이 인덱스에 쓰여지지 않는 경우(희소 인덱스 참조)나 변경된 속성들이 인덱스에 프로젝션되지 않아 아이템 수정이 인덱스에 반영되지 않는 경우입니다.

DynamoDB는 각 테이블에 대해 용량 모드를 지정할 수 있게 합니다. 예측이 덜 가능한 워크로드에 적합한 온디맨드 용량 모드에서는, 서비스가 귀하를 위해 용량을 관리하며 사용한 만큼만 지불하면 됩니다. 프로비저닝된 용량 모드에서는 테이블의 읽기 및 쓰기 용량을 지정해야 하며 프로비저닝된 용량을 기준으로 비용을 지불합니다.

DynamoDB 테이블이나 인덱스에서 아이템을 읽거나 쓸 때마다, 읽기 또는 쓰기 작업을 수행하는 데 필요한 용량은 읽기 용량 단위(RCU) 또는 쓰기 용량 단위(WCU)로 표현됩니다. 하나의 RCU는 최대 4KB 크기의 아이템에 대해 초당 하나의 강력한 일관성 읽기, 또는 초당 두 개의 최종적 일관성 읽기를 나타냅니다. 트랜잭션 읽기 요청은 4KB까지의 아이템에 대해 초당 하나의 읽기를 수행하는 데 두 개의 RCU가 필요합니다. 하나의 WCU는 최대 1KB 크기의 아이템에 대해 초당 하나의 쓰기를 나타냅니다. 트랜잭션 쓰기 요청은 1KB까지의 아이템에 대해 초당 하나의 쓰기를 수행하는 데 두 개의 WCU가 필요합니다.

이는 총 크기가 8KB인 하나 이상의 아이템을 단일 강력한 일관성 읽기로 가져오는 경우 2개의 RCU를 소비한다는 것을 의미합니다. 8KB 크기의 아이템을 일반(비트랜잭션) 삽입하는 경우 8개의 WCU를 소비하게 됩니다.
프로비저닝된 용량 모드에서는 테이블이 지원하는 RCU와 WCU의 수를 선택합니다. 만약 애플리케이션이 초당 1000개의 4KB 아이템을 쓰기해야 한다면, 테이블의 프로비저닝된 쓰기 용량은 최소 4000 WCU가 되어야 합니다. 테이블에 충분하지 않은 읽기 또는 쓰기 용량이 프로비저닝되었을 때, DynamoDB 서비스는 읽기와 쓰기 작업을 제한합니다. 이는 성능 저하를 초래할 수 있으며 경우에 따라 클라이언트 애플리케이션에서 제한 예외가 발생할 수 있습니다. 이러한 이유로, 테이블을 설계할 때 애플리케이션의 I/O 요구사항을 이해하는 것이 중요합니다.

하지만 읽기와 쓰기 용량은 기존 테이블에서 동적으로 변경될 수 있습니다. 만약 애플리케이션이 갑자기 사용량이 급증하여 제한이 발생하면, 프로비저닝된 용량을 늘려 새로운 워크로드를 처리할 수 있습니다. 마찬가지로, 어떤 이유로 부하가 감소하면 프로비저닝된 용량을 줄일 수 있습니다. 테이블의 읽기 또는 쓰기 용량의 이러한 동적 변경은 간단한 API 호출을 통해, 또는 DynamoDB Auto Scaling을 통해 자동으로 달성할 수 있습니다. 또한, 24시간마다 한 번씩 테이블의 용량 모드를 변경할 수 있습니다.

테이블의 I/O 특성을 동적으로 변경할 수 있는 이 기능은 DynamoDB와 전통적인 RDBMS 간의 주요 차별점입니다. RDBMS에서는 I/O 처리량이 데이터베이스 엔진이 실행되는 기본 하드웨어에 따라 고정됩니다. 이는 많은 경우에 DynamoDB가 전통적인 RDBMS보다 훨씬 더 비용 효율적일 수 있다는 것을 의미합니다. RDBMS는 일반적으로 최대 사용량에 맞춰 프로비저닝되어 대부분의 시간 동안 충분히 활용되지 않은 상태로 유지되기 때문입니다.

## 읽은 후, 얻어가는 것 들

- DynamoDB도 핫 파티션이 존재한다. 카디널리티 높은 것을 파티션 키로 지정하라
- 로컬 보조 인덱스는 테이블의 정의된 것과 동일한 파티션 키를 사용하지만 정렬 키는 다른 것을 사용할 수 있다.
- 로컬 보조 인덱스는 테이블 생성시에만 만들 수 있다. 글로벌 보조 인덱스는 언제든지 생성 / 삭제 가능
- 관계형 데이터 베이스와 같이 인덱스 설정시 추가적인 저장 공간과 쓰기 용량을 소비한다.
- 이 때, 인덱스를 만들 때 원본 테이블의 속성 중 일부를 선택한다. 그리고 보조 인덱스로 아이템을 가져올 때는, 선택한 속성만 조회된다.
    - 트레이드 오프가 있을 것 같다. 많은 속성을 포함시키면 인덱스만 불러와서 데이터를 가져올 수 있겠지만, 쓰기 + 저장공간을 소비한다.
- 하나의 읽기 용량 단위는 최대 4KB 크기의 아이템을 초당 하나의 강력한 일관성 읽기, 혹은 초당 두 개의 최종적 일관성 읽기를 할 수 있다.
- 하나의 쓰기 용량 단위는 최대 1KB 크기의 아이템에 대해 초당 하나의 쓰기를 한다.
- 이 RCU와 WCU를 테이블을 만들기 전에 지정하여 프로비저닝을 했을 때, 이 용량을 초과하면 테이블은 쓰로틀링이 걸린다.
- 근데 이것은 동적으로 늘리거나 줄일 수 있다. 심지어 DynamoDB 오토 스케일링을 사용하면 쉽게 자동 조정이 된다.
- 그래서 전통적인 RDBMS보다 비용 효율적일 수도 있다.

### 그래서 LSI는 왜 쓰는가?

- 강력한 일관성 읽기를 사용해야하는 데이터 일관성이 중요한 데이터들을 다룰 때, 추가적인 쿼리 방법을 위해 쓰일 것 같다.
    - 파티션키는 동일하지만, 정렬키는 다른 것을 사용할 수 있기 때문
- 하지만 테이블 생성 이후에 추가가 되지 않기 떼문에, 신중한 설계가 필요하다.