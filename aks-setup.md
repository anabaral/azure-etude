# AKS setup 방법

여기서는 
- VM에 AKS용 배스천 역할을 할 수 있도록 셋업을 하고 
- AKS를 생성 및 몇 가지 정보를 확인하는

방법을 기술합니다.

사실 거의 대부분의 내용은 여기 참조 : https://docs.microsoft.com/ko-kr/azure/aks/kubernetes-walkthrough


## bastion setup

먼저 다음을 가정합니다:
- bastion 역할을 할 VM 은 있음
- OS는 Ubuntu
- 거기서 sudo 가능한 계정으로 로그인 했음

### azure cli 설치

여러 방법으로 설치 가능한데 두 가지 정도 소개

- 가장 간단한(?) 방법은
```
$ curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

- apt를 통해 관리하고 싶다면 
```
# Ubuntu 가정
$ sudo apt-get update
$ sudo apt-get install ca-certificates curl apt-transport-https lsb-release gnupg
$ curl -sL https://packages.microsoft.com/keys/microsoft.asc | \
           gpg --dearmor | \
           sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg > /dev/null
$ AZ_REPO=$(lsb_release -cs)
$ echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $AZ_REPO main" | \
       sudo tee /etc/apt/sources.list.d/azure-cli.list
$ sudo apt-get install azure-cli
```

### kubectl 설치

- 그냥 쉘로 설치하려면
```
$ curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
$ chmod +x ./kubectl
$ sudo mv ./kubectl /usr/local/bin/kubectl
```

- apt 로 관리하고 싶다면
```
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
$ sudo apt-get update
$ sudo apt-get install -y kubectl
```

### helm 설치

웬만한 오픈소스 어플리케이션 하나 이상 설치하려면 어차피 쓰게 되니까 설치하는 걸 권장
```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ sudo ./get_helm.sh
```

### docker 설치

컨테이너 이미지를 만들려면 필요하니까 설치하는 걸 권장.
```
$ sudo apt-get update
$ sudo apt-get remove docker docker-engine docker.io # 이미 깔려 있다면 지우기 위해
$ sudo apt install docker.io
$ sudo systemctl enable docker
$ sudo systemctl start docker
```

### Azure 로그인
로그인 명령을 입력하면 브라우저로 2차인증을 하도록 유도됩니다.
```
$ az login
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code GSY5H99NS to authenticate.
```

여기까지 하면 배스천에서 필요한 작업은 끝난 것 같습니다.

## 그룹 만들기 

내가 쓸 그룹을 만들 명령어. 상황에 따라 이미 있어 불필요할 지 모르나 일단 명령어는 기억해 둡시다.
```
$ az group create --name myresourcegroup --location koreacentral
```



## AKS 설치 및 설정

### 클러스터 생성
명령은 간단히 실행됩니다. 권한만 있다면..

그리고 메시지를 보면 키를 따로 저장해 두어야 할 필요도 있어 보이는데, 왜 키를 생성해야 하는지는 현 시점에선 모르겠다. AKS 노드에 직접 접근하기 위한 키인가?
```
$ az aks create --resource-group 04226 --name myAKSCluster --node-count 1 --enable-addons monitoring --generate-ssh-keys
SSH key files '/home/azureuser/.ssh/id_rsa' and '/home/azureuser/.ssh/id_rsa.pub' have been generated under ~/.ssh to allow SSH access to the VM. If using machines without permanent storage like Azure Cloud Shell without an attached file share, back up your keys to a safe location
{
  "aadProfile": null,
  "addonProfiles": {
    "KubeDashboard": {
      "config": null,
      "enabled": true,
      "identity": null
    },
    "omsagent": {
      "config": {
        "logAnalyticsWorkspaceResourceID": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/defaultresourcegroup-se/providers/microsoft.operationalinsights/workspaces/defaultworkspace-3ac347d8-a75f-4611-8ca8-161a69189283-se"
      },
      "enabled": true,
      "identity": null
    }
  },
  "agentPoolProfiles": [
    {
      "availabilityZones": null,
      "count": 1,
      "enableAutoScaling": null,
      "enableNodePublicIp": false,
      "maxCount": null,
      "maxPods": 110,
      "minCount": null,
      "mode": "System",
      "name": "nodepool1",
      "nodeLabels": {},
      "nodeTaints": null,
      "orchestratorVersion": "1.16.13",
      "osDiskSizeGb": 128,
      "osType": "Linux",
      "provisioningState": "Succeeded",
      "scaleSetEvictionPolicy": null,
      "scaleSetPriority": null,
      "spotMaxPrice": null,
      "tags": null,
      "type": "VirtualMachineScaleSets",
      "vmSize": "Standard_DS2_v2",
      "vnetSubnetId": null
    }
  ],
  "apiServerAccessProfile": null,
  "autoScalerProfile": null,
  "diskEncryptionSetId": null,
  "dnsPrefix": "myAKSClust-04226-3ac347",
  "enablePodSecurityPolicy": null,
  "enableRbac": true,
  "fqdn": "myaksclust-04226-3ac347-ce40ceb2.hcp.koreacentral.azmk8s.io",
  "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/04226/providers/Microsoft.ContainerService/managedClusters/myAKSCluster",
  "identity": null,
  "identityProfile": null,
  "kubernetesVersion": "1.16.13",
  "linuxProfile": {
    "adminUsername": "azureuser",
    "ssh": {
      "publicKeys": [
        {
          "keyData": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC0SfwvY/5LpwR2F2U33xW/QUC87e3UHgCGJ6GPIRcsSCkAdZvWq6bzCHGzu0xlu/3MbTFWC9bniwP7IjfHMHkHFHroZIOIgPyFAdeodw/ndgZE7gkeL7Yew9fAwe2UUKG8vKLgj9Z5m6N4Oyh7St0xy3ryenyoKu/HKnedKuzycj6622cC4N3IktHmZbIZNEAdECPLhT2aUm13z64R5iZCzYuKrH+2zNSvbKj0UDtEfC8FFnvaJ+ciP07AynIWQbxN3w2MpKE3KknWPabk16DlTw5NgIKKhSUaf+5QIDIqvgm+YmYVdLDepOLlKZJoFqBdeCS3yh5XRrJzJvZQLJ4B"
        }
      ]
    }
  },
  "location": "koreacentral",
  "maxAgentPools": 10,
  "name": "myAKSCluster",
  "networkProfile": {
    "dnsServiceIp": "10.0.0.10",
    "dockerBridgeCidr": "172.17.0.1/16",
    "loadBalancerProfile": {
      "allocatedOutboundPorts": null,
      "effectiveOutboundIps": [
        {
          "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/MC_04226_myAKSCluster_koreacentral/providers/Microsoft.Network/publicIPAddresses/02b18ee1-f88c-4c82-9dfe-72e29758f957",
          "resourceGroup": "MC_04226_myAKSCluster_koreacentral"
        }
      ],
      "idleTimeoutInMinutes": null,
      "managedOutboundIps": {
        "count": 1
      },
      "outboundIpPrefixes": null,
      "outboundIps": null
    },
    "loadBalancerSku": "Standard",
    "networkMode": null,
    "networkPlugin": "kubenet",
    "networkPolicy": null,
    "outboundType": "loadBalancer",
    "podCidr": "10.244.0.0/16",
    "serviceCidr": "10.0.0.0/16"
  },
  "nodeResourceGroup": "MC_04226_myAKSCluster_koreacentral",
  "privateFqdn": null,
  "provisioningState": "Succeeded",
  "resourceGroup": "04226",
  "servicePrincipalProfile": {
    "clientId": "40d26ffd-ce1f-4ea4-bc8b-7fe7e96a24ac",
    "secret": null
  },
  "sku": {
    "name": "Basic",
    "tier": "Free"
  },
  "tags": null,
  "type": "Microsoft.ContainerService/ManagedClusters",
  "windowsProfile": null
}
```


### kubectl 권한 셋업

클러스터로부터 필요한 권한과 config를 받아오는 명령이 따로 있습니다.
```
$ az aks get-credentials --resource-group 04226 --name myAKSCluster

