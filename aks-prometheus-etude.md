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


