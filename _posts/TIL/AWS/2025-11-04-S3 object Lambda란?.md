---
layout: post
category: AWS
---

DEA 공부를 하다가 람다 관련 새로운 기술 질문들이 나와 정리해보았습니다.

![](https://velog.velcdn.com/images/leehjhjhj/post/94c32972-f274-49b3-97a5-e40599888082/image.png)

Amazon S3 Object Lambda는 S3에 저장된 객체를 가져올 때, AWS Lambda 함수를 통해 실시간으로 데이터를 가공하거나 변환하여 반환할 수 있도록 해주는 기능입니다.

GET, HEAD, LIST 요청에 대해 지원되며, 각 요청 결과를 Lambda 함수로 커스터마이즈할 수 있습니다.

즉, Lambda Edge처럼 S3 요청을 가로채서 Lambda가 변형 한 뒤, 이를 반환하는 기능입니다.

## 대표 활용 사례

- 데이터에서 민감(PII) 정보 제거 후 반환
- 특정 요청에 맞는 이미지 리사이즈, 워터마킹, 형식 변환(XML→JSON 등), 파일 압축·해제
- 사용자별 맞춤형 데이터 제공 및 동적 권한 부여

굳이 왜쓰지? 싶지만 S3에 저장된 원본 객체는 그대로 둘 수 있다는 장점도 있고, 프록시나 데이터 복제 등등 하지 않고 Lambda만 작성해주면 되기 때문에 장점이 있을 것 같습니다.