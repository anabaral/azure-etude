# AKS 에서 keycloak (chart ver. 9.0.1) 을 설치해 보자.

AKS에서 최신 버전의 keycloak 설치가..
처음엔 별로 다르지 않을 거라 생각했는데 많이 달라서 따로 문서로 작성했다.

keycloak 버전은 다음과 같다:
- 버전 11.0.0 (이미지 버전도 동일)
- 차트 버전 9.0.1


먼저 AKS storageclass의 특성부터 설명.
```
$ kubectl get sc
NAME                PROVISIONER                AGE
azurefile           kubernetes.io/azure-file   3h20m  # 되긴 하는데.. 왜 소유자가 root야?? initContainer chown 명령도 안먹고..
azurefile-premium   kubernetes.io/azure-file   3h20m
default (default)   kubernetes.io/azure-disk   3h15m  # 이것은 둘 이상의 pod에서 접근 불가함
managed-premium     kubernetes.io/azure-disk   3h15m  # 이것은 둘 이상의 pod에서 접근 불가함
```
- default 계열을 쓰더라도 pod 가 하나일 때는 별 문제 없이 쓸 수 있다. (나도 둘 이상 띄워보고 나서 문제를 발견)
- azurefile 을 써도 container 들의 runAsUser 값이 root이면 별 무리없이 쓸 수 있다.
- azurefile-premium 까지는 안해봤는데, 검색해 보니 같은 문제를 안고 있는 것 같아서 시간 낭비 안하려고..

정황상 새로 만들지 않으면 안될 거 같아서, 그렇다고 맨땅에 만들고 싶진 않아서 살짝 고쳐 만듬.
```
$ kubectl get sc azurefile -o yaml > keycloak-sc.yaml   # 받아서 편집하자

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
  - uid=1000
  - gid=1000
volumeBindingMode: Immediate
```

새로 만든 걸 적용
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
  storageClassName: azurefile-keycloak # 새로만든 걸로
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
  storageClassName: azurefile-keycloak # 새로만든 걸로
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
  storageClassName: azurefile-keycloak # 새로만든 걸로
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```

새 keycloak 은 admin 사용자 관리에서 이전 버전과 차이가 난다.   <br>
이전 버전은 그냥 admin 사용자 이름과 비번을 정해주면 그냥 만들고 끝, 이었는데   <br>
새 버전은 일단 띄운 후 컨테이너 안에서 특정 쉘을 실행해 생성해 주어야 한다. <br>
아니면 localhost 접근으로 만들라는데 public cloud 환경에서 어떻게 하겠어.. <br>
근데 생성후 심지어 서버 재시작을 해야 하는데 아까 admin 만든다고 만들어진 파일 위치가 
/opt/jboss/keycloak/standalone/configuration 인데 PV 가 아닌 컨테이너 이미지 영역에 있어서.. 되돌아간다.. <br>
그것 때문에 아래와 같은 꼼수를 사용한다.

```
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
  - mountPath: /opt/jboss/keycloak/standalone/configuration_temp   # 일단 띄우고 복사하기 위해
    name: conf
  - mountPath: /opt/jboss/keycloak/themes_temp   # 일단 띄우고 복사하기 위해
    name: theme
# keycloak chart 9.x 는 admin 사용자조차 로컬 접속해서 추가 후 재시작하는 방식으로 설정하는데
# 하필 재시작 시 생성되게 만드는 파일이 /opt/jboss/keycloak/standalone/configuration 에
# 만들어지고, 이 위치는 persistent volume화하기 까다로움.
# data 처럼 비운 채로 시작하는 곳이 아니라 theme 처럼 초기 파일을 복사해야 하기 때문.
# 그래서 초기 설치 시엔 _temp 붙여 넣고 add admin 후 재시작 전에 파일 카피, mount 교체 필요.
```

설치...
```
$ kubectl create -f keycloak-pvc-9.0.1.yaml
$ helm install keycloak -n mta-infra -f keycloak-values-9.0.1.yaml codecentric/keycloak

# 다 뜬 후에
$ kubectl exec -it keycloak-0 -- bash
bash-4.4$ cd /opt/jboss/keycloak/bin
bash-4.4$ sh add-user-keycloak.sh -r master -u admin
Password:
Added 'admin' to ...
bash-4.4$ cd /opt/jboss/keycloak/standalone/
bash-4.4$ ls
configuration  configuration_temp  data  deployments  lib  log  tmp  # 위에 configuration_temp 라고 설정한 이유
bash-4.4$ cp -a configuration/* configuration_temp/command
# storageclass를 azurefile로 쓸 경우 여기서 권한 문제 날 수 있음..

# 아직 나가지 말고
$ cd /opt/jboss/keycloak/
$ cp -a themes/* themes_temp/

# 끝나고 configuration_temp --> configuration 으로 다시 마운트, themes_temp --> themes 로 다시 마운트
$ kubectl edit sts keycloak
...
```

정상 접속 되는지 확인해 보자.

근데.. 뭐지 이 멍청한 결과는..
(구글크롬 자바스크립트 콘솔창)
![구글크롬_자바스크립트_콘솔창](https://github.com/anabaral/azure-etude/blob/master/img/keycloak_error_https.png?raw=true)
자기가 http 포함시켜 놓고 못 읽게 하네..
