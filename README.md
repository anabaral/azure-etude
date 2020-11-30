# AZURE 에서 뭔가 해 보자

## AKS 를 가지고 뭘 하기

일단 AKS 를 건드려 봅시다.

배스천을 설정하고 클러스터를 만드는 방법은 아래 링크에 소개:

[AKS 셋업](https://github.com/anabaral/azure-etude/blob/master/aks-setup.md)


클러스터에서 서비스를 제대로 만들기 위해 알아야 할 것은 엄청 많지만 대략 나누면 다음 정도입니다.
- 서비스하는 어플리케이션 구성하기
- 이 서비스를 외부에서 접속 가능하게 연결하기

### AKS에서 서비스 어플리케이션 구성

K8S에서 어플리케이션은 대략 다음을 구성해야 합니다. 경우에 따라 빠지는 것 혹은 추가로 필요한 것이 있을 수 있습니다.

![k8s에서의 Application](https://github.com/anabaral/azure-etude/blob/master/img/k8s-app.svg)

- Deployment 혹은 Statefulset, Daemonset 의 형태로 어플리케이션 배포형태를 결정하고 배포기술파일을 작성해야 합니다.
- Deployment 는 Pod 라는 형태로 인스턴스화 합니다.  
  인스턴스라는 것은, java object instance나 VM instance 처럼 같은 게 여러 개 뜰 수 있다는 의미입니다. 보통 replica count 수로 이 개수를 조절합니다.
- Pod는 여러 컨테이너로 구성되게 됩니다.
- 컨테이너 각각에 대해 컨테이너 이미지가 필요합니다.  
  이미지는 이론적으로는 맨땅에서 만들 수도 있다고 하는데 보통은 기존의 이미지를 가져와 add-on 해서 만듭니다.
- 이미지의 파일구조가 그대로 컨테이너의 파일구조가 되는데, 여기 무언가 쓸 수 있지만 Pod가 종료하면 내용을 잃어버립니다.
- 그래서 영구저장을 위한 스토리지가 필요하고, Persistent Volume 이나 Persistent Volume Claim 의 구성이 필요합니다.


### AKS에서 서비스를 외부 연결

K8S에서 서비스를 외부 연결하는 방법은 NodePort 이용하는 방법과 LoadBalancer 이용하는 방법이 있는데 일반적인 것은 후자입니다.

AKS 등 Kubernetes 에서 Load Balancer 를 사용하는 방법은 다음과 같이 세 가지로 나눌 수 있습니다:

![K8s에서의 Load Balancer](https://github.com/anabaral/azure-etude/blob/master/img/k8s-lb-1.svg)

- 그림 왼쪽, 서비스 타입을 LoadBalancer로 설정하면 어느 포트를 잡아서 그 포트로 접속하면 서비스를 접근할 수 있습니다.  
  클러스터 통틀어 하나의 서비스만 사용한다면 80/443 포트로 서비스하면 이것으로 충분합니다.
- 그림 중간, 여러 서비스를 하나의 포트로 서비스할 때 Google 사이트에서 제공하는 kubernetes-ingress 서비스를 설치해 사용합니다.  
  (https://github.com/kubernetes/ingress-nginx 참조)  
  이러면 80/443 포트로 접근하게 하고  Ingress 설정과 요청 경로에 따라 서로 다른 서비스로 분기시킬 수 있습니다.
- 그림 오른쪽, Load Balancer 에 해당하는 기능을 벤더 제공 솔루션이 대체합니다.  
  이 방식을 하면 말 그대로 부하 분산기 CPU/Memory 자원이 클러스터 외부에 존재하기에 부담이 나누어지며, 당연히 과금도 별도로 일어납니다.


