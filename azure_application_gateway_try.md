# AKS 에서 Application gateway 설정해 보기

다음 링크를 참조함:
https://docs.microsoft.com/ko-kr/azure/developer/terraform/create-k8s-cluster-with-aks-applicationgateway-ingress

이 링크가 더 요긴함:
https://zerobig-k8s.tistory.com/55

미리 언급하는데 아래 나오는 내용대로 따라가면 새로운 AKS 클러스터가 생성됩니다. 기존의 AKS 클러스터에 붙이기를 원했었기에
조금 황당했는데.. 아마 몇 가지 수정이 필요할 것 같습니다.

이게 배스천과 Cloud shell 에서의 경과가 약간 다른데, 알고 보니 배스천에 설치되는 terraform 버전 문제였음. 아래 상세히 나옵니다.

먼저 준비:
```
# 배스천에서 쉽게 설치하는 방법
$ sudo snap install terraform  # 이게 설치되는 최신 버전이 2020-08-20 현재 v0.11 대 버전
Download snap "terraform" (216) from channel "stable"           53%  139kB/s 1m40s

# 최신 버전을 설치하려면
$ curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
$ sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
$ sudo apt-get update && sudo apt-get install terraform
# 그런데 이 terraform 은 이상하게도 sudo로만 실행이 됨. ===> 이건 재로그인하면 언제 그랬냐는 듯이 해결됨. PATH 문제?
```

왜 테라폼을 쓰는 지 모르겠지만.. 아래 작성되는 파일들을 보면 따로 손으로 하려면 해야 할 게 많아서일지도 모르겠습니다.

그리고 필수적인 것은 아니지만 복붙을 편하게 하기 위한 스크립팅에 사용하게 jq를 설치
```
$ sudo snap install jq
```


## 파일 작성

```
$ vi main.tf
provider "azurerm" {
  # The "feature" block is required for AzureRM provider 2.x.
  # If you are using version 1.x, the "features" block is not allowed.
  version = "~>2.0"
  features {}
}

terraform {
    backend "azurerm" {}
}
```

```
$ vi variables.tf
variable "resource_group_name" {
  description = "Name of the resource group."
}

variable "location" {
  description = "Location of the cluster."
}

variable "aks_service_principal_app_id" {
  description = "Application ID/Client ID  of the service principal. Used by AKS to manage AKS related resources on Azure like vms, subnets."
}

variable "aks_service_principal_client_secret" {
  description = "Secret of the service principal. Used by AKS to manage Azure."
}

variable "aks_service_principal_object_id" {
  description = "Object ID of the service principal."
}

variable "virtual_network_name" {
  description = "Virtual network name"
  default     = "aksVirtualNetwork"
}

variable "virtual_network_address_prefix" {
  description = "Containers DNS server IP address."
  default     = "15.0.0.0/8"
}

variable "aks_subnet_name" {
  description = "AKS Subnet Name."
  default     = "kubesubnet"
}

variable "aks_subnet_address_prefix" {
  description = "Containers DNS server IP address."
  default     = "15.0.0.0/16"
}

variable "app_gateway_subnet_address_prefix" {
  description = "Containers DNS server IP address."
  default     = "15.1.0.0/16"
}

variable "app_gateway_name" {
  description = "Name of the Application Gateway."
  default = "ApplicationGateway1"
}

variable "app_gateway_sku" {
  description = "Name of the Application Gateway SKU."
  default = "Standard_v2"
}

variable "app_gateway_tier" {
  description = "Tier of the Application Gateway SKU."
  default = "Standard_v2"
}

variable "aks_name" {
  description = "Name of the AKS cluster."
  default     = "aks-cluster1"
}
variable "aks_dns_prefix" {
  description = "Optional DNS prefix to use with hosted Kubernetes API server FQDN."
  default     = "aks"
}

variable "aks_agent_os_disk_size" {
  description = "Disk size (in GB) to provision for each of the agent pool nodes. This value ranges from 0 to 1023. Specifying 0 applies the default disk size for that agentVMSize."
  default     = 40
}

variable "aks_agent_count" {
  description = "The number of agent nodes for the cluster."
  default     = 3
}

variable "aks_agent_vm_size" {
  description = "The size of the Virtual Machine."
  default     = "Standard_D3_v2"
}

variable "kubernetes_version" {
  description = "The version of Kubernetes."
  default     = "1.11.5"
}

variable "aks_service_cidr" {
  description = "A CIDR notation IP range from which to assign service cluster IPs."
  default     = "10.0.0.0/16"
}

variable "aks_dns_service_ip" {
  description = "Containers DNS server IP address."
  default     = "10.0.0.10"
}

variable "aks_docker_bridge_cidr" {
  description = "A CIDR notation IP for Docker bridge."
  default     = "172.17.0.1/16"
}

variable "aks_enable_rbac" {
  description = "Enable RBAC on the AKS cluster. Defaults to false."
  default     = "false"
}

variable "vm_user_name" {
  description = "User name for the VM"
  default     = "vmuser1"
}

variable "public_ssh_key_path" {
  description = "Public key path for SSH."
  default     = "~/.ssh/id_rsa.pub"
}

variable "tags" {
  type = map(string)

  default = {
    source = "terraform"
  }
}
```

