# JS-Console 연계

JS-Console 은 Keycloak 인증을 HTML/Javascript 기반으로 확인 및 테스트하고 자기 것으로 만들기 위한 간단한 어플리케이션입니다.
- Apache 기반에 웹파일로만 구성되어 있으며 파일도 몇 없습니다.

## 설치

https://github.com/anabaral/mta-js-console  여기 링크를 참조합니다.
- 이 저장소는 글 쓰는 현재 Private 상태인데 내용상 비공개일 필요는 없으므로 곧 공개하겠습니다.  

## 설정 (아직 성공 못함)

저도 해 봤는데 체크아웃 하고 바로 쓸 수 있는 상태는 아닙니다.
- 기본적으로는 다음 파일들의 수정이 필요합니다:
  * index.html (keycloak.js 파일의 위치 -- keycloak 사이트 URL 반영)
  * keycloak.json (keycloak 에 client 셋업한 후 얻는 json 내용을 반영)
- 위의 파일들을 수정하고도 실제로 작동을 하지 않습니다.
  * https://{keycloak_site}/auth/keycloak.js 파일을 분석해 보니 토큰을 받기 위한 호출을 하는데
  * 즉 https://keycloak.chatops.ga/auth/realms/chatops/protocol/openid-connect/token 를 호출할 때
  * client_secret 에 해당하는 정보를 보내지 않습니다. 
  * keycloak 서버는 그것 때문에 401 Error를 응답합니다.
  * keycloak.js 는 ( https://github.com/keycloak/keycloak/blob/master/adapters/oidc/js/src/main/resources/keycloak.js )  
    히스토리 등을 봐선 딱히 그걸 버그라고 간주하지 않는 것 같습니다만, 예전엔 어떻게 썼었는 지 모르겠네요

    
참조할 만한 사이트: https://bakery-it.tistory.com/43
- 여기 읽어보면 keycloak.js 를 사용하는 클라이언트는 secret을 사용할 수 없다고 나와 있음...!!
- 그래서 keycloak에서 access type을 public으로 맞추지 않는 한 로그인 불가.
  ![](./img/keycloak-client-accesstype.png)
- 그렇다고 access type을 public으로 맞추면 이번엔 CORS 문제가 나옴. js-console의 apache 설정을 건드려야 하나..?)

