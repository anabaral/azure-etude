# Application Gateway 설정 시도

AWS에서의 ALB 와 같은 기능을 AKS에서 얻고 싶음. 
이를 위해 L7 수준이라고 알려져 있는 Application Gateway를 셋업하려 함.

AWS는 적당히 만들수 있었던 것 같은데 이건 좀 다르다..

처음엔 기존에 있는 AKS가 속한 vnet 안에 만들어보려고 시도.
```
$ az network application-gateway create -g 04226 -n myAG --vnet-name 04226-vnet
Deployment failed. Correlation ID: 15c9801b-6dd3-4a27-adcd-300822b270c5. {
  "error": {
    "code": "ApplicationGatewaySubnetCannotHaveOtherResources",
    "message": "Subnet /subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/04226-vnet/subnets/default cannot be used for application gateway /subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG since it has other resources deployed. Subnet used for application gateway can only have other application gateways.",
    "details": []
  }
}
```
실패.

혹시나 해서 기존 vnet에 서브넷을 만드는 것도 시도.
```
$ az network vnet subnet create --name myAGsubnet --vnet-name 04226-vnet -g 04226 --address-prefix 10.2.2.0/24
Subnet 'myAGsubnet' is not valid in virtual network '04226-vnet'.
```
역시 실패. (이건 좀 다른 이유때문인 듯. IP 부여가 달랐거나)


vnet을 만들어 보자.
```
$ az network vnet create --name 04226-ag-vnet -g 04226 --location koreacentral
{
  "newVNet": {
    "addressSpace": {
      "addressPrefixes": [
        "10.0.0.0/16"
      ]
    },
    "bgpCommunities": null,
    "ddosProtectionPlan": null,
    "dhcpOptions": {
      "dnsServers": []
    },
    "enableDdosProtection": false,
    "enableVmProtection": false,
    "etag": "W/\"c7fb05b1-2cec-4d34-bd91-9ebbc5eb0973\"",
    "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/04226-ag-vnet",
    "ipAllocations": null,
    "location": "koreacentral",
    "name": "04226-ag-vnet",
    "provisioningState": "Succeeded",
    "resourceGroup": "04226",
    "resourceGuid": "bbf5cc82-2c0f-4d34-939b-8742cd5f2322",
    "subnets": [],
    "tags": {},
    "type": "Microsoft.Network/virtualNetworks",
    "virtualNetworkPeerings": []
  }
}
$  az network vnet list -o table -g 04226
Name           ResourceGroup    Location      NumSubnets    Prefixes     DnsServers    DDOSProtection    VMProtection
-------------  ---------------  ------------  ------------  -----------  ------------  ----------------  --------------
04226-ag-vnet  04226            koreacentral  0             10.0.0.0/16                False             False
04226-vnet     04226            koreacentral  1             10.2.0.0/24                False             False
```
원래 있던 vnet과 새로 만든 vnet이 같이 보임.

이번엔 subnet 만들어줘야 함
```
$ az network vnet subnet create -g 04226 --vnet-name 04226-ag-vnet -n default  --address-prefix 10.0.2.0/24
{
  "addressPrefix": "10.0.2.0/24",
  "addressPrefixes": null,
  "delegations": [],
  "etag": "W/\"511e2973-e6f9-4e10-af0a-d44f3b2b2569\"",
  "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/04226-ag-vnet/subnets/default",
  "ipAllocations": null,
  "ipConfigurationProfiles": null,
  "ipConfigurations": null,
  "name": "default",
  "natGateway": null,
  "networkSecurityGroup": null,
  "privateEndpointNetworkPolicies": "Enabled",
  "privateEndpoints": null,
  "privateLinkServiceNetworkPolicies": "Enabled",
  "provisioningState": "Succeeded",
  "purpose": null,
  "resourceGroup": "04226",
  "resourceNavigationLinks": null,
  "routeTable": null,
  "serviceAssociationLinks": null,
  "serviceEndpointPolicies": null,
  "serviceEndpoints": null,
  "type": "Microsoft.Network/virtualNetworks/subnets"
}
```