$ kubectl config get-contexts
CURRENT   NAME           CLUSTER        AUTHINFO                         NAMESPACE
*         myAKSCluster   myAKSCluster   clusterUser_04226_myAKSCluster

$ kubectl get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-21253863-vmss000000   Ready    agent   15m   v1.16.13
```

### cluster 정보 얻기
이걸 실행하면 아까 클러스터 생성할 때 출력되었던 내용이 다시 찍힌다.
```
$ az aks show --resource-group 04226 --name myAKSCluster
```


### 클러스터 업그레이드

다음과 같이 업그레이드 가능 버전을 확인.
```
$ az aks get-upgrades --resource-group 04226 --name myAKSCluster --output table
Name     ResourceGroup    MasterVersion    Upgrades
-------  ---------------  ---------------  ----------------
default  04226            1.16.13           1.17.7, 1.17.9
```
업그레이드가 불가할 경우에는 에러가 난다고..

업그레이드는 이렇게 함.
```
$ az aks upgrade --resource-group 04226 --name myAKSCluster --kubernetes-version 1.17.7
- Running..
<클러스터 정보 출력>

$ az aks get-upgrades --resource-group 04226 --name myAKSCluster --output table   Name     ResourceGroup    MasterVersion    Upgrades
-------  ---------------  ---------------  ----------------------------------------
default  04226            1.17.7           1.17.9, 1.18.4(preview), 1.18.6(preview)
```
실제 해 보니 클러스터를 새로 생성하는 것보다 업그레이드가 (생성 직후에 하는 건데도) 훨씬 오래 걸림. (생성할 때 3분, 업그레이드 10분 초과)

한편 위와 보이듯이 현재 버전에 따라서 업그레이드 가능 버전이 다름.

이미 알고 있을 수 있지만 현재 버전 확인하는 방법은 이것도 있음
```
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.6", GitCommit:"dff82dc0de47299ab66c83c626e08b245ab19037", GitTreeState:"clean", BuildDate:"2020-07-15T16:58:53Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.7", GitCommit:"5737fe2e0b8e92698351a853b0d07f9c39b96736", GitTreeState:"clean", BuildDate:"2020-06-24T19:54:11Z", GoVersion:"go1.13.6", Compiler:"gc", Platform:"linux/amd64"}
```

### 클러스터 노드 개수 조정

azure 클러스터는 노드풀 이란 걸 가지고 있는데, 이게 AWS에서의 스케일링 그룹과 비슷한 역할을 합니다.
```
$ az aks nodepool scale -g 04226 --cluster-name myAKS -n nodepool1
...노드풀 정보 출력...
```
위에 노드 수를 명시하지 않았는데, 이럴 경우 기본값을 부여하는 것 같습니다. 여기서는 3개로 조정되었습니다.
명시하고 싶으면 별도 옵션을 사용합니다.
```
$ az aks nodepool scale -g 04226 --cluster-name myAKS -n nodepool1 --node-count 1
```
노드 수를 0으로 줄 수는 없습니다. AWS에서는 가능했고 비용 절감 측면에서 기대했는데 불가하네요. 
다만 노드 스케일이 AWS의 스케일링 그룹과 의미가 동일하지는 않으므로 다른 이 있는지도 모르겠습니다.
```
$ az aks nodepool scale -g 04226 --cluster-name myAKS -n nodepool1 --node-count 0
Operation failed with status: 'Bad Request'. Details: The value of parameter agentPoolProfile.count is invalid. Please see https://aka.ms/aks-naming-rules for more details.
$
```



### 클러스터 삭제
```
$ az aks delete --resource-group 04226 --name myAKSCluster 
Are you sure you want to perform this operation? (y/n): y
 - Running ..
```

