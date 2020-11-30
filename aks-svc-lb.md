# AKS 에서의 Load Balancer 사용 방식들

AKS 에서 Load Balancer 사용하는 방법은 다음과 같이 정리할 수 있다:

![K8s에서의 Load Balancer](https://github.com/anabaral/azure-etude/blob/master/img/k8s-lb-1.svg)

- 그림 왼쪽, 서비스 타입을 LoadBalancer로 설정하면 어느 포트를 잡아서 그 포트로 접속하면 서비스를 접근할 수 있다.
- 그림 중간, 여러 서비스를 하나의 포트로 서비스할 때 Google 사이트에서 제공하는 kubernetes-ingress 서비스를 설치해 사용한다.  
  (https://github.com/kubernetes/ingress-nginx 참조)  
  이러면 80/443 포트로 접근하게 하고  Ingress 설정과 요청 경로에 따라 서로 다른 서비스로 분기시킨다.
- 그림 오른쪽, Load Balancer 에 해당하는 기능을 벤더 제공 솔루션이 대체한다.


# AKS 에서 service: Load balancer type으로 시작해 

가장 기본적인 것부터 차근차근 하자. Applcation Gateway 만드는 거 이전에..

## service type: LoadBalancer

AKS 에서 IP를 공개하는 가장 기본적인 방법임.

aws 쪽에서 잘 설치해 왔던 keycloak을 설치하되, service.type=LoadBalancer 로 두고 설치하면 EXTERNAL-IP 가 생성된다.

설치 과정은 따로 두었음: https://github.com/anabaral/azure-etude/blob/master/aks-keycloak-install.md

아무튼 확인.
```
$ kubectl get svc keycloak-http
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                      AGE
keycloak-http   LoadBalancer   10.0.168.216   52.141.26.2   80:31497/TCP,8443:30961/TCP,9990:31379/TCP   30m
```

On-Premise 에서야 이렇게 만들어도 의미가 없지만 Azure에서는 이 IP를 자동으로 공용 IP로 등록해 준다.
화면에서도 확인할 수 있지만 다음 명령으로 확인:
```
# ip-address 에만 관심 있으면
$ az network public-ip list | jq '.[]| select(.tags.service == "mta-infra/keycloak-http") | [ .ipAddress , .name ]'
[
  "52.141.29.136",
  "kubernetes-a43d2d7e02c96496abd21e57848879c4"
]

# 전체 정보를 보고 싶으면 다음을 입력함. 내용은 여기선 생략
$ az network public-ip list | jq '.[]| select(.tags.service == "mta-infra/keycloak-http") '
```

이렇게 하면 우여곡절 끝네 접속은 된다. 하지만 이것은 Ingress를 거치는 것은 아니다.


