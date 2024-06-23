---
layout: post
category: AWS
---
- 민감한 정보이지만 어쩔 수 없이 퍼블릭을 열고, 객체 조회를 해야할 필요가 발생한다.
- 이러한 상황에서는 **버킷 정책**을 통해서 보안을 강구할 수 있다.
- 거기서도 `Referer` 라는 조건은 요청이 특정 출처에서 왔는지 확인한다.

```
{
    "Version": "2012-10-17",
    "Id": "Policy1577077078140",
    "Statement":
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::naver/*",
            "Condition": {
                "StringLike": {
                    "aws:Referer": [
                        "https://www.naver.com/admin/*",
                    ]
                }
            }
        }
}
```

- Referer 헤더는 HTTP 요청을 보낸 URL을 나타내며, 해당 Referer의 요청만 GetObject를 허용한다.

# CloudFront Cors
- S3에서 CORS 정책을 허용해도, 이미지가 보이지 않는 현상이 발생한다. S3 객체 url을 열면 이미지가 보이지만 클라우드 프론트 url을 열면 보이지 않는다.
- CloudFront는 클라이언트로부터 받은 쿠키, 헤더, 쿼리 문자열을 원본으로 전달하지 않는다.
- 즉 CloudFront가 S3 버킷에서 반환된 CORS 헤더를 그대로 전달하지 않아서 Cors에러가 발생할 수 있다.

## 해결 방법
- 배포에 들어가서 동작 편집을 누른다.
- 캐시 키 및 원본 요청에서 새로운 캐시 정책을 만들어준다.
- 캐시 키 설정에서 `[헤더 - 다음 헤더 포함]`에서 `Origin`을 넣어준다.
    - 캐시 키 설정이란?

        - 캐시 키 설정은 CloudFront에서 캐시 키에 포함하는 최종 사용자 요청의 값을 지정합니다.
        이러한 값에는 HTTP 헤더, URL 쿼리 문자열 및 쿠키가 포함될 수 있습니다.

        - 캐시 키 설정은 뷰어 요청이 캐시 적중을 일으키는지 여부를 확인하는 데 도움이 되며,
        이를 통해 캐시 적중률을 높일 수 있습니다.
        캐시 키 설정에 더 적은 값을 포함하면 캐시 적중률을 높일 수 있습니다.

        - 캐시 키에 포함하는 값은 CloudFront가 오리진으로 보내는 요청(오리진 요청이라고 함)에 자동으로 포함됩니다.
        오리진 요청 정책을 사용하여 캐시 키에 영향을 주지 않고 오리진 요청을 제어할 수 있습니다.

- 만든 캐시 정책을 적용하고, 원본 요청 정책에 `CORS-S3Origin`을 선택한다. 해당 원본 요청 정책에는 다음과 같은 헤더가 포함된다.
    - origin
    - access-control-request-headers
    - access-control-request-method
- 기본 `CachingOptimized`은 성능 최적화를 위한 최소한의 헤더만 포함하도록 하는 정책이다.
- 따라서 Origin 헤더를 포함하도록 한 커스텀 정책은 CORS 관련 헤더를 클라이언트 요청에서 원본으로 전달해준다.
- 참고로 Origin 헤더에는 다음과 같은 정보가 들어간다.
    - 프로토콜: 요청이 사용한 프로토콜 (예: http, https)
    - 호스트: 요청을 보낸 도메인 (예: example.com)
    - 포트 번호: 요청을 보낸 포트 번호 (생략될 수 있음, 기본 포트인 경우 생략됨) (예: 80 for HTTP, 443 for HTTPS)
