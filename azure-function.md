# Azure 에서 Function 을 만들어 써 보자.

## 목적: 정해진 시각에 정해진 Kubernetes Cluster를 중지

한 일:
- Azure Function App 추가 
- Function (python) 추가
  * Microsoft Visual Code (설치 및) 실행
  * 특정 디렉터리 지정해 Function 생성 및 디플로이
- Function에 필요한 패키지 추가
  * cd {function app 디렉터리}    ( 대략 function 코드 있는 디렉터리의 상위 디렉터리)
  * .\venv\Script\activate.bat 실행
  * 가상환경 프롬프트 적용된 상태에서 pip install 
    + pip install azure-identity
    + pip install azure-mgmt-containerservice
    + pip install azure-functions    (이건 이미 깔려있는 듯)
- Function code 작성
  * _init_.py  (코드를 분리해 작성해야 할 것 같았는데 별 생각 없어서 그냥 작성함)
  * 코드 내용:
  ```
  import datetime
  import logging
  import re

  import azure.functions as func
  from azure.identity import DefaultAzureCredential
  from azure.identity import ManagedIdentityCredential
  from azure.mgmt.containerservice import ContainerServiceClient


  def main(mytimer: func.TimerRequest) -> None:
      utc_timestamp = datetime.datetime.utcnow().replace(
          tzinfo=datetime.timezone.utc).isoformat()

      if mytimer.past_due:
          logging.info('The timer is past due!')


      # 여기서는 DefaultAzureCredential()을 썼지만 실제로는 Service Principal의 것을 써야 할 듯.
      containerservice_client = ContainerServiceClient(DefaultAzureCredential(), "<my-subscription-id>")

      AKS_NAME = 'aks-cluster-ChatOps'
      AKS_GROUP = 'RG-ChatOps'

      # 테스트 로직. 
      # 여러 클러스터들 중 내가 원하는 걸 찾아본다.
      it = containerservice_client.managed_clusters.list()
      for item in it:
          group = re.search('resourcegroups/([^/]+)/', item.id).group(1)
          name = re.search('/([^/]+)$', item.id).group(1)
          print(f'{group} {name}')
          if group == AKS_GROUP and name == AKS_NAME :
              break

      # 실제 실행하고 싶은 로직.
      # 원하는 클러스터를 중지함.
      aks = containerservice_client.managed_clusters.get(AKS_GROUP, AKS_NAME )
      aks_state_code = aks.power_state.as_dict()['code'] 
      logging.info(f'kubernetes cluster state = {aks_state_code}')
      if aks_state_code != 'Stopped' :
          containerservice_client.managed_clusters.begin_stop(AKS_GROUP, AKS_NAME )

      logging.info('Python_timer trigger function ran at %s', utc_timestamp)

  ```
  * 로직은 단순함. 클러스터의 상태를 체크하여 'Stop' 상태가 아니면 Stop을 시도함.  
    (아마 나중에 개선의 여지가 있을 것)
  * 인증/권한에 권한 관련한 내용이 단촐한 것은 이 함수 앱 자체에 바로 권한을 부여할 생각이기 때문.  
    이론상 identity에 권한을 부여하면 이 함수 앱 자체가 권한을 가지므로 `DefaultAzureCredentials()` 만으로 동작이 가능해져야 맞음.

- 함수 앱에 권한 부여
  * (아직 성공 못했음)
  * azure portal 화면에서 해당 function app 으로 들어감
  * 'ID' 블레이드 선택
  * '시스템 할당 항목' 과 '사용자 할당 항목'이 있는데 '시스템 할당 항목'을 켬
  * 그러면 개체 ID가 활성화되고 [Azure 권한 할당] 이 가능해짐
  * Azure 권한 할당:
    + 이 단계에서 아직 답이 없음.
    + 현재 파악되는 요령은, JSON으로 확인했을 때 허용되는 Action에 다음이 포함되면 성공.
      - "Microsoft.ContainerService/managedClusters/stop"
      - "Microsoft.ContainerService/managedClusters/read"
    + 그런데 'Azure Kubernetes Service RBAC 클러스터 관리자' 역할의 경우, "Microsoft.ContainerService/managedClusters/*" 이 있는데 안됨. 뭐지?
    + 아 그럼 사용자 지정 권한을 만들면 되지 않나? 싶었는데..  
      Premium 급으로 구독을 업그레이드 해야 함. ㅠ_ㅠ


