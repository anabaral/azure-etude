# Azure 컨테이너 인스턴스 에서 앱 띄워보기

정말 기초 중의 기초만으로..

아래 링크에 사용되는 이미지를 기본으로 해서 조금 고쳐서 쓸 생각입니다.
https://github.com/anabaral/azure-etude/blob/master/aks-app.md 

아래 진행한 것들은 정확히는 시간 순서로 한 것은 아닙니다. 이렇게 했다가 에러가 나니 고치고 하는 과정을 거쳤기에 이미지 만들기와 컨테이너 만들기가 번갈아서 진행된 측면이 있습니다.

## 컨테이너 이미지 만들기

Dockerfile 제일 밑에 다음이 필요합니다. AKS에서는 Deployment 설정에 포함되어 있던 것입니다.
```
ENTRYPOINT [ "/bin/bash", "-ec", "npm start" ]
```

빌드 스크립트도 조정했습니다. 이름과 버전 바꾼 정도이지만:
```
$ cat build.sh
az acr login -n tuna01

NAME=node-ctr
VERSION=0.1
docker build -t ${NAME}:${VERSION} .
if [ $? -ne 0 ]; then echo 'fail' && exit 1 ; fi
docker tag ${NAME}:${VERSION} tuna01.azurecr.io/${NAME}:${VERSION}
if [ $? -ne 0 ]; then echo 'fail' && exit 1 ; fi
docker push tuna01.azurecr.io/${NAME}:${VERSION}
if [ $? -ne 0 ]; then echo 'fail' && exit 1 ; fi
```

nodejs 소스도 건드려야 합니다. 컨테이너를 하나만 띄울 거기 때문에 초기화할 때 mongodb를 찾지 않도록 바꿔줍니다:
```
//아래 부분들을 찾아 주석처리
//__var mongoose = require('mongoose');                               // mongoose for mongodb
//__var database = require('./config/database');                      // load the database config
//__mongoose.connect(database.remoteUrl);                         // connect to local MongoDB instance. A remoteUrl is also available (modulus.io)
```

만약 mongodb가 필요하다면 (저처럼 봇 접속에만 한정하지 않고 기존 소스의 로직을 적극 활용한다면) 소스를 건드리지 말고 다른 방향으로 셋업을 해주어야겠죠. 
- mongodb 를 띄움
- Dockerfile 을 수정해서 AKS에서 환경변수 설정한 것을 옮겨주기. 하나만 예로 들자면:
  ```
  ENV DATABASE_HOST mynode-mongodb
  ```


## 컨테이너 만들기

처음엔 웹 화면으로 진행했는데 결국 명령어로 실행했습니다:
```
$ az container create -g rg-tuna -n ctr-tuna-03 --image tuna01.azurecr.io/node-ctr:0.1  --ip-address Public --ports 3000 --environment-variables 'PORT'='3000'
```

위에서 ```--ip-address Public``` 주지 않으면 그 뒤의 포트 관련 설정이 다 안 먹힙니다.

그리고 웹 화면으로 진행할 때 경험한 것이긴 한데 IP address 를 공용(public)으로 주지 않으면 Virtual Network 및 Subnet 을 (아마도 새로 만들어서) 구성해야 하는데 거기에
무언가 숨은 제약이 있어서 배포 시점에 실패합니다. 사유는 대략:
- ServiceUnavailable 이라고 상태가 나오고
- 메시지는 "요청된 리소스를 지금은 'koreacentral' 위치에서 사용할 수 없습니다. 다른 리소스 요청으로 또는 다른 위치에서 다시 시도하세요. 요청된 리소스: '1' CPU '1.5'GB 메모리 'Linux' OS 가상 네트워크""
  라고만 나옵니다.

처음엔 리소스 증가 신청을 해야 하나, 혹은 virtual network을 기존의 것을 쓰면 안되나, 등등 여러 모로 시행착오를 했는데 결국 공용 IP주소 선택으로 넘어갔으며 아직 정확한 원인을 모릅니다.


