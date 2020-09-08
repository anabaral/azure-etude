# Azure에 ingress controller 테스트 좀더 괜찮은 버전

이 문서는 https://github.com/anabaral/azure-etude/blob/master/aks-ing-try1.md 에서 조금 변형해서 테스트합니다.

이 테스트 와중에 이전 문서에서 성공하지 못했던 사유도 알게 되었습니다.

예전 문서처럼 public ip 를 만들면서 시작할 필요 없습니다.

ingress-controller 를 먼저 만들 겁니다. 
- kubernetes 사이트에서 배포하는 것입니다. 앞서의 문서처럼 helm으로 설치하겠습니다.
- 설치가 되면 service type이 LoadBalancer이므로 External-IP 가 하나 만들어집니다. 
- 몰랐는데, 만들어진 External-IP는 public ip 자원에서 찾을 수 있습니다. 즉 Azure와 상호작용 하고 있다는 방증이죠.
```
$ vi  ingress-values.yaml
controller:
  replicaCount: 2
  nodeSelector:
    beta.kubernetes.io/os: linux
defaultBackend:
  nodeSelector:
    beta.kubernetes.io/os: linux

$ kubectl create ns ingress-basic # 안 만들었다면 만듭니다. 다른 네임스페이스에 설치해도 되긴 합니다.
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx  # 추가 안했다면 합니다.
$ helm install nginx-ingress -n ingress-basic -f ingress-values.yaml ingress-nginx/ingress-nginx
```

서비스 확인해 보죠.
```
$ kubectl get svc -n ingress-basic
NAME                                               TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
nginx-ingress-ingress-nginx-controller             LoadBalancer   10.0.220.130   52.141.29.222   80:30125/TCP,443:31261/TCP   8h
nginx-ingress-ingress-nginx-controller-admission   ClusterIP      10.0.93.83     <none>          443/TCP                      8h
```

저 IP를 기억하거나 저 IP를 가지는 Azure 리소스 명을 기억해 둬야 합니다. DNS 설정에 써야 하니까요.

그리고 keycloak을 설치하시죠. 참조는 https://github.com/anabaral/azure-etude/blob/master/aks-keycloak-install.md <br>
keycloak은 mta-infra 네임스페이스에 설치했다고 가정합니다.

SSL 통신은 꼭 필수적인 건 아니고 결국 제대로 하려면 자체 인증서보다 공인 인증서를 써야 하지만 일단 구색을 갖춰보죠.
이게 싫다면 생략하고 밑의 Ingress를 적당히 고쳐도 됩니다.
```
openssl genrsa -out mta-infra-tls.key 2048
openssl req -new -key mta-infra-tls.key -out mta-infra-tls.csr
openssl req -x509 -days 365 -key mta-infra-tls.key -in mta-infra-tls.csr -out mta-infra-tls.crt
# kubecrl delete secret -n mta-infra mta-infra-tls
kubectl create secret tls -n mta-infra mta-infra-tls --cert mta-infra-tls.crt --key mta-infra-tls.key
```

keycloak의 ingress 구성을 합니다. 
앞서 말했듯이 SSL 통신을 안하려면 tls 와 redirect 설정을 빼면 됩니다.
```
$ vi keycloak-ing.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
  name: keycloak-ingress
  namespace: mta-infra
spec:
  rules:
  - host: keycloak.sk-az.net
    http:
      paths:
      - backend:
          serviceName: keycloak-http
          servicePort: http
        path: /
  tls:
  - hosts:
    - keycloak.sk-az.net
    secretName: mta-infra-tls
```
참고사항:
* kubernetes 에서 배포하는 nginx 기반 ingress-controller는 path가 '/' 가 기본입니다. 오히려 '/*' 는 에러. (regex 켜면 다르겠죠)
* ingress가 생성되거나 수정될 때 마다 ingress controller 의 로그에 다음과 같은 내용이 뜨므로 변경이 제대로 먹히는 지를 파악할 수 있습니다:
```
I0908 10:03:42.476094       6 event.go:278] Event(v1.ObjectReference{Kind:"Ingress", Namespace:"
mta-infra", Name:"keycloak-ingress", UID:"36e0b749-649e-4496-9428-a6a075c93822", APIVersion:"net
working.k8s.io/v1beta1", ResourceVersion:"48732", FieldPath:""}): type: 'Normal' reason: 'UPDATE
' Ingress mta-infra/keycloak-ingress
```



DNS 영역을 생성합니다.
- 명령어로 해도 됩니다만 어차피 화면에서 끝맺음할 거라서 콘솔화면에서 했습니다.
- MC_04226_myAKSCluster_koreacentral 그룹에 생성하여 k8s 제거할 때 같이 제거되도록 했습니다.
그다음 DNS 레코드셋을 생성합니다. 
- 여기 대상을 ingress 를 만들면서 생성된 public ip 로 맞추면 됩니다. ip로 지정할 수도 있고 Alias로 지정할 수도 있습니다.

사실 앞서 문서에서도 여기까지는 잘 했는데 이게 문제였습니다.

위의 도메인도 azure 내부의 네임서버에서는 서비스합니다. 그래서 azure 내부의 배스천 서버 같은 곳에서는 
nslookup 으로 ip 리졸브를 할 수 있습니다. (그것도 경우에 따라 안되는 때가 있던데..)

그러나 public internet 에서 내가 만든 도메인을 사용하려면 'App Service 도메인' 에 도메인 등록을 해야 하더군요.
- 이 단계부터 공인 도메인 관리국(?) 같은 데 이게 등록되는 것 같습니다.
- 이 단계부터 추가할 때 국가, 전화번호, 이메일 등등 정보를 물어봅니다.
- 이 단계부터 별도 과금이 들어가는 것 같습니다. 연 11불 남짓?

여기 등록하고 그 하위에 위에 설정한 DNS 영역이 이어지면 그때부터 일반 PC 나 일반 네임서버에서 내 도메인을 IP 리졸브 할 수 있습니다.

여기까지 알았을 때 App Service 도메인을 "AKS가 생성한" 리소스 그룹에 등록했다는 생각이 떠올랐네요.. <br>
지우면 6개월 정도 못쓴다던데..<br>
그래서 리소스 이동을 시도했는데 결과가 어떻게 될 지 모르겠습니다.


아무튼.. 이제 도메인에 인증서만 장착하면 제대로 서비스가 될 것 같습니다.




