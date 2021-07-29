# Jenkins 에서 Pipeline 설정

이 문서는 2021-07 작업을 기반으로 합니다. 몇 가지 가정이 있는데 문서에 보일 것입니다.


## 주의사항

당신이 2021년에 이 문서를 보고 있다면 염두에 두세요:

- 제발 Bitnami의 Jenkins를 설치하지 마세요.  
  겉으로는 그럴듯 한데 뭔가 심각하게 빠져 있어서 Pipeline 설정할 때 뇌가 뽀개져 버릴 겁니다.  
  특히 kubernetes pod 로 jenkins agent를 만들때 필요한 준비가 제대로 안 되어 있습니다.
- Azure Kubernetes 즉 AKS 를 사용하려고 생각한다면 재고해 보세요.
  Azure 기본 storageclass 들이 제공하는 스토리지는 엄밀히 말해 Linux/Unix 파일시스템이 아닌 것 같습니다.
  읽고 쓰는 건 얼추 되는데 무슨 퍼미션 같은 문제가 갑자기 튀어나와 우리 뇌를 휘젓습니다.
  아주 안되는 것도 아니고 다 되는 것도 아니고... 이런 것 가지고 수명을 단축시키지 마시길.
  ![빡침](./img/sally-with-fury.png)


(2022년쯤이면 혹시 나아질 지도 모르겠습니다)


## Jenkins 설치

다음 명령으로 설치합니다.

```
$ helm repo list | grep jenkins
jenkinsci                               https://charts.jenkins.io

$ vi jenkins-values.yaml
controller:
  adminUser: admin
  adminPassword: ~아싸라비아콜롬비아~
  ingress:
    enabled: true
    hostName: jenkins.sk-az.net
    annotations:             # 여기 annotation들은 cert-manager를 위한 설정인데 여기선 설명 안함
      cert-manager.io/cluster-issuer: "letsencrypt-prod"      
      appgw.ingress.kubernetes.io/ssl-redirect: "true"
      kubernetes.io/ingress.class: nginx
      kubernetes.io/tls-acme: "true"
    tls:               # 아 이걸 빠뜨렸었네.. 어쩐지 tls가 설정 안되더라고.. 저도 안해봐서 이건 설명 안합니다
  installPlugins:      # 아 이것도 빠뜨렸었네요.. 플러그인은 여기서 미리 설치하는 게 좋습니다. 뭐가 필요한 지 미리 알 수 있다면.
serviceAccount:
  name: jenkins
serviceAccountAgent:   # 이거 만들라고 명시한건데 안만들었네요
  name: jenkins-robot
  create: true         # 아 이걸 빠뜨렸었구나.. 좀 잘 볼 걸
$ helm install jenkins jenkinsci/jenkins -n ca  -f jenkins-values.yaml
```

`jenkins-values.yaml` 파일 작성을 위한 참조: https://github.com/jenkinsci/helm-charts/blob/main/charts/jenkins/VALUES_SUMMARY.md


## 추가 설정

### Ingress

TLS가 설정이 빠져서 추가 설정합니다:
```
$ kubectl edit ingress -n ca jenkins
...
  tls:                  # 적절한 위치가 어딘지는 구글링으로 찾아보세요
  - hosts:
    - jenkins.sk-sz.net
    secretName: jenkins-tls
...
```

### PVC

PersistentVolumeClaim 을 추가합니다:
```
$ vi ca-workspace-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/azure-file
  finalizers:
  - kubernetes.io/pvc-protection
  name: ca-workspace
  namespace: ca
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
  storageClassName: azurefile   # 이것 가지고 고생함
  volumeMode: Filesystem
```
`storageClassName` 이 중요한데, 우리가 쓸 수 있는 게 대략 
- `default` 하고 `azurefile` 중에 하나를 택하거나 
- 이 둘 중 하나를 기반으로 커스텀 생성하는 것입니다.

그런데 여러 시도가 무색해져서 그냥 순정(?)으로 택했습니다.

참고로 뭘 잘못했는 지는 모르지만 퍼미션 문제로 골치 썩입니다. 결국 이해한 것은
- 뭔지 모르나 퍼미션 문제를 일으키는 Operation이 존재함. (read나 write는 아닌 것 같음. 777이래도 에러가 나곤 함)
- 그 Operation은 파일/디렉터리의 소유자 (혹은 그룹) 만이 실행할 수 있고
- Linux/Unix 에서는 root는 그걸 무시해야 하는데 root도 지배받음. (테스트해 보면 root 가 chmod /chgrp을 못함)
- azurefile 에서 경험한 것인데, default(azure disk) 에서도 미묘하게 다르면서 비슷한 문제를 겪었으며 그것보다는 azurefile이 낫다고 판단하고 바꾼 것임.
- 이건 문서상으로 봤던 건데, Azure 제공 스토리지에서 symbolic link 혹은 hard link 만드는 게 잘 안된다고 함. (둘 중 하나는 될 지도 모름)


