# Document API
- 생성한 인덱스를 관리할 목적
## 문서 파라미터
- ID: 지정하지 않으면 자동으로 ID 부여
- 버전관리 
    - 색인된 모든 문서는 버전을 가지고 있고, 재색인하면 _version이 올라감
    - 갱신, 삭제할 때 마다 올라감
- 오퍼레이션 타입: ID가 존재하면 update를 치게되고, 없으면 create를 하게된다. 그런데 그냥 같은 ID 색인시 update말고 색인 실패를 원하면 `op_type`을 이용한다.
- 타임아웃 설정: 색인 진행 중일 시, 기다리는 시간 설정
- 인덱스 매핑 정보 자동 생성:
    - action.auto_create_index: 기본 값 true, 인덱스 자동 설정
    - index.mapper.dynamic: 동적 매핑 사용 여부 설정
## Index API
- 문서를 특정 인덱스에 추가하는데 사용
- `_shards`: 몇 개의 샤드에 명령이 수행됐는가?
## Get API
- 특정 문서를 인덱스에서 조회할 때 사용
- GET `movie_dynamic/_doc/1`
- 일반적으로 모든 필드가 `_source`에 들어가는데, 크기가 크면 `_source_exclude` 옵션을 이용
## Delete API
- 하나 지울 때, DELETE`movie_dynamic/_doc/1`, 전체는 인덱스명을 입력
- delete by query api로 검색 후 삭제 가능
## Update API
- `ctx._source`를 사용해서 문서를 수정할 수 있다.
- Update는 일반적인 업데이트가 아니고, 재색인 작업을 거치는 것이라 `_source`가 활성화되어있어야 한다.
## Bulk API
- 다수의 문서를 색인하가니 식제 가능
- 그러나 트랜잭션이 조금 문제. 중간에 멈춰도 롤백이 되지 않는다.
## Reindex API
- 가장 일반적인 사용 법은 인덱스 -> 인덱스 간 복사
- 쿼리를 사용해서 특정 결과만 이동 가능
- 검색시 정렬 -> 리소스가 많이 들기에 색인시 정렬할 채로 하는 것이 좋음
- 이 API로 새로운 인덱스를 만들 때 새로운 정렬 방식으로 데이터 정렬 후 복사 가능