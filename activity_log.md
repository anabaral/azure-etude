# Azure Activity log 을 접근하는 방법

## CLI 접근


## python 접근

다음과 같이 접근해 보았음.

```
import datetime
from azure.mgmt.monitor import MonitorManagementClient
from azure.identity import DefaultAzureCredential


subscription_id = "3ac347d8-xxxx-xxxx-xxxx-161a69189283"
resource_group_name = "04226"
vm_name = "04226-test-vm"


resource_id = (
    "subscriptions/{}/"
    "resourceGroups/{}/"
    "providers/Microsoft.Compute/virtualMachines/{}"
).format(subscription_id, resource_group_name, vm_name) # 괄호를 쓰면 역슬래시 없이도 긴 문장을 쪼개 쓸 수 있었네..
credentials = DefaultAzureCredential()
al_client = MonitorManagementClient(
    credentials,
    subscription_id
)
```

이렇게 준비를 하고 나서 가상머신을 지정한 다음을 실행해 보면
```
>>> log = al_client.activity_logs.list(filter="eventtimestamp ge '2021-03-20t04:36:37.6407898 z' and "
...     " resourceUri eq '/subscriptions/3ac347d8-xxxx-xxxx-xxxx-161a69189283/resourceGroups/04226/providers/Microsoft.Compute/virtualMachines/04226-test-vm'", 
...     select=None)
>>> event_data = next(log)
>>> print(event_data)  # 아래 출력은 한 줄로 나오는데 정리했음
{
  "additional_properties": {
    "channels": "Operation"
  },
  "authorization": "<azure.mgmt.monitor.v2015_04_01.models._models_py3.SenderAuthorization object at 0x000002D947264E50>",
  "claims": {
    "aud": "https://management.core.windows.net/",
    "iss": "https://sts.windows.net/458503d6-xxxx-xxxx-xxxx-35ad3a0aa1f3/",
    "iat": "1619052190",
    "nbf": "1619052190",
    "exp": "1619056090",
    "http://schemas.microsoft.com/claims/authnclassreference": "1",
    "aio": "AVQAq/8TAAxxxxxxxx7cx/jcO4e71XXXXXXXXXXXXAHJlqPMCYq3mS02NcTd7cgnzfk0qIy/IOQulXXXXXXXXXXXXgOHOTFrG5DjcclRWqilrlqP1iha5v0=",
    "http://schemas.microsoft.com/claims/authnmethodsreferences": "pwd,mfa",
    "appid": "c44b4083-xxxx-xxxx-xxxx-974e53cbdf3c",
    "appidacr": "2",
    "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname": "이",
    "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname": "상은",
    "groups": "108909d1-xxxx-xxxx-xxxx-0f80877005ae",
    "ipaddr": "39.118.91.74",
    "name": "이상은",
    "http://schemas.microsoft.com/identity/claims/objectidentifier": "44a81845-xxxx-xxxx-xxxx-88277fc4c46b",
    "puid": "10032000CC3E01FE",
    "rh": "0.AVMA1gOFXXXXXXXXXXXXOgqh84NAS8SwO8FJtH2XTlPL3zxTALs.",
    "http://schemas.microsoft.com/identity/claims/scope": "user_impersonation",
    "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier": "qp1bUbjYDKmVp8hFTIg8l6O3TAkwGX2_7WTUU70gWN8",
    "http://schemas.microsoft.com/identity/claims/tenantid": "458503d6-xxxx-xxxx-xxxx-35ad3a0aa1f3",
    "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name": "ds04226@infrads.onmicrosoft.com",
    "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn": "ds04226@infrads.onmicrosoft.com",
    "uti": "bxxxxxxxxk-UaJ-BElMIAA",
    "ver": "1.0",
    "xms_tcdt": "1592975843"
  },
  "caller": "ds04226@infrads.onmicrosoft.com",
  "description": "",
  "id": "/subscriptions/3ac347d8-xxxx-xxxx-xxxx-161a69189283/resourceGroups/04226/providers/Microsoft.Compute/virtualMachines/04226-test-vm/events/57f58abd-xxxx-xxxx-xxxx-9e0aa794439f/ticks/637546514219973593",
  "event_data_id": "57f58abd-xxxx-xxxx-xxxx-9e0aa794439f",
  "correlation_id": "f3b980f2-xxxx-xxxx-xxxx-d57500ca8551",
  "event_name": "<azure.mgmt.monitor.v2015_04_01.models._models_py3.LocalizableString object at 0x000002D947264A00>",
  "category": "<azure.mgmt.monitor.v2015_04_01.models._models_py3.LocalizableString object at 0x000002D947B12F10>",
  "http_request": None,
  "level": "Informational",
  "resource_group_name": "04226",
  "resource_provider_name": "<azure.mgmt.monitor.v2015_04_01.models._models_py3.LocalizableString object at 0x000002D947B12820>",
  "resource_id": "/subscriptions/3ac347d8-xxxx-xxxx-xxxx-161a69189283/resourceGroups/04226/providers/Microsoft.Compute/virtualMachines/04226-test-vm",
  "resource_type": "<azure.mgmt.monitor.v2015_04_01.models._models_py3.LocalizableString object at 0x000002D947B12D90>",
  "operation_id": "70df015e-f3cc-4d4b-88c9-caeee1c2f0db",
  "operation_name": "<azure.mgmt.monitor.v2015_04_01.models._models_py3.LocalizableString object at 0x000002D947B125E0>",
  "properties": {
    "eventCategory": "Administrative",
    "entity": "/subscriptions/3ac347d8-xxxx-xxxx-xxxx-161a69189283/resourceGroups/04226/providers/Microsoft.Compute/virtualMachines/04226-test-vm",
    "message": "Microsoft.Compute/virtualMachines/deallocate/action",
    "hierarchy": "3ac347d8-xxxx-xxxx-xxxx-161a69189283"
  },
  "status": <azure.mgmt.monitor.v2015_04_01.models._models_py3.LocalizableString object at 0x000002D947B12430>,
  "sub_status": <azure.mgmt.monitor.v2015_04_01.models._models_py3.LocalizableString object at 0x000002D947B12EE0>,
  "event_timestamp": datetime.datetime(2021, 4, 22, 1, 23, 41, 997359, tzinfo=<isodate.tzinfo.Utc object at 0x000002D946C17190>),
  "submission_timestamp": datetime.datetime(2021, 4, 22, 1, 25, 4, 129435, tzinfo=<isodate.tzinfo.Utc object at 0x000002D946C17190>),
  "subscription_id": "3ac347d8-xxxx-xxxx-xxxx-161a69189283",
  "tenant_id": "458503d6-xxxx-xxxx-xxxx-35ad3a0aa1f3"
}
```
이렇게 나옵니다. (아이디에 해당하는 내용은 일부 XXXX 처리했음)

조금 내용을 더 들여다 보면
```
>>> event_data.category.value
'Administrative'
>>> event_data.resource_type.value
'Microsoft.Compute/virtualMachines'
>>> event_data.operation_name.value
'Microsoft.Compute/virtualMachines/deallocate/action'
>>> event_data.event_name.value
'EndRequest'
>>> event_data.status.value
'Succeeded'
>>> event_data.sub_status.value
''
```
이 정도..
