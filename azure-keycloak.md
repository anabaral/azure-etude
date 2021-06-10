# Azure AKS 에서 keycloak 설치

이 문서를 작성하다 보니 이미 예전에 AKS에서 keycloak을 설치한 적이 있네요.
- https://github.com/anabaral/azure-etude/blob/master/aks-keycloak-install.md
- 당시 설치한 건 APP 버전 기준 11.0.0 인데 2021-06 현재 13.0.1 버전이 최신
- 잊고 있던 문제가 존재하는 걸 확인해서.. 조금 더 조사가 필요할 듯 합니다.
- keycloak 관련해서는 Github 내 다른 리포지토리에서 많이 다루었기에 위 문서는 딱 '다른'점만 불친절하게 설명하고 말았습니다.  
  설치과정 포함해서 다시 처음부터 다루어야 할 듯.


빝나미 사이트에서 확인해 보면 https://bitnami.com/stack/keycloak/helm  
Azure Marketplace 에 keycloak이 있는 것 같습니다.  
다만 둘의 차이가 지원의 차이 같으니 bitnami 것을 우선으로 받아 보겠습니다.


## 헬름 저장소 확인

```
$ helm repo list
$ helm repo add bitnami https://charts.bitnami.com/bitnami   # bitnami가 없으면

$ helm search repo bitnami/keycloak
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
bitnami/keycloak        3.0.4           13.0.1          Keycloak is a high performance Java-based ident...
```

## 시험 설치

그냥 아무런 옵션 없이 설치해 볼게요.
```
$ kubectl create ns sso
$ helm install keycloak bitnami/keycloak --namespace sso 
```

설치 시 나오는 가이드 중에 제일 중요한 것은 임시 비번 관련한 것입니다.
```
# 암호 얻는 방법
echo Username: user
echo Password: $(kubectl get secret --namespace sso keycloak -o jsonpath="{.data.admin-password}" | base64 --decode)
```

아무 옵션 없이 설치할 경우 다음이 확인됩니다:
- 해당 네임스페이스에 `LoadBalancer` 타입의 서비스가 만들어집니다.
  우리는 domain name 기준으로 서비스할 생각이므로 ingress 를 생성하고 거기 연결시켜야겠네요.  
  그 전에 당장 로그인 및 비번 변경 정도는 가능하겠죠.
  ```
  $ kubectl get svc -n sso
  NAME                           TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
  keycloak                       LoadBalancer   10.0.13.160    20.194.113.146   80:30357/TCP,443:32027/TCP   114s
  ```
- 기본 DB로 postgresql 이 설치됩니다. 이 정도면 괜찮은 DB 같은데요.
  ```
  $ kubectl get po -n sso
  NAME                    READY   STATUS    RESTARTS   AGE
  keycloak-0              1/1     Running   0          3m2s
  keycloak-postgresql-0   1/1     Running   0          3m2s
  ```
- 이 DB를 위한 PVC도 만들어집니다.
  ```
  $ kubectl get pvc -n sso
  NAME                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
  data-keycloak-postgresql-0   Bound    pvc-5da16d1a-fdf7-45dd-9dfb-d1abbc3677fb   8Gi        RWO            default        4m53s
  ```


다시 삭제하고 다시 설치할 건데, 그 전에 수집할 정보가 있습니다.  
예전 설치 경험(https://github.com/anabaral/azure-etude/blob/master/aks-keycloak-install.md) 문서를 보면 다음이 필요함을 알 수 있습니다:
- storageclass를 실행 uid 를 맞추도록 새로 만들어야 함.
- theme 에 해당하는 디렉터리를 찾아서 볼륨을 미리 생성해야 함.
- data, configuration 에 해당하는 디렉터리는 생성하지 않고 보류. (버전 별 특성이 있어 만들던 볼륨이라)

정보를 수집하기 위해 다음을 실행합니다:
```
$ kubectl exec -it -n sso keycloak-0 -- bash
```
알아낸 정보는 다음과 같습니다:
- uid=1001 (그런데 /etc/passwd 에는 없는 사용자임)
- basedir=/opt/bitnami/keycloak/standalone
- themedir=/opt/bitnami/keycloak/themes


설치되었던 걸 다시 삭제합니다.
```
$ helm delete keycloak -n sso
$ kubectl delete pvc data-keycloak-postgresql-0 -n sso
```

StorageClass 생성
```
$ vi keycloak-sc.yaml
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  labels:
    kubernetes.io/cluster-service: "true"
  name: azurefile-keycloak               # 이름 바꾸고
parameters:
  skuName: Standard_LRS
provisioner: kubernetes.io/azure-file
reclaimPolicy: Delete
mountOptions:                            # 요게 중요. keycloak 유저에 맞추자.
  - uid=1001
  - gid=1001
volumeBindingMode: Immediate

$ kubectl create -f keycloak-sc.yaml
```

themedir 생성
```
$ vi keycloak-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: keycloak-storage-theme
  labels:
    app: keycloak
  namespace: sso
spec:
  storageClassName: azurefile-keycloak # 새로만든 걸로
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi

$ kubectl create -f keycloak-pvc.yaml
```

설치용 파일 
```
$ keycloak-values-13.0.1.yaml
service:
  type: ClusterIP
ingress:
  enabled: true
  hostname: keycloak.sk-az.net
  certManager: true
  annotations:
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
extraVolumes: |
  - name: theme
    persistentVolumeClaim:
      claimName: keycloak-storage-theme
extraVolumeMounts: |
  - mountPath: /opt/bitnami/keycloak/themes_temp
    name: theme
```

설치
```
$ helm install keycloak bitnami/keycloak -n sso -f keycloak-values-13.0.1.yaml
```

다 좋은데 https 접근이 안되네. 위의 `ingress.certManager` 설정은 ingress에 고작 한 줄 추가해 줌: `kubernetes.io/tls-acme: "true"`  
작업을 해 줘야겠다.
```
# clusterissuer는 이미 만든거 그대로 사용


# certificate 를 만들자
$ vi keycloak-cert.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: sso-prd-tls
  namespace: sso #### namespace별 생성
spec:
  secretName:  sso-prd-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: keycloak.sk-az.net
  dnsNames:
  - keycloak.sk-az.net


$ kubectl create -f keycloak-cert.yaml
$ kubectl describe certificate -n sso sso-prd-tls
# 'The certificate has been successfully issued' 메시지 확인

$ kubectl get ing -n sso keycloak -o yaml > keycloak-ing.yaml
$ vi keycloak-ing.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    appgw.ingress.kubernetes.io/ssl-redirect: "true"  # 이건 설치할 때 
    kubernetes.io/tls-acme: "true"                    # 이건 설치할 때 
    kubernetes.io/ingress.class: nginx                # 추가
    cert-manager.io/issuer: letsencrypt-prod          # 추가
    meta.helm.sh/release-name: keycloak
    meta.helm.sh/release-namespace: sso
  labels:
    app.kubernetes.io/component: keycloak
    app.kubernetes.io/instance: keycloak
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: keycloak
    helm.sh/chart: keycloak-3.0.4
  name: keycloak
  namespace: sso
spec:
  rules:
  - host: keycloak.sk-az.net
    http:
      paths:
      - backend:
          serviceName: keycloak
          servicePort: http
        path: /
        pathType: ImplementationSpecific
  tls:                                              # 추가
  - hosts:                                          # 추가
    - cloudadaptor.sk-az.net                        # 추가
    secretName: sso-prd-tls                         # 추가
```







