# Python 으로 kubernetes 제어하기

## 프로토타이핑

항상 쉽게 시작하는 게 좋음.

python 에 kubernetes가 깔린 이미지를 구해서 Deployment든 StatefulSet으로든 띄워놓았다고 가정함. 
그런게 있으면 다행이고 없으면 만들어야 함...
어쨌든 여기선 그게 있다고 가정하고 다음과 같이 확인한다.
```
$ kubectl get po -n app | findstr controller
controller-6746f5b4f9-zp6cr   1/1     Running   0          4m30s
```

이 pod의 실행 `serviceAccount` 가 `controller`라고 가정하자. 없으면 만든다.
```
$ kubectl create sa -n app controller
```

물론 위의 pod 에 해당하는 Deployment나 StatefulSet이 이 `serviceAccount`로 실행한다고 설정해 놓아야 함.
```
$ kubectl edit deploy -n app controller      # 한 예입니다
...
spec:
  template:
    spec:
      ...
      serviceAccount: controller
...
```

이 `controller`에 권한을 부여해야 하는데, 언제나 그렇듯 디테일하게 권한을 조정하는 것은 무척 까다로운 일이니 실환경에서 그렇게 하기로 하고 여기선 다 주자.

```
$ vi controller-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding           # 네임스페이스에 구애받지 않는 권한이어야 하므로 클러스터롤이어야 함
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: controller-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin              # 모든 권한 다 줘
subjects:
- kind: ServiceAccount
  name: controller
  namespace: app
```

이제 이렇게 설정이 되면 권한 문제가 없겠지? 이 POD로 들어가서 뭔가 실행해 보자.  
```
$ kubectl exec -it  -n app controller-6746f5b4f9-zp6cr -- python
>>> 
```
(이 때 POD가 충분한 메모리를 가져야 함. 처음에 200Mi 정도 줬더니 툭하면 죽더라)  

다음 코드를 한 번 실행해 보자.
```
from kubernetes import client, config
#import ssl
# 권한 얻기
config = client.Configuration()
config.api_key['authorization'] = open('/var/run/secrets/kubernetes.io/serviceaccount/token').read()
config.api_key_prefix['authorization'] = 'Bearer'
config.host = 'https://10.0.0.1'
config.ssl_ca_cert = '/var/run/secrets/kubernetes.io/serviceaccount/ca.crt'
config.verify_ssl=True
```
여기까지가 준비

지원되는 API 버전 확인:
```
client.CoreApi().get_api_versions().versions
```

API Client 얻기
```
# api_client 얻기
api_client = client.CoreV1Api(client.ApiClient(config))
```

아래부터는 얻은 API Client를 이용해서 하는 작업들임.


```
# 특정 namespace에 해당하는 pod 들 정보 얻기
ret = api_client.list_namespaced_pod("app", watch=False)

# pod 리스트 및 그 개수 및 요약목록 얻기
ret = api_client.list_pod_for_all_namespaces(watch=False)
print(len(ret.items))
for i in ret.items:
  print("%s\t%s\t%s" % (i.status.pod_ip, i.metadata.namespace, i.metadata.name))
```

