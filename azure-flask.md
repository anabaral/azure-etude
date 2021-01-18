# Azure 에서 flask application 을 띄워보자.

## 발단

flask app 을 사용할 기회가 생겨서 써보려고 합니다.

기존에는 다음과 같이 사용했던 환경이 있는데
  1. 개인 PC에서 Virtualbox 를 띄우고 개인 개발 (DB도 PC에 두거나 혹은 개발서버에서)
  1. 사무실 환경에 개발서버를 두고 통합 개발 (개발DB 존재)
  1. 완성된 어플리케이션은 실 운영 서버에 디플로이
개인개발까지는 터치하지 않고 개발서버는 Azure Cloud에 한 번 둬 보는 걸로 하려 합니다.

## 개요

그런데 이걸 VM을 띄워서 하는 것은 비용은 조금 절감되겠지만 가용성 측면에서 좀 아니다 싶어서
최소 3-Node의 Kubernetes 를 띄우고 거기에 Pod 형태로 뜨게 해 보려고 합니다.

그런데 지금까지는 -- 이를테면 Nodejs -- 어느 정도 정식 버전의 helm 배포판이 있어서 그걸 이용했었는데
Flask는 개인이 오픈한 이미지는 있어도 정식 릴리즈 스러운 게 안 보이는지라..

이미지를 적당히 만들어서 셋업해야 할 것 같아요..

일단 PC에서의 환경을 설정하는 것부터..

## PC에서

https://docs.microsoft.com/ko-kr/azure/developer/python/configure-local-development-environment?tabs=bash#required-components  
좀 찾아보니 다음이 필요할 것 같습니다.
- 활성 구독이 있는 Azure 계정
  . 있습니다. 계속 쓸 수 있을 지는 모르겠지만
- Python 2.7+ 또는 3.5.3+  <-- 3.8 정도는 깔아줘야 할 듯
- Azure CLI 설치
  . https://docs.microsoft.com/ko-kr/cli/azure/install-azure-cli-windows?tabs=azure-cli
- git 설치 (github을 써야 하니까)

여기까지 깔았으면 터미널을 열고 로그인 한 번 해 봅니다.
```
az login
```

그리고 나면 python 프로그램이 이 권한을 가지고 실행할 수 있게 해야 하고
```
(아직)
```

python-azure 연결 라이브러리 설치해야 하고
```
pip install (아직)
```

----------- 여기까지가 PC에서 테스트가능하게 하는 준비 ------------

그러고 나면 k8s 환경 셋업해야 하고
```
(아직)
```

그게 되면 k8s 의 python 프로그램이 권한을 가지게 해 봐야 하고
```
(아직)
```

키값이 k8s 환경 안에 암호화되어 있어야 하니까 그걸 좀 해 줘야 하고
```
(아직)
```

쓸데없는 비용이 나가지 않도록 한밤중에는 k8s 노드 수를 0개로 조정하고 (혹은 k8s를 중단하거나)
```
(아직)
```





