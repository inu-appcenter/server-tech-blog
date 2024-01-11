## 문제점
1. FCM 시크릿키는 숨겨야 함
2. secret.json을 이그노어 함
3. 그럼 github actions의 CICD에서는 이그노어 된 파일을 볼 수 없음
4. yml의 환경변수가 아니라 json이라서 ELB 환경변수에 넣기도 힘듬

## 해결 방법
1. 깃헙 시크릿 키에 json파일을 저장
2. deploy.yml에서 빌드하기전에 src/main/resources/firebase 경로에 시크릿 키에서 가져온 json 넣는 스크립트 코드 작성

이렇게 하려 했는데 github actions에서 시크릿에 json 넣는것을 피하라 했다
```
To help ensure that GitHub redacts your secret in logs, 
avoid using structured data as the values of secrets.
For example, avoid creating secrets that contain JSON or encoded Git blobs.
```
그래서 해결하다가 중간에 바꿨다
1. 깃헙 시크릿에 json파일을 base64로 인코딩 하여 저장
2. deploy.yml에서 빌드하기전에 시크릿에서 가져온 json base64로 디코딩
3. src/main/resources/firebase 에 디코딩한 json저장

```
      - name: Setup Firebase service key
        run: |
          mkdir -p src/main/resources/firebase
          echo ${{ secrets.FIREBASE_SERVICE_KEY_BASE64_ENCODE }} | base64 -d > src/main/resources/firebase/firebase-service-key.json
```

