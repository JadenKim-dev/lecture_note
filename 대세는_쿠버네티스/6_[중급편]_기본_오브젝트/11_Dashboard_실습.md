# Kubernetes Dashboard 실습

다음의 커맨드로 2.0.0 버전의 쿠버네티스 대시보드를 설치한다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.0/aio/deploy/recommended.yaml
```

위 커맨드를 실행하면 kubernetes-dashboard 네임스페이스 내에 필요한 오브젝트들이 생성된다.

다음의 구성파일로 ClusterRoleBinding을 생성해서, cluster-admin 롤을 Service Account에 바인딩한다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard2
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard
```

이제 인증서에 등록할 정보를 가져오기 위해 kubeconfig 내에 포함되어 있는 client crt, client key 정보를 추출해서 각각 파일로 저장한다.

```bash
$ grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> client.crt
$ grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> client.key
```

다음의 커맨드로 client.crt, client.key 두 파일의 내용을 합쳐서 하나의 p12 파일로 만들 수 있다.

```bash
$ openssl pkcs12 -export -clcerts -inkey client.key -in client.crt -out client.p12 -name "k8s-master-30"
```

해당 파일을 이용하여 인증서를 등록하면 큐버네티스 대시보드에 https로 접근할 수 있다.

대시보드에 접근하면 토큰 값을 입력하는 창이 나온다.  
다음의 커맨드로 secret 객체에 포함된 토큰 값을 추출한다.

```bash
$ kubectl -n kubernetes-dashboard get secret kubernetes-dashboard-token- \-o jsonpath='{.data.token}' | base64 --decode
```
