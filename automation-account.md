# automation account 가지고 뭔가 해 보기

회사에서 주어진 테스트 계정 가지고 한 번 해 보자.

명령어 참조는 다음 링크 참조함:  
https://docs.microsoft.com/en-us/cli/azure/ext/automation/automation/runbook?view=azure-cli-latest

```
C:\Users\rindon>az automation account
CommandNotFoundError: 'automation' is misspelled or not recognized by the system.
Still stuck? Run 'az --help' to view all commands or go to 'https://aka.ms/cli_ref' to learn more
```
명령이 없다는데?

일단 업그레이드를 해 보자.
```
C:\Users\rindon>az upgrade
This command is in preview. It may be changed/removed in a future release.
Your current Azure CLI version is 2.17.1. Latest version available is 2.19.1.
Please check the release notes first: https://docs.microsoft.com/cli/azure/release-notes-azure-cli
Do you want to continue? (Y/n):
```

좀더 찾아보니 2021-02 현재 확장판으로 제공되는 것 같음.
```
C:\Users\rindon>az extension list
[]

C:\Users\rindon>az extension list-available | findstr auto
    "name": "automation",

C:\Users\rindon>az extension add --name automation
 - Installing ..
The installed extension 'automation' is experimental and not covered by customer support. Please use with discretion.
```

자동계정을 create하는 명령이 있긴 한데 내겐 이미 portal.azure.com 화면에서 만들어 둔 자동계정이 있으므로 이걸 확인해 보자.
```
C:\Users\rindon>az automation account list -o table
Command group 'automation account' is experimental and under development. Reference and support levels: https://aka.ms/CLI_refstatus
CreationTime                      LastModifiedTime                  Location      Name        ResourceGroup
--------------------------------  --------------------------------  ------------  ----------  ---------------
2021-02-10T04:48:05.776666+00:00  2021-02-16T15:35:25.200000+00:00  koreacentral  auto-04226  04226
```

현재는 Runbook을 CLI에서 어떻게 추가하는 지 모르겠음. 명령이 없는 것 같기도 함.  
자동계정에 등록되어 있는 (import 해 둔) Runbook들을 확인해 보자.
```
C:\Users\rindon>az automation runbook list --automation-account-name auto-04226 -g 04226 -o table
Command group 'automation runbook' is experimental and under development. Reference and support levels: https://aka.ms/CLI_refstatus
CreationTime                      LastModifiedTime                  Location      LogActivityTrace    LogProgress    LogVerbose    Name                            ResourceGroup    RunbookType      State
--------------------------------  --------------------------------  ------------  ------------------  -------------  ------------  ------------------------------  ---------------  ---------------  ---------
2021-02-10T04:48:09.196666+00:00  2021-02-26T01:21:20.890000+00:00  koreacentral  0                   False          False         AzureAutomationTutorial         04226            GraphPowerShell  Published
2021-02-10T04:48:08.976666+00:00  2021-02-10T06:18:26.730000+00:00  koreacentral  0                   False          False         AzureAutomationTutorialPython2  04226            Python2          Published
2021-02-10T04:48:08.913333+00:00  2021-02-10T04:48:08.916666+00:00  koreacentral  0                   False          False         AzureAutomationTutorialScript   04226            PowerShell       Published
2021-02-26T02:28:17.210000+00:00  2021-02-26T02:28:17.210000+00:00  koreacentral  0                   False          False         StartAzureV2Vm                  04226            GraphPowerShell  New
```
위에 보면 가장 마지막의 Runbook은 방금전에 portal.azure.com 화면에서 import 하고 아직 publish 하지 않은 Runbook임. (상태가 New)

이걸 publish 해 보자.
```
C:\Users\rindon>az automation runbook publish --automation-account-name auto-04226 -g 04226 -n StartAzureV2Vm
Command group 'automation runbook' is experimental and under development. Reference and support levels: https://aka.ms/CLI_refstatus

C:\Users\rindon>az automation runbook list --automation-account-name auto-04226 -g 04226 -o table |  findstr StartAzureV2Vm
WARNING: Command group 'automation runbook' is experimental and under development. Reference and support levels: https://aka.ms/CLI_refstatus
2021-02-26T02:28:17.210000+00:00  2021-02-26T02:30:06.580000+00:00  koreacentral  0                   False          False         StartAzureV2Vm                  04226            GraphPowerShell  Published
# 이렇게 상태가 바뀜
```

Runbook을 실행해 보자.  
아래는 실수한 예인데 일단 명령어는 확인해 보겠음:
```
C:\Users\rindon>az automation runbook start --automation-account-name auto-04226 -g 04226 -n StartAzureV2Vm
Command group 'automation runbook' is experimental and under development. Reference and support levels: https://aka.ms/CLI_refstatus
{
  "creationTime": "2021-02-26T02:57:40.526666+00:00",
  "endTime": null,
  "exception": null,
  "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Automation/automationAccounts/auto-04226/jobs/d93a1dc6-ab06-4366-a5b6-fb58f8e3646c",
  "jobId": "769a1f58-c9d7-4a91-9db4-22e67a958a3b",
  "lastModifiedTime": "2021-02-26T02:57:40.526666+00:00",
  "lastStatusModifiedTime": "2021-02-26T02:57:40.526666+00:00",
  "name": "d93a1dc6-ab06-4366-a5b6-fb58f8e3646c",
  "parameters": {},
  "provisioningState": "Processing",
  "resourceGroup": "04226",
  "runOn": null,
  "runbook": {
    "name": "StartAzureV2Vm"
  },
  "startTime": null,
  "startedBy": null,
  "status": "New",
  "statusDetails": "None",
  "type": "Microsoft.Automation/AutomationAccounts/Jobs"
}
```
아무런 파라미터를 부여하지 않고 실행하면, 내가 제어할 수 있는 모든 VM을 다 기동하는 것 같다.  
내 경우에는 내가 속한 구독의 모든 VM을 기동 시도하는데, 내가 속한 테스트 환경이 한 구독을 여럿이 나눠 사용하는 성격이라 모든 VM을 다 시도하더라.
더 이상한 것은, 실패했다고 출력에 나오는데 기동이 된다.
```
# 출력
test-bastion-1 failed to start.  Error: 
...
```
위의 VM은 실제로는 구동이 되었음.

