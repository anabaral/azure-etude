# nodejs 로 앱 하나 만들기

Readme.md 문서에서 설명한 대로 Deployment Descriptor 파일을 작성해 뭔가 만들려면 상당히 고달픈 과정을 거쳐야 하므로 우리는 helm을 이용해 초기 셋업을 할 것입니다.

네임스페이스 하나 만들고요
```
$ kubectl create ns selee
```

## helm 설치

먼저 저장소 설정을 하고 설치합니다.

보통 bitnami 사이트로 리포지토리를 설정하고 설치하는데 AKS에서 설치하는 것은 아래의 명령이 따로 있네요:
```
$ helm repo add azure-marketplace https://marketplace.azurecr.io/helm/v1/repo
$ helm install -n selee mynode azure-marketplace/node
```
근데 알 수 없는 에러가 납니다(14.0.0버전, 2020-12-01 현재). 그냥 bitnami 거 설치할게요.
```
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm install -n selee mynode bitnami/node
```
## 설치된 것 확인

설치하면 가차없이 실행까지 됩니다. 떠 있는 것들을 확인해 보죠.

```
$ kubectl -n selee get deploy
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
mynode           1/1     1            1           2m56s
mynode-mongodb   1/1     1            1           2m56s

$ kubectl get po -n selee
NAME                            READY   STATUS    RESTARTS   AGE
mynode-58ccbb59b5-p5mnk         1/1     Running   3          5m2s
mynode-mongodb-bd8bd754-6g8j5   1/1     Running   0          5m2s

$ kubectl get svc -n selee
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
mynode           ClusterIP   10.0.10.218    <none>        80/TCP      5m59s
mynode-mongodb   ClusterIP   10.0.115.134   <none>        27017/TCP   5m59s
```

... mongodb가 기본으로 깔리네요. 이게 이 어플리케이션에서의 주 DB 입니다.  

이제 조금씩 어떤 구조인 지 확인해 보겠습니다만 사실 다음 링크 내용도 참조하시면 (저한테) 수월합니다.  
https://github.com/anabaral/aws-etude/blob/master/nodejs-etude.md

디플로이 구조를 확인하는 방법
```
$ kubectl get deploy -n selee mynode -o yaml | less
```

## 브라우저 확인

