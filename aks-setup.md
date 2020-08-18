# AKS setup 방법

사실 거의 대부분의 내용은 여기 참조 : https://docs.microsoft.com/ko-kr/azure/aks/kubernetes-walkthrough

## azure cli 설치
여러 방법이 있으나 가장 간단한(?) 방법을 사용해 보자.
```
$ curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

## 로그인
로그인 명령을 입력하면 브라우저로 2차인증을 하도록 유도됨.
```
$ az login
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code GSY5H99NS to authenticate.
```

## 그룹 만들기 
내가 받은 테스트용 계정에는 이미 04226 이라는 그룹이 있으므로 생략하지만 일단 명령어는 기억해 두자
```
$ az group create --name myresourcegroup --location koreacentral
```

## aks cluster 생성
명령은 간단히 실행된다. 나중에 node-count 변경하는 방법을 찾아봐야 할 텐데.. 
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

## kubectl 설치
kubectl 등을 설치해야 제어가 가능하니 설치. 이 때 배스천의 /usr/local/bin/ 을 건드려야 하니 sudo 필요.
```
$ sudo az aks install-cli
```

## kubectl 권한 셋업
config를 받아오는 명령이 따로 있다.
```
$ az aks get-credentials --resource-group 04226 --name myAKSCluster

$ kubectl config get-contexts
CURRENT   NAME           CLUSTER        AUTHINFO                         NAMESPACE
*         myAKSCluster   myAKSCluster   clusterUser_04226_myAKSCluster

$ kubectl get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-21253863-vmss000000   Ready    agent   15m   v1.16.13
```

## cluster 정보 얻기
이걸 실행하면 아까 클러스터 생성할 때 출력되었던 내용이 다시 찍힌다.
```
$ az aks show --resource-group 04226 --name myAKSCluster
```

## docker 설치
아까 했어야 했던 일인데, docker가 없는 걸 지금 깨달았다..
```
$ sudo apt install docker.io
Reading package lists... Done
(생략)
Setting up docker.io (19.03.6-0ubuntu1~18.04.1) ...
Adding group `docker' (GID 116) ...
Done.
Created symlink /etc/systemd/system/sockets.target.wants/docker.socket → /lib/systemd/system/docker.socket.
docker.service is a disabled or a static unit, not starting it.
Processing triggers for systemd (237-3ubuntu10.41) ...
Processing triggers for man-db (2.8.3-2ubuntu0.1) ...
Processing triggers for ureadahead (0.100.0-21) ...
```
서비스 시작은 수동으로 해야 하나봄. 뭐 배스천에서는 이미지 빌드만 하게 될테니까..

## 클러스터 업그레이드

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

## 클러스터 노드 개수 조정

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



## 클러스터 삭제
```
$ az aks delete --resource-group 04226 --name myAKSCluster 
Are you sure you want to perform this operation? (y/n): y
 - Running ..
```
