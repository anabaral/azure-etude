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
  ```
  ## 헬름 저장소 (없을경우) 추가
  $ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx 
  ```

* 헬름 설치에 쓰인 파라미터 파일
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

* 설치
  ```
  $ kubectl create ns ingress
  $ helm install nginx-ingress -n ingress -f ingress-values.yaml ingress-nginx/ingress-nginx
  ... 메시지 나오는 걸 잘 보길 ...
  ```

* 서비스 내용을 확인하고 

* 인그레스 설정







