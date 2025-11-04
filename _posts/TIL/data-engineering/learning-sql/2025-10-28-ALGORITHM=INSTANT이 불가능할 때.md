---
layout: post
category: data-engineering
subcategory: learning-sql
---

한 테이블에서 컬럼을 추가하려고 `ALGORITHM=INSTANT` 를 사용해봤는데 

다음과 같은 에러가 발생했습니다.

> [1846] ALGORITHM=INSTANT is not supported. Reason: InnoDB presently supports one FULLTEXT index creation at a time. Try ALGORITHM=COPY/INPLACE.
> 

찾아보니 이유가 다음과 같았습니다.

우선 ALGORITHM=INSTANT은 무엇인지 짧게 알아보고 가겠습니다.

### ALGORITHM=INSTANT 란?

`ALGORITHM=INSTANT`의 원리는 테이블의 실제 데이터가 아닌, 메타데이터만 수정합니다.

즉, 새 컬럼을 추가할 때 기존 데이터 파일을 건드리지 않고, MySQL의 메타데이터에 새로운 컬럼 정보를 등록하는 방식입니다.

이 때문에 테이블 크기와 무관하게 작업이 거의 즉시 완료되고, 데이터 복사나 재구성이 발생하지 않아 부하가 크게 줄어듭니다.

### 지원이 안되는 상황

- 테이블에 **FULLTEXT 인덱스**가 이미 하나 이상 존재하거나, 인덱스 작업이 동시에 진행 중인 경우
- MySQL 8.0.29 미만, Aurora MySQL, MariaDB 환경 등에서 지원 미비
- 메타데이터만으로 처리가 불가능한 구조 (예: NOT NULL에 기본값 없음, 컬럼 중간/앞에 위치 변형 등)