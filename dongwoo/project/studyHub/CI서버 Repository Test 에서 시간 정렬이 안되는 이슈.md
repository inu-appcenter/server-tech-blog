# CI서버 Repository Test 에서 시간 정렬이 안되는 이슈


지난번에 Repository Test 를 CI서버에서 돌릴때 배포 환경과 테스트 환경을 맞추는 작업을 했었다(https://velog.io/@wellbeing-dough/spring-boot-RepositoryTest-GitHub-Actions%EC%97%90-%EB%8F%8C%EB%A6%AC%EA%B8%B0)

잘 돌아가나 했는데 테스트에서![](https://velog.velcdn.com/images/wellbeing-dough/post/30460f0b-fb77-4e25-a890-573f2defa36a/image.png)
이렇게 최신순 정렬한 테스트 코드가 전부 테스트가 실패했다 분명히 로컬에선 잘 돌아가는데
참고: github actions에서 테스트 결과 아카이빙 하려면 (https://velog.io/@wellbeing-dough/Github-Actions-%ED%85%8C%EC%8A%A4%ED%8A%B8-%EC%83%81%EC%84%B8-%EA%B2%B0%EA%B3%BC-%EC%95%84%EC%B9%B4%EC%9D%B4%EB%B9%99%ED%95%98%EA%B8%B0)

세부 로그를 살펴보니
![](https://velog.velcdn.com/images/wellbeing-dough/post/7cd978da-8422-457f-baf8-965c60c1de0d/image.png)
최신순들이 전부 순서가 바뀌어 있었다

![](https://velog.velcdn.com/images/wellbeing-dough/post/0ddc8210-4a34-485c-9816-a3f8722d8fcf/image.png)
해당 코드의 저장하는 부분인데 뭔가 두개의 데이터의 저장의 차이가 밀리세컨드 차이라서 발생하는 문제같았다 그래서 Thread를 2초간 멈춰봤는데 테스를 통과하는것이였다...

H2환경에서 created_date, modified_date는 이렇게 밀리세컨드까지 다 나온다
![](https://velog.velcdn.com/images/wellbeing-dough/post/a699dc3d-876e-4877-8ad5-12008c5a3ada/image.png)
하지만 배포환경의 MySQL에서는 이렇게 세컨드까지만 나온다
![](https://velog.velcdn.com/images/wellbeing-dough/post/fc2cf520-7f3b-40f8-a8bd-98b36aeaeb21/image.png)

```java
    created_date    TIMESTAMP(3)     DEFAULT NULL,
    modified_date   TIMESTAMP(3)     DEFAULT NULL
```

이렇게 TIMESTAMP(3)으로 하면된다
timestamp(1)은 소수점 첫번째자리,timestamp(2)는 소수점 두번째 자리를 표현한다 최대 6자리 까지 정확도를 높일 수 있다

참고로 DATETIME을 쓰지 않고 TIMESTAMP의 차이는
1. Timestamp 타입은 1970-01-01 00:00:01 ~ 2038-01-19 08:44:07까지의 데이터만 지원, Datetime 타입은 1000-01-01 00:00:00 ~ 9999-12-31 23:59:59 사이의 데이터를 지원
2. (MySQL 5.6.4 버전 이상 기준) Timestamp는 4 bytes+ 3byte(초 단위를 저장하기 위함) 크기가 필요함, Datetime 타입은 5byte + 3byte(초 단위를 저장하기 위함) 크기가 필요함
3. Timestamp 타입의 값은 현재 시각 → UTC 시각으로 변환되며 Datetime은 변환되지 않음 만약 글로벌 서비스에서 Datetime을 사용하여 날짜를 표현할 경우, 한국에서 17:00시에 작성된 글이 미국에서도 그대로 17:00시에 저장된 것처럼 보임
4. Timestamp 타입을 갖는 쿼리는 캐시로 저장되나 Datetime 타입을 갖는 쿼리는 캐시로 저장되지 않음

그래서 TIMESTAMP를 사용했다

참고: https://dev.mysql.com/doc/refman/8.0/en/fractional-seconds.html