```
$ vi resources.tf
# # Locals block for hardcoded names.
locals {
    backend_address_pool_name      = "${azurerm_virtual_network.test.name}-beap"
    frontend_port_name             = "${azurerm_virtual_network.test.name}-feport"
    frontend_ip_configuration_name = "${azurerm_virtual_network.test.name}-feip"
    http_setting_name              = "${azurerm_virtual_network.test.name}-be-htst"
    listener_name                  = "${azurerm_virtual_network.test.name}-httplstn"
    request_routing_rule_name      = "${azurerm_virtual_network.test.name}-rqrt"
    app_gateway_subnet_name = "appgwsubnet"
}

data "azurerm_resource_group" "rg" {
  name = var.resource_group_name
}

# User Assigned Identities
resource "azurerm_user_assigned_identity" "testIdentity" {
  resource_group_name = data.azurerm_resource_group.rg.name
  location            = data.azurerm_resource_group.rg.location

  name = "identity1"

  tags = var.tags
}

resource "azurerm_virtual_network" "test" {
  name                = var.virtual_network_name
  location            = data.azurerm_resource_group.rg.location
  resource_group_name = data.azurerm_resource_group.rg.name
  address_space       = [var.virtual_network_address_prefix]

  subnet {
    name           = var.aks_subnet_name
    address_prefix = var.aks_subnet_address_prefix
  }

  subnet {
    name           = "appgwsubnet"
    address_prefix = var.app_gateway_subnet_address_prefix
  }

  tags = var.tags
}

data "azurerm_subnet" "kubesubnet" {
  name                 = var.aks_subnet_name
  virtual_network_name = azurerm_virtual_network.test.name
  resource_group_name  = data.azurerm_resource_group.rg.name
}

data "azurerm_subnet" "appgwsubnet" {
  name                 = "appgwsubnet"
  virtual_network_name = azurerm_virtual_network.test.name
  resource_group_name  = data.azurerm_resource_group.rg.name
}

# Public Ip
resource "azurerm_public_ip" "test" {
  name                         = "publicIp1"
  location                     = data.azurerm_resource_group.rg.location
  resource_group_name          = data.azurerm_resource_group.rg.name
  allocation_method            = "Static"
  sku                          = "Standard"

  tags = var.tags
}

resource "azurerm_application_gateway" "network" {
  name                = var.app_gateway_name
  resource_group_name = data.azurerm_resource_group.rg.name
  location            = data.azurerm_resource_group.rg.location

  sku {
    name     = var.app_gateway_sku
    tier     = "Standard_v2"
    capacity = 2
  }

  gateway_ip_configuration {
    name      = "appGatewayIpConfig"
    subnet_id = data.azurerm_subnet.appgwsubnet.id
  }

  frontend_port {
    name = local.frontend_port_name
    port = 80
  }

  frontend_port {
    name = "httpsPort"
    port = 443
  }

  frontend_ip_configuration {
    name                 = local.frontend_ip_configuration_name
    public_ip_address_id = azurerm_public_ip.test.id
  }

  backend_address_pool {
    name = local.backend_address_pool_name
  }

  backend_http_settings {
    name                  = local.http_setting_name
    cookie_based_affinity = "Disabled"
    port                  = 80
    protocol              = "Http"
    request_timeout       = 1
  }

  http_listener {
    name                           = local.listener_name
    frontend_ip_configuration_name = local.frontend_ip_configuration_name
    frontend_port_name             = local.frontend_port_name
    protocol                       = "Http"
  }

  request_routing_rule {
    name                       = local.request_routing_rule_name
    rule_type                  = "Basic"
    http_listener_name         = local.listener_name
    backend_address_pool_name  = local.backend_address_pool_name
    backend_http_settings_name = local.http_setting_name
  }

  tags = var.tags

  depends_on = [azurerm_virtual_network.test, azurerm_public_ip.test]
}

resource "azurerm_role_assignment" "ra1" {
  scope                = data.azurerm_subnet.kubesubnet.id
  role_definition_name = "Network Contributor"
  principal_id         = var.aks_service_principal_object_id

  depends_on = [azurerm_virtual_network.test]
}

resource "azurerm_role_assignment" "ra2" {
  scope                = azurerm_user_assigned_identity.testIdentity.id
  role_definition_name = "Managed Identity Operator"
  principal_id         = var.aks_service_principal_object_id
  depends_on           = [azurerm_user_assigned_identity.testIdentity]
}

resource "azurerm_role_assignment" "ra3" {
  scope                = azurerm_application_gateway.network.id
  role_definition_name = "Contributor"
  principal_id         = azurerm_user_assigned_identity.testIdentity.principal_id
  depends_on           = [azurerm_user_assigned_identity.testIdentity, azurerm_application_gateway.network]
}

resource "azurerm_role_assignment" "ra4" {
  scope                = data.azurerm_resource_group.rg.id
  role_definition_name = "Reader"
  principal_id         = azurerm_user_assigned_identity.testIdentity.principal_id
  depends_on           = [azurerm_user_assigned_identity.testIdentity, azurerm_application_gateway.network]
}

resource "azurerm_kubernetes_cluster" "k8s" {
  name       = var.aks_name
  location   = data.azurerm_resource_group.rg.location
  dns_prefix = var.aks_dns_prefix

  resource_group_name = data.azurerm_resource_group.rg.name

  linux_profile {
    admin_username = var.vm_user_name

    ssh_key {
      key_data = file(var.public_ssh_key_path)
    }
  }

  addon_profile {
    http_application_routing {
      enabled = false
    }
  }

  default_node_pool {
    name            = "agentpool"
    node_count      = var.aks_agent_count
    vm_size         = var.aks_agent_vm_size
    os_disk_size_gb = var.aks_agent_os_disk_size
    vnet_subnet_id  = data.azurerm_subnet.kubesubnet.id
  }

  service_principal {
    client_id     = var.aks_service_principal_app_id
    client_secret = var.aks_service_principal_client_secret
  }

  network_profile {
    network_plugin     = "azure"
    dns_service_ip     = var.aks_dns_service_ip
    docker_bridge_cidr = var.aks_docker_bridge_cidr
    service_cidr       = var.aks_service_cidr
  }

  depends_on = [azurerm_virtual_network.test, azurerm_application_gateway.network]
  tags       = var.tags
}
```

```
$ vi output.tf
output "client_key" {
    value = azurerm_kubernetes_cluster.k8s.kube_config.0.client_key
}

output "client_certificate" {
    value = azurerm_kubernetes_cluster.k8s.kube_config.0.client_certificate
}

output "cluster_ca_certificate" {
    value = azurerm_kubernetes_cluster.k8s.kube_config.0.cluster_ca_certificate
}

output "cluster_username" {
    value = azurerm_kubernetes_cluster.k8s.kube_config.0.username
}

output "cluster_password" {
    value = azurerm_kubernetes_cluster.k8s.kube_config.0.password
}

output "kube_config" {
    value = azurerm_kubernetes_cluster.k8s.kube_config_raw
}

output "host" {
    value = azurerm_kubernetes_cluster.k8s.kube_config.0.host
}

output "identity_resource_id" {
    value = azurerm_user_assigned_identity.testIdentity.id
}

output "identity_client_id" {
    value = azurerm_user_assigned_identity.testIdentity.client_id
}
```

스토리지 계정 및 key1 값을 알아내어 아래에 적용
```
$ ACCOUNT_KEY=$(az storage account keys list --resource-group 04226 --account-name 04226diag --query [0].value -o tsv)
$ az storage container create -n tfstate --account-name 04226diag --account-key $ACCOUNT_KEY
{
  "created": true
}
```


