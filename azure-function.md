# Azure 에서 Function 을 만들어 써 보자.

## 목적: 정해진 시각에 정해진 Kubernetes Cluster를 중지

실행하고 싶은 코드:
```
pip install azure-mgmt-containerservice

import re
from azure.identity import DefaultAzureCredential
from azure.mgmt.containerservice import ContainerServiceClient

# 여기서는 DefaultAzureCredential()을 썼지만 실제로는 Service Principal의 것을 써야 할 듯.
containerservice_client = ContainerServiceClient(DefaultAzureCredential(), "<subscription-id-of-mine>") 

AKS_NAME = 'aks-04226'
AKS_GROUP = '04226'

# 테스트 로직. 
# 여러 클러스터들 중 내가 원하는 걸 찾아본다.
it = containerservice_client.managed_clusters.list()
for item in it:
  group = re.search('resourcegroups/([^/]+)/', item.id).group(1)
  name = re.search('/([^/]+)$', item.id).group(1)
  print(f'{group} {name}')
  if group == AKS_NAME and name == AKS_GROUP :
    break

# 실제 실행하고 싶은 로직.
# 원하는 클러스터를 중지함.
aks = containerservice_client.managed_clusters.get(AKS_NAME, AKS_GROUP)
aks_state_code = aks.power_state.as_dict()['code'] 
if aks_state_code != 'Stopped' :
  containerservice_client.managed_clusters.begin_stop(AKS_NAME, AKS_GROUP)
```

