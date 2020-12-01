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

(작성 중)

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

```
이렇게만 해도 이미지는 만들어집니다. 물론 빌드하고 그 결과를 Registry에 부어넣어야 쓸 수 있겠죠.





