---
layout: post
title: 8퍼센트 성능 개선
---

8퍼센트에 합류한 지 2달이 다 되어간다. 5월에 진행한 8퍼센트 서버 성능 개선에 대한 기록을 남겨보려고 한다.

스타트업에게 중요한 덕목 중 하나는 빠른 실행력이다. 특히나 빠른 성장 중에 있다면 비즈니스적인 개발에 힘을 쏟게 마련이다.
로직 구현을 빠르게 하다 보면 성능을 신경 쓰지 못하기도 한다. 성능을 신경 쓰지 않고 개발을 한다고 해서 성능 문제가 바로 생기지도 않다.
흔히 말하는 기술 부채 중의 하나로 볼 수도 있겠다. 기술 부채가 무엇인지에 대해서는 8퍼센트 CTO 이호성 님의 [기술 부채](https://brunch.co.kr/@leehosung/2) 포스팅을 참고하면 도움이 될 것 같다.

보통 처음에는 특별히 신경을 쓰지 않아도 서비스가 잘 돌아간다. 성능 문제가 보이기 시작한다면 서비스가 성장하고 있다고 볼 수 있기 때문에 기분 좋게 성능 개선을 진행하면 된다.
Python Django로 구현된 [8퍼센트](https://8percent.kr/) 서버에 대한 이야기를 해보겠다.

5월 둘째 주 이전까지는 성능 문제가 별로 보이지 않았다. 5월 둘째 주에 성능 문제를 겪기 시작했다. 신규 채권이 오픈되는 평일 오후 1시에 사용자가 몰리면서 웹서버와 DB서버의 부하가 증가했고 심한 경우 서비스 접속이 되지 않았다.
단순히 특정 시점에서의 문제라기 보다 성능을 크게 염두에 두지 않은 코드들이 점점 쌓이다가 문제가 터진 것으로 봐야 할 것이다.

원인 파악을 먼저 진행했다. AWS에서 제공하는 CloudWatch 지표와 RDS의 slow query log를 확인했다. 웹서버, DB 서버의 CPU 사용률이 거의 100% 가까이 올라가는 것을 확인했고 1초 이상 걸리는 DB query를 확인했다.
이것만으로는 부족해서 newrelic과 django-debug-toolbar를 코드에 추가했다. newrelic으로 평균 응답 시간이 느린 페이지를 확인하고 django-debug-toolbar를 사용해서 코드를 확인할 수 있었다.
페이지별로 차이는 있지만 대체로 하나의 페이지를 만들어내기 위해 수백 개의 DB query를 발생시키고 있었다.

위의 내용을 바탕으로 적용한 개선 방법은 아래와 같다.

- select_related, values, values_list 적절히 사용
- 불필요한 count query 제거
- model에 @cached_property 활용
- loop 통한 연산 대신 aggregate 사용
- cache 활용

위의 5가지 방법 모두 DB query를 줄이는데 도움이 된다. 조금 더 자세히 살펴보자.

## 1. select_related, values, values_list 적절히 사용

Post 모델에 User 모델이 foreign key로 연결되어 1대 1 관계라고 가정한 예는 아래와 같다. 

```python
authors = set()

# bad example
posts = Post.objects.all().orderby('-created_date')[:100]
for post in posts:
    authors.add(post.user)

# good example
posts = Post.objects.all().select_related('user').orderby('-created_date')[:100]
for post in posts:
    authors.add(post.user)
```

최근 블로그 포스팅 100개의 작성자를 authors에 저장하는 코드다. select_related를 사용하지 않아도 authors의 결과는 동일하다.
하지만 성능 차이는 꽤 난다. select_related를 사용하지 않으면 post 테이블 데이터를 읽는 select 쿼리 1번을 한 후 user 테이블의 데이터를 읽는 select 쿼리를 최대 100번 더 하게 된다.
총 101번의 쿼리가 발생하는 것이다. select_related를 사용하면 쿼리가 실행되는 시점에 1개의 select 쿼리로 post, user 테이블의 데이터를 모두 읽어 들인다.
101번에서 1번으로 쿼리가 줄어든다. 여기에는 네트워크 지연 시간과 DB에서의 lookup 시간이 포함되어 제법 차이가 날 수밖에 없다.
many to many나 many to one 관계에서는 select_related 대신 prefetch_related를 사용하면 된다.

아래의 2개 예는 values, values_list를 사용한 것이다.

```python
post_infos = Post.objects.all().values('title', 'created_date', 'user__name')
    .orderby('-created_date')[:30]
post_infos = Post.objects.all().values_list('title', 'created_date', 'user__name')
    .orderby('-created_date')[:30]
```

최근 블로그 포스팅 30개의 제목, 작성 시간, 작성자 이름을 읽는 코드다. values, values_list 모두 해당 모델의 인스턴스를 반환하는 대신 인자로 전달한 필드들에 대한 값만을 반환한다.
values와 values_list의 차이점은 결과가 dictionary array인지 tuple array인지이다. 상황에 맞게 사용하면 된다.
모델 인스턴스 생성 오버헤드가 없고 위의 예처럼 join이 필요한 경우에는 한 번의 쿼리로 필요한 값들을 읽을 수 있어서 유용하다. 

## 2. 불필요한 count query 제거

```python
posts = Post.objects.all().select_related('user').orderby('-created_date')[:100]
if len(posts) == 0:
    authors = set()
else:
    authors = set()
    for post in posts:
        authors.add(post.user)
```

위의 예를 보면 len(posts)를 하는 시점에 count 값을 읽는 select 쿼리를 하고 count 값이 0이 아닌 경우에 쿼리를 한 번 더 하게 된다.
블로그 포스팅이 하나라도 있으면 대부분 2개의 쿼리를 하게 되는 것이다. 하나의 쿼리지만 줄이는 것이 좋다.

## 3. model에 @cached_property 활용

cached_property 데코레이터를 쿼리를 발생시키는 모델의 메소드에 적용하면 해당 메소드를 호출할 때마다 발생하는 여러 번의 쿼리를 1번으로 줄일 수 있다. 모델 인스턴스가 살아있는 동안 캐싱된다. 

## 4. loop 통한 연산 대신 aggregate 사용

```python
# bad example
total_view_count = 0
for post in Post.objects.all():
    total_view_count += post.view_count

# good example
total_view_count = Post.objects.all().aggregate(Sum('view_count'))
```

위의 예는 view_count의 총합을 구하는 코드이다. bad example처럼 직접 loop 돌려서 view_count를 더하려고 하면 전체 post를 읽어 들인 후에 post별로 view_count를 읽어서 더하게 된다.
모델 인스턴스 생성 오버헤드가 post 개수만큼 발생한다.

## 5. cache 활용

항상 최신 결과일 필요가 없는 페이지의 경우 view를 통으로 캐싱해도 된다. 자주 호출되는 페이지라면 1분 단위 캐싱만 하더라도 효과가 좋다.


## 느낀 점

Python Django 기반 웹서버의 성능 개선 과정을 정리해보았다. RDBMS를 사용하는 대부분의 웹서버라면 비슷한 과정을 거치지 않을까 싶다.
분석이 가능한 여러 툴을 사용해서 병목이 되는 지점을 찾고 개선을 진행해나가면 된다. 이번에 CloudWatch, newrelic, django-debug-toolbar를 사용해서 분석을 진행했다.
과거 Ruby on Rails 기반 웹서버의 성능 개선시에는 CloudWatch, newrelic, ruby-prof, rack-mini-profiler, bullet 등의 도구를 사용했다.
언어별로 도구의 차이만 있을 뿐이다. 이런 경험을 토대로 봤을 때 병목은 DB를 다루는 부분인 경우가 대부분이다.
그래서 위의 개선 방법들처럼 DB 쿼리를 효율적으로 하는 코드를 염두에 두고서 개발을 해나가는 것이 좋다.

