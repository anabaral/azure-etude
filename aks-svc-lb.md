# AKS 에서 service: Load balancer type으로 시작해 

가장 기본적인 것부터 차근차근 하자. Applcation Gateway 만드는 거 이전에..

## service type: LoadBalancer

AKS 에서 IP를 공개하는 가장 기본적인 방법임.

aws 쪽에서 잘 설치해 왔던 keycloak을 설치하되, service.type=LoadBalancer 로 두고 설치하면 EXTERNAL-IP 가 생성된다.

그것 외에도 (버전 변경과 함께) 몇 가지 바뀐 점이 있어서 설명
```
$ vi keycloak-pvc-9.0.1.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: keycloak-storage-data
  labels:
    app: keycloak
  namespace: mta-infra
spec:
  storageClassName: default  # 이것 default 말고 다른 걸로 바꿔야 함. 둘 이상의 pod에서 접근 불가
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: keycloak-storage-theme
  labels:
    app: keycloak
  namespace: mta-infra
spec:
  storageClassName: default  # 이것 default 말고 다른 걸로 바꿔야 함. 둘 이상의 pod에서 접근 불가
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: keycloak-storage-config
  labels:
    app: keycloak
  namespace: mta-infra
spec:
  storageClassName: default  # 이것 default 말고 다른 걸로 바꿔야 함. 둘 이상의 pod에서 접근 불가
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi



$ vi keycloak-values-9.0.1.yaml
replicas: 2
postgresql:
  enabled: true
  postgresqlUsername: keycloak
  postgresqlPassword: keycloak
  postgresqlDatabase: keycloak
serviceAccount:
  create: true
service:
  type: LoadBalancer
extraVolumes: |
  - name: data
    persistentVolumeClaim:
      claimName: keycloak-storage-data
  - name: conf
    persistentVolumeClaim:
      claimName: keycloak-storage-config
  - name: theme
    persistentVolumeClaim:
      claimName: keycloak-storage-theme

extraVolumeMounts: |
  - mountPath: /opt/jboss/keycloak/standalone/data
    name: data
  - mountPath: /opt/jboss/keycloak/standalone/configuration_temp
    name: conf
  - mountPath: /opt/jboss/keycloak/themes
    name: theme
# keycloak chart 9.x 는 admin 사용자조차 로컬 접속해서 추가 후 재시작하는 방식으로 설정하는데
# 하필 재시작 시 생성되게 만드는 파일이 /opt/jboss/keycloak/standalone/configuration 에
# 만들어지고, 이 위치는 persistent volume화하기 까다로움.
# data 처럼 비운 채로 시작하는 곳이 아니라 theme 처럼 초기 파일을 복사해야 하기 때문.
# 그래서 초기 설치 시엔 _temp 붙여 넣고 add admin 후 재시작 전에 파일 카피, mount 교체 필요.
```

설치...
```
$ helm install keycloak -n mta-infra -f keycloak-values-9.0.1.yaml codecentric/keycloak
```
이후에 몇 가지 위에 적힌 것처럼 해줘야 하는 부분이 있는데 이건 겪어보면 암...

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


