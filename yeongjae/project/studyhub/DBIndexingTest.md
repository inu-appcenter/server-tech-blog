## **개요**

스터디 허브 프로젝트의 성능 개선을 위해 DB 인덱스에 관해 학습하던 중 문득 성능 차이를 눈으로 보고싶다는 생각이 들어 직접 구현하게 되었습니다.

---

## **프로젝트 설계**

DB 인덱스 성능 테스트가 목적이기 때문에 설계를 간단하게 구성했습니다.
<br></br>

![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/33c3f0dc-7146-416e-b4a3-c980deeb52f1)
<br></br>

![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/417b2970-0b2e-41b1-891f-bbaecd2127c9)
<br></br>

![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/54b4d096-f3ee-4416-a163-15e5258135c8)
<br></br>

![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/cb2b2ba9-f0e4-4cef-854d-72489de4de7c)
<br></br>

데이터 입력은 아래와 같은 로직으로 진행됩니다.


~~~
1. 클라이언트가 Post 요청을 보낸다.
2. 서버측에서는 이를 받아 랜덤한 문자열을 가지는 UserEntity를 100만개를 생성한다.
3. DB에 UserEntity 100만개를 saveAll 메소드를 이용해 저장한다.
~~~

<br></br>

---

## **조회 성능 테스트**

![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/60491b63-7615-47d6-90e7-b6443ef67933)

<br></br>

![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/9c533463-ed54-4a45-a901-58ddd9dc1b36)



성능 테스트는 다음과 같은 방식으로 진행됩니다.

~~~
1. 인덱싱하지 않은 상태로 DB에서 name을 조회해 동일한 name이 있을 경우 반환한 뒤 수행 시간을 측정한다.
2. 인덱싱한 상태로 DB에서 name을 조회해 동일한 name이 있을 경우 반환한 뒤 수행 시간을 측정한다.
~~~


### **인덱스 하기 전**

![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/f3daef1d-0561-4d25-ad4c-d2b3c4e32abc)
<br></br>

![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/689bd70b-96d0-43a7-86c2-4367a56dec90)

**데이터 100만개를 삽입하기 위한 요청을 보내고 데이터베이스에서 100만개가 삽입된 것을 확인했습니다.**

![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/c2762216-f68b-4c96-8e4a-b9cf1e49c6b6)

**DB 내 삽입된 문자열 중 네개를 선택해 조회해 보겠습니다.**

![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/867ac460-5da4-464b-ae64-2435442cea47)
![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/2bdb63d7-f0de-4ace-bf3d-72d499a3b636)
![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/8502a710-5f00-4857-a7c6-ad133a401427)
![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/59ba8dfb-9932-40e9-ab4e-d06ce6425765)

<br></br>

**네개의 응답 속도의 평균을 내보면 94ms로 추산됩니다.**

### **인덱스 한 이후**

**이번엔 DB 인덱싱을 한 뒤 조회해 보겠습니다.**
![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/9cc768c0-e992-41f2-ba58-7f5f4317f70b)
![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/c3468ff7-e8c5-4c7d-ac3d-ba767ae7fe43)

기본으로 생성된 primary_key 에 대한 인덱스 이외에 하나의 인덱스가 더 생성된 것을 확인할 수 있습니다.

인덱스 생성이 완료됐으므로 DB 내 삽입된 문자열 중 네개를 선택해 조회해 보겠습니다.


![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/6fa48001-9585-41a4-836f-f49740665d63)
![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/965822ab-e556-4c8d-a64d-358605ddd353)
![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/0153a7eb-9ced-44d7-9502-c614aa8fc9d5)
![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/d8147da9-35be-4d28-afed-798d1e020f1d)

**WOW! 평균 10ms 로 줄어든 것을 볼 수 있습니다!**

---


## **삽입 성능 테스트**

인덱스는 조회 성능은 개선하지만 인덱스 유지비용 때문에 삽입, 수정, 삭제 성능이 떨어진다는 단점이 있습니다.
<br></br>
조회 성능 테스트에서 했던 방식과 같은 방식으로 데이터 100만개를 삽입한 뒤, 데이터를 1000개만 삽입하는 API를 따로 만들어 인덱스 하기 전과후의 성능을 비교하겠습니다. 

<br></br>

![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/0f952d77-a628-4e1e-8fa8-41f4724cf555)
![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/097cb13d-94fc-4046-a767-371b7aa93e57)


## **인덱스 하기 전**
![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/b772b4ec-3217-4c60-bca8-80e39ca7ff5c)
![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/d273bcd6-f1e6-4d18-94b3-23023f5d17e9)
![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/ff849ffd-46d7-45b7-b5b6-75b3d879f3c1)


## **인덱스 한 이후**
1000개의 데이터를 삽입했을 때 인덱스 하기 전에는 평균적으로 300ms 초반, 인덱스 한 이후에는 400ms 초반대로 추산되는 것을 볼 수 있습니다.
![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/da708296-eb01-4ae4-a197-c44ac49df7c8)
![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/f277069b-00df-4239-809f-bb074f2270ca)
![image](https://github.com/inu-appcenter/server-tech-blog/assets/97587573/4efba241-5b7d-4785-8673-b2cc5989ecb1)




---
## **결론**
처음 테스트할 때 데이터를 10만개만 넣고 테스트하니 성능 차이가 미비해 100만개로 늘려 테스트하니 성능 차이가 확실하게 보였습니다.

만약 스터디허브에 적용하게 된다면 삽입, 수정, 삭제가 자주 일어나는 StudyPost 보다는 UserEntity에 적용하는 것이 더 좋지 않을까 생각됩니다. (100만명의 유저가 있을까...??)