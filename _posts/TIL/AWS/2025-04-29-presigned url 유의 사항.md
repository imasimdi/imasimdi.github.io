---
layout: post
category: AWS
---


최근 회사에서 프론트엔드의 이미지 업로드를 위한 S3키가 유출되는 사고가 발생했습니다.
원인을 찾기 위해 다양한 방법으로 추적했지만 계속해서 키가 유출되는 상황이 반복되었고, 결국 업로드 방식 자체를 변경하기로 결정했습니다.

프론트엔드에서 직접 AWS SDK를 불러와 객체를 업로드하는 방식에서 백엔드에서 Presigned URL을 생성하여 프론트엔드에 제공하는 방식으로 변경하였습니다. 이렇게 되면 프론트엔드는 AWS 인증 정보 없이 해당 URL로 직접 업로드를 할 수 있게되어 프론트엔드에 S3 키를 두지 않아도 되는 장점이 있습니다.

보안은 개선되었지만, 업로드된 객체에 접근할 때 403 Forbidden 에러가 발생하는 새로운 문제가 발생했습니다.

원인은 바로 객체 ACL 설정을 누락한 것입니다.

```python

presigned_url = s3_client.generate_presigned_url(
            'put_object',
            Params={
                'Bucket': bucket_name,
                'Key': file_key,
                'ContentType': request.content_type,
                'ACL': 'public-read' # 이 설정이 누락
            },
            ExpiresIn=request.expiration
        )
```

Presigned URL 생성 시 ACL 파라미터를 명시적으로 지정하지 않으면, 기본값은 private으로 설정됩니다.
이는 객체 소유자와 버킷 소유자만 해당 객체에 접근할 수 있음을 의미합니다.

버킷 정책이 Public으로 설정되어 있더라도, 개별 객체에 private ACL이 적용되면 그 객체는 private 상태가 유지됩니다. 이것이 바로 403 Forbidden 에러가 발생한 근본 원인이었습니다.

정책 우선 순위는 다음과 같습니다.

1.객체 수준 ACL (가장 구체적인 수준)
2. 버킷 정책
3. IAM 정책
4. S3 버킷 ACL (가장 일반적인 수준)

Presigned URL 생성 시 ACL을 public-read로 설정했다면, 실제 파일 업로드 요청에도 반드시 동일한 ACL 헤더를 포함해야 합니다.

- x-amz-acl : public-read

이 헤더가 누락되면 S3는 서명 불일치로 인해 403 SignatureDoesNotMatch 에러를 반환합니다. Presigned URL은 지정된 파라미터와 정확히 일치하는 요청에 대해서만 유효하기 때문입니다.

또한, Content-Type도 presigned url을 생성했을 당시의 content type으로 넣어주어야 합니다. 이미지 관련 MIME 타입은 다음과 같습니다.

- image/jpeg: JPEG/JPG 이미지
- image/png: PNG 이미지
- image/gif: GIF 이미지
- image/webp: WebP 이미지
- image/svg+xml: SVG 이미지(벡터 이미지)
- image/tiff: TIFF 이미지
- image/bmp: BMP 이미지
- image/heic: HEIC 이미지(iOS 기기에서 주로 사용)
