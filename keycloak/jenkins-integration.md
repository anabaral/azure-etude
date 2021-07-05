# Keycloak - Jenkins 연동

Jenkins 에서 Keycloak 을 통한 로그인을 하도록 설정해보겠습니다.

## Keycloak Side

![Keycloak_Client설정](./img/jenkins-keycloak-01.png)

이 글을 처음 작성하는 현재 `ChatOps` 렐름에 `devops` 클라이언트가 이미 생성되어 있습니다. 
이걸 적절히 편집할 건데, 우선 Settings 탭에서 다음을 손봅니다:
- Access Type ID: confidential
- Valid Redirect URIs: `https://jenkins.chatops.ga/*`
- Web Origins: `https://jenkins.chatops.ga/*`
- 저장.


![Keycloak_OIDC_JSON갈무리](./img/jenkins-keycloak-02.png)
동일 클라이언트에서 Installation 탭이 있습니다. 여기서 
- Format Option: Keycloak OIDC JSON 을 선택하면
- 나오는 JSON 텍스트를 갈무리 해 둡니다.


## Jenkins Side

![Jenkins_JSON붙여넣기](./img/jenkins-keycloak-11.png)
- Jenkins 관리 / 시스템 설정 에 들어가서 Global Keycloak Settings 카테고리로 간 후
  * Keycloak JSON 란을 위에 갈무리한 텍스트로 채웁니다.
  * 저장.


![Jenkins_Auth선택](./img/jenkins-keycloak-12.png)
- Jenkins 관리 / Configure Global Security 로 들어가서
  * Security Realm 의 선택항목을 `Keycloak Authentication Plugin` 으로 선택합니다.
  * 위의 것에 문제가 있을 경우 [저장]한 후 로그인을 못할 수 있으니 꼼꼼히 체크하세요.
  * 저장.

로그아웃 하면 다시 로그인 창 뜰 때 Keycloak 것이 뜰 겁니다.


## Troubleshooting

보안 설정 바꿀 때, 혹은 되돌릴 때 잘못하면 로그인을 못하는 상황이 벌어집니다. 이 때 간단 처치하는 방법을 적습니다.
다만 이 방법은 Jenkins 버전이 올라감에 따라 바뀌거나 없어질 수 있습니다.

- Jenkins Pod 에 들어갑니다.
  ```
  $ kubectl exec -it -n cicd jenkins-xxxxxxxxxx-yyyyy -c jenkins -- bash
  jenkins@jenkins-xxxxxxxxxx-yyyyy:/$ 
  ```
- `config.xml` 파일이 있는 곳을 찾아갑니다. 제 경우는 대략
  ```
  jenkins:/$ cd /bitnami/jenkins/home
  jenkins:/$ ls config.xml
  config.xml
  ```
- 파일 내용을 확인합니다. 요런 형태면 변경 가능합니다.
  ```
  jenkins:/$ more config.xml
  ...
    <useSecurity>true</useSecurity>
  ...
  ```
- 파일을 변경해 보안사용을 해제합니다.
  ```
  jenkins:/$ sed -i 's/^  <useSecurity>true<\/useSecurity>$/  <useSecurity>false<\/useSecurity>/' config.xml
  jenkins:/$ more config.xml  # 고쳐졌는지 확인
  ```
- Pod 를 나가서 해당 Pod를 재시작 합니다.
  ```
  jenkins:/$ exit
  $ kubectl delete po -n cicd jenkins-xxxxxxxxxx-yyyyy 
  ```
- 몇 분의 시간을 기다린 후 접속하면 로그인 없이 접속됩니다.
- 당연히 다른 사람들도 접속 가능하니 위험한 상태죠. 얼른 보안 접속 상태로 설정을 바꿉시다.
  * Configure Global Security 화면에서 
  * Security Realm 항목 수정 (초기 설정은 아마 Jenkins’ own user database )
  * Authorization 항목 수정 (초기 설정은 아마 Logged-in users can do anything 이고 anonymous read access 는 막는 걸로)


## 더 해 볼 만한 것

* Keycloak 에서 Role을 부여할 수 있습니다.
* Jenkins의 Configure Global Security 화면에서, Authorization 을 부여할 수 있습니다.
  - 현재 기본값은 '로그인한 사람은 누구나 관리자권한' 상태입니다.
  - 'Matrix based securiry' 로 바꾸는 게 맞겠죠.