## 테라폼 초기화 시도 
(참조: https://docs.microsoft.com/ko-kr/azure/aks/update-credentials )

Service Principal 관련 정보를 먼저 얻고 이를 이용해 초기화함 (Service Principal은 한글로 서비스 주체 라고 부름)
다음은 
```
$ SUBSCRIPTION=$(az account show --query id -o tsv)
$ az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/${SUBSCRIPTION}"
Creating a role assignment under the scope of "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283"
  Retrying role assignment creation: 1/36
{
  "appId": "81de2f9f-a0f9-4c72-8846-e3ce70a29511",
  "displayName": "azure-cli-2020-08-20-12-32-01",
  "name": "http://azure-cli-2020-08-20-12-32-01",
  "password": "xn12zT.....................fXvIW0",
  "tenant": "458503d6-ae80-47cb-bc1e-35ad3a0aa1f3"
}
# 위의 displayName 을 이용해
$ OBJECT_ID=$(az ad sp list --display-name azure-cli-2020-08-20-12-32-01 | jq -r '.[].objectId')
53fbda23-9e1e-4d74-949a-06d3200e36f9

# 파일 작성
$ vi terraform.tfvars
aks_name = "myAKSCluster"  # 이 라인은 참조처에는 없는데 필요함. 지정하지 않으면 variables.tf 에 있는 기본값 "aks-cluster1" 이 들어감.

resource_group_name = "<Name of the Resource Group already created>" # 이건 내 리소스 그룹 (내 경우엔 04226) 을 입력

location = "<Location of the Resource Group>" # 리소스 그룹의 위치 (내 경우엔 koreacentral)

aks_service_principal_app_id = "<Service Principal AppId>"  # 위의 appId 입력

aks_service_principal_client_secret = "<Service Principal Client Secret>"  # 위의 password 입력

aks_service_principal_object_id = "<Service Principal Object Id>"     # 위의 $OBJECT_ID 입력
```


여기까지 하고 terraform init 을 시도하는 데 에러가 난 적이 있음.

```
# 위의 ACCOUNT_KEY 를 사용
$ terraform init -backend-config="storage_account_name=04226diag" -backend-config="container_name=tfstate" -backend-config="access_key=<YourStorageAccountAccessKey 즉 ACCOUNT_KEY>" -backend-config="key=codelab.microsoft.tfstate" 
Error: Error parsing /home/azureuser/terraform-aks-appgw-ingress/output.tf: At 2:13: Unknown token: 2:13 IDENT azurerm_kubernetes_cluster.k8s.kube_config.0.client_key
```

이게 이상한 것이, 배스천에서 시도하면 에러가 나고 Cloud shell 에서 실행하면 에러가 안나거나 다른 에러가 남.
둘은 환경적으로 뭔가 차이가 있는데 이를테면

```
# 배스천에서는
$ terraform workspace show  # 존재하는 건 확인되지만
default
$ terraform workspace list  # 에러
Failed to load root config module: Error parsing /home/azureuser/terraform-aks-appgw-ingress/output.tf: At 2:13: Unknown token: 2:13 IDENT azurerm_kubernetes_cluster.k8s.kube_config.0.client_key

# Cloud shell 에서는
$ terraform workspace show  # 존재하는 건 확인되고
default
$ terraform workspace list  # 정상
* default
```

... 정말 고생고생한 후 알아낸 사실. 배스천에서 snap으로 설치한 terraform v0.11 버전에서 에러 나던 게 v0.12 나 v0.13 버전은 제대로 실행됨(!!!)

```
$ terraform init -backend-config="storage_account_name=04226diag" -backend-config="container_name=tfstate" -backend-config="access_key=<04226_diag 에 달린 storage account key 쓰면 됨>" -backend-config="key=codelab.microsoft.tfstate"
Initializing the backend...

Initializing provider plugins...

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

## plan / apply


아래는 실패한 예..
```
$ terraform plan -out out.plan
Acquiring state lock. This may take a few moments...
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.azurerm_resource_group.rg: Refreshing state...

------------------------------------------------------------------------

Error: Error in function call

  on resources.tf line 171, in resource "azurerm_kubernetes_cluster" "k8s":
 171:       key_data = file(var.public_ssh_key_path)
    |----------------
    | var.public_ssh_key_path is "~/.ssh/id_rsa.pub"

Call to function "file" failed: no file exists at
/home/ds04226/.ssh/id_rsa.pub.
```
처음에 배스천에서 AKS를 생성하고 Cloud shell에서 terraform 을 실행하다 보니  --generate-ssh-keys 옵션으로 생성한 ssh 키가 Cloud shell에 없어서 나는 에러

AKS를 생성했던 배스천에서 파일 복사 후
```
$ vi ~/.ssh/id_rsa.pub
```
다시 시도하면 됨.

아래는 다른 실패사례.
```
$ terraform plan -out out.plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.azurerm_resource_group.rg: Refreshing state... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226]
azurerm_user_assigned_identity.testIdentity: Refreshing state... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/04226/providers/Microso      ft.ManagedIdentity/userAssignedIdentities/identity1]
azurerm_public_ip.test: Refreshing state... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/publicIPAd      dresses/publicIp1]
data.azurerm_subnet.appgwsubnet: Refreshing state... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/v      irtualNetworks/aksVirtualNetwork/subnets/appgwsubnet]
data.azurerm_subnet.kubesubnet: Refreshing state... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/vi      rtualNetworks/aksVirtualNetwork/subnets/kubesubnet]
azurerm_role_assignment.ra2: Refreshing state... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/04226/providers/Microsoft.ManagedIdenti      ty/userAssignedIdentities/identity1/providers/Microsoft.Authorization/roleAssignments/3a363938-b660-131e-8f3f-e19e94313406]