그리고 AG 생성.
```
$ az network application-gateway create -g 04226 -n myAG --vnet-name 04226-ag-vnet
{- Finished ..
  "applicationGateway": {
    "authenticationCertificates": [],
    "backendAddressPools": [
      {
        "etag": "W/\"b8046de5-8157-49ee-aacc-818b3b8b0388\"",
        "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/backendAddressPools/appGatewayBackendPool",
        "name": "appGatewayBackendPool",
        "properties": {
          "backendAddresses": [],
          "provisioningState": "Succeeded",
          "requestRoutingRules": [
            {
              "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/requestRoutingRules/rule1",
              "resourceGroup": "04226"
            }
          ]
        },
        "resourceGroup": "04226",
        "type": "Microsoft.Network/applicationGateways/backendAddressPools"
      }
    ],
    "backendHttpSettingsCollection": [
      {
        "etag": "W/\"b8046de5-8157-49ee-aacc-818b3b8b0388\"",
        "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/backendHttpSettingsCollection/appGatewayBackendHttpSettings",
        "name": "appGatewayBackendHttpSettings",
        "properties": {
          "connectionDraining": {
            "drainTimeoutInSec": 1,
            "enabled": false
          },
          "cookieBasedAffinity": "Disabled",
          "pickHostNameFromBackendAddress": false,
          "port": 80,
          "protocol": "Http",
          "provisioningState": "Succeeded",
          "requestRoutingRules": [
            {
              "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/requestRoutingRules/rule1",
              "resourceGroup": "04226"
            }
          ],
          "requestTimeout": 30
        },
        "resourceGroup": "04226",
        "type": "Microsoft.Network/applicationGateways/backendHttpSettingsCollection"
      }
    ],
    "frontendIPConfigurations": [
      {
        "etag": "W/\"b8046de5-8157-49ee-aacc-818b3b8b0388\"",
        "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/frontendIPConfigurations/appGatewayFrontendIP",
        "name": "appGatewayFrontendIP",
        "properties": {
          "httpListeners": [
            {
              "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/httpListeners/appGatewayHttpListener",
              "resourceGroup": "04226"
            }
          ],
          "privateIPAddress": "10.0.2.6",
          "privateIPAllocationMethod": "Dynamic",
          "provisioningState": "Succeeded",
          "subnet": {
            "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/04226-ag-vnet/subnets/default",
            "resourceGroup": "04226"
          }
        },
        "resourceGroup": "04226",
        "type": "Microsoft.Network/applicationGateways/frontendIPConfigurations"
      }
    ],
    "frontendPorts": [
      {
        "etag": "W/\"b8046de5-8157-49ee-aacc-818b3b8b0388\"",
        "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/frontendPorts/appGatewayFrontendPort",
        "name": "appGatewayFrontendPort",
        "properties": {
          "httpListeners": [
            {
              "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/httpListeners/appGatewayHttpListener",
              "resourceGroup": "04226"
            }
          ],
          "port": 80,
          "provisioningState": "Succeeded"
        },
        "resourceGroup": "04226",
        "type": "Microsoft.Network/applicationGateways/frontendPorts"
      }
    ],
    "gatewayIPConfigurations": [
      {
        "etag": "W/\"b8046de5-8157-49ee-aacc-818b3b8b0388\"",
        "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/gatewayIPConfigurations/appGatewayFrontendIP",
        "name": "appGatewayFrontendIP",
        "properties": {
          "provisioningState": "Succeeded",
          "subnet": {
            "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/04226-ag-vnet/subnets/default",
            "resourceGroup": "04226"
          }
        },
        "resourceGroup": "04226",
        "type": "Microsoft.Network/applicationGateways/gatewayIPConfigurations"
      }
    ],
    "httpListeners": [
      {
        "etag": "W/\"b8046de5-8157-49ee-aacc-818b3b8b0388\"",
        "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/httpListeners/appGatewayHttpListener",
        "name": "appGatewayHttpListener",
        "properties": {
          "frontendIPConfiguration": {
            "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/frontendIPConfigurations/appGatewayFrontendIP",
            "resourceGroup": "04226"
          },
          "frontendPort": {
            "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/frontendPorts/appGatewayFrontendPort",
            "resourceGroup": "04226"
          },
          "hostNames": [],
          "protocol": "Http",
          "provisioningState": "Succeeded",
          "requestRoutingRules": [
            {
              "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/requestRoutingRules/rule1",
              "resourceGroup": "04226"
            }
          ],
          "requireServerNameIndication": false
        },
        "resourceGroup": "04226",
        "type": "Microsoft.Network/applicationGateways/httpListeners"
      }
    ],
    "operationalState": "Running",
    "probes": [],
    "provisioningState": "Succeeded",
    "redirectConfigurations": [],
    "requestRoutingRules": [
      {
        "etag": "W/\"b8046de5-8157-49ee-aacc-818b3b8b0388\"",
        "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/requestRoutingRules/rule1",
        "name": "rule1",
        "properties": {
          "backendAddressPool": {
            "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/backendAddressPools/appGatewayBackendPool",
            "resourceGroup": "04226"
          },
          "backendHttpSettings": {
            "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/backendHttpSettingsCollection/appGatewayBackendHttpSettings",
            "resourceGroup": "04226"
          },
          "httpListener": {
            "id": "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG/httpListeners/appGatewayHttpListener",
            "resourceGroup": "04226"
          },
          "provisioningState": "Succeeded",
          "ruleType": "Basic"
        },
        "resourceGroup": "04226",
        "type": "Microsoft.Network/applicationGateways/requestRoutingRules"
      }
    ],
    "resourceGuid": "e2f2b812-c38a-4cf7-a341-7e868d459baa",
    "rewriteRuleSets": [],
    "sku": {
      "capacity": 2,
      "name": "Standard_Medium",
      "tier": "Standard"
    },
    "sslCertificates": [],
    "urlPathMaps": []
  }
}
```