```
# 무슨 일이 가능하지? 속성이나 메소드가 뭐가 있는지 보자:
>>> dir(api_client)
['__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', 'api_client', 'connect_delete_namespaced_pod_proxy','connect_delete_namespaced_pod_proxy_with_http_info', 'connect_delete_namespaced_pod_proxy_with_path', 'connect_delete_namespaced_pod_proxy_with_path_with_http_info', 'connect_delete_namespaced_service_proxy', 'connect_delete_namespaced_service_proxy_with_http_info', 'connect_delete_namespaced_service_proxy_with_path', 'connect_delete_namespaced_service_proxy_with_path_with_http_info', 'connect_delete_node_proxy', 'connect_delete_node_proxy_with_http_info', 'connect_delete_node_proxy_with_path', 'connect_delete_node_proxy_with_path_with_http_info', 'connect_get_namespaced_pod_attach', 'connect_get_namespaced_pod_attach_with_http_info', 'connect_get_namespaced_pod_exec', 'connect_get_namespaced_pod_exec_with_http_info', 'connect_get_namespaced_pod_portforward', 'connect_get_namespaced_pod_portforward_with_http_info', 'connect_get_namespaced_pod_proxy', 'connect_get_namespaced_pod_proxy_with_http_info', 'connect_get_namespaced_pod_proxy_with_path', 'connect_get_namespaced_pod_proxy_with_path_with_http_info', 'connect_get_namespaced_service_proxy', 'connect_get_namespaced_service_proxy_with_http_info', 'connect_get_namespaced_service_proxy_with_path', 'connect_get_namespaced_service_proxy_with_path_with_http_info', 'connect_get_node_proxy', 'connect_get_node_proxy_with_http_info', 'connect_get_node_proxy_with_path', 'connect_get_node_proxy_with_path_with_http_info', 'connect_head_namespaced_pod_proxy', 'connect_head_namespaced_pod_proxy_with_http_info', 'connect_head_namespaced_pod_proxy_with_path', 'connect_head_namespaced_pod_proxy_with_path_with_http_info', 'connect_head_namespaced_service_proxy', 'connect_head_namespaced_service_proxy_with_http_info', 'connect_head_namespaced_service_proxy_with_path', 'connect_head_namespaced_service_proxy_with_path_with_http_info', 'connect_head_node_proxy', 'connect_head_node_proxy_with_http_info', 'connect_head_node_proxy_with_path', 'connect_head_node_proxy_with_path_with_http_info', 'connect_options_namespaced_pod_proxy', 'connect_options_namespaced_pod_proxy_with_http_info', 'connect_options_namespaced_pod_proxy_with_path', 'connect_options_namespaced_pod_proxy_with_path_with_http_info', 'connect_options_namespaced_service_proxy', 'connect_options_namespaced_service_proxy_with_http_info', 'connect_options_namespaced_service_proxy_with_path', 'connect_options_namespaced_service_proxy_with_path_with_http_info', 'connect_options_node_proxy', 'connect_options_node_proxy_with_http_info', 'connect_options_node_proxy_with_path', 'connect_options_node_proxy_with_path_with_http_info', 'connect_patch_namespaced_pod_proxy', 'connect_patch_namespaced_pod_proxy_with_http_info', 'connect_patch_namespaced_pod_proxy_with_path', 'connect_patch_namespaced_pod_proxy_with_path_with_http_info', 'connect_patch_namespaced_service_proxy', 'connect_patch_namespaced_service_proxy_with_http_info', 'connect_patch_namespaced_service_proxy_with_path', 'connect_patch_namespaced_service_proxy_with_path_with_http_info', 'connect_patch_node_proxy', 'connect_patch_node_proxy_with_http_info', 'connect_patch_node_proxy_with_path', 'connect_patch_node_proxy_with_path_with_http_info', 'connect_post_namespaced_pod_attach', 'connect_post_namespaced_pod_attach_with_http_info', 'connect_post_namespaced_pod_exec', 'connect_post_namespaced_pod_exec_with_http_info', 'connect_post_namespaced_pod_portforward', 'connect_post_namespaced_pod_portforward_with_http_info', 'connect_post_namespaced_pod_proxy', 'connect_post_namespaced_pod_proxy_with_http_info', 'connect_post_namespaced_pod_proxy_with_path', 'connect_post_namespaced_pod_proxy_with_path_with_http_info', 'connect_post_namespaced_service_proxy', 'connect_post_namespaced_service_proxy_with_http_info', 'connect_post_namespaced_service_proxy_with_path', 'connect_post_namespaced_service_proxy_with_path_with_http_info', 'connect_post_node_proxy', 'connect_post_node_proxy_with_http_info', 'connect_post_node_proxy_with_path', 'connect_post_node_proxy_with_path_with_http_info', 'connect_put_namespaced_pod_proxy', 'connect_put_namespaced_pod_proxy_with_http_info', 'connect_put_namespaced_pod_proxy_with_path', 'connect_put_namespaced_pod_proxy_with_path_with_http_info', 'connect_put_namespaced_service_proxy', 'connect_put_namespaced_service_proxy_with_http_info', 'connect_put_namespaced_service_proxy_with_path', 'connect_put_namespaced_service_proxy_with_path_with_http_info', 'connect_put_node_proxy', 'connect_put_node_proxy_with_http_info', 'connect_put_node_proxy_with_path', 'connect_put_node_proxy_with_path_with_http_info', 'create_namespace', 'create_namespace_with_http_info', 'create_namespaced_binding', 'create_namespaced_binding_with_http_info', 'create_namespaced_config_map', 'create_namespaced_config_map_with_http_info', 'create_namespaced_endpoints', 'create_namespaced_endpoints_with_http_info', 'create_namespaced_event', 'create_namespaced_event_with_http_info', 'create_namespaced_limit_range', 'create_namespaced_limit_range_with_http_info', 'create_namespaced_persistent_volume_claim', 'create_namespaced_persistent_volume_claim_with_http_info', 'create_namespaced_pod', 'create_namespaced_pod_binding', 'create_namespaced_pod_binding_with_http_info', 'create_namespaced_pod_eviction', 'create_namespaced_pod_eviction_with_http_info', 'create_namespaced_pod_template', 'create_namespaced_pod_template_with_http_info', 'create_namespaced_pod_with_http_info', 'create_namespaced_replication_controller', 'create_namespaced_replication_controller_with_http_info', 'create_namespaced_resource_quota', 'create_namespaced_resource_quota_with_http_info', 'create_namespaced_secret', 'create_namespaced_secret_with_http_info', 'create_namespaced_service', 'create_namespaced_service_account', 'create_namespaced_service_account_token', 'create_namespaced_service_account_token_with_http_info', 'create_namespaced_service_account_with_http_info', 'create_namespaced_service_with_http_info', 'create_node', 'create_node_with_http_info', 'create_persistent_volume', 'create_persistent_volume_with_http_info', 'delete_collection_namespaced_config_map', 'delete_collection_namespaced_config_map_with_http_info', 'delete_collection_namespaced_endpoints', 'delete_collection_namespaced_endpoints_with_http_info', 'delete_collection_namespaced_event', 'delete_collection_namespaced_event_with_http_info', 'delete_collection_namespaced_limit_range', 'delete_collection_namespaced_limit_range_with_http_info', 'delete_collection_namespaced_persistent_volume_claim', 'delete_collection_namespaced_persistent_volume_claim_with_http_info', 'delete_collection_namespaced_pod', 'delete_collection_namespaced_pod_template', 'delete_collection_namespaced_pod_template_with_http_info', 'delete_collection_namespaced_pod_with_http_info', 'delete_collection_namespaced_replication_controller', 'delete_collection_namespaced_replication_controller_with_http_info', 'delete_collection_namespaced_resource_quota', 'delete_collection_namespaced_resource_quota_with_http_info', 'delete_collection_namespaced_secret', 'delete_collection_namespaced_secret_with_http_info', 'delete_collection_namespaced_service_account', 'delete_collection_namespaced_service_account_with_http_info', 'delete_collection_node', 'delete_collection_node_with_http_info', 'delete_collection_persistent_volume', 'delete_collection_persistent_volume_with_http_info', 'delete_namespace', 'delete_namespace_with_http_info', 'delete_namespaced_config_map', 'delete_namespaced_config_map_with_http_info', 'delete_namespaced_endpoints', 'delete_namespaced_endpoints_with_http_info', 'delete_namespaced_event', 'delete_namespaced_event_with_http_info', 'delete_namespaced_limit_range', 'delete_namespaced_limit_range_with_http_info', 'delete_namespaced_persistent_volume_claim', 'delete_namespaced_persistent_volume_claim_with_http_info', 'delete_namespaced_pod', 'delete_namespaced_pod_template', 'delete_namespaced_pod_template_with_http_info', 'delete_namespaced_pod_with_http_info', 'delete_namespaced_replication_controller', 'delete_namespaced_replication_controller_with_http_info', 'delete_namespaced_resource_quota', 'delete_namespaced_resource_quota_with_http_info', 'delete_namespaced_secret', 'delete_namespaced_secret_with_http_info', 'delete_namespaced_service', 'delete_namespaced_service_account', 'delete_namespaced_service_account_with_http_info', 'delete_namespaced_service_with_http_info', 'delete_node', 'delete_node_with_http_info', 'delete_persistent_volume', 'delete_persistent_volume_with_http_info', 'get_api_resources', 'get_api_resources_with_http_info', 'list_component_status', 'list_component_status_with_http_info', 'list_config_map_for_all_namespaces', 'list_config_map_for_all_namespaces_with_http_info', 'list_endpoints_for_all_namespaces','list_endpoints_for_all_namespaces_with_http_info', 'list_event_for_all_namespaces', 'list_event_for_all_namespaces_with_http_info', 'list_limit_range_for_all_namespaces', 'list_limit_range_for_all_namespaces_with_http_info', 'list_namespace', 'list_namespace_with_http_info', 'list_namespaced_config_map', 'list_namespaced_config_map_with_http_info', 'list_namespaced_endpoints', 'list_namespaced_endpoints_with_http_info', 'list_namespaced_event', 'list_namespaced_event_with_http_info', 'list_namespaced_limit_range', 'list_namespaced_limit_range_with_http_info', 'list_namespaced_persistent_volume_claim', 'list_namespaced_persistent_volume_claim_with_http_info', 'list_namespaced_pod', 'list_namespaced_pod_template', 'list_namespaced_pod_template_with_http_info', 'list_namespaced_pod_with_http_info', 'list_namespaced_replication_controller', 'list_namespaced_replication_controller_with_http_info', 'list_namespaced_resource_quota', 'list_namespaced_resource_quota_with_http_info', 'list_namespaced_secret', 'list_namespaced_secret_with_http_info', 'list_namespaced_service', 'list_namespaced_service_account', 'list_namespaced_service_account_with_http_info', 'list_namespaced_service_with_http_info', 'list_node', 'list_node_with_http_info', 'list_persistent_volume', 'list_persistent_volume_claim_for_all_namespaces', 'list_persistent_volume_claim_for_all_namespaces_with_http_info', 'list_persistent_volume_with_http_info', 'list_pod_for_all_namespaces', 'list_pod_for_all_namespaces_with_http_info', 'list_pod_template_for_all_namespaces', 'list_pod_template_for_all_namespaces_with_http_info', 'list_replication_controller_for_all_namespaces', 'list_replication_controller_for_all_namespaces_with_http_info', 'list_resource_quota_for_all_namespaces', 'list_resource_quota_for_all_namespaces_with_http_info', 'list_secret_for_all_namespaces', 'list_secret_for_all_namespaces_with_http_info', 'list_service_account_for_all_namespaces', 'list_service_account_for_all_namespaces_with_http_info', 'list_service_for_all_namespaces', 'list_service_for_all_namespaces_with_http_info', 'patch_namespace', 'patch_namespace_status', 'patch_namespace_status_with_http_info', 'patch_namespace_with_http_info', 'patch_namespaced_config_map', 'patch_namespaced_config_map_with_http_info', 'patch_namespaced_endpoints', 'patch_namespaced_endpoints_with_http_info', 'patch_namespaced_event', 'patch_namespaced_event_with_http_info', 'patch_namespaced_limit_range', 'patch_namespaced_limit_range_with_http_info', 'patch_namespaced_persistent_volume_claim', 'patch_namespaced_persistent_volume_claim_status', 'patch_namespaced_persistent_volume_claim_status_with_http_info', 'patch_namespaced_persistent_volume_claim_with_http_info', 'patch_namespaced_pod', 'patch_namespaced_pod_status', 'patch_namespaced_pod_status_with_http_info', 'patch_namespaced_pod_template', 'patch_namespaced_pod_template_with_http_info', 'patch_namespaced_pod_with_http_info', 'patch_namespaced_replication_controller', 'patch_namespaced_replication_controller_scale', 'patch_namespaced_replication_controller_scale_with_http_info', 'patch_namespaced_replication_controller_status', 'patch_namespaced_replication_controller_status_with_http_info', 'patch_namespaced_replication_controller_with_http_info', 'patch_namespaced_resource_quota', 'patch_namespaced_resource_quota_status', 'patch_namespaced_resource_quota_status_with_http_info', 'patch_namespaced_resource_quota_with_http_info', 'patch_namespaced_secret', 'patch_namespaced_secret_with_http_info', 'patch_namespaced_service', 'patch_namespaced_service_account', 'patch_namespaced_service_account_with_http_info', 'patch_namespaced_service_status', 'patch_namespaced_service_status_with_http_info', 'patch_namespaced_service_with_http_info', 'patch_node', 'patch_node_status', 'patch_node_status_with_http_info', 'patch_node_with_http_info', 'patch_persistent_volume', 'patch_persistent_volume_status', 'patch_persistent_volume_status_with_http_info', 'patch_persistent_volume_with_http_info', 'read_component_status', 'read_component_status_with_http_info', 'read_namespace', 'read_namespace_status', 'read_namespace_status_with_http_info', 'read_namespace_with_http_info', 'read_namespaced_config_map', 'read_namespaced_config_map_with_http_info', 'read_namespaced_endpoints', 'read_namespaced_endpoints_with_http_info', 'read_namespaced_event', 'read_namespaced_event_with_http_info', 'read_namespaced_limit_range', 'read_namespaced_limit_range_with_http_info', 'read_namespaced_persistent_volume_claim', 'read_namespaced_persistent_volume_claim_status', 'read_namespaced_persistent_volume_claim_status_with_http_info', 'read_namespaced_persistent_volume_claim_with_http_info', 'read_namespaced_pod', 'read_namespaced_pod_log', 'read_namespaced_pod_log_with_http_info', 'read_namespaced_pod_status', 'read_namespaced_pod_status_with_http_info', 'read_namespaced_pod_template', 'read_namespaced_pod_template_with_http_info', 'read_namespaced_pod_with_http_info', 'read_namespaced_replication_controller', 'read_namespaced_replication_controller_scale', 'read_namespaced_replication_controller_scale_with_http_info', 'read_namespaced_replication_controller_status', 'read_namespaced_replication_controller_status_with_http_info', 'read_namespaced_replication_controller_with_http_info', 'read_namespaced_resource_quota', 'read_namespaced_resource_quota_status', 'read_namespaced_resource_quota_status_with_http_info', 'read_namespaced_resource_quota_with_http_info', 'read_namespaced_secret', 'read_namespaced_secret_with_http_info', 'read_namespaced_service', 'read_namespaced_service_account', 'read_namespaced_service_account_with_http_info', 'read_namespaced_service_status', 'read_namespaced_service_status_with_http_info', 'read_namespaced_service_with_http_info', 'read_node', 'read_node_status', 'read_node_status_with_http_info', 'read_node_with_http_info', 'read_persistent_volume', 'read_persistent_volume_status', 'read_persistent_volume_status_with_http_info', 'read_persistent_volume_with_http_info', 'replace_namespace', 'replace_namespace_finalize', 'replace_namespace_finalize_with_http_info', 'replace_namespace_status', 'replace_namespace_status_with_http_info', 'replace_namespace_with_http_info', 'replace_namespaced_config_map', 'replace_namespaced_config_map_with_http_info', 'replace_namespaced_endpoints', 'replace_namespaced_endpoints_with_http_info', 'replace_namespaced_event', 'replace_namespaced_event_with_http_info', 'replace_namespaced_limit_range', 'replace_namespaced_limit_range_with_http_info', 'replace_namespaced_persistent_volume_claim', 'replace_namespaced_persistent_volume_claim_status', 'replace_namespaced_persistent_volume_claim_status_with_http_info', 'replace_namespaced_persistent_volume_claim_with_http_info', 'replace_namespaced_pod', 'replace_namespaced_pod_status', 'replace_namespaced_pod_status_with_http_info', 'replace_namespaced_pod_template', 'replace_namespaced_pod_template_with_http_info', 'replace_namespaced_pod_with_http_info', 'replace_namespaced_replication_controller', 'replace_namespaced_replication_controller_scale', 'replace_namespaced_replication_controller_scale_with_http_info', 'replace_namespaced_replication_controller_status', 'replace_namespaced_replication_controller_status_with_http_info', 'replace_namespaced_replication_controller_with_http_info', 'replace_namespaced_resource_quota', 'replace_namespaced_resource_quota_status', 'replace_namespaced_resource_quota_status_with_http_info', 'replace_namespaced_resource_quota_with_http_info', 'replace_namespaced_secret', 'replace_namespaced_secret_with_http_info', 'replace_namespaced_service', 'replace_namespaced_service_account', 'replace_namespaced_service_account_with_http_info', 'replace_namespaced_service_status', 'replace_namespaced_service_status_with_http_info', 'replace_namespaced_service_with_http_info', 'replace_node', 'replace_node_status', 'replace_node_status_with_http_info', 'replace_node_with_http_info', 'replace_persistent_volume', 'replace_persistent_volume_status', 'replace_persistent_volume_status_with_http_info', 'replace_persistent_volume_with_http_info']
```

