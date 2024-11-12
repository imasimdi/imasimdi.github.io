---
layout: post
category: AWS
---

# 도입 배경

- 여전히 스파이크 트래픽을 잘 핸들링하려고 고민 중에 있었는데, 주로 특정 시간에 상품 쪽에서 트래픽이 몰리는 경향이 있어서 상품 상세 정보 만이라도 캐시를 적용하고자 하였다.
- 또한 현재 조회수 update를 `get` 한번당 update 쿼리가 나갔는데, 캐싱을 통해서 x의 배수일 때만 실제 DB Update가 일어나게 하여 쿼리 수를 줄이고 싶었다.
- 캐시 정책을 살펴보고 캐시를 적용하는데 있어서 어떤 방식이 좋은지 결정을 하였다.

# 캐시 정책

## Cache-Aside (Lazy Loading)

### 로직

- 데이터 요청이 들어오면 먼저 캐시를 확인
- 캐시에 없으면 DB에서 조회 후 캐시에 저장
- 캐시에 있으면 바로 반환
- 데이터 업데이트 시 캐시를 삭제하거나 업데이트

### 장단점

- 가장 구현이 단순하고 직관적이다.
- 또한 캐시에 장애가 생겨도 애플리케이션에 지장이 없다.
- 동시에 같은 데이터를 요청하면 캐시에 데이터가 없을 시 여러 번 DB 조회가 발생한다.

### 쓰는 곳

- 읽기가 많고 쓰기가 적은 곳
- 실시간성이 딱히 중요하지 않은 곳 → 데이터 변화가 많이 일어나지 않는 곳
    - 제품 목록이나 사용자 프로필 등

## Write Through

### 로직

- 데이터 쓰기 시 DB와 캐시를 동시에 업데이트
- 읽기는 항상 캐시에서 먼저 확인한 후 없으면 DB에서 조회
- 마지막으로 캐시 업데이트

### 장단점

- 데이터 일관성이 매우 높다. → 캐시와 DB 동기화 보장
- 자연스럽게 캐시 미스 발생 빈도도 낮아진다.
- 하지만 그만큼 쓰기 지연이 발생. write시 캐시와 DB를 동시에 업데이트 해야하기 때문
- 자주 업데이트되는 데이터는 오버헤드가 발생할 수도 있다.

### 쓰는 곳

- 데이터 일관성이 매우 중요한 경우
- 읽기와 쓰기 비율이 비슷할 경우
- 금융이나 재고 등 정확성이 제일 중요한 데이터에 사용

## Write Behind

### 로직

- 데이터 쓰기시 캐시만 즉시 업데이트
- DB업데이트는 배치 처리 등으로 비동기적으로 처리
- 읽기는 캐시에서 먼저 확인

### 장단점

- DB와 캐시 모두 동시에 업데이트하는 Write Through와 달리 캐시만 즉이 업데이트 하기 때문에 쓰기 성능이 매우 좋다.
- 따라서 DB 부하도 조금 감소하는 편
- 그러나 캐시서버가 장애가 발생하면 데이터 유실이 된다.
- 자연스럽게 데이터 정합성 보장이 어렵다.

### 쓰는 곳

- 대량의 쓰기가 발생하는 곳
- 로그 데이터나 통계 데이터 등등 실시간 분석에 쓰일 때

## Time Based

### 로직

- 캐시된 데이터에 TTL을 생성
- TTL 완료시 자동으로 캐시에서 제거
- 다음 요청시 새로운 데이터로 캐시 갱신

### 장단점

- 가장 구현이 간단할 것 같다.
- 오래된 캐시가 자동으로 제거되기 때문에 효율적
- 그러나 TTL이 만료되기 전까지는 캐시가 사라지기 않기 때문에 정합성 문제가 발생한다.
- 반대로 유효한 데이터도 삭제될 수도 있다.
- TTL이 한 시점에 몰려있다면 갱신 시점에 부하가 올 수 있다.
- 적절한 TTL을 찾는데 어려울 수 있다.

### 쓰는 곳

- 어느정도 데이터 불일치를 트레이드 오프로 볼 수 있는 곳
    - 세션 데이터, 토큰
- 실시간성이 중요하지 않는 통계 데이터 등

## 실제로는 어떻게 사용하는 것이 좋을까?

- 하나의 방식을 사용하는 것 보다는 여러 방식을 혼합하는게 좋아보인다.
- 기본적으로는 Cache aside를 사용하나, 중요한 데이터는 Write Through, 로그는 Write behind를 사용하는 식
- 공지사항이나 메인 배너 등 사라질 시점이 정확하다면 Time Based를 사용하면 좋을 것 같다.
- 상품 정보 조회(재고 or 옵션 제외)는 Cache Aside를 적용하기로 결정하였다. 

# 도입

## Redis Client

- 우선 Elastic Cache를 생성해주고, Redis Client 객체를 생성하였다.

```python
class RedisClient:
    def __init__(self):
        self.redis_client = Redis(
            host=settings.redis_host,  
            port=settings.redis_port,
            db=settings.redis_db,
            decode_responses=True,
            encoding='utf-8',
            socket_timeout=0.1,
            socket_connect_timeout=0.1 
        )
        self.default_ttl = 3600

    def get_connection(self) -> Redis:
        try:
            self.redis_client.ping()
            return self.redis_client
        except Exception as e:
            print(e)
```