### Plugin 설치

helm 설치할 때 같이 설치하면 좋았는데 이것저것 시도하면서 설치해야 했기에 관리메뉴에서 했습니다.

아래 항목들을 설치합니다 (모두 필요한 건 아닐 수도 있는데 걍 하시죠 뭐가 없어서 안되는 지 찾는 거 너무 힘드니까)
- credentials
- credentials-binding
- github
- kubernetes-client-api
- kubernetes
- pipeline-model-definition
그리고 의존성 관계로 필요한 것들 다 설치합니다. 설치 후 재시작하는 게 좋습니다.


### 설정 확인

굳이 필요한 건 아닌데 확인차 여기 들어가 봅시다:
- Jenkins 관리 > 노드 관리 > Configure Clouds 로 들어가서
- [Kubernetes Cloud Detail] 버튼을 눌러 펼친 후
  다음 항목들에 값이 적절히 들어가 있는지 확인합니다:
  * Kubernetes URL
  * Jenkins URL
  * Jenkins tunnel
- 이게 Bitnami 설치 에서는 없었습니다. 설정해 줘도 Jenkins tunnel은 끝내 오류를 냈습니다. (이새끼들!)


## 파이프라인 구성

### 비번 등록

하나의 파이프라인을 설정하기 위해서 최소한 다음 비번 등록이 필요합니다.
- 소스 저장소(여기서는 Github) 접근용 비번
- 이미지 저장소(azure registry) 접근용 비번

일단 이걸 체크해 봅시다:
- Jenkins 관리 > Configure Credential Providers 로 들어가 
  ![configure_credential_providers](./img/jenkins-credentials-01.png)
- Providers = Only Selected 선택
- Types = Only Selected 선택
- 'Username with password' 가 체크 되어 있는 걸 확인합니다.
- (있죠? Bitnami 건 이것조차 체크가 안되어 있어서.. 구글링에서 뻔히 보이는 걸 못하고 한참 헤맸다구요. 썩을!)

체크가 되어 있으면 Credentials 을 추가해 보죠:
- Jenkins 관리 > Manage Credentials 로 들어가  
  ![manage_credentials](./img/jenkins-credentials-02.png)
- Add credentials 선택
- Kind: username and password
- Scope: 선택 (저는 Global 로 했음)
- 소스 저장소용 입력
  * Username: 
  * Password:
  * Id: git-credentials
  * 참고로 본인이 일상적으로 사용하는 비번보다는 github 에서 read-only 권한만 부여한 token을 생성해서 쓰는 게 좋습니다. 
    구글링 하면 친절하게 나와요. 절대 캡쳐하기 귀찮아서 그러는 거임.
- 이미지 저장소용 입력
  * Username: 
  * Password:
  * Id: registry04226-userpass
  * 참고로 이미지 저장소용 비번은 다음과 같이 Azure Portal이나 Azure CLI를 통해 생성해 두어야 합니다:  
    ![azure_이미지_저장소_](./img/jenkins-credentials-03.png)


### serviceAccount 설정

helm 설치 할 때 잘했으면 이게 필요 없는데.. 아무튼 만들어 봅시다.
```
$ vi jenkins-robot.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: ca
  name: jenkins-robot-role
rules:
- apiGroups:
  - "apps"
  - ""
  resources:
  #- services
  #- deployments
  #- persistentvolumeclaims
  - '*'
  verbs:
  - '*'
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jenkins-robot-rolebinding
  namespace: ca
subjects:
- kind: ServiceAccount
  name: jenkins-robot
  namespace: ca
roleRef:
  kind: Role
  name: jenkins-robot-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-robot
  namespace: ca
---
```

구조는 이렇게 설명됩니다:
- `Role`이 권한을 부여하는 거고,
- `RoleBinding`이 어느 서비스계정에게 이 권한을 줄 건지 매핑을 정합니다.  
- 우리는 Pod가 어느 서비스계정으로서 기능할 건지만 정하면 되는데 그게 `jenkins-robot` 입니다.

`apiGroups` 는 [API groups and versioning](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-groups-and-versioning) 에서 설명되는 개념인데
간단히 요약하자면
```
$ kubectl get svc -n ca cloudadaptor -o yaml | head -3
apiVersion: v1
kind: Service
metadata:
```
이 리소스 (Service) 의 api group은 "" 이고
```
$ kubectl get deploy -n ca cloudadaptor -o yaml | head -3
apiVersion: apps/v1
kind: Deployment
metadata:
```
이 리소스(Deployment) 의 api group은 "apps" 입니다.

