# pipline
NHN Pipeline Devtools Pipeline > Jenkins 연계한 CICD

## pipeline 생성 과정 및 운영


### 1. pipeline 정보 입력
> * 파이프라인 이름 및 설명 설정
![](이미지/생성1.파이프라인정보.png)

### 2. 빌드할 소스코드 선택
> * 빌드할 소스가 있는 소스 저장소 설정
> * jenkins 빌드 시 해당 STEP 건너뛰기
![](이미지/생성2.소스설정.png)

### 3. 빌드도구 선택
#### - jenkins를 활용할 경우
> * jenkins 활용 시 미리 jenkins 설치와 job을 생성하는 과정이 필요함 <br>
> * jenkins 에서 이미지 저장소 접근을 위한 credential 추가 <br>
![](이미지/docker_credential.PNG)
> * pipeline job 예시<br>
> 
```
node {
    def GITLAB_CODE_REPO_URL = 'http://github.com/soobnii/sht.git';
    def app;
    stage('Clone Sources') { // github에서 소스 얻어오기
        dir("soob/source") {
            git branch: "master", url: GITLAB_CODE_REPO_URL
        }
    }
    stage('code build') { // maven으로 코드 빌드
        dir("soob/source"){
           sh 'mvn package' 
        }
    }
    stage('docker build') { // Dockerfile로 빌드 (이미지 생성)
        dir("soob/source"){
            app = docker.build("cbb06093-kr1-registry.container.cloud.toast.com/tomcat8-soob")
        }
    }
    stage('docker push') { // 이미지 push
        docker.withRegistry('http://cbb06093-kr1-registry.container.cloud.toast.com', 'cbb06093-kr1-registry.container.cloud.toast.com') {            
            app.push("${env.BUILD_NUMBER}")            
            app.push("latest")        
        }
    }
}
```
![](이미지/생성3.빌드설정.png)


### 4. 배포 환경 선택
> * 쿠버네티스 클러스터 선택 및 이미지 배포방식 설정 <br>
> * yaml 형식으로 입력
> * 미리 서비스파드가 실행중인 상태로, Deployment 실행 시 RollingUpdate 됨
> 
```
apiVersion: apps/v1
kind: Deployment
metadata:
    labels:
        app: soob-app
    name: soob-app
    namespace: soob
spec:
    replicas: 1
    selector:
        matchLabels:
            app: soob-app
    template:
        metadata:
            labels:
                app: soob-app
        spec:
            containers:
                -
                    image: 'cbb06093-kr1-registry.container.cloud.toast.com/tomcat8-soob:latest'
                    name: soob-app
                    ports:
                        -
                            containerPort: 8080
                            name: soob-app
            imagePullSecrets:
                -
                    name: nhn-registry-auth
```
> * (참조) 최초 생성 시 secret_deploy_service.yaml 참조 
> 
```
kubectl create -f secret_deploy_service.yaml
```
> * (참조) Secret 상성 시 인코딩된 value 구하는 방법
> 
```
 docker login cbb06093-kr1-registry.container.cloud.toast.com
 cd .docker
 base64 config.json
```

> * (참조) config.json
> 
```
{
	"auths": {
		"cbb06093-kr1-registry.container.cloud.toast.com": {
			"auth": "aW5mbzEwMEBpbmplaW5jLmNvLmtyOkRkMWJmZFZvWEJNNHdRUUw="
		}
	}
}
```
![](이미지/생성4.배포설정.png)

### 5. 자동 실행 설정
> * 자동 실행 유형 : GitHub / 이미지 저장소 (webhooks 등록 필요)
> * 설정한 GitHub 레포지토리에 이벤트 발생 or 이미지 저장소의 특정 이미지 UPDATE 시 해당 파이프라인이 자동으로 실행됨
> * 단, 파이프라인의 자동 실행 방지는 **미사용** 으로 설정되어 있어야 함 <br>
![](이미지/자동실행설정.png)
![](이미지/자동실행설정_webhooks등록.png)

### 6.스테이지 추가
> * 파이프라인 생성 후 스테이지 추가, 변경 가능
> * 추가 시 Deploy 단계에서 **PATH, SCALE, ROLLOUT UNDO, DELETE** 작업 추가 가능
![](이미지/최종스테이지(jenkins).png)
![](이미지/최종스테이지(NHN).png)