아무튼, 대상을 제대로 제한하려면 parameter를 주어야 한다.  
그리고 parameter 값도 신경써야 한다.  
```
# 잘못된 parameter 값 사례
C:\Users\rindon>az automation runbook start --automation-account-name auto-04226 -g 04226 -n StartAzureV2Vm --parameters ResourceGroupName=04226
C:\Users\rindon>az automation runbook start --automation-account-name auto-04226 -g 04226 -n StartAzureV2Vm --parameters ResourceGroupName="04226"
# 값이 잘 넘어간 것처럼 보이지만 (아마도 python에서) 숫자로 인지해서 '0' 을 빼고 실행되고 
# 결과적으로 '4226'이란 리소스 그룹 없는데요? 에러 발생

# 잘 실행한 예
C:\Users\rindon>az automation runbook start --automation-account-name auto-04226 -g 04226 -n StartAzureV2Vm --parameters ResourceGroupName='04226'
Command group 'automation runbook' is experimental and under development. Reference and support levels: https://aka.ms/CLI_refstatus
{
  "creationTime": "2021-02-26T05:10:12.056666+00:00",
  "endTime": null,
  "exception": null,
  "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Automation/automationAccounts/auto-04226/jobs/70fab187-d9f2-45b0-b2b8-f6b2db3c82c7",
  "jobId": "81041ec8-9631-4d7e-85ca-e2fdbf050bc7",
  "lastModifiedTime": "2021-02-26T05:10:12.056666+00:00",
  "lastStatusModifiedTime": "2021-02-26T05:10:12.056666+00:00",
  "name": "70fab187-d9f2-45b0-b2b8-f6b2db3c82c7",
  "parameters": {
    "ResourceGroupName": "'04226'"
  },
  "provisioningState": "Processing",
  "resourceGroup": "04226",
  "runOn": null,
  "runbook": {
    "name": "StartAzureV2Vm"
  },
  "startTime": null,
  "startedBy": null,
  "status": "New",
  "statusDetails": "None",
  "type": "Microsoft.Automation/AutomationAccounts/Jobs"
}
```

Runbook의 실행 인스턴스는 'job'으로 분류되므로 이를 확인하려면 다음과 같이 함:
```
C:\Users\rindon>az automation job list --automation-account-name auto-04226 -g 04226 -o table
Command group 'automation job' is experimental and under development. Reference and support levels: https://aka.ms/CLI_refstatus
CreationTime                      EndTime                           JobId                                 LastModifiedTime                  Name                                  ProvisioningState    ResourceGroup    StartTime                         Status
--------------------------------  --------------------------------  ------------------------------------  --------------------------------  ------------------------------------  -------------------  ---------------  --------------------------------  ---------
2021-02-26T05:10:12.069000+00:00  2021-02-26T05:11:05.191675+00:00  81041ec8-9631-4d7e-85ca-e2fdbf050bc7  2021-02-26T05:11:05.191675+00:00  70fab187-d9f2-45b0-b2b8-f6b2db3c82c7  Succeeded            04226            2021-02-26T05:10:14.993508+00:00  Completed
2021-02-26T05:09:28.075930+00:00  2021-02-26T05:09:43.703297+00:00  d9a44999-78f8-4246-8bd5-811e11a94698  2021-02-26T05:09:43.703297+00:00  3747f725-b0e1-4e07-b42a-a5abfbf1449b  Succeeded            04226            2021-02-26T05:09:34.788228+00:00  Completed
2021-02-26T05:05:29.953981+00:00  2021-02-26T05:05:32.913445+00:00  062e6ff3-7b15-4683-bc00-fbbcf915e1e5  2021-02-26T05:05:32.913445+00:00  cf74c1e1-523e-443e-8f17-97bb7c654a2f  Succeeded            04226            2021-02-26T05:05:31.648356+00:00  Completed
2021-02-26T02:57:40.538007+00:00  2021-02-26T02:59:37.158544+00:00  769a1f58-c9d7-4a91-9db4-22e67a958a3b  2021-02-26T02:59:37.158544+00:00  d93a1dc6-ab06-4366-a5b6-fb58f8e3646c  Succeeded            04226            2021-02-26T02:57:47.615562+00:00  Completed
2021-02-10T06:18:32.609512+00:00  2021-02-10T06:18:36.757735+00:00  30b3c31e-81e5-465b-abbc-9a4ba08e58ce  2021-02-10T06:18:36.757735+00:00  efe0d1c4-402a-499d-af94-7abb33bf32cc  Succeeded            04226            2021-02-10T06:18:33.472570+00:00  Completed
2021-02-10T06:15:36.207412+00:00  2021-02-10T06:15:57.821449+00:00  c0d64592-3291-413b-bf2c-0f2111354cf9  2021-02-10T06:15:57.821449+00:00  1450d84d-7f7f-40de-9b52-4de0999faff5  Succeeded            04226            2021-02-10T06:15:42.580417+00:00  Completed
```