이게 다가 아니었네..? 여기는 POD 얻어서 뭔가 하는 것 외엔 이렇다할 게 없음.

```
# 또다른 api_client 얻기
>>> api_client = client.AppsV1Api(client.ApiClient(config))

>>> dir(api_client)
['__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', 'api_client', 'create_namespaced_controller_revision', 'create_namespaced_controller_revision_with_http_info', 'create_namespaced_daemon_set', 'create_namespaced_daemon_set_with_http_info', 'create_namespaced_deployment', 'create_namespaced_deployment_with_http_info', 'create_namespaced_replica_set', 'create_namespaced_replica_set_with_http_info', 'create_namespaced_stateful_set', 'create_namespaced_stateful_set_with_http_info', 'delete_collection_namespaced_controller_revision', 'delete_collection_namespaced_controller_revision_with_http_info', 'delete_collection_namespaced_daemon_set', 'delete_collection_namespaced_daemon_set_with_http_info', 'delete_collection_namespaced_deployment', 'delete_collection_namespaced_deployment_with_http_info', 'delete_collection_namespaced_replica_set', 'delete_collection_namespaced_replica_set_with_http_info', 'delete_collection_namespaced_stateful_set', 'delete_collection_namespaced_stateful_set_with_http_info', 'delete_namespaced_controller_revision', 'delete_namespaced_controller_revision_with_http_info', 'delete_namespaced_daemon_set', 'delete_namespaced_daemon_set_with_http_info', 'delete_namespaced_deployment', 'delete_namespaced_deployment_with_http_info', 'delete_namespaced_replica_set', 'delete_namespaced_replica_set_with_http_info', 'delete_namespaced_stateful_set', 'delete_namespaced_stateful_set_with_http_info', 'get_api_resources', 'get_api_resources_with_http_info', 'list_controller_revision_for_all_namespaces', 'list_controller_revision_for_all_namespaces_with_http_info', 'list_daemon_set_for_all_namespaces', 'list_daemon_set_for_all_namespaces_with_http_info', 'list_deployment_for_all_namespaces', 'list_deployment_for_all_namespaces_with_http_info', 'list_namespaced_controller_revision', 'list_namespaced_controller_revision_with_http_info', 'list_namespaced_daemon_set', 'list_namespaced_daemon_set_with_http_info', 'list_namespaced_deployment', 'list_namespaced_deployment_with_http_info', 'list_namespaced_replica_set', 'list_namespaced_replica_set_with_http_info', 'list_namespaced_stateful_set', 'list_namespaced_stateful_set_with_http_info', 'list_replica_set_for_all_namespaces', 'list_replica_set_for_all_namespaces_with_http_info', 'list_stateful_set_for_all_namespaces', 'list_stateful_set_for_all_namespaces_with_http_info', 'patch_namespaced_controller_revision', 'patch_namespaced_controller_revision_with_http_info', 'patch_namespaced_daemon_set', 'patch_namespaced_daemon_set_status', 'patch_namespaced_daemon_set_status_with_http_info', 'patch_namespaced_daemon_set_with_http_info', 'patch_namespaced_deployment', 'patch_namespaced_deployment_scale', 'patch_namespaced_deployment_scale_with_http_info', 'patch_namespaced_deployment_status', 'patch_namespaced_deployment_status_with_http_info', 'patch_namespaced_deployment_with_http_info', 'patch_namespaced_replica_set', 'patch_namespaced_replica_set_scale', 'patch_namespaced_replica_set_scale_with_http_info', 'patch_namespaced_replica_set_status', 'patch_namespaced_replica_set_status_with_http_info', 'patch_namespaced_replica_set_with_http_info', 'patch_namespaced_stateful_set', 'patch_namespaced_stateful_set_scale', 'patch_namespaced_stateful_set_scale_with_http_info', 'patch_namespaced_stateful_set_status', 'patch_namespaced_stateful_set_status_with_http_info', 'patch_namespaced_stateful_set_with_http_info', 'read_namespaced_controller_revision', 'read_namespaced_controller_revision_with_http_info', 'read_namespaced_daemon_set', 'read_namespaced_daemon_set_status', 'read_namespaced_daemon_set_status_with_http_info', 'read_namespaced_daemon_set_with_http_info', 'read_namespaced_deployment', 'read_namespaced_deployment_scale', 'read_namespaced_deployment_scale_with_http_info', 'read_namespaced_deployment_status', 'read_namespaced_deployment_status_with_http_info', 'read_namespaced_deployment_with_http_info', 'read_namespaced_replica_set', 'read_namespaced_replica_set_scale', 'read_namespaced_replica_set_scale_with_http_info', 'read_namespaced_replica_set_status', 'read_namespaced_replica_set_status_with_http_info', 'read_namespaced_replica_set_with_http_info', 'read_namespaced_stateful_set', 'read_namespaced_stateful_set_scale', 'read_namespaced_stateful_set_scale_with_http_info', 'read_namespaced_stateful_set_status', 'read_namespaced_stateful_set_status_with_http_info', 'read_namespaced_stateful_set_with_http_info', 'replace_namespaced_controller_revision', 'replace_namespaced_controller_revision_with_http_info', 'replace_namespaced_daemon_set', 'replace_namespaced_daemon_set_status', 'replace_namespaced_daemon_set_status_with_http_info', 'replace_namespaced_daemon_set_with_http_info', 'replace_namespaced_deployment', 'replace_namespaced_deployment_scale', 'replace_namespaced_deployment_scale_with_http_info', 'replace_namespaced_deployment_status', 'replace_namespaced_deployment_status_with_http_info', 'replace_namespaced_deployment_with_http_info', 'replace_namespaced_replica_set', 'replace_namespaced_replica_set_scale', 'replace_namespaced_replica_set_scale_with_http_info', 'replace_namespaced_replica_set_status', 'replace_namespaced_replica_set_status_with_http_info', 'replace_namespaced_replica_set_with_http_info', 'replace_namespaced_stateful_set', 'replace_namespaced_stateful_set_scale', 'replace_namespaced_stateful_set_scale_with_http_info', 'replace_namespaced_stateful_set_status', 'replace_namespaced_stateful_set_status_with_http_info', 'replace_namespaced_stateful_set_with_http_info']
```

