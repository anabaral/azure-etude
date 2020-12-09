# Azure 컨테이너 인스턴스 에서 앱 띄워보기

정말 기초 중의 기초만으로..

아래 링크에 사용되는 이미지를 기본으로 해서 조금 고쳐서 쓸 생각입니다.
https://github.com/anabaral/azure-etude/blob/master/aks-app.md 

아래 진행한 것들은 정확히는 시간 순서로 한 것은 아닙니다. 이렇게 했다가 에러가 나니 고치고 하는 과정을 거쳤기에 이미지 만들기와 컨테이너 만들기가 번갈아서 진행된 측면이 있습니다.

## 컨테이너 이미지 만들기

### Dockerfile

Dockerfile 제일 밑에 다음이 필요합니다. 
```
ENTRYPOINT [ "/bin/bash", "-ec", "npm start" ]
```

AKS에서는 Deployment 설정에 다음과 같이 포함되어 있던 것입니다.
```
spec:
  template:
    spec:
      containers:
      - command:
        - /bin/bash
        - -ec
        - npm start
```

### Build script

빌드 스크립트도 조정했습니다.
```
$ cat build.sh
az acr login -n tuna01

NAME=node-ctr
VERSION=0.1
docker build -t ${NAME}:${VERSION} .
if [ $? -ne 0 ]; then echo 'fail' && exit 1 ; fi
docker tag ${NAME}:${VERSION} tuna01.azurecr.io/${NAME}:${VERSION}
if [ $? -ne 0 ]; then echo 'fail' && exit 1 ; fi
docker push tuna01.azurecr.io/${NAME}:${VERSION}
if [ $? -ne 0 ]; then echo 'fail' && exit 1 ; fi
```
AKS 때와 비교하면 이름과 버전 바꾼 정도입니다.

### 프로그램 소스

nodejs 소스도 건드려야 합니다.  
AKS에서는 별 생각 없이 같이 뜬 mongodb를 방치했는데, 여기선 컨테이너 하나만 띄울 거라 nodejs가 초기화하면서 mongodb를 찾으면 에러납니다.

```
//아래 부분들을 찾아 주석처리
//__var mongoose = require('mongoose');                               // mongoose for mongodb
//__var database = require('./config/database');                      // load the database config
//__mongoose.connect(database.remoteUrl);                         // connect to local MongoDB instance. A remoteUrl is also available (modulus.io)
```

만약 mongodb가 필요하다면 (즉 저처럼 봇 접속에만 한정하지 않고 기존 소스의 로직을 적극 활용한다면) 소스를 건드리지 말고 다른 방향으로 셋업을 해주어야겠죠. 
- mongodb 를 따로 시동 (또다른 컨테이너로 띄우든 관리형 서비스로 띄우든)
- Dockerfile 을 수정해서 AKS에서 환경변수 설정한 것을 옮기고 값 조정. 하나만 예로 들자면:
  ```
  ENV DATABASE_HOST mynode-mongodb
  ```


## 컨테이너 만들기

처음엔 웹 화면으로 진행했는데 결국 명령어로 실행했습니다:
```
$ az container create -g rg-tuna -n ctr-tuna-03 --image tuna01.azurecr.io/node-ctr:0.1  --ip-address Public --ports 3000 --environment-variables 'PORT'='3000'
```

위에서 ```--ip-address Public``` 주지 않으면 그 뒤의 포트 관련 설정이 다 안 먹힙니다.

그리고 웹 화면으로 진행할 때 경험한 것이긴 한데 IP address 를 공용(public)으로 주지 않으면 Virtual Network 및 Subnet 을 (아마도 새로 만들어서) 구성해야 하는데 거기에
무언가 숨은 제약이 있어서 배포 시점에 실패합니다. 사유는 대략:
- ServiceUnavailable 이라고 상태가 나오고
- 메시지는 "요청된 리소스를 지금은 'koreacentral' 위치에서 사용할 수 없습니다. 다른 리소스 요청으로 또는 다른 위치에서 다시 시도하세요. 요청된 리소스: '1' CPU '1.5'GB 메모리 'Linux' OS 가상 네트워크""
  라고만 나옵니다.

처음엔 리소스 증가 신청을 해야 하나, 혹은 virtual network을 기존의 것을 쓰면 안되나, 등등 여러 모로 시행착오를 했는데  
결국 공용 IP주소 선택으로 넘어갔습니다.  
아직도 private address 선택시 안되는 정확한 원인을 모릅니다.