이게 둘로 나뉘어지기에 위의 `Role` 정의에서도 apiGroup 을 둘로 나누어야 했습니다.  
그 외에 `resources` 와 `verbs` 는 어느 자원에 어느 동작을 허용해 줄 것인가를 나타내는데  
하나씩 추가해 가면서 빌드 실패를 열 번 넘게 지켜보다 그냥 저렇게 줬습니다...  
(helm 설치 할 적에 세팅만 잘했더라면)


### 파이프라인 작성

초기화면에서 '새로운 Item' > 'Pipeline' 을 선택해서 생성합니다.

다른 옵션을 일단은 체크할 필요 없고 그냥 스크립트만 작성해 봅니다. 

스크립트가 완성되면 이를 Github에 올리는 것도 고려해 보세요.

```
def label = 'jenkins-agent'

def this_time = new java.text.SimpleDateFormat("yyyyMMdd_HHmmss")
                .format(java.util.Calendar.getInstance(
                        java.util.TimeZone.getTimeZone("KST"), java.util.Locale.KOREA).getTime()
                  )

podTemplate(label:label, serviceAccount: "jenkins-robot", namespace: "ca", runAsUser: "0",
  containers: [
    //containerTemplate(name: 'docker', image: "docker:20.10.7", ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'docker', image: "desmart/dind", ttyEnabled: true, privileged: true, command: 'dockerd --host=unix:///var/run/docker.sock '),
    containerTemplate(name: 'kubectl', image: "lachlanevenson/k8s-kubectl:v1.21.2", ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'jnlp', image: "jenkins/inbound-agent:4.3-4-jdk11")
  ],
  volumes: [
    // docker 관련 mount는 docker in docker 이미지를 사용하면서 불필요해짐
    //hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
    //hostPathVolume(hostPath: '/var/lib/docker', mountPath: '/var/lib/docker'),
    hostPathVolume(hostPath: '/etc/hosts', mountPath: '/etc/hosts')
  ],
  workspaceVolume: persistentVolumeClaimWorkspaceVolume(claimName: "ca-workspace", readOnly: false)
) {


    node(label){
      stage('Checkout') {
        checkout([$class: 'GitSCM', 
            branches: [[name: '*/master']], 
            doGenerateSubmoduleConfigurations: false, 
            extensions: [[$class: 'CleanCheckout']], 
            submoduleCfg: [], 
            userRemoteConfigs: [[credentialsId: 'git-credentials', url: 'https://github.com/opmdash/cloudadaptor.git']]
        ])
      }
      stage('Build'){
        container('docker') {
            sh "docker build -t cloudadaptor:${this_time} -f doc/build-ca/Dockerfile . "
            sh "docker tag cloudadaptor:${this_time} registry04226.azurecr.io/cloudadaptor:${this_time}"
        }
      }
      stage('Push'){
        container('docker') {
          try {
            withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: "registry04226-userpass", usernameVariable: 'ACR_USER', passwordVariable: 'ACR_PASSWORD']]){
                sh "docker login -u $ACR_USER -p $ACR_PASSWORD https://registry04226.azurecr.io"
                sh "docker push registry04226.azurecr.io/cloudadaptor:${this_time}"
            }
          }catch(Exception e){
            echo e.toString()
            throw e
          }
        }
      }
      stage('Deploy'){
          container('kubectl') {
            try {
              sh "kubectl apply -f doc/ca-deploy.yaml"
              sh "kubectl set image -n ca  deployment/cloudadaptor cloudadaptor=registry04226.azurecr.io/cloudadaptor:${this_time}"
              sh "kubectl apply -f doc/ca-svc.yaml"
            }catch(Exception e){
              echo e.toString()
              throw e
            }
          }
      }
    }

}
```

부가설명을 덧붙이겠습니다:
- jnlp container 명시
  * `containerTemplate(name: 'jnlp' ... )` 사실 명시하지 않아도 기본으로 Pod에 추가됩니다. 
  * 하지만 기본으로 추가되는 컨테이너 이미지 안의 jdk가 jdk8 인데 이게 master보다 낮은 버전이라서 에러가 나네요..
  * jdk 11 기반의 이미지를 써야 했기에 명시했습니다.