Error: Error: Subnet "kubesubnet" (Virtual Network "aksVirtualNetwork" / Resource Group "04226") was not found

  on resources.tf line 45, in data "azurerm_subnet" "kubesubnet":
  45: data "azurerm_subnet" "kubesubnet" {



Error: Error: Subnet "appgwsubnet" (Virtual Network "aksVirtualNetwork" / Resource Group "04226") was not found

  on resources.tf line 51, in data "azurerm_subnet" "appgwsubnet":
  51: data "azurerm_subnet" "appgwsubnet" {


```
별 걸 다 의심하면서 이것저것 시도해 봤는데 결론은 terraform이 뭔가 이상하다...

위의 ```*.tf``` 파일들을 보면 알겠지만 순차적 실행하는 코드가 아니기에 terraform이 뭔가 잘못 판단하면 A가 없어서 B가 안되고 B가 없어서 A가 안되는 식의 에러가 나는 것 같다.

거의 terraform 명령어 공부하듯이 뒤져가면서 시도한 것은 다음
```
$ terraform plan -out out.plan -target azurerm_virtual_network.test
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.azurerm_resource_group.rg: Refreshing state... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226]

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # azurerm_virtual_network.test will be created
  + resource "azurerm_virtual_network" "test" {
      + address_space       = [
          + "15.0.0.0/8",
        ]
      + guid                = (known after apply)
      + id                  = (known after apply)
      + location            = "koreacentral"
      + name                = "aksVirtualNetwork"
      + resource_group_name = "04226"
      + subnet              = [
          + {
              + address_prefix = "15.0.0.0/16"
              + id             = (known after apply)
              + name           = "kubesubnet"
              + security_group = ""
            },
          + {
              + address_prefix = "15.1.0.0/16"
              + id             = (known after apply)
              + name           = "appgwsubnet"
              + security_group = ""
            },
        ]
      + tags                = {
          + "source" = "terraform"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  - identity_client_id   = "df40eaed-bfda-4605-ad21-3c390428e40d" -> null
  - identity_resource_id = "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/04226/providers/Microsoft.ManagedIdentity/userAssignedIdentities/identity1" -> null

Warning: Resource targeting is in effect

You are creating a plan with the -target option, which means that the result
of this plan may not represent all of the changes requested by the current
configuration.

The -target option is not for routine use, and is provided only for
exceptional situations such as recovering from errors or mistakes, or when
Terraform specifically suggests to use it as part of an error message.


------------------------------------------------------------------------

This plan was saved to: out.plan

To perform exactly these actions, run the following command to apply:
    terraform apply "out.plan"

```

```
$ terraform apply out.plan
azurerm_virtual_network.test: Creating...
azurerm_virtual_network.test: Still creating... [10s elapsed]
azurerm_virtual_network.test: Creation complete after 10s [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/aksVirtualNetwork]

Warning: Applied changes may be incomplete

The plan was created with the -target option in effect, so some changes
requested in the configuration may have been ignored and the output values may
not be fully updated. Run the following command to verify that no other
changes are pending:
    terraform plan

Note that the -target option is not suitable for routine use, and is provided
only for exceptional situations such as recovering from errors or mistakes, or
when Terraform specifically suggests to use it as part of an error message.


Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

identity_client_id = df40eaed-bfda-4605-ad21-3c390428e40d
identity_resource_id = /subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/04226/providers/Microsoft.ManagedIdentity/userAssignedIdentities/identity1
```
즉 서브넷을 포함한 가상 네트워크를 먼저 만들어 준다.
그리고 나서 전체를 다시 시도


```
$ terraform plan -out out.plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.azurerm_resource_group.rg: Refreshing state... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226]
azurerm_public_ip.test: Refreshing state... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/publicIPAddresses/publicIp1]
azurerm_virtual_network.test: Refreshing state... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/aksVirtualNetwork]
azurerm_user_assigned_identity.testIdentity: Refreshing state... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/04226/providers/Microsoft.ManagedIdentity/userAssignedIdentities/identity1]
azurerm_role_assignment.ra2: Refreshing state... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/04226/providers/Microsoft.ManagedIdentity/userAssignedIdentities/identity1/providers/Microsoft.Authorization/roleAssignments/3a363938-b660-131e-8f3f-e19e94313406]
data.azurerm_subnet.appgwsubnet: Refreshing state... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/aksVirtualNetwork/subnets/appgwsubnet]
data.azurerm_subnet.kubesubnet: Refreshing state... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/aksVirtualNetwork/subnets/kubesubnet]

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # azurerm_application_gateway.network will be created
  + resource "azurerm_application_gateway" "network" {
      + id                  = (known after apply)
      + location            = "koreacentral"
      + name                = "ApplicationGateway1"
      + resource_group_name = "04226"
      + tags                = {
          + "source" = "terraform"
        }

      + backend_address_pool {
          + id   = (known after apply)
          + name = "aksVirtualNetwork-beap"
        }

      + backend_http_settings {
          + cookie_based_affinity               = "Disabled"
          + id                                  = (known after apply)
          + name                                = "aksVirtualNetwork-be-htst"
          + pick_host_name_from_backend_address = false
          + port                                = 80
          + probe_id                            = (known after apply)
          + protocol                            = "Http"
          + request_timeout                     = 1
        }

      + frontend_ip_configuration {
          + id                            = (known after apply)
          + name                          = "aksVirtualNetwork-feip"
          + private_ip_address            = (known after apply)
          + private_ip_address_allocation = (known after apply)
          + public_ip_address_id          = (known after apply)
          + subnet_id                     = (known after apply)
        }

      + frontend_port {
          + id   = (known after apply)
          + name = "aksVirtualNetwork-feport"
          + port = 80
        }
      + frontend_port {
          + id   = (known after apply)
          + name = "httpsPort"
          + port = 443
        }

      + gateway_ip_configuration {
          + id        = (known after apply)
          + name      = "appGatewayIpConfig"
          + subnet_id = "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/aksVirtualNetwork/subnets/appgwsubnet"
        }

      + http_listener {
          + frontend_ip_configuration_id   = (known after apply)
          + frontend_ip_configuration_name = "aksVirtualNetwork-feip"
          + frontend_port_id               = (known after apply)
          + frontend_port_name             = "aksVirtualNetwork-feport"
          + id                             = (known after apply)
          + name                           = "aksVirtualNetwork-httplstn"
          + protocol                       = "Http"
          + ssl_certificate_id             = (known after apply)
        }

      + request_routing_rule {
          + backend_address_pool_id    = (known after apply)
          + backend_address_pool_name  = "aksVirtualNetwork-beap"
          + backend_http_settings_id   = (known after apply)
          + backend_http_settings_name = "aksVirtualNetwork-be-htst"
          + http_listener_id           = (known after apply)
          + http_listener_name         = "aksVirtualNetwork-httplstn"
          + id                         = (known after apply)
          + name                       = "aksVirtualNetwork-rqrt"
          + redirect_configuration_id  = (known after apply)
          + rewrite_rule_set_id        = (known after apply)
          + rule_type                  = "Basic"
          + url_path_map_id            = (known after apply)
        }

      + sku {
          + capacity = 2
          + name     = "Standard_v2"
          + tier     = "Standard_v2"
        }

      + ssl_policy {
          + cipher_suites        = (known after apply)
          + disabled_protocols   = (known after apply)
          + min_protocol_version = (known after apply)
          + policy_name          = (known after apply)
          + policy_type          = (known after apply)
        }
    }

  # azurerm_kubernetes_cluster.k8s will be created
  + resource "azurerm_kubernetes_cluster" "k8s" {
      + dns_prefix              = "aks"
      + fqdn                    = (known after apply)
      + id                      = (known after apply)
      + kube_admin_config       = (known after apply)
      + kube_admin_config_raw   = (sensitive value)
      + kube_config             = (known after apply)
      + kube_config_raw         = (sensitive value)
      + kubelet_identity        = (known after apply)
      + kubernetes_version      = (known after apply)
      + location                = "koreacentral"
      + name                    = "aks-cluster1"
      + node_resource_group     = (known after apply)
      + private_cluster_enabled = (known after apply)
      + private_fqdn            = (known after apply)
      + private_link_enabled    = (known after apply)
      + resource_group_name     = "04226"
      + sku_tier                = "Free"
      + tags                    = {
          + "source" = "terraform"
        }

      + addon_profile {

          + http_application_routing {
              + enabled                            = false
              + http_application_routing_zone_name = (known after apply)
            }
        }

      + auto_scaler_profile {
          + balance_similar_node_groups      = (known after apply)
          + max_graceful_termination_sec     = (known after apply)
          + scale_down_delay_after_add       = (known after apply)
          + scale_down_delay_after_delete    = (known after apply)
          + scale_down_delay_after_failure   = (known after apply)
          + scale_down_unneeded              = (known after apply)
          + scale_down_unready               = (known after apply)
          + scale_down_utilization_threshold = (known after apply)
          + scan_interval                    = (known after apply)
        }

      + default_node_pool {
          + max_pods             = (known after apply)
          + name                 = "agentpool"
          + node_count           = 3
          + orchestrator_version = (known after apply)
          + os_disk_size_gb      = 40
          + type                 = "VirtualMachineScaleSets"
          + vm_size              = "Standard_D3_v2"
          + vnet_subnet_id       = "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/aksVirtualNetwork/subnets/kubesubnet"
        }

      + linux_profile {
          + admin_username = "vmuser1"

          + ssh_key {
              + key_data = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC0SfwvY/5LpwR2F2U33xW/QUC87e3UHgCGJ6GPIRcsSCkAdZvWq6bzCHGzu0xlu/3MbTFWC9bniwP7IjfHMHkHFHroZIOIgPyFAdeodw/ndgZE7gkeL7Yew9fAwe2UUKG8vKLgj9Z5m6N4Oyh7St0xy3ryenyoKu/HKnedKuzycj6622cC4N3IktHmZbIZNEAdECPLhT2aUm13z64R5iZCzYuKrH+2zNSvbKj0UDtEfC8FFnvaJ+ciP07AynIWQbxN3w2MpKE3KknWPabk16DlTw5NgIKKhSUaf+5QIDIqvgm+YmYVdLDepOLlKZJoFqBdeCS3yh5XRrJzJvZQLJ4B"
            }
        }

      + network_profile {
          + dns_service_ip     = "10.0.0.10"
          + docker_bridge_cidr = "172.17.0.1/16"
          + load_balancer_sku  = "standard"
          + network_plugin     = "azure"
          + network_policy     = (known after apply)
          + outbound_type      = "loadBalancer"
          + pod_cidr           = (known after apply)
          + service_cidr       = "10.0.0.0/16"

          + load_balancer_profile {
              + effective_outbound_ips    = (known after apply)
              + idle_timeout_in_minutes   = (known after apply)
              + managed_outbound_ip_count = (known after apply)
              + outbound_ip_address_ids   = (known after apply)
              + outbound_ip_prefix_ids    = (known after apply)
              + outbound_ports_allocated  = (known after apply)
            }
        }

      + role_based_access_control {
          + enabled = (known after apply)

          + azure_active_directory {
              + admin_group_object_ids = (known after apply)
              + client_app_id          = (known after apply)
              + managed                = (known after apply)
              + server_app_id          = (known after apply)
              + server_app_secret      = (sensitive value)
              + tenant_id              = (known after apply)
            }
        }

      + service_principal {
          + client_id     = "81de2f9f-a0f9-4c72-8846-e3ce70a29511"
          + client_secret = (sensitive value)
        }

      + windows_profile {
          + admin_password = (sensitive value)
          + admin_username = (known after apply)
        }
    }

  # azurerm_public_ip.test will be created
  + resource "azurerm_public_ip" "test" {
      + allocation_method       = "Static"
      + fqdn                    = (known after apply)
      + id                      = (known after apply)
      + idle_timeout_in_minutes = 4
      + ip_address              = (known after apply)
      + ip_version              = "IPv4"
      + location                = "koreacentral"
      + name                    = "publicIp1"
      + resource_group_name     = "04226"
      + sku                     = "Standard"
      + tags                    = {
          + "source" = "terraform"
        }
    }

  # azurerm_role_assignment.ra1 will be created
  + resource "azurerm_role_assignment" "ra1" {
      + id                               = (known after apply)
      + name                             = (known after apply)
      + principal_id                     = "53fbda23-9e1e-4d74-949a-06d3200e36f9"
      + principal_type                   = (known after apply)
      + role_definition_id               = (known after apply)
      + role_definition_name             = "Network Contributor"
      + scope                            = "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/aksVirtualNetwork/subnets/kubesubnet"
      + skip_service_principal_aad_check = (known after apply)
    }

  # azurerm_role_assignment.ra2 will be created
  + resource "azurerm_role_assignment" "ra2" {
      + id                               = (known after apply)
      + name                             = (known after apply)
      + principal_id                     = "53fbda23-9e1e-4d74-949a-06d3200e36f9"
      + principal_type                   = (known after apply)
      + role_definition_id               = (known after apply)
      + role_definition_name             = "Managed Identity Operator"
      + scope                            = (known after apply)
      + skip_service_principal_aad_check = (known after apply)
    }

  # azurerm_role_assignment.ra3 will be created
  + resource "azurerm_role_assignment" "ra3" {
      + id                               = (known after apply)
      + name                             = (known after apply)
      + principal_id                     = (known after apply)
      + principal_type                   = (known after apply)
      + role_definition_id               = (known after apply)
      + role_definition_name             = "Contributor"
      + scope                            = (known after apply)
      + skip_service_principal_aad_check = (known after apply)
    }

  # azurerm_role_assignment.ra4 will be created
  + resource "azurerm_role_assignment" "ra4" {
      + id                               = (known after apply)
      + name                             = (known after apply)
      + principal_id                     = (known after apply)
      + principal_type                   = (known after apply)
      + role_definition_id               = (known after apply)
      + role_definition_name             = "Reader"
      + scope                            = "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226"
      + skip_service_principal_aad_check = (known after apply)
    }

  # azurerm_user_assigned_identity.testIdentity will be created
  + resource "azurerm_user_assigned_identity" "testIdentity" {
      + client_id           = (known after apply)
      + id                  = (known after apply)
      + location            = "koreacentral"
      + name                = "identity1"
      + principal_id        = (known after apply)
      + resource_group_name = "04226"
      + tags                = {
          + "source" = "terraform"
        }
    }

