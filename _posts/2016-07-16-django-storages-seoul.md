---
layout: post
title: django-storages S3 서울 리전 연동
---

[8퍼센트](https://8percent.kr/?utm_source=github_page&utm_medium=blog&utm_campaign=06_004) 인프라를 AWS 도쿄에서 [서울로 이전](/2016/06/06/aws-seoul.html)할 때 S3와 CloudFront도 이전을 했다.
하지만 django-storages 연동 문제로 S3와 CloudFront는 한동안 도쿄 리전을 사용할 수 밖에 없었다.

## 원인 파악

기존 리전들의 S3는 Signature Version 4와 Version 2 모두 지원하는 반면 비교적 최근에 생긴 서울, 프랑크푸르트 등 리전의 S3는 Signature Version 4만 지원한다.
django-storages의 리전 설정을 서울로 변경하게 되면 Signature Version 2를 사용하게 되어 모든 응답이 400 Bad Request로 오게 된다.
Signature Version 4를 사용하도록 해주어야 한다.

## 문제 해결

django-storages를 Signature Version 4를 사용하여 서울 리전과 연동하도록 하기 위해 아래의 코드를 사용했다.

### settings.py

```python
DEFAULT_FILE_STORAGE = 'eight.utils.s3_storage.S3Storage'

AWS_ACCESS_KEY_ID = 'access_key_id'  # access key
AWS_SECRET_ACCESS_KEY = 'secret_access_key'  # secret access key
AWS_REGION = 'ap-northeast-2'
AWS_STORAGE_BUCKET_NAME = '8percent-bucket'  # 실제 bucket 아님
AWS_QUERYSTRING_AUTH = False
```

### eight/utils/s3_storage.py

```python
import os

from django.conf import settings
from storages.backends.s3boto import S3BotoStorage

os.environ['S3_USE_SIGV4'] = 'True'


class S3Storage(S3BotoStorage):

    @property
    def connection(self):
        if self._connection is None:
        self._connection = self.connection_class(
            self.access_key, self.secret_key,
            calling_format=self.calling_format,
            host='s3.%s.amazonaws.com' % settings.AWS_REGION
        )
        return self._connection
```
> host를 s3.amazonaws.com 대신 s3.ap-northeast-2.amazonaws.com로 하게 되면 Signature Version 4로 연동을 하게 된다.

위의 코드 적용으로 AWS 이전을 마무리할 수 있었다.
S3BotoStorage를 상속받아서 connection 메소드를 오버라이딩하는 코드다.

## 더 간단한 해결 방법

이 글을 쓰려고 검색을 하다가 더 간단한 방법을 찾아서 소개하려고 한다.

```python
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto.S3BotoStorage'

AWS_ACCESS_KEY_ID = 'access_key_id'  # access key
AWS_SECRET_ACCESS_KEY = 'secret_access_key'  # secret access key
AWS_REGION = 'ap-northeast-2'
AWS_STORAGE_BUCKET_NAME = '8percent-bucket'  # 실제 bucket 아님
AWS_QUERYSTRING_AUTH = False
AWS_S3_HOST = 's3.%s.amazonaws.com' % settings.AWS_REGION
```

AWS_S3_HOST 설정으로 host 변경이 된다. S3BotoStorage 코드를 분석해보니 이 설정에 따라 host 변경이 된다.
이 방법으로 변경 후에 테스트 코드를 돌려보니 정상적으로 파일 업로드, 다운로드가 잘 된다.
좀 더 테스트를 해보고 문제가 없다면 8퍼센트에도 적용하려고 한다.
문제 해결하던 시점에는 이 방법이 잘 안됐던 것으로 기억하는데 지금은 잘 돼서 조금 의아하긴 하다.

## 맺음말

[8퍼센트](https://8percent.kr/?utm_source=github_page&utm_medium=blog&utm_campaign=06_004)에 합류하면서 python, django를 제대로 쓰기 시작했다.
python, django 뉴비이기 때문에 이번 처럼 짤막한 문제 해결 방법이나 지식, 팁 등을 계속 공유해보려고 한다.
사실 개발이나 분석은 그냥 하면 되지만 그것을 정리해서 이렇게 글로 남기는 것이 더 힘든 것 같다.

## 참고 링크

[https://github.com/jschneier/django-storages/issues/28](http://devpotenup.blogspot.kr/2016/04/aws-region-django-s3.html)
[http://devpotenup.blogspot.kr/2016/04/aws-region-django-s3.html](http://devpotenup.blogspot.kr/2016/04/aws-region-django-s3.html)
