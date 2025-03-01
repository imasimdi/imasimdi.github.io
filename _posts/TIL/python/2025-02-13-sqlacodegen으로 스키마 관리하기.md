---
layout: post
category: python
category-show: 아티클
tags: [article]
---

## 문제점

저희 회사에서 현재 행사 시스템을 전면 개편하고 있습니다.

레거시를 활용한 개편인 만큼, 기존 테이블을 수정하거나 새로운 테이블을 생성하는 경우가 잦았습니다.
어느정도 개발이 되고 나니깐 문제점이 발생했습니다. 문서에 기록은 하였지만 어느 컬럼이 생기고, 어느 인덱스를 바꾸거나 추가했는지 정확하게 추적이 되지 않았습니다.

변경점이 많은 대형 프로젝트인 만큼, 변경된 스키마 추적이 한번 놓치기 시작하니깐 겉잡을 수 없이 되어버렸습니다. 더군다나 저희는 현재 스키마 관리를 하는 시스템이 갖춰있지 않습니다. python 스키마 관리 툴인`alembic` 도 사용하지 않고, 스키마 변경에 대한 문서화도 되어있지 않습니다.

또한 sqlalcemy의 `Base` 객체들을 선언하지 않고 쿼리만으로 레포지토리 로직을 짜기 때문에, 코드 버전 관리 차원에서도 테이블 변경을 추적하기 힘들었습니다.
이제와서 alembic을 도입하자니 위험성이 너무 크고, 지금처럼 매번 프로젝트마다 변경점을 기록한 뒤 수동으로 DDL을 작성하여 싱크를 맞추는 작업은 한계가 있었습니다.
많은 고민의 결과, 최근 객체 매핑을 작업을 하던 중 아이디어가 떠올랐습니다. 바로 SQLAlchemy로 객체 매핑을 한 뒤에 전/후를 비교하는 방법입니다.

## 원리와 기대점

1. 개발 DB와 운영 DB의 스키마를 `sqlacodegen`을 통해서 modes.py를 추출합니다.
2. 두 models.py를 비교하여 차이점을 기록하는 markdown을 생성합니다. (sh. run.sh)
3. 차이점을 바탕으로 DDL도 생성시켜줍니다. (sh make_ddl.sh)

기대점은 다음과 같습니다.

- 현재 운영 <-> 개발 DB 간의 스키마 차이를 쉽게 파악이 가능합니다.
- 스키마 관리를 해당 툴과 함께 SQLAlchemy의 Base객체를 사용하여 효율적인 관리가 가능합니다.
- 개발 후 운영 배포시 누락할 수 있는 컬럼, 인덱스를 파악할 수 있습니다.

이러한 아이디어를 바탕으로 클로드의 도움을 받아 코드를 짜보았습니다. 

https://github.com/Crafta-Corp/schema-migration-tool

README에도 나와있듯, `make_base.sh` 에 DB URL을 추가하여 준비를 해주고, 
`sh run.sh` 를 실행하면 다음과 같은 일이 벌어집니다.

두 DB의 스키마를 읽어서 `Base` 객체로 변환해줍니다. 변환된 객체들을 sources의 `prod_models.py` 과 `dev_models.py` 로 만들어 줍니다. 만들면 이러한 모습의 파일이 생성됩니다.

- `dev_models.py`예
    
    ```python
    class Agree(Base):
        __tablename__ = 'agree'
    
        agree_id = Column(Integer, primary_key=True)
        user_id = Column(Integer, nullable=False, server_default=text("'0'"))
        crt_date = Column(DateTime)
        
     ...
    ```
    

이후에 `compare_models_to_file.py` 가 실행되어 두 스키마의 모델들을 비교해주고, 레포트를 작성해줍니다. 예시는 다음과 같습니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/d95ebdfd-4389-472f-8831-4fa8b347a920/image.png)


`sh make_ddl.sh` 를 사용해서 앞서 만들어진 차이점을 바탕으로 DDL도 만들어줍니다.

이후에 레포트를 보면서`schema_changes_prod_to_dev.sql` 를 검토 한 뒤, 마이그레이션을 진행해주면 됩니다.

## 후기

초기 실행 결과, 약 70여 개의 컬럼과 인덱스 불일치가 발견되었습니다. 해당 툴을 활용하여 동기화 작업을 진행하였고 현재는 프로젝트 관련 테이블의 변경점만 남겨둔 상태입니다.
향후 객체 매핑을 통해 코드 수준에서 테이블 변경점을 관리하고, 해당 툴을 활용하여 동기화를 유지할 예정입니다.
입사 이후 개발 환경과 운영 데이터베이스 간 동기화 문제를 해결하고자 했었는데, 이렇게 아이디어를 바탕으로 손쉽게 해결할 수 있었어서 매우 만족스럽습니다. 물론 다양한 해결책이 존재할 수 있으나, 회사의 특수한 환경과 요구사항에 맞게 해당 문제를 해결했다는 점, 그리고 팀원들에게 도움을 주었다는 것이 성취감을 느끼게 해주었습니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/85ac7998-ea88-4d97-bfe2-49da40bdaa7b/image.png)