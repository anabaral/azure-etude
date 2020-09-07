# 이것저것 확인

지금부터 볼 azure 환경은 내가 속한 조직에서 테스트용으로 부여해 준 것이라 새 계정을 생성했을 때 보게 되는 것과 다를 수 있음.

## vnet 확인

```
$ az network vnet list -o table -g 04226
Name        ResourceGroup    Location      NumSubnets    Prefixes     DnsServers    DDOSProtection    VMProtection
----------  ---------------  ------------  ------------  -----------  ------------  ----------------  --------------
04226-vnet  04226            koreacentral  1             10.2.0.0/24                False             False
```

## subnet 확인

```
$ az network vnet subnet list -g 04226 --vnet-name 04226-vnet -o table
AddressPrefix    Name     PrivateEndpointNetworkPolicies    PrivateLinkServiceNetworkPolicies    ProvisioningState    ResourceGroup
---------------  -------  --------------------------------  -----------------------------------  -------------------  ---------------
10.2.0.0/24      default  Enabled                           Enabled                              Succeeded            04226
```

## storage account 확인

```
# 먼저 이름만 보자
$ az storage account list | jq '.[].name'  # 필터링 안하면 엄청 양이 많음
...
"04226diag"
...
$ az storage account show -n 04226diag   # 내 것만 보자
```