```
# list replica set in namespace 'app'
>>> [(x.metadata.name, x.spec.replicas) for x in api_client.list_namespaced_replica_set('app').items]
[('controller-5d7d5b6c', 0), ('controller-6746f5b4f9', 1), ('controller-77f7ff6445', 0), ('mongodb-6fb499c65', 1), ('mta-admin-775f9b49c', 1), ('mta-admin-86495799f7', 0), ('mta-alert-6df5c8774b', 0), ('mta-alert-78f5959488', 0), ('mta-alert-7d5d78988c', 1), ('mta-alert-7df795c9f', 0), ('nodejs-bot-5db6ffc7', 0), ('nodejs-bot-6548c48869', 0), ('nodejs-bot-65b96fb78', 1), ('nodejs-bot-6d8fdb9f8c', 0), ('user-5987cd4f4f', 1)]
# 없어진 리플리카셋도 남아있네..?
```

```
# get deployment 'controller' in namespace 'app'
>>> api_client.read_namespaced_deployment('controller','app')

# 참고로 위의 결과는 아래와 사실상 같다. ('api_version' , 'kind' 는 밑에 값이 없음)
>>> [x for x in api_client.list_namespaced_deployment('app').items if x.metadata.name == 'controller' ][0]
```





참조사이트: http://jason-heo.github.io/python/2020/12/19/kubernetes-python-api.html  


좀더 많은 예제를 보고 싶다면 https://github.com/kubernetes-client/python

API 리스트는 https://github.com/kubernetes-client/python/blob/master/kubernetes/README.md

