# 일반적인 앱에서의 연동

다음은 현재까지 식별된 연동방식입니다.
![식별된 연동들](./img/keycloak-auth.svg)

첫 번째 방식은 js-console (https://github.com/anabaral/azure-etude/blob/master/keycloak/js-console-integration.md) 연동에서 사용한 방식입니다.
- keycloak측에서 제공하는 자바스크립트 (keycloak.js) 를 사용
- access_type = 'public' 에서만 가능
- 현재 찾은 바로는 node.js 에서는 서버사이드 지원 모듈이 존재하는데, python 은 뭔가 부족함

두번째 방식은 브라우저에서 1차로 연결하는 서버가 로그인을 제공하는 방식입니다.
- 브라우저에서 전달한 ID/PW 를 가지고 keycloak 인증은 1차 서버가 수행합니다.
- python-keycloak 모듈이 필요합니다.
- 2차 서버는 1차서버에서 받은 token(주로 access_token) 을 전달받아 인증을 할 수 있습니다. (권한정보까지는 받는 방법을 모르겠어요) 

세번째 방식은.. 아직 모든 걸 확인한 것은 아닙니다만
- client_id, client_secret 정보를 가지고 토큰을 얻어냅니다.
- 유저 로그인이 없이 토큰을 얻는 걸로 보아 관리자급의 권한을 가지는 게 아닌가 싶습니다.
- access_type = 'confidential' 에서만 가능