## 실패사례

잘못해서 vnet을 다른 region(e.g. eastus) 에 만들었을 경우에는 다음과 같은 에러가 떨어짐.
```
$ az network application-gateway create -g 04226 -n myAG --vnet-name 04226-ag-vnet
Deployment failed. Correlation ID: d644dbf7-dfdb-48e7-bc61-3036e7fb1ba5. {
  "error": {
    "code": "InvalidResourceReference",
    "message": "Resource /subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/04226-AG-VNET referenced by resource /subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG was not found. Please make sure that the referenced resource exists, and that both resources are in the same region.",
    "details": [
      {
        "code": "NotFound",
        "message": "Resource /subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/04226-AG-VNET not found."
      }
    ]
  }
}
```

혹시 subnet은 불필요하지 않을까 싶어 vnet만 만들고 거기 속하는 subnet을 만들지 않은 채로 생성하려 하면 다음 에러가 나옴.
$ az network application-gateway create -g 04226 -n myAG --vnet-name 04226-ag-vnet
Deployment failed. Correlation ID: 2e963bf5-564f-4f74-b819-1ccb0b9e30ba. {
  "error": {
    "code": "InvalidResourceReference",
    "message": "Resource /subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/04226-ag-vnet/subnets/default referenced by resource /subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/myAG was not found. Please make sure that the referenced resource exists, and that both resources are in the same region.",
    "details": []
  }
}

## 비고

이렇게 만들고 나서도 AG가 아직 public-ip 를 부여받지 못한 상태임.
가이드에는 이것 또한 만들어 주라고 했었는데.. 혹시 기본으로 생성되나 싶어 안 만들었었음..

현재는 배스천만 갖고 있음
```
$ az network public-ip list -g 04226 -o table
Name               ResourceGroup    Location      Zones    Address        AddressVersion    AllocationMethod    IdleTimeoutInMinutes    ProvisioningState
-----------------  ---------------  ------------  -------  -------------  ----------------  ------------------  ----------------------  -------------------
test-bastion-1-ip  04226            koreacentral           52.231.31.197  IPv4              Dynamic             4                       Succeeded
```

그러나 이렇게 하는 게 과연 맞나 하는 생각은 듬.
AKS 에서 ingress 만들때마다 수동으로 매핑해 줘야 하나 싶기도 하고.. 수동으로 매핑은 또 어떻게 하지?




