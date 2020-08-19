# AKS 에서 Application gateway 설정해 보기

다음 링크를 참조함:
https://docs.microsoft.com/ko-kr/azure/developer/terraform/create-k8s-cluster-with-aks-applicationgateway-ingress

※ 나중에 알게 된 거지만 위의 링크 가이드는 그냥 배스천에서는 안된다. Azure Portal 에서 실행하는 Cloud shell에서 실행하는 건데 
   거기엔 약간의 환경적인 준비가 되어 있다. 그걸 몰라서 고생함. 

먼저 준비:
```
$ sudo snap install terraform  # 이것은 cloud shell 에서는 불필요
Download snap "terraform" (216) from channel "stable"           53%  139kB/s 1m40s
```

왜 테라폼을 쓰는 지 모르겠지만.. 파일을 작성
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


테라폼 초기화 시도 하는데 에러가 나네? <-- 이건 Cloud shell 에서 실행하지 않은 것과 관련이 있음
```
$ terraform init -backend-config="storage_account_name=04226diag" -backend-config="container_name=tfstate" -backend-config="access_key=<YourStorageAccountAccessKey>" -backend-config="key=codelab.microsoft.tfstate" 
Error: Error parsing /home/azureuser/terraform-aks-appgw-ingress/output.tf: At 2:13: Unknown token: 2:13 IDENT azurerm_kubernetes_cluster.k8s.kube_config.0.client_key
```

Cloud shell 에서 실행할 때는 다른 에러가 났었는데.. access_key가 잘못되었다고 하면서.. 그런데 실제 잘못되지는 않았고..
계속 재시도하다 어느 순간 되기 시작했는데 된 이유는 모르겠음.
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

```
$ vi terraform.tfvars
resource_group_name = "<Name of the Resource Group already created>"

location = "<Location of the Resource Group>"

aks_service_principal_app_id = "<Service Principal AppId>"  # 만들어 놓은 AKS의 servicePrincipalProfile 속성에 적힌 것을 입력함

aks_service_principal_client_secret = "<Service Principal Client Secret>"  # 만들어 놓은 AKS의 servicePrincipalProfile 속성에.. null이네? 빈간

aks_service_principal_object_id = "<Service Principal Object Id>"
```

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
실행이 안되는데, 본래 AKS를 Cloud shell에서 (--generate-ssh-keys 옵션 쓰고) 생성했다면 나지 않았을 에러.

별 수 없이 AKS를 생성했던 배스천에서 파일 복사 후
```
$ vi ~/.ssh/id_rsa.pub
```

다시 시도.
```
$ terraform plan -out out.planAcquiring state lock. This may take a few moments...Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.azurerm_resource_group.rg: Refreshing state...

------------------------------------------------------------------------

Error: expected "service_principal.0.client_secret" to not be an empty string, got

  on resources.tf line 160, in resource "azurerm_kubernetes_cluster" "k8s":
 160: resource "azurerm_kubernetes_cluster" "k8s" {
```
날 수 있는 에러는 다 나네. 

https://docs.microsoft.com/ko-kr/azure/aks/update-credentials
이걸 참조하여 service principal 을 만들어야 하나보다.

```
# service principal client id 구하기
$ SP_ID=$(az aks show --resource-group 04226 --name myAKS  --query servicePrincipalProfile.clientId -o tsv)

# 만료날자 확인 (생성시점으로부터 5년)
$ az ad sp credential list --id $SP_ID --query "[].endDate" -o tsv

# 리셋하면서 client secret 얻자
$ SP_SECRET=$(az ad sp credential reset --name $SP_ID --query password -o tsv)
$ echo $SP_SECRET
<결과>
$ vi terraform.tfvars
<위의 결과를 적용>
$ terraform plan -out out.plan
Acquiring state lock. This may take a few moments...
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

data.azurerm_resource_group.rg: Refreshing state...

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create
 <= read (data resources)

Terraform will perform the following actions:

  # data.azurerm_subnet.appgwsubnet will be read during apply
  # (config refers to values not yet known)
 <= data "azurerm_subnet" "appgwsubnet"  {
      + address_prefix                                 = (known after apply)
      + address_prefixes                               = (known after apply)
      + enforce_private_link_endpoint_network_policies = (known after apply)
      + enforce_private_link_service_network_policies  = (known after apply)
      + id                                             = (known after apply)
      + name                                           = "appgwsubnet"
      + network_security_group_id                      = (known after apply)
      + resource_group_name                            = "04226"
      + route_table_id                                 = (known after apply)
      + service_endpoints                              = (known after apply)
      + virtual_network_name                           = "aksVirtualNetwork"

      + timeouts {
          + read = (known after apply)
        }
    }

  # data.azurerm_subnet.kubesubnet will be read during apply
  # (config refers to values not yet known)
 <= data "azurerm_subnet" "kubesubnet"  {
      + address_prefix                                 = (known after apply)
      + address_prefixes                               = (known after apply)
      + enforce_private_link_endpoint_network_policies = (known after apply)
      + enforce_private_link_service_network_policies  = (known after apply)
      + id                                             = (known after apply)
      + name                                           = "kubesubnet"
      + network_security_group_id                      = (known after apply)
      + resource_group_name                            = "04226"
      + route_table_id                                 = (known after apply)
      + service_endpoints                              = (known after apply)
      + virtual_network_name                           = "aksVirtualNetwork"

      + timeouts {
          + read = (known after apply)
        }
    }

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
          + subnet_id = (known after apply)
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
          + vnet_subnet_id       = (known after apply)
        }

      + linux_profile {
          + admin_username = "vmuser1"

          + ssh_key {
              + key_data = <<~EOT
                    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC0SfwvY/5LpwR2F2U33xW/QUC87e3UHgCGJ6GPIRcsSCkAdZvWq6bzCHGzu0xlu/3MbTFWC9bniwP7IjfHMHkHFHroZIOIgPyFAdeodw/ndgZE7gkeL7Yew9fAwe2UUKG8vKLgj9Z5m6N4Oyh7St0xy3ryenyoKu/HKnedKuzycj6622cC4N3IktHmZbIZNEAdECPLhT2aUm13z64R5iZCzYuKrH+2zNSvbKj0UDtEfC8FFnvaJ+ciP07AynIWQbxN3w2MpKE3KknWPabk16DlTw5NgIKKhSUaf+5QIDIqvgm+YmYVdLDepOLlKZJoFqBdeCS3yh5XRrJzJvZQLJ4B
                EOT
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
          + client_id     = "8ce78bdc-cc25-4a18-a342-17dd50574a3f"
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
      + principal_type                   = (known after apply)
      + role_definition_id               = (known after apply)
      + role_definition_name             = "Network Contributor"
      + scope                            = (known after apply)
      + skip_service_principal_aad_check = (known after apply)
    }

  # azurerm_role_assignment.ra2 will be created
  + resource "azurerm_role_assignment" "ra2" {
      + id                               = (known after apply)
      + name                             = (known after apply)
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

Plan: 9 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

This plan was saved to: out.plan

To perform exactly these actions, run the following command to apply:
    terraform apply "out.plan"

```
오 성공했나.









