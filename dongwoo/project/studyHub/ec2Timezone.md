## 1. 문제상황
- 배치 시스템으로 매일 새벽 4시에 모집글 상태 자동 변환을 구현하고 테스트도 잘 통과 했지만 배포된 서버의 시간이 한국 시간과 다른것을 확인
  ![](https://velog.velcdn.com/images/wellbeing-dough/post/8781d09c-2f3e-4a6c-834d-ae500b187752/image.png)
  도대체 어디나라 시간일까 궁금해서 알아봤는데 영국의 시간이라고 함 조심스럽게 런던에 그리니치 천문대 기준이 아닌가? 라는 생각이 들었음

## 2. 해결
- sudo ln -sf /usr/share/zoneinfo/Asia/Seoul /etc/localtime 로 Timezone을 한국 표준(KST)로 변경하는 리눅스 명령어가 있는데 우리는 CICD 블루그린 배포로 매 배포마다 ec2가 달라짐(돈을 아끼기 위해...) 매 배포마다 한국 표준으로 설정할 수 없어서 deploy.yml에 적용하면 됨

```yaml
     # EB에 CD 하기 위해 추가 작성
      - name: Generate deployment package
        run: |   #|작성하면 명령어를 여러줄 적을 수 있다
          mkdir deploy #깃헙에 있는 서버에 deploy폴더 만들고
          cp build/libs/*.jar deploy/application.jar #아까 자르파일로 구은거 application.jar로 복사
          cp Procfile deploy/Procfile #Procfile을 deploy로 옮긴다
          cp -r .ebextensions deploy/.ebextensions #eb~저 파일도 deploy에 이동
          cd deploy && zip -r deploy.zip . #이모든 파일을 dploy.zip으로 압축한다
      - name: Deploy to EB #ELB에 배포하자
        uses: einaregilsson/beanstalk-deploy@v21 #이것도 라이브러리 사용해서 간단하게 배포
        with:
          aws_access_key: ${{ secrets.AWS_ACCESS_KEY }} #밑에서 설명
          aws_secret_key: ${{ secrets.AWS_SECRET_KEY }}
          application_name: aws-v5-beanstalk # 엘리스틱 빈스톡 애플리케이션 이름!
          environment_name: Aws-v5-beanstalk-env # 엘리스틱 빈스톡 환경 이름!
          version_label: aws-v5-${{steps.current-time.outputs.formattedTime}} #버전명이 같으면 안되서 시간넣음
          region: ap-northeast-2 #서울
          deployment_package: deploy/deploy.zip #최종적으로 S3에 접근해서 deploy.zip을 던지겠다
```
CD부분만 가져왔는데 저기서 .ebextensions에 명령어파일을 하나 더 만들어서 실행 해보자
![](https://velog.velcdn.com/images/wellbeing-dough/post/093b7ccc-0e9f-4c3f-a392-9cedf7da883b/image.png)
01_update_timezone.config 만들고
```config
commands:
  01_update_timezone:
    command: "sudo ln -sf /usr/share/zoneinfo/Asia/Seoul /etc/localtime"
```
명령어를 설정 해 준다 그러면 ebextensions가 deploy.zip으로 명령어가 같이 압축되서 deployment_package에 잘 들어갈 수 있다
![](https://velog.velcdn.com/images/wellbeing-dough/post/0f26e898-cd51-4731-8d06-067d93f73dcb/image.png)
현재 시간이랑 잘 맞는다