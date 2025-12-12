---
layout: post
category: data-engineering
subcategory: learning-sql
tags: [Postgresql]
---

Mysql의 TIMESTAMP와 DATETIME 처럼, 이 둘의 차이는 어떤 것이 있을까? 생각하여 간단하게 정리해보았습니다.

## timestamp (without time zone)

- 시간대 정보 없이 날짜와 시간만 저장
- 입력한 값 그대로 저장되고, 조회 시에도 그대로 반환
- 서버나 클라이언트의 시간대 설정과 무관

```sql
INSERT INTO events (created_at) VALUES ('2025-11-28 14:30:00');
```

- 저장: 2025-11-28 14:30:00
- 조회: 2025-11-28 14:30:00 (항상 같음)
## timestamptz (with time zone)

- 시간대 정보를 포함하여 저장 (내부적으로는 UTC로 변환)
- 조회 시 세션의 시간대(SET timezone)에 맞춰 자동 변환
- 글로벌 서비스에 적합
- 한국 시간대에서 입력 (UTC+9)

```sql
SET timezone = 'Asia/Seoul';
INSERT INTO events (created_at) VALUES ('2025-11-28 14:30:00+09');
```

- 저장: 2025-11-28 05:30:00 UTC (내부적으로 UTC 변환)
- 한국에서 조회: 2025-11-28 14:30:00+09
- 미국에서 조회: 2025-11-27 21:30:00-08 (PST 기준)