---
layout: post
category: python
---

## Session.refresh() 주요 용도

세션 내 객체를 데이터베이스에서 최신 상태로 다시 불러옵니다. 주로 다른 세션/트랜잭션에서 변경된 데이터를 반영할 때 사용하며, `expire()`와 달리 즉시 SELECT 쿼리를 실행합니다.

## flush()와 refresh() 관계

- **같은 세션/트랜잭션**: `flush()`만으로 충분. 변경사항이 메모리에 반영되고 autoflush로 쿼리 시 자동 처리됩니다.
- **다른 세션**: `flush()` 후에도 변경이 보이지 않습니다. `commit()` 후 `refresh()`가 필요합니다.

## Bulk Update 특성

| 상황 | 세션 추적 | 객체 상태 | refresh 필요 |
| --- | --- | --- | --- |
| `obj.status = value` | O | 메모리 즉시 반영 | ❌ 불필요 |
| `query().filter(...).update({})` | X (직접 SQL) | stale 상태 | ✅ 필수 |

## 실전 워크플로우 예시

```python
1. 같은 세션: flush만
obj.name = 'updated'
session.flush()  # DB 버퍼 전송, 객체는 이미 최신

# 2. 다른 세션 동기화
session2.refresh(obj)  # commit된 DB 값으로 갱신

# 3. Bulk update 후
session.query(Model).filter(...).update({"status": "done"})
session.refresh(existing_obj)  # 기존 객체 동기화
```

근데 refresh는 select를 한번 강제로 불러오기 때문에.. 업데이트 칠 때는 flush만 쓰는게 좋지 않나 싶어요.

그리고, refresh가 필요 없는 필드는 그냥 flush 만으로도 충분합니다. 단지, 예전 데이터를 보고 있을 뿐.. 근데 특정 필드만 업데이트를 할 수도 있어요.`expire(['specific_attr'])`를 사용하면 됩니다.

근데 이러면 너무 관리가 힘들어질 것 같지 않나요? 업데이트 이후에 싱싱한 객체를 보려면 그냥 refresh 한번 치는 게 좋아보입니다.

## 그럼 merge는요?

SQLAlchemy의 `session.merge()`는 detached(세션 밖) 객체를 현재 세션에 병합하며, refresh와 달리 DB에서 최신 데이터를 우선 로드한 후 변경을 적용합니다

```python
detached_user = User(id=1, name='detached_change')  # 세션 밖 객체
merged = session.merge(detached_user)  # DB 최신 로드 후 병합
merged.name  # DB 값 + 'detached_change'
session.flush()  # DB 반영
```

그래서 read session으로 불러와서 update SQL을 부르고 merge + refresh를 하면?

select이 2번 나가요! 아주 비효율적이기 때문에, 애초에 Write Session으로 객체를 불러오도록 하거나, 객체의 setter로 직접 변경하는 것이(트레이드 오프는 있지만) 성능에는 좋아보입니다.