번듯하게보다는 좀 쉬운 쪽으로 브라우저 접근을 해 보겠습니다.  
가장 쉬운 방법은 위의 [nodejs-etude](https://github.com/anabaral/aws-etude/blob/master/nodejs-etude.md) 링크에 있는, 즉 내 PC에 kubectl 을 설치하고 port-forward 하는 방법이지만 그것보다는 좀더 운영환경에 가깝게..

```
$ kubectl edit svc -n selee mynode # 다음만 수정합니다
spec:
...
  type: ClusterIP --> LoadBalancer 로 수정

$ kubectl get svc -n selee
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
mynode           LoadBalancer   10.0.10.218    40.82.137.156   80:30254/TCP   95m  # 외부 IP가 생기고 포트 80이 열림
mynode-mongodb   ClusterIP      10.0.115.134   <none>          27017/TCP      95m
```

이제 ```40.82.137.156:80 ``` 으로 접속하면 페이지가 나옵니다.  
이게 첫 화면(Readme.md)에서 소개했던 서비스하는 방식 첫번째 입니다.

## DNS 연결

이걸 한 번 domain name 으로 접근 가능하게 해보겠습니다.

디테일은 아직 작성 못하였는데 골자만 적으면 다음과 같습니다.
- 처음에는 Azure Portal 화면에서 DNS영역 추가 했었는데 이걸 인지 못해서
- freenom.com 에 접속해 (회원가입 아니면 소셜로그인) 무료 도메인을 얻음: tuna-az.ga
- 무료 도메인에 레코드셋 추가, ip를 위의 EXTERNAL-IP 로 지정
- 그러고도 인지 못하길래 (어쩌면 시간이 필요했는지도 모르지만) 도메인용 네임서버를 Azure에서 주는 네임서버로 세팅
- 현재 www.tuna-az.ga 로 접속됨
- 이 도메인은 3개월 한시적으로 얻은 거라 2021년 1분기 끝날 때쯤이면 없어집니다.

## 이미지 바꿔보기

디플로이 구조를 잘 뜯어보면 다음을 알 수 있습니다:
- 실행 컨테이너는 node라는 이름의 하나인데 초기화컨테이너(initContainer) 라는 것도 두 개가 실행됨(git-clone-repository, npm-install)
- 두 컨테이너는 이미지가 다르기 때문에 서로 다른 공간을 쓰지만  
  /app 디렉터리를 공유하고 있음(mountPath 항목)  
  사실 같은 pod 내의 컨테이너들은 네트워크도 공유함. (127.0.0.1 의 포트만 나눠가지는 식으로) 
- 초기화컨테이너가 하는 일이 /app 디렉터리로 뭔가 가져오는 겁니다. git clone 명령을 사용합니다.  
  * 이 방식은 나름 편리한데 우리는 커스텀 이미지를 만드는 실습을 할 거기 때문에 이걸 변형할 생각입니다.
  * 즉 초기화컨테이너를 없애는 대신 /app 에 들어갈 내용을 직접 새 이미지에 부어 배포할 겁니다.

다음을 수행합니다:
```
$ mkdir build
$ cd build
$ git clone https://github.com/bitnami/sample-mean.git --branch master  # deploy 초기화컨테이너 선언에 포함되었던 명령 응용
```

Dockerfile 작성
```
$ vi Dockerfile
FROM docker.io/bitnami/node:14.15.1-debian-10-r8

ADD sample-mean /app
RUN cd /app && npm install
```
주의할 것은 initContainer 들이 실행하는 순서대로 Dockerfile 에서도 실행되어야 합니다.
여기서는 소스들이 먼저 존재해야 npm install 이 그 안의 파일을 읽기 때문에 중요합니다.

이제 빌드하고 그 결과를 AKS에서 읽을 수 있는 공용 Registry에 부어넣어야 하는데, 이를 위해 Container Registry가 필요합니다.  
```tuna01.azurecr.io```  

그리고 다음을 실행합니다.
```
$ az acr login -n tuna01

$ VERSION=0.1
$ docker build -t node-ex:${VERSION} .
$ docker tag node-ex:0.1 tuna01.azurecr.io/node-ex:${VERSION}
$ docker push tuna01.azurecr.io/node-ex:${VERSION}
```

나중에 조금 더 다듬어서 쉘로 실행하는 게 좋겠네요.
결과를 확인하면 다음과 같습니다:
```
$ docker images
REPOSITORY                  TAG                    IMAGE ID            CREATED             SIZE
node-ex                     0.1                    3bebff9805e7        59 minutes ago      695MB
tuna01.azurecr.io/node-ex   0.1                    3bebff9805e7        59 minutes ago      695MB
bitnami/node                14.15.1-debian-10-r8   567c627a8cf7        5 days ago          695MB
```  
오해하지 말 것은, 위의 리스트는 배스천 서버에 남아있는 캐시를 토대로 보여주는 것입니다.  
```docker rmi``` 등으로 위의 것들을 지워도 ACR에 push했던 이미지는 없어지지 않습니다.  
반대로 ACR에서 이미지를 삭제해도 위의 캐시가 남아있기도 해서, 때로는 저걸 기반으로 다시 살리기도 합니다.

이제 우리가 helm으로 설치했던 nodejs 가 우리가 만든 이미지로 다시 뜨도록 해 보죠:
해야 할 것은
- Deployment descriptor를 건드려서
  * node 컨테이너의 이미지를 우리가 만든 이미지로 바꾸고
  * git-clone-repository 초기화컨테이너는 없애고
  * npm-install 초기화컨테이너도 없애고

등입니다.
이건 ``` kubectl edit deploy -n selee mynode ``` 명령어로 하기도 하는데, 우리는 백업을 만드는 겸 해서 다음과 같이 하겠습니다:
```
$ kubectl get deploy -n selee mynode -o yaml > selee.mynode-deploy.yaml
$ vi selee.mynode-deploy.yaml
...
... metadata.resourceVersion 삭제
... metadata.selfLink 삭제
... metadata.uid 삭제
...
spec:
  template:
    spec:
      containers:
        #image: docker.io/bitnami/node:14.15.1-debian-10-r8
        image: tuna01.azurecr.io/node-ex:0.1
        #imagePullPolicy: IfNotPresent
        imagePullPolicy: Always          # <-- 이것 중요. 특히 시행착오 반복할 때는 필요함. 이게 IfNotPresent 면 이미지를 갱신해도 다시 읽지 않음.
...
#        volumeMounts:
#        - mountPath: /app   # app은 주석처리
...
#      initContainers:
#        이 이하는 모두 주석처리
#      ...
#        volumeMounts:
#        - mountPath: /app
#          name: app
#        workingDir: /app     # 여기까지
      volumes:
#      - emptyDir: {}
#        name: app            # app은 주석처리
      - emptyDir: {}
        name: data

$ kubectl apply -f selee.mynode-deploy.yaml   # 작성된 것 적용
```

아직 한 문제가 남았습니다:
```
$ kubectl get po -n selee
NAME                            READY   STATUS         RESTARTS   AGE
mynode-58ccbb59b5-p5mnk         1/1     Running        3          18h
mynode-688c75f6c7-zjl5d         0/1     ErrImagePull   0          27s   <--- 에러가 나네요?
mynode-mongodb-bd8bd754-6g8j5   1/1     Running        0          18h

$ kubectl describe po -n selee mynode-688c75f6c7-zjl5d
...
Events:
  ...
  Normal   Pulling    15s (x2 over 30s)  kubelet            Pulling image "tuna01.azurecr.io/node-ex:0.1"
  Warning  Failed     15s (x2 over 29s)  kubelet            Failed to pull image "tuna01.azurecr.io/node-ex:0.1": rpc error: code = Unknown desc = Error response from daemon: Get https://tuna01.azurecr.io/v2/node-ex/manifests/0.1: unauthorized: authentication required, visit https://aka.ms/acr/authorization for more information.
  Warning  Failed     15s (x2 over 29s)  kubelet            Error: ErrImagePull
```

k8s 에서 pull 할 때 권한이 없다고 에러 납니다. 이건 이것대로 권한을 부여해야 합니다.  
미리 알았으면 처음에 ACR을 먼저 만들고 AKS를 만들 때 ```--attach-acr``` 옵션으로 연결했으면 되었는데 지금은 그렇게 못하니까 업데이트로 해야 합니다:  
```
$ az aks update -n aks-tuna -g rg-tuna --attach-acr tuna01
AADSTS50076: Due to a configuration change made by your administrator, or because you moved to a new location, you must use multi-factor authentication to access '797f4846-ba00-4fd7-ba43-dac1f8f63013'.
Trace ID: b501db49-5be5-418d-873f-233fa4fc0302
Correlation ID: 45eab984-8a70-42e6-9a10-47c2fdfbe843
Timestamp: 2020-12-02 06:23:00Z
```
위와 같은 에러는 좀 특이한데, '797f4846-ba00-4fd7-ba43-dac1f8f63013' 이게 Azure Resource Manager의 식별자라네요. 즉 multi-factor 인증으로 자신을 증명하지 않은 상태에서는 ARM에 접근할 수 없다는 것입니다.
이걸 별다른 노력 없이 해결하기 위해서는 Azure Active Directory 화면의 '속성' 메뉴로 가서 '보안 기본값 관리' 값을 '아니오'로 바꿔야 합니다.
그건 좀 부담스럽죠. 그 대신 본인이 명시적으로 2-factor 인증을 성공시키는 걸로 하겠습니다.
```
$ az login --tenant 488c43cb-891e-4986-8b6b-40c2e5c7b87f   # 자신의 tenantId 를 입력. 이 옵션이 없으면 1-factor 인증으로 마치기도 함
```  
이렇게 하면 디바이스 코드 입력 외에 별도 메일로 전달되는 코드 입력까지 마쳐야 인증이 완료됩니다.

인증이 성공하고 나면 다시 시도합니다.
```
$ az aks update -n aks-tuna -g rg-tuna --attach-acr tuna01
... 다시 클러스터 정보 출력 ...
```

여기까지 하면 "내 이미지" 기반으로 어플리케이션을 띄운 것까지는 된 셈입니다.  
앞서의 DNS 라든지 다른 작업이 정상적으로 완료되었다면 ```http://www.tuna-az.ga/``` 으로 접속이 가능할 겁니다.  
물론 이게 끝은 아니죠. 

## 개발!

일단 "남의 소스"로 어플리케이션을 띄웠지만, 이걸 조금 수정해서 우리가 원하는 기능을 추가해 보는 것이 최종 목표입니다.  

화면이 있는 어플리케이션을 만드는 것도 좋지만, 그러려면 HTML/Javascript 등 추가로 알아야 할 게 많으니까, 그게 우리의 메인 토픽은 아니니까  
우리는 Nodejs 를 이용해 적당한 API 를 개발할 겁니다.  
이걸 가지고 Slack bot 이나 KakaoTalk Chatbot 을 만들어볼 겁니다.  
(물론 Hello World 수준으로) 


