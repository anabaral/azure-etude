# Cert-Manager 설정

Cert-Manager 설치는 helm 으로 하면 간단히 되니까 따로 설명을 하지 않겠습니다.  
여기서는 내가 원하는 도메인에 신뢰받는 인증서를 어떻게 연결하는지만 다룹니다.

지웠다 깔았다 시행착오를 반복하였기에 조금 잘못 설명하는 부분이 있을 수 있습니다.

가장 이상적인 절차는 다음과 같습니다:
1. Ingress에 cert-manager 이나 TLS 관련 설정이 없는 채 존재
2. ClusterIssuer 를 올바른 형식으로 생성
3. Certificate 를 올바른 형식으로 생성.  
   이 때 신경쓸 건 없지만 다음 딸림 리소스들이 생성됩니다.
   - 이 때 지정한 이름의 secret이 같은 namespace에 없으면 생성됩니다. (이름에 hash가 붙음) 
   - CertificateRequest 라는 리소스가 생성됩니다
   - Order 라는 리소스도 생성됩니다
4. Ingress 를 올바른 형식으로 수정 적용


처음의 ingress 설정
```
apiVersion: networking.k8s.io/v1    # 이렇게 해도 extensions/v1beta1 으로 들어가던데 뭐가 잘못된 건지 모르겠네요
kind: Ingress
metadata:
  annotations:
    appgw.ingress.kubernetes.io/ssl-redirect: "false"
  name: my-example
  namespace: my-ns
spec:
  rules:
  - host: my-example.my-az.net
    http:
      paths:
      - backend:
          service:
            name: my-example
            port:
              number: 8080
        path: /
        pathType: ImplementationSpecific
```

인터넷에서 찾은 ClusterIssuer 설정
```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: '<내_이메일_주소>'
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

인터넷에서 찾은 Certificate 설정 (Optional ※)
```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-prd-tls
  namespace: my-ns
spec:
  secretName: my-prd-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: my-example.my-az.net
  dnsNames:
  - my-example.my-az.net
```

정상적으로 인증서가 등록되었는지 확인하는 방법
```
$ kubectl describe certificate -n my-ns my-prd-tls | tail
Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Issuing    31s   cert-manager  Issuing certificate as Secret does not exist
  Normal  Generated  31s   cert-manager  Stored new private key in temporary Secret resource "my-prd-tls-9zwfb"
  Normal  Requested  31s   cert-manager  Created new CertificateRequest resource "my-prd-tls-b827r"
  Normal  Issuing    4s    cert-manager  The certificate has been successfully issued
```
위에처럼 `The certificate has been successfully issued` 메시지를 보면 성공입니다.

※ Certifiacte 는 미리 만들 필요 없습니다. (2021년 현재 최신 K8s 와 Cert-manager 기준입니다. 옛버전은 아닐 수도 있습니다)
   2021-06-15 실수로 알아낸 것인데 Ingress 설정에서 tls 이름을 잘못 지정했더니 바로 Certificate 와 Secret이 생성되었습니다.
   즉 ClusterIssuer 와 Ingress 를 제외한 어떤 것도 굳이 만들지 않아도 됩니다.

이제 ingress 설정을 cert-manager에 맞게 수정합니다.
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/issuer: "letsencrypt-prod"
    kubernetes.io/tls-acme: "true"
  name: my-example
  namespace: my-ns
spec:
  rules:
  - host: my-example.sk-az.net
    http:
      paths:
      - backend:
          service:
            name: my-example
            port:
              number: 8080
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - my-example.my-az.net
    secretName: my-prd-tls
```

## 시행착오 기록

ClusterIssuer가 staging용이 있기에 이걸 먼저 시도했었습니다.  
시도할 거면 도메인 이름이라도 시험적인 이름을 써야 했는데...
```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: '<내_이메일_주소>'
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx   # 이것도 꼭 nginx여야 하나? 싶었는데 kubernetes 제공 ingress-controller를 쓴다면 그냥 이걸로...
```
이걸 기반으로 진행하면 루트 인증서가 신뢰받지 못하는 것이어서 낭패를 봅니다:  
![루트인증서가신뢰못받는](https://github.com/anabaral/azure-etude/blob/master/img/cert-with-tls-wrong.png)

잘못 만들어 놓은 것을 수정할 때도 유의가 필요합니다. 가장 쉬운 접근은 다 지우고 다시 시도하는 것인데  
이 때 Ingress도 괜한 것을 붙잡지 않도록 cert-manager 및 TLS 관련 설정을 없애 놓고 시작합니다.

위에 언급한 `CertificateRequest` 나 `Order` 같은 리소스에 문제가 생겼는지 점검도 필요합니다.  
`Certificate` 리소스가 제대로 'issued' 상태에 이르렀는지 확인하고 나서 `Ingress` 를 맞게 수정하세요.

아래는 잘 된 인증서의 사례입니다:  
![성공사례](https://github.com/anabaral/azure-etude/blob/master/img/cert-with-tls-valid.png)

이건 `Ingress` 설정에 뭔가 문제가 있을 때 나옵니다:  
![fake_cert](https://github.com/anabaral/azure-etude/blob/master/img/cert-without-tls-wrong.png)