- docker in docker 이미지 사용 `image: "desmart/dind"`
  * 제가 원래 찾았던 이미지는 그냥 docker 였습니다. 이 이미지는 다음 volume 설정을 필요로 합니다:
    ```
    hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
    hostPathVolume(hostPath: '/var/lib/docker', mountPath: '/var/lib/docker'),
    ```
  * 구글링 해 보면 이 설정이 더 보편적이기 때문에 처음엔 이걸 시도했는데, 두 가지 문제가 발견됩니다.
    + `/var/run/docker.sock` 이 sock파일이 아니라 디렉터리로 공유되더라구요. 
      당연히 에러가 나는데 뭐가 잘못된 건지 모르겠습니다.
      기분상으로는 azure storage의 몹쓸 문제가 아닌가 싶습니다.
    + docker daemon이 안 뜬다고 에러가 납니다. 이것 역시 원인은 확실히 모르겠습니다.
      그냥 의심에.. 예전에 (AWS에서) 잘 되던 케이스는 K8S 노드에 기본으로 docker daemon이 떠있던 게 
      아니었을까 싶네요. 여긴 당연히 없고.
  * 결국 이 문제들을 한번에 해결하기 위해서는 컨테이너 안에서 문제가 해결되어 있는 이미지가 필요했습니다.
- `workspaceVolume` 설정의 `ca-workspace` PVC 가 저기 위에서 만들어 놓은 PVC입니다.
- 소스 잘 보면 `credentialsId` 를 쓴 곳이 두 군데 있습니다. 앞서 만들어 놓은 credential 을 사용한 겁니다.
- 다른 곳에서도 시행착오를 엄청 했는데.. 그것들은 인터넷으로부터 제대로 된 예제를 찾지 못한 탓이라.. 설명할 건 별로 없네요..


### Role 에 대한 부연설명 

![Jenkins가 사용하는 Kubernetes ServiceAccount 구조](./img/jenkins-sa-arch.svg)

위에서 다 설명하지는 않았는데 실제 필요한 serviceAccount 는 하나 혹은 두 개입니다.
- 첫 번째는 Jenkins-Agent 를 생성하고 정보조회하고 삭제하는 권한과 연결된 SA입니다.
  * 이건 필수입니다. 
- 두 번째는 Jenkins-Agent가 현재 deployment 나 statement, job 등을 제어하고자 할 때 필요한 SA(및 권한)입니다.
  * 위의 스크립트에서 `serviceAccount: "jenkins-robot",` 라고 기술된 부분에서 지정하여 사용합니다.
  * 만약 만드는 Jenkins Pipeline 스크립트 안에서 이미지 빌드 및 푸쉬까지만 행하고 
    디플로이는 수작업으로 혹은 ArgoCD 같은 CD Tool에게 맡긴다면, 두 번째 SA는 필요 없습니다.
    그럴 때는 그냥 첫번째 SA를 재사용하면 편합니다.
  * 만약 만드는 Jenkins Pipeline 스크립트 안에서 어플리케이션 디플로이까지 하도록 하려면,
    두 번째 SA를 (그리고 role, rolebinding)을 잘 설정해야 합니다.
  * 심지어 스크립트에서 디플로이하려는 어플리케이션이 Jenkins와 다른 Namespace에 존재하는 것이면
    그냥 role, rolebinding 이 아니라 clusterrole, clusterrolebinding 을 설정해야 합니다.


### Git push 에 대한 부연설명

위의 환경과 조금 다른 환경에서 (2021 ChatOps) pipeline설정을 해야 했었는데, 거기는 Jenkins가 설치된 네임스페이스와
대상 어플리케이션의 네임스페이스가 달랐습니다 (이게 일반적이긴 하죠)  
이런 환경에서는 jenkins agent 가 deploy까지 수행하려면 cluster role 권한을 부여받아야 하는데 그건 좀 부담스러워서
대신 Deployment Descriptor 를 git에 올리고 argoCD에서 sync하는 전략을 택했습니다.
그 관련해 성공한 코드를 참조삼아 올립니다:
```
      stage('Deploy'){
        try {
            sh "git config --global user.email 'anabaral@gmail.com'"
            sh "git config --global user.name 'ChatAdmin'"
            sh "sed -i 's#\\(image: harbor.chatops.ga/app/controller\\):[0-9]\\{8\\}_[0-9]\\{6\\}#\\1:${this_time}#' doc/deploy/ctr-deploy.yaml "
          withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: "Gitea", 
                            usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASSWORD']]){
            sh "git add ${deploy_file}"
            sh "git commit -m 'auto image change by jenkins'"
            sh "git remote set-url origin https://${GIT_USER}:${GIT_PASSWORD}@gitea.chatops.ga/selee/controller.git"
            sh "git push origin HEAD:master"
          }
          //sh "kubectl set image -n app  deployment/controller controller=harbor.chatops.ga/app/controller:${this_time}"
        }catch(Exception e){
          echo e.toString()
          throw e
        }
      }
```
분명히 이것보다 더 간결하고 깔끔한 코드가 있을텐데, 검색으로 찾는데 실패해서 시행착오를 거쳐 만든 것이라 나중에 개선할 여지가 있습니다.







