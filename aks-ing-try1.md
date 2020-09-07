# 고정ip에 ingress controller 깔면서 시험

이 글을 참조했습니다. https://docs.microsoft.com/ko-kr/azure/aks/ingress-static-ip

AKS를 만든 가정하에 그 AKS가 노드그룹으로 사용하는 리소스 그룹 안에 지정하면 나중에 AKS 지울 때 같이 지워집니다.
```
$ NODE_RES_GRP=$(az aks show --resource-group 04226 --name myAKSCluster --query nodeResourceGroup -o tsv)
$ az network public-ip create --resource-group ${NODE_RES_GRP} --name keycloak-public-ip --sku Standard --allocation-method static --query publicIp.ipAddress -o tsv
52.141.29.111
```
만들어진 IP를 확인합니다. 앞으로 쓰게 되니까.. (다행히 잊어도 콘솔화면에서 찾을 수 있습니다)

DNS 영역하고 레코드셋을 만들어 봅시다.
```
$ az network dns zone create -g MC_04226_myAKSCluster_koreacentral -n azure.skmta.net
$ az network dns record-set a add-record --resource-group MC_04226_myAKSCluster_koreacentral --zone-name azure.skmta.net --record-set-name keycloak --ipv4-address 52.141.29.111
```

```
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

$ vi ingress-values.yaml
controller:
  replicaCount: 1
  nodeSelector:
    beta.kubernetes.io/os: linux
  service:
    loadBalancerIP: 52.141.29.111
    annotations:
      service.beta.kubernetes.io/azure-dns-label-name: keycloak # 꼭 한 단어일 필요는 없지만.. 해보면 한계를 알 수 있습니다
defaultBackend:
  nodeSelector:
    beta.kubernetes.io/os: linux
```

위의 파일을 이용해 설치를 해 봅시다.
```
$ helm install nginx-ingress -n ingress-basic -f ingress-values.yaml ingress-nginx/ingress-nginx
NAME: nginx-ingress
LAST DEPLOYED: Mon Sep  7 12:54:18 2020
NAMESPACE: ingress-basic
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace ingress-basic get services -o wide -w nginx-ingress-ingress-nginx-controller'

An example Ingress that makes use of the controller:

  apiVersion: networking.k8s.io/v1beta1
  kind: Ingress
  metadata:
    annotations:
      kubernetes.io/ingress.class: nginx
    name: example
    namespace: foo
  spec:
    rules:
      - host: www.example.com
        http:
          paths:
            - backend:
                serviceName: exampleService
                servicePort: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
        - hosts:
            - www.example.com
          secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```

만들어진 도메인을 확인해 봅시다:
```
$ az network public-ip list --resource-group MC_04226_myAKSCluster_koreacentral --query "[?name=='keycloak-public-ip'].[dnsSettings.fqdn]" -o tsv
keycloak.koreacentral.cloudapp.azure.com
```

위에 도메인 만든 거하고 다르죠?

위에 도메인이.. 권한을 안 줘서 그런지 무슨 문제가 있는지 접속이 안됩니다. 
```
$ ping keycloak.azure.skmta.net
ping: keycloak.azure.skmta.net: Name or service not known
```
접속이 안되는 정도가 아니라 아예 이름을 못찾네요.

반면 만들어진 도메인으로 연결해 보면 잘 됩니다.<br>
아, 물론 Ingress를 만들어야 확인 되겠죠.

```
$ vi keycloak-ing.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    #kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
  name: keycloak-ingress
  namespace: mta-infra
spec:
  tls:
  - hosts:
    - keycloak.koreacentral.cloudapp.azure.com
    #- azure-keycloak.skmta.net
    secretName: mta-infra-tls
  rules:
  - host: keycloak.koreacentral.cloudapp.azure.com
    #host: azure-keycloak.skmta.net
    http:
      paths:
      - backend:
          serviceName: keycloak-http
          servicePort: http
        path: /
```

확인은 물론 브라우저로 가능하고, curl로도 됩니다.
```
$ curl keycloak.koreacentral.cloudapp.azure.com
<html>
<head><title>308 Permanent Redirect</title></head>
<body>
<center><h1>308 Permanent Redirect</h1></center>
<hr><center>nginx/1.19.2</center>
</body>
</html>
```