말끔하게 지우고 CLI로 다시 시도해 봤는데 역시 에러가 납니다:
```
$ az network vnet create -g rg-tuna -n vn-ctr-tuna-01 --address-prefix 10.168.0.0/16 \
    --subnet-name sub-ctr-tuna-01 --subnet-prefix 10.168.0.0/24
$ az container create -g rg-tuna -n ctr-tuna-01 --image tuna01.azurecr.io/node-ctr:0.1  \
    --location koreacentral --vnet vn-ctr-tuna-01 --subnet sub-ctr-tuna-01 \
    --ip-address Private --ports 3000 --environment-variables 'PORT'='3000'
UnknownError: The requested resource is not available in the location 'koreacentral' at this moment. Please retry with a different resource request or in another location. Resource requested: '1' CPU '1.5' GB memory 'Linux' OS virtual network
```




## Troubleshooting

### 컨테이너 생성 시 만들어졌던 가상네트워크 지우기

앞에서 컨테이너 인스턴스를 만들 때 IP address 를 private 로 하려 시도했었는데 그 때 가상 네트워크(Virtual Network)과 서브넷(Subnet)을 생성해야 했습니다.  
문제는 이걸 삭제하려고 하니 자꾸 이상한 에러가 나면서 삭제가 안됩니다.
* 처음에 웹 콘솔에서 서브넷 속성을 변경하려 할 때 나던 에러:
  ```
  서브넷 sub-ctr-tuna-01이(가) 사용 중이므로 업데이트될 수 없습니다.
  ```
* 아예 서브넷을 삭제하려 할 때 나던 에러:
  ```
  Subnet sub-ctr-tuna-01 is in use by /subscriptions/ca36f0eb-8274-4e37-a0ed-f78461bad6d2/resourceGroups/rg-tuna/providers/Microsoft.Network/networkProfiles/ctr-tuna-01-networkProfile/containerNetworkInterfaceConfigurations/eth0/ipConfigurations/ipconfigprofile1 
  and cannot be deleted. In order to delete the subnet, delete all the resources within the subnet. 
  See aka.ms/deletesubnet.
  ```
  ...네트워크 프로필이 어떻다고?
* 가상 네트워크 째로 없애려 해도 비슷한 에러가 뜸:
  ```
  Subnet sub-ctr-tuna-01 is in use by /subscriptions/ca36f0eb-8274-4e37-a0ed-f78461bad6d2/resourceGroups/rg-tuna/providers/Microsoft.Network/networkProfiles/ctr-tuna-01-networkProfile/containerNetworkInterfaceConfigurations/eth0/ipConfigurations/ipconfigprofile1 
  and cannot be deleted. In order to delete the subnet, delete all the resources within the subnet. 
  See aka.ms/deletesubnet.
  ```
  명령줄에서 실행해도 비슷.

구글신에게 물어보니, 똑같이 네트워크 프로필을 지워야 한다고 합니다. 다만 답해준 사람이 친절하게도 방법을 대략 알려줬습니다.
```
$ az network profile list -g rg-tuna -o table   # 그런 게 있다는 걸 확인!
Location      Name                        ProvisioningState    ResourceGroup    ResourceGuid
------------  --------------------------  -------------------  ---------------  ------------------------------------
koreacentral  ctr-tuna-01-networkProfile  Succeeded            rg-tuna          aa2dd796-7649-42be-b171-11628e028360
koreacentral  ctr-tuna-1-networkProfile   Succeeded            rg-tuna          c9f9dc74-276a-493c-a9f8-a0ec1cbbbb91
koreacentral  xxx-networkProfile          Succeeded            rg-tuna          09546f7f-be27-4f09-83f3-a6536183dd76
```

이것들이 뭔지는 잘 모르겠으나, 이름의 접두사 'ctr-tuna-01', 'ctr-tuna-1', 'xxx' 들은 
컨테이너 인스턴스 생성 중에 만들었던 vnet 들의 이름입니다. 아무튼 지금은 쓰지도 않고 필요없으니 다 지워버리기로.

```
$ az network profile delete -g rg-tuna -n xxx-networkProfile
Are you sure you want to perform this operation? (y/n): y
$ az network profile delete -g rg-tuna -n ctr-tuna-1-networkProfile
Are you sure you want to perform this operation? (y/n): y
$ az network profile delete -g rg-tuna -n ctr-tuna-01-networkProfile
Are you sure you want to perform this operation? (y/n): y
```

이제 시원하게 다 날려버립시다:
```
$ az network vnet list -o table
Name               ResourceGroup                     Location      NumSubnets    Prefixes                          DnsServers    DDOSProtection    VMProtection
-----------------  --------------------------------  ------------  ------------  --------------------------------  ------------  ----------------  --------------
aks-vnet-11324664  MC_rg-tuna_aks-tuna_koreacentral  koreacentral  1             10.0.0.0/8                                      False             False
vn-ctr-tuna        rg-tuna                           koreacentral  1             192.168.128.0/18, 192.169.0.0/16                False             False
vn-tuna            rg-tuna                           koreacentral  2             192.168.0.0/16, 100.64.0.0/16                   False             False
$ az network vnet delete -g rg-tuna -n vn-ctr-tuna
```
다른 것들은 아직 쓰는 거니 지우지 말도록 주의합니다.