- `socket_timeout`과 `socket_connect_timeout`을 두어서 캐시 서버가 죽었을 때, timeout 까지 시간이 너무 길지 않게 제한을 두었다.
- 그리고 `get`, `set`, `delete`, `mget`, `mset`, `bulk_delete` 메서드를 생성하여 재사용성을 높였다.

```python
def mget(self, keys: List[str]) -> List[Optional[dict]]:
       try:
           with self.redis_client.pipeline() as pipe:
               for key in keys:
                   pipe.get(key)
               results = pipe.execute()    
               return [
                   RedisClient.deserialize_value(json.loads(data)) 
                   if data else None 
                   for data in results
               ]
       except Exception as e:
           print(f"mget error: {e}")
           return [None] * len(keys)
```
- 예시로 `mget`을 가져왔는데, `pipe` 메서드를 통해서 네트워크 통신 횟수를 줄였다. 여러 키를 한번에 조회하는 일이 많아서 해당 메서드를 사용하기로 결정했다.
- `set`, `get`시 datetime을 string 객체로 만들어주기 위해 serializer, deserializer를 만들었다.

```python
@staticmethod
def serialize_datetime(obj: Any) -> Any:
    if isinstance(obj, (datetime, date)):
        result = obj.isoformat()
        return result
    return obj

@staticmethod
def serialize_dict(data: Dict) -> Dict:
    return {
        key: RedisClient.serialize_datetime(value)
        for key, value in data.items()
        }

@staticmethod
def deserialize_datetime(obj: Any) -> Any:
    if isinstance(obj, str):
        try:
            if 'T' in obj:
                return datetime.fromisoformat(obj)
            return date.fromisoformat(obj)
        except ValueError:
            return obj
    return obj

@staticmethod
def deserialize_dict(data: Dict) -> Dict:
    return {
        key: RedisClient.deserialize_datetime(value)
        for key, value in data.items()
    }
```

# 조회수 캐시

- hit을 시작으로 하는 캐시를 생성하고, 실제 조회 API가 호출되었을 때 hit 캐시를 불러와 `+1`를 해주고 캐시 업데이트를 해준다.
- 그리고 실제 DB Update는 x의 배수에만 업데이트 쿼리가 발생하게 설정하였다.
- 실제 DB 업데이트는 x의 배수로 되지만, 화면에 보이는 조회수는 조회수 키로 조회한 데이터로 보여지기 때문에 유저들이 느끼는 조회수 카운팅은 전과 다를 것이 없다.

```python
if data.hit is not None:
    hit = data.hit + 1
    if hit % x == 0:
        hit_thread = threading.Thread(target=updateHit, args=[str(prod_id), hit])
        hit_thread.start()
```

# 고려 및 고민했던 점

## 캐시 삭제 메서드

- Cache Aside 정책을 선택하였으니, 데이터가 변경되었을 때 캐시 데이터를 삭제해주었어야 했다.
- 데이터 변경 메서드에서 키를 만들고 `delete()`를 직접 호출하는 것 보다 `delete_~~_info()` 와 같이 직접 메서드를 만들어 변경에 용이하게 만들었다.

```python
def delete_prod_info(self, prod_id: int):
    redis_key = "prod:" + str(prod_id)
    self.delete(redis_key)

def change_prod_data(data):
    ...
    redis_client.delete_prod_info(prod_id=data.prod_id)
    ...
```

## Cache Read Fail Fallback

- 캐시 서버가 부하로 인해 죽게되었을 때를 가정하여, 캐시 서버가 없어도 정상적으로 API가 동작하도록 만들어야 했다.
- 방법은 간단한데 캐시 읽기 실패 시 데이터베이스로 대체하는 것이다.
- 만약에 `mget` 을 실행할 때 Redis server에 응답이 없거나 에러가 발생할 때 예외처리에서 return을 `[None] * len(keys)` 로 반환한다.
- 만약 redis에서 받아온 데이터가 `None`일 때 데이터베이스에서 가져오는 형식으로 진행하여 Cache Read Fail Fallback를 보장했다.
- 또한 앞서 redis client를 생성할 때 `socket_timeout`과 `socket_connect_timeout`를 명시하여 너무 오래 handshaking을 지속하는 것을 방지하였다.

# 결과

- 결과는 다음과 같다.

## 상품 상세 페이지

- 캐시 적중시 DB QUERY **13 -> 0** 개로 축소
- API CALL 속도 평균 25.9% 상승 (캐싱 적용으로 평균적으로 1/4 가량의 시간 절약)
    - **캐싱 전 성능**

        - 평균 실행 시간: 221.19ms
        - 최소 실행 시간: 208.92ms
        - 최대 실행 시간: 242.76ms
        - 표준편차: 12.15ms

    - **캐싱 후 성능**

        - 평균 실행 시간: 163.92ms
        - 최소 실행 시간: 146.39ms
        - 최대 실행 시간: 175.59ms
        - 표준편차: 9.85ms

## 조회수

- **쿼리 수 비교**
    - 캐싱 전: 100번의 조회 = 100번의 DB update 쿼리
    - 캐싱 후: 16번의 DB update 쿼리

# 후기

- 기존에 있던 API에 캐시를 도입하다 보니깐, 바인딩된 데이터를 수정 / 삭제시키는 메서드들이 너무 많았다.
- 그래서 역추적을 통해서 해당 데이털가 어디서 변경되고, 어디서 수정되는지 파악을하고 삭제 메서드를 도입하는 것이 가장 힘들었다.
- 하지만 트래픽이 몰릴 때 상당한 쿼리 수의 DB call을 줄일 수 있었고, API call 속도 단축도 시킬 수 있어서 좋은 개선이었다.