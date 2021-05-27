# AKS 에 Prometheus 설치, 설정 및 사용

## 설치

AKS 설치는 아래 링크를 참조하여 진행:  
https://github.com/anabaral/azure-etude/blob/master/aks-setup.md

prometheus 설치 참조 링크는 https://github.com/helm/charts/tree/master/stable/prometheus  
(DEPRECATED and moved to https://github.com/prometheus-community/helm-charts 라고 나오는데 설정 옵션은 여전히 위 링크에 있음)  

실제 설치 내역은 아래 적음.
```
# 현재 헬름 저장소 확인
ds04226@Azure:~$ helm repo list
# 헬름 저장소 등록
ds04226@Azure:~$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

ds04226@Azure:~$ vi prometheus-values.yaml
alertmanager:
  persistentVolume:
    storageClass: "default"
server:
  persistentVolume:
    size: 4Gi
    storageClass: "default"

ds04226@Azure:~$ helm install prometheus prometheus-community/prometheus -f prometheus-values.yaml
```
위에선 네임스페이스를 지정하지 않았는데 그러면 기본 네임스페이스에 저장됨 (AKS 셋업 직후라면 default)

하지만 네임스페이스를 지정하는 게 낫겠지?

```
ds04226@Azure:~$ helm install prometheus    # 과감히 삭제.
ds04226@Azure:~$ kubectl create namespace monitoring
ds04226@Azure:~$ helm install prometheus prometheus-community/prometheus -n monitoring -f prometheus-values.yaml
```

다음을 체크해 보자:
- PersistentVolume 생성된 것
- PersistentVolumeClaim 생성된 것
- Prometheus-Server Pod 의 deployment descriptor 에서 PV 와 PVC를 확인

이제 뭘 해 볼까?  
가장 먼저 할 수 있는 일은 prometheus 서버에 직접 붙어 이것저것을 확인하는 것.  
그러기 위해서는 해당되는 서비스를 외부에 오픈해야 한다. [README.md](https://github.com/anabaral/azure-etude#aks%EC%97%90%EC%84%9C-%EC%84%9C%EB%B9%84%EC%8A%A4%EB%A5%BC-%EC%99%B8%EB%B6%80-%EC%97%B0%EA%B2%B0) 에서 서술했던 방법들 중 하나를 쓰면 되는데  
만약 kubernetes cluster에 ingress 관련하여 아무런 설정이 안된 상태라면 그 중 쉬우면서도 간단히 응용이 되는 2번째 방법으로 가보자.

방법은 이미 [고정ip에 ingress controller 깔면서 시험](https://github.com/anabaral/azure-etude/blob/master/aks-ing-try1.md) 에서 기술했었는데 

* public ip 생성
```
$ NODE_RES_GRP=$(az aks show --resource-group 04226 --name myAKSCluster --query nodeResourceGroup -o tsv)
$ az network public-ip create --resource-group ${NODE_RES_GRP} --name prometheus-public-ip --sku Standard --allocation-method static --query publicIp.ipAddress -o tsv
NNN.NNN.NNN.NNN
```

* DNS 영역 생성
  - 위 링크에서처럼 명령어로 할 수도 있고 포털 화면에서 생성해도 됨
  - 궁극적으로 prometheus.xxxx.yyyy 같은 도메인명으로 public ip 가 등록되면 됨 (주소로 등록해도 되는데 별칭 -- prometheus-public-ip -- 으로 등록해 보자)

* ingress controller 설치
  - 헬름 저장소 추가 (없을경우)
    ```
    $ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx 
    ```

  - ingress controller를 헬름으로 설치하는데 쓰인 파라미터 파일
    ```
    $ cat > ingress-values.yaml
    controller:
      replicaCount: 1
      nodeSelector:
        beta.kubernetes.io/os: linux
      service:
        loadBalancerIP: NNN.NNN.NNN.NNN  # 위에서 생성된 IP 기입
        annotations:
          service.beta.kubernetes.io/azure-dns-label-name: ingress # 이걸 왜 주었더라? 기억이...
    defaultBackend:
      nodeSelector:
        beta.kubernetes.io/os: linux
    ^D
    ```

  - ingress controller 설치 실행
    ```
    $ kubectl create ns ingress
    $ helm install nginx-ingress -n ingress-basic -f ingress-values.yaml ingress-nginx/ingress-nginx
    ... 메시지 나오는 걸 잘 보길 ...
    ```

* prometheus 헬름 설치
  - 저장소 추가
    ``` 
    $ helm repo list
    $ helm repo add bitnami https://charts.bitnami.com/bitnami
    ```
  - 파일 작성
    ```
    $ cat prometheus-values.yaml  # 작성된 파일 보기
    alertmanager:
      persistentVolume:
        storageClass: "default"
    server:
      persistentVolume:
        size: 4Gi
        storageClass: "default"
    ```
  - 설치 실행
    ```
    $ helm install prometheus prometheus-community/prometheus -n monitoring -f prometheus-values.yaml
    ```


## 설정

* 생성된 서비스 내용을 확인하고 
  ```
  ds04226@Azure:~$ kubectl get svc -n monitoring prometheus-server -o yaml
  apiVersion: v1
  kind: Service
  metadata:
    ...
    name: prometheus-server
    namespace: monitoring
    ...
  spec:
    clusterIP: 10.0.2.102
    ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 9090
    selector:
      app: prometheus
      component: server
      release: prometheus
    sessionAffinity: None
    type: ClusterIP
  status:
    loadBalancer: {}
  ```

* 인그레스 설정에 서비스를 반영
  - Azure DNS 설정 : 좀 날림으로 진행했음. 기존에 쓰던 설정이 남아있어서 prometheus.sk-az.net 으로 레코드만 추가
  - 다음을 작성
    ```
    ds04226@Azure:~$ cat prometheus-ing.yaml
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      annotations:
        appgw.ingress.kubernetes.io/ssl-redirect: "false"
      name: alertmanager-ingress
      namespace: monitoring
    spec:
      #tls:   # tls 설정은 아직 보류
      #- hosts:
      #  - prometheus.sk-az.net
      #  secretName: mta-infra-tls
      rules:
      - host: prometheus.sk-az.net
        http:
          paths:
          - backend:
              serviceName: prometheus-server    # 이름을 맞춤
              servicePort: http                 # 포트를 맞춤 (이름 혹은 숫자)
            path: /
    ```
  - 변경된 설정 적용
    ```
    $ kubectl apply -f prometheus-ing.yaml
    ```

## 사용

* ingress까지 설정이 되면 다음과 같이 `http://prometheus.sk-az.net` 으로 (혹은 따로 정한 도메인명으로) 접속이 가능합니다. 
  ![](https://github.com/anabaral/azure-etude/blob/master/img/prometheus-01.png)
* 초기화면이 쿼리를 시험해 볼 수 있는 graph 화면입니다.
* 가장 단순한 결과가 나오도록 쿼리를 실행한 예입니다:
  ![](https://github.com/anabaral/azure-etude/blob/master/img/prometheus-02.png)
* 이를 `curl` 로 재현하면 다음과 같이 나옵니다:
  ```
  $ curl --data-urlencode 'query=node_cpu_seconds_total{cpu="0",mode="idle"}' 'http://prometheus.sk-az.net/api/v1/query'  | jq
  {
    "status": "success",
    "data": {
      "resultType": "vector",
      "result": [
        {
          "metric": {
            "__name__": "node_cpu_seconds_total",
            "app": "prometheus",
            "app_kubernetes_io_managed_by": "Helm",
            "chart": "prometheus-13.8.0",
            "component": "node-exporter",
            "cpu": "0",
            "heritage": "Helm",
            "instance": "10.240.0.4:9100",
            "job": "kubernetes-service-endpoints",
            "kubernetes_name": "prometheus-node-exporter",
            "kubernetes_namespace": "monitoring",
            "kubernetes_node": "aks-nodepool1-17385196-vmss000005",
            "mode": "idle",
            "release": "prometheus"
          },
          "value": [
            1622021952.209,
            "13560.21"
          ]
        }
      ]
    }
  }
  ```
  * 쓸만하게 되려면 시각 혹은 시간 구간을 지정해야 하는데 그건 다음에...

## 메트릭들

메트릭은 exporter 를 호출하면 얻을 수 있는데, prometheus-server 도 그 자신이 exporter 역할을 한다. 즉
`curl http://prometheus.sk-az.net/metrics ` 과 같이 호출해 보면 자신이 측정하는 메트릭을 응답으로 내보낸다.

메트릭 목록을 얻고 싶다면 `curl http://prometheus.sk-az.net/api/v1/label/__name__/values` 를 호출하면 되는데 조금 더 다듬으면
```
ds04226@Azure:~$ cat prometheus-metrics-list.py
#!/usr/bin/python
import json
import urllib.request

hdr = {}
request = urllib.request.Request('http://prometheus.sk-az.net/api/v1/label/__name__/values', headers = hdr)
response = urllib.request.urlopen(request)
json = json.load(response)
if "status" in json and json["status"] == "success" :
    for name in json["data"]:
        print(name)

ds04226@Azure:~$ python prometheus-metrics-list.py | wc -l
797
```
797개나 되는 걸 어떻게 분석하지...?


