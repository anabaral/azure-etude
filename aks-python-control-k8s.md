# Python 으로 kubernetes 제어하기

## 프로토타이핑

항상 쉽게 시작하는 게 좋음.

python 에 kubernetes가 깔린 이미지를 구해서 Deployment든 StatefulSet으로든 띄워놓았다고 가정함. 
그런게 있으면 다행이고 없으면 만들어야 함...
어쨌든 여기선 그게 있다고 가정하고 다음과 같이 확인한다.
```
$ kubectl get po -n app | findstr controller
controller-6746f5b4f9-zp6cr   1/1     Running   0          4m30s
```

이 pod의 실행 `serviceAccount` 가 `controller`라고 가정하자. 없으면 만든다.
```
$ kubectl create sa -n app controller
```

물론 위의 pod 에 해당하는 Deployment나 StatefulSet이 이 `serviceAccount`로 실행한다고 설정해 놓아야 함.
```
$ kubectl edit deploy -n app controller      # 한 예입니다
...
spec:
  template:
    spec:
      ...
      serviceAccount: controller
...
```

이 `controller`에 권한을 부여해야 하는데, 언제나 그렇듯 디테일하게 권한을 조정하는 것은 무척 까다로운 일이니 실환경에서 그렇게 하기로 하고 여기선 다 주자.

```
$ vi controller-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding           # 네임스페이스에 구애받지 않는 권한이어야 하므로 클러스터롤이어야 함
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: controller-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin              # 모든 권한 다 줘
subjects:
- kind: ServiceAccount
  name: controller
  namespace: app
```

이제 이렇게 설정이 되면 권한 문제가 없겠지? 이 POD로 들어가서 뭔가 실행해 보자.  
```
$ kubectl exec -it  -n app controller-6746f5b4f9-zp6cr -- python
>>> 
```
(이 때 POD가 충분한 메모리를 가져야 함. 처음에 200Mi 정도 줬더니 툭하면 죽더라)  

다음 코드를 한 번 실행해 보자.
```
from kubernetes import client, config
#import ssl
# 권한 얻기
config = client.Configuration()
config.api_key['authorization'] = open('/var/run/secrets/kubernetes.io/serviceaccount/token').read()
config.api_key_prefix['authorization'] = 'Bearer'
config.host = 'https://10.0.0.1'
config.ssl_ca_cert = '/var/run/secrets/kubernetes.io/serviceaccount/ca.crt'
config.verify_ssl=True

# api_client 얻기
api_client = client.CoreV1Api(client.ApiClient(config))
# 특정 namespace에 해당하는 pod 들 정보 얻기
ret = api_client.list_namespaced_pod("app", watch=False)
```
참조사이트: http://jason-heo.github.io/python/2020/12/19/kubernetes-python-api.html  





