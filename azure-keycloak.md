# Azure에서 keycloak 설치

빝나미 사이트에서 확인해 보면 https://bitnami.com/stack/keycloak/helm  
Azure Marketplace 에 keycloak이 있는 것 같습니다.  
다만 둘의 차이가 지원의 차이 같으니 bitnami 것을 우선으로 받아 보겠습니다.

```
$ helm repo list
$ helm repo add bitnami https://charts.bitnami.com/bitnami   # bitnami가 없으면

$ helm search repo bitnami/keycloak
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
bitnami/keycloak        3.0.4           13.0.1          Keycloak is a high performance Java-based ident...
```

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