Plan: 8 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + client_certificate     = (known after apply)
  + client_key             = (known after apply)
  + cluster_ca_certificate = (known after apply)
  + cluster_password       = (known after apply)
  + cluster_username       = (known after apply)
  + host                   = (known after apply)
  ~ identity_client_id     = "df40eaed-bfda-4605-ad21-3c390428e40d" -> (known after apply)
  ~ identity_resource_id   = "/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/04226/providers/Microsoft.ManagedIdentity/userAssignedIdentities/identity1" -> (known after apply)
  + kube_config            = (known after apply)

------------------------------------------------------------------------

This plan was saved to: out.plan

To perform exactly these actions, run the following command to apply:
    terraform apply "out.plan"


```
이제 생성.

```
$ terraform apply out.plan
Acquiring state lock. This may take a few moments...
azurerm_user_assigned_identity.testIdentity: Creating...
azurerm_public_ip.test: Creating...
azurerm_virtual_network.test: Creating...
azurerm_public_ip.test: Creation complete after 4s [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/publicIPAddresses/publicIp1]
azurerm_user_assigned_identity.testIdentity: Creation complete after 9s [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/04226/providers/Microsoft.ManagedIdentity/userAssignedIdentities/identity1]
azurerm_role_assignment.ra2: Creating...
azurerm_virtual_network.test: Still creating... [10s elapsed]
azurerm_virtual_network.test: Creation complete after 15s [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/aksVirtualNetwork]
data.azurerm_subnet.kubesubnet: Refreshing state...
data.azurerm_subnet.appgwsubnet: Refreshing state...
azurerm_role_assignment.ra1: Creating...
azurerm_application_gateway.network: Creating...
azurerm_application_gateway.network: Still creating... [10s elapsed]
azurerm_application_gateway.network: Still creating... [20s elapsed]
azurerm_application_gateway.network: Still creating... [30s elapsed]
azurerm_application_gateway.network: Still creating... [40s elapsed]
azurerm_application_gateway.network: Still creating... [50s elapsed]
azurerm_application_gateway.network: Still creating... [1m0s elapsed]
azurerm_application_gateway.network: Still creating... [1m10s elapsed]
azurerm_application_gateway.network: Still creating... [1m20s elapsed]
azurerm_application_gateway.network: Still creating... [1m30s elapsed]
azurerm_application_gateway.network: Still creating... [1m40s elapsed]
azurerm_application_gateway.network: Still creating... [1m50s elapsed]
azurerm_application_gateway.network: Still creating... [2m0s elapsed]
azurerm_application_gateway.network: Still creating... [2m10s elapsed]
azurerm_application_gateway.network: Still creating... [2m20s elapsed]
azurerm_application_gateway.network: Still creating... [2m30s elapsed]
azurerm_application_gateway.network: Still creating... [2m40s elapsed]
azurerm_application_gateway.network: Still creating... [2m50s elapsed]
azurerm_application_gateway.network: Still creating... [3m0s elapsed]
azurerm_application_gateway.network: Still creating... [3m10s elapsed]
azurerm_application_gateway.network: Still creating... [3m20s elapsed]
azurerm_application_gateway.network: Still creating... [3m30s elapsed]
azurerm_application_gateway.network: Still creating... [3m40s elapsed]
azurerm_application_gateway.network: Still creating... [3m50s elapsed]
azurerm_application_gateway.network: Still creating... [4m0s elapsed]
azurerm_application_gateway.network: Still creating... [4m10s elapsed]
azurerm_application_gateway.network: Still creating... [4m20s elapsed]
azurerm_application_gateway.network: Still creating... [4m30s elapsed]
azurerm_application_gateway.network: Still creating... [4m40s elapsed]
azurerm_application_gateway.network: Still creating... [4m50s elapsed]
azurerm_application_gateway.network: Still creating... [5m0s elapsed]
azurerm_application_gateway.network: Still creating... [5m10s elapsed]
azurerm_application_gateway.network: Still creating... [5m20s elapsed]
azurerm_application_gateway.network: Still creating... [5m30s elapsed]
azurerm_application_gateway.network: Creation complete after 5m36s [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/ApplicationGateway1]
azurerm_role_assignment.ra4: Creating...
azurerm_role_assignment.ra3: Creating...
azurerm_kubernetes_cluster.k8s: Creating...
azurerm_role_assignment.ra4: Still creating... [10s elapsed]
azurerm_role_assignment.ra3: Still creating... [10s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [10s elapsed]
azurerm_role_assignment.ra4: Still creating... [20s elapsed]
azurerm_role_assignment.ra3: Still creating... [20s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [20s elapsed]
azurerm_role_assignment.ra4: Creation complete after 26s [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Authorization/roleAssignments/e83734fb-4d43-28c2-faf2-f20fbe867d76]
azurerm_role_assignment.ra3: Creation complete after 27s [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/applicationGateways/ApplicationGateway1/providers/Microsoft.Authorization/roleAssignments/9caac22a-7e69-0785-5cf0-9d227c76ccc1]
azurerm_kubernetes_cluster.k8s: Still creating... [30s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [40s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [50s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [1m0s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [1m10s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [1m20s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [1m30s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [1m40s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [1m50s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [2m0s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [2m10s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [2m20s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [2m30s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [2m40s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [2m50s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [3m0s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [3m10s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [3m20s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [3m30s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [3m40s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [3m50s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [4m0s elapsed]
azurerm_kubernetes_cluster.k8s: Still creating... [4m10s elapsed]
azurerm_kubernetes_cluster.k8s: Creation complete after 4m11s [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/04226/providers/Microsoft.ContainerService/managedClusters/aks-cluster1]

Error: authorization.RoleAssignmentsClient#Create: Failure responding to request: StatusCode=400 -- Original Error: autorest/azure: Service returned an error. Status=400 Code="InvalidPrincipalId" Message="A valid principal ID must be provided for role assignment."

  on resources.tf line 131, in resource "azurerm_role_assignment" "ra1":
 131: resource "azurerm_role_assignment" "ra1" {



Error: authorization.RoleAssignmentsClient#Create: Failure responding to request: StatusCode=400 -- Original Error: autorest/azure: Service returned an error. Status=400 Code="InvalidPrincipalId" Message="A valid principal ID must be provided for role assignment."

  on resources.tf line 139, in resource "azurerm_role_assignment" "ra2":
 139: resource "azurerm_role_assignment" "ra2" {

```

뭘 또 해야 하는거지... 

일단 이걸 참조함. 잘못 찾는 건지도 모르지만
https://docs.microsoft.com/ko-kr/azure/role-based-access-control/role-assignments-template

```
# "나"의 오브젝트 아이디를 찾자.
$ az ad user show --id "ds04226@infrads.onmicrosoft.com" --query objectId --output tsv

# 이걸 적용
$ vi terraform.tfvars
<object_id 에 위의 값을 입력>

# 설정이 바뀌었으니 plan부터 다시 해야 
$ terraform plan -out out.plan
...
$ $ terraform apply out.plan
Acquiring state lock. This may take a few moments...
azurerm_role_assignment.ra1: Creating...
azurerm_role_assignment.ra2: Creating...
azurerm_kubernetes_cluster.k8s: Modifying... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/04226/providers/Microsoft.ContainerService/managedClusters/aks-cluster1]
azurerm_role_assignment.ra1: Still creating... [10s elapsed]
azurerm_role_assignment.ra2: Still creating... [10s elapsed]
azurerm_kubernetes_cluster.k8s: Still modifying... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-...erService/managedClusters/aks-cluster1, 10s elapsed]
azurerm_role_assignment.ra2: Still creating... [20s elapsed]
azurerm_role_assignment.ra1: Still creating... [20s elapsed]
azurerm_kubernetes_cluster.k8s: Still modifying... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-...erService/managedClusters/aks-cluster1, 20s elapsed]
azurerm_role_assignment.ra2: Creation complete after 26s [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/04226/providers/Microsoft.ManagedIdentity/userAssignedIdentities/identity1/providers/Microsoft.Authorization/roleAssignments/386f034b-037b-8699-3ff4-8451d1e35741]
azurerm_role_assignment.ra1: Creation complete after 26s [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/04226/providers/Microsoft.Network/virtualNetworks/aksVirtualNetwork/subnets/kubesubnet/providers/Microsoft.Authorization/roleAssignments/0308a06a-f0f3-3d9f-2760-26ccc7f16196]
azurerm_kubernetes_cluster.k8s: Still modifying... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-...erService/managedClusters/aks-cluster1, 30s elapsed]
azurerm_kubernetes_cluster.k8s: Still modifying... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-...erService/managedClusters/aks-cluster1, 40s elapsed]
azurerm_kubernetes_cluster.k8s: Still modifying... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-...erService/managedClusters/aks-cluster1, 50s elapsed]
azurerm_kubernetes_cluster.k8s: Still modifying... [id=/subscriptions/3ac347d8-a75f-4611-8ca8-...erService/managedClusters/aks-cluster1, 1m0s elapsed]
azurerm_kubernetes_cluster.k8s: Modifications complete after 1m8s [id=/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/04226/providers/Microsoft.ContainerService/managedClusters/aks-cluster1]

Apply complete! Resources: 2 added, 1 changed, 0 destroyed.

Outputs:

client_certificate = LS0tLS1CRUdJTiB........RVJUSUZJQ0FURS0tLS0tCg==
client_key = LS0tLS1CRUdJTi........S0VZLS0tLS0K
cluster_ca_certificate = LS0tLS1CRUdJTiBDRVJ........JQ0FURS0tLS0tCg==
cluster_password = 788f4f10e........a35bd
cluster_username = clusterUser_04226_aks-cluster1
host = https://aks-65b53683.hcp.koreacentral.azmk8s.io:443
identity_client_id = 9c70915d-0055-4be7-826f-af3c37b292f4
identity_resource_id = /subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/04226/providers/Microsoft.ManagedIdentity/userAssignedIdentities/identity1
kube_config = apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1........URS0tLS0tCg==
    server: https://aks-65b53683.hcp.koreacentral.azmk8s.io:443
  name: aks-cluster1
contexts:
- context:
    cluster: aks-cluster1
    user: clusterUser_04226_aks-cluster1
  name: aks-cluster1
current-context: aks-cluster1
kind: Config
preferences: {}
users:
- name: clusterUser_04226_aks-cluster1
  user:
    client-certificate-data: LS0tLS1CRU........Q0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0E........0VZLS0tLS0K
    token: 788f4........15a35bd
```
에러 없이 끝났다.
관리화면에 들어가서 리소스 그룹쪽을 찾아보면 ApplicationGateway1 라는 이름의 게이트웨이가 생성된 것을 확인할 수 있음.
public ip 부여 되어 있고..

그러나

cluster가 aks-cluster1 이라고??? <br>
확인해 보니 aks-cluster1 클러스터가 만들어졌음.<br>
허허허...<br>
이미 생성된 AKS에 연결시켜주는 줄 알았지..

```
$ az aks get-credentials --resource-group 04226 --name aks-cluster1

$ az aks delete -g 04226 -n myAKSCluster # 원래 만들어 놓았던 걸 삭제
```


이제 연결이 되나 보자.

keycloak을 설치하고, ingress 를 셋업한다.

설치는.. 아래 설치는 최신 차트버전 9.0.1 로 설치했던 건데 고생을 심하게 해서
최근 설치는 다음과 같이 예전 버전으로 설치했다.
다른 페이지에 있는 설치본이 이 버전임  
```
$ helm install -n mta-infra keycloak --version 8.2.2 -f keycloak-values-8.2.2.yaml codecentric/keycloak
```

설치 과정은 다른 페이지에 있으니가 생략하고 인그레스만 보이면:
```
$ cat keycloak-ing.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: keycloak-ingress
  namespace: mta-infra
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: keycloak-http
          servicePort: 80
        path: /
    host: azure-keycloak.skmta.net
```

근데 설치가 잘 안되네?
```
$ kubectl describe po -n mta-infra keycloak-0 | tail -3
  Warning  FailedScheduling  51m (x10 over 63m)    default-scheduler  persistentvolumeclaim "keycloak-storage-data" not found
  Warning  FailedScheduling  50m                   default-scheduler  persistentvolumeclaim "keycloak-storage-theme" not found
  Warning  FailedScheduling  2m23s (x36 over 50m)  default-scheduler  pod has unbound immediate PersistentVolumeClaims

$ kubectl describe pvc -n mta-infra data-keycloak-postgresql-0 | tail -2
  Warning  ProvisioningFailed  60m                  persistentvolume-controller  Failed to provision volume with StorageClass "default": azure.BearerAuthorizer#WithAuthorization: Failed to refresh the Token for request to http://localhost:7788/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/mc_04226_myaks_koreacentral/providers/Microsoft.Compute/disks/kubernetes-dynamic-pvc-2d87ef96-cc44-4411-8e0f-8580d9da6a96?api-version=2018-09-30: StatusCode=401 -- Original Error: adal: Refresh request failed. Status Code = '401'. Response body: {"error":"invalid_client","error_description":"AADSTS7000215: Invalid client secret is provided.\r\nTrace ID: 35878c00-6057-4fc6-857e-660e2b811800\r\nCorrelation ID: 4224c6fc-ff5f-489f-b1ea-95f83e911935\r\nTimestamp: 2020-08-19 04:22:28Z","error_codes":[7000215],"timestamp":"2020-08-19 04:22:28Z","trace_id":"35878c00-6057-4fc6-857e-660e2b811800","correlation_id":"4224c6fc-ff5f-489f-b1ea-95f83e911935","error_uri":"https://login.microsoftonline.com/error?code=7000215"}
  Warning  ProvisioningFailed  2m3s (x26 over 58m)  persistentvolume-controller  (combined from similar events): Failed to provision volume with StorageClass "default": azure.BearerAuthorizer#WithAuthorization: Failed to refresh the Token for request to http://localhost:7788/subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourceGroups/mc_04226_myaks_koreacentral/providers/Microsoft.Compute/disks/kubernetes-dynamic-pvc-2d87ef96-cc44-4411-8e0f-8580d9da6a96?api-version=2018-09-30: StatusCode=401 -- Original Error: adal: Refresh request failed. Status Code = '401'. Response body: {"error":"invalid_client","error_description":"AADSTS7000215: Invalid client secret is provided.\r\nTrace ID: 50a8d7a6-bbe9-42e4-99db-1b6368b73b00\r\nCorrelation ID: 4e77c0ee-2847-481b-b833-32ce85f3e367\r\nTimestamp: 2020-08-19 05:20:58Z","error_codes":[7000215],"timestamp":"2020-08-19 05:20:58Z","trace_id":"50a8d7a6-bbe9-42e4-99db-1b6368b73b00","correlation_id":"4e77c0ee-2847-481b-b833-32ce85f3e367","error_uri":"https://login.microsoftonline.com/error?code=7000215"}
```

이상해서 이것저것 뒤져봤는데 이걸 안했었던 것 같다.
https://docs.microsoft.com/ko-kr/azure/aks/update-credentials#update-aks-cluster-with-new-service-principal-credentials
```
$ SP_ID=$(az aks show --resource-group 04226 --name myAKS --query servicePrincipalProfile.clientId -o tsv)
$ az aks update-credentials --resource-group 04226 --name myAKS --reset-service-principal \
   --service-principal $SP_ID --client-secret "$SP_SECRET"
<별도 출력은 없음>
$ kubectl get pvc        # Pending 상태였던 pvc가 잘 바운드 됨을 확인
NAME                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-keycloak-postgresql-0   Bound    pvc-2d87ef96-cc44-4411-8e0f-8580d9da6a96   8Gi        RWO            default        73m
keycloak-storage-data        Bound    pvc-77dc59f1-836c-423f-b1a0-7bd642dbcd5c   1Gi        RWO            default        28m
keycloak-storage-theme       Bound    pvc-73bddc1a-ba51-41bd-a42c-9fd522fade25   1Gi        RWO            default        28m
```
위에서 $SP_SECRET 은 아까 구했던 값을  terraform.tfvars 파일에 적어두었으니 그걸 베끼면 된다. (다시 reset해서 구하면 값이 바뀌어 버리니..)


근데 keycloak 하고 어떻게 연결하는 지 방법을 못찾겠네..?

일단 처음 참조처에서 예제를 실행해 보자. 
https://docs.microsoft.com/en-us/azure/developer/terraform/create-k8s-cluster-with-aks-applicationgateway-ingress#install-ingress-controller-helm-chart
```
# default namespace 에 설정하더라:
$ kubectl create -f https://raw.githubusercontent.com/Azure/aad-pod-identity/master/deploy/infra/deployment-rbac.yaml
serviceaccount/aad-pod-id-nmi-service-account created
customresourcedefinition.apiextensions.k8s.io/azureassignedidentities.aadpodidentity.k8s.io created
customresourcedefinition.apiextensions.k8s.io/azureidentitybindings.aadpodidentity.k8s.io created
customresourcedefinition.apiextensions.k8s.io/azureidentities.aadpodidentity.k8s.io created
customresourcedefinition.apiextensions.k8s.io/azurepodidentityexceptions.aadpodidentity.k8s.io created
clusterrole.rbac.authorization.k8s.io/aad-pod-id-nmi-role created
clusterrolebinding.rbac.authorization.k8s.io/aad-pod-id-nmi-binding created
daemonset.apps/nmi created
serviceaccount/aad-pod-id-mic-service-account created
clusterrole.rbac.authorization.k8s.io/aad-pod-id-mic-role created
clusterrolebinding.rbac.authorization.k8s.io/aad-pod-id-mic-binding created
deployment.apps/mic created
```

어? Azure AKS 용 ingress가 따로 있나?
그럼 내가 깔았던 건 뭐지? ... 아무튼 현재는 AKS를 새로 깔았으니 무시하자.

```
$ helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/
$ helm repo update

$ wget https://raw.githubusercontent.com/Azure/application-gateway-kubernetes-ingress/master/docs/examples/sample-helm-config.yaml -O helm-config.yaml
$ vi helm-config.yaml
# This file contains the essential configs for the ingress controller helm chart

# Verbosity level of the App Gateway Ingress Controller
verbosityLevel: 3

################################################################################
# Specify which application gateway the ingress controller will manage
#
appgw:
    subscriptionId: 3ac347d8-a75f-4611-8ca8-161a69189283     # 구독ID : az account list --output table 명령으로도 볼 수 있음
    resourceGroup: "04226"                                   # 이것을 따옴표 없이 숫자로 쓰면 다르게 인식됨. 주의
    name: ApplicationGateway1                                # 만들어 놓은 AG
    usePrivateIP: false

    # Setting appgw.shared to "true" will create an AzureIngressProhibitedTarget CRD.
    # This prohibits AGIC from applying config for any host/path.
    # Use "kubectl get AzureIngressProhibitedTargets" to view and change this.
    shared: false

################################################################################
# Specify which kubernetes namespace the ingress controller will watch
# Default value is "default"
# Leaving this variable out or setting it to blank or empty string would
# result in Ingress Controller observing all acessible namespaces.
#
# kubernetes:
#   watchNamespace: <namespace>

################################################################################
# Specify the authentication with Azure Resource Manager
#
# Two authentication methods are available:
# - Option 1: AAD-Pod-Identity (https://github.com/Azure/aad-pod-identity)
#armAuth:
#    type: aadPodIdentity           # 처음에 이걸로 될 줄 알고 시도했었음.
#    identityResourceID: /subscriptions/3ac347d8-a75f-4611-8ca8-161a69189283/resourcegroups/04226/providers/Microsoft.ManagedIdentity/userAssignedIdentities/identity1
#    identityClientID:  9c70915d-0055-4be7-826f-af3c37b292f4

## Alternatively you can use Service Principal credentials
armAuth:
    type: servicePrincipal
    secretJSON: "ewogICJjbGllbnRJZCI6ICI1ZT.....3JlLndpbmRvd3MubmV0LyIKfQo="
    # <<Generate this value with: "az ad sp create-for-rbac --subscription <subscription-uuid> --sdk-auth | base64 -w0" >>
    # --subscription 옵션은 없어야 실행이 되더라.
    # --sdk-auth 옵션이 있어야 json 안에 몇 가지 항목이 더 들어감.
# Specify if the cluster is RBAC enabled or not
rbac:
    enabled: true # true/false
################################################################################

$ helm install -n default ag-ingress-controller -f helm-config.yaml application-gateway-kubernetes-ingress/ingress-azure
NAME: ag-ingress-controller
LAST DEPLOYED: Wed Aug 19 06:12:30 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing ingress-azure:1.2.0.

Your release is named ag-ingress-controller.
The controller is deployed in deployment ag-ingress-controller-ingress-azure.

Configuration Details:
----------------------
 * AzureRM Authentication Method:
    - Use AAD-Pod-Identity
 * Application Gateway:
    - Subscription ID : 3ac347d8-a75f-4611-8ca8-161a69189283
    - Resource Group  : 2198    <====  이건 리소스 그룹명을 "04226" 이 아닌 04226 으로 입력했을 때 잘못 인식된 케이스
    - Application Gateway Name : ApplicationGateway1
 * Kubernetes Ingress Controller:
    - Watching All Namespaces
    - Verbosity level: 3

Please make sure the associated aadpodidentity and aadpodidbinding is configured.
For more information on AAD-Pod-Identity, please visit https://github.com/Azure/aad-pod-identity
```

설치는 되는데 정상작동 안하고 있음...? (리소스 그룹과 az ad sp 명령 잘못 준 탓)

```
$ kubectl get  po -n default -l release=ag-ingress-controller
NAME                                                   READY   STATUS    RESTARTS   AGE
ag-ingress-controller-ingress-azure-59c89485c8-gmp4k   0/1     Running   1          10m
```

위에 지적한 사항들 잘 주면 설치 제대로 됨.

아래 로그는 제대로 값을 안 주었을 때 나오는 예.
```
$ kubectl logs -n default ag-ingress-controller-ingress-azure-54474ffd4b-87kws | tail
...
E0819 06:53:37.405949       1 client.go:170] Code="ErrorApplicationGatewayUnexpectedStatusCode" Message="Unexpected status code '401' while performing a GET on Application Gateway." InnerError="network.ApplicationGatewaysClient#Get: Failure responding to request: StatusCode=401 -- Original Error: autorest/azure: Service returned an error. Status=401 Code="AuthenticationFailed" Message="Authentication failed. The 'Authorization' header is missing.""
```


## 삭제는?

무엇이 생기는 지 식별해 두자.
* 공용 IP 주소 (public ip) "publicIp1" 
* 관리ID (identity) "identity1"
* 애플리케이션 게이트웨이 (application gateway)
* 서브넷 -- 이건 아직 잘 모름


## Ingress 연결은?

아직 ingress 연결은 못한 채임... 

아까 만든 ingress 는 존재하니, 일단 내 PC의 hosts 파일에 다음 라인을 넣어 보고 
```
20.194.15.35   azure-keycloak.skmta.net   # IP는 Application Gateway 찾아가서 정보 확인한 것임
```

연결해 보는데 ```502 Bad Gateway``` 에러가 난다.

관리콘솔에서 Application Gateway 찾아가면 502 에러가 났음을 알려주고 backend 상태를 로그로 보여줌






