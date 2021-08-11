# ArgoCD 에서 비번 변경

다음과 같이 `ArgoCD Server` POD에 들어가서 직접 로그인 및 변경 명령을 수행합니다.

```
$ kubectl exec -it -n cicd argocd-server-76b7c65745-cwlb5 -- bash
argocd@argocd-server-76b7c65745-cwlb5:~$ argocd
argocd controls a Argo CD server
...
설명
...

argocd@argocd-server-76b7c65745-cwlb5:~$ argocd login argocd.chatops.ga
WARN[0001] Failed to invoke grpc call. Use flag --grpc-web in grpc calls. To avoid this warning message, use flag --grpc-web.
Username: admin
Password:
'admin:login' logged in successfully
Context 'argocd.chatops.ga' updated

argocd@argocd-server-76b7c65745-cwlb5:~$ argocd account update-password
WARN[0003] Failed to invoke grpc call. Use flag --grpc-web in grpc calls. To avoid this warning message, use flag --grpc-web.
*** Enter new password:
*** Confirm new password:
Password updated
Context 'argocd.chatops.ga' updated

```
