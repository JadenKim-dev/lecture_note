# Authentication 실습

## Client crt 실습

### Client key, crt 가져오기

마스터 노드에서 다음 경로에 kubeconfig 가 위치해있다.  
`/etc/kubernetes/admin.conf`

admin.conf의 파일 내용을 확인해보면, 그 안에 CA crt, Client crt, Client CA가 Base64 인코딩 되어 저장되어 있다.

```properties
cluster.certificate-authority-data : CA.crt (Base64)
user.client-certificate-data: Client.crt (Base64)
user.client-key-data: Client.key (Base64)
```

```bash
$ cat /etc/kubernetes/admin.conf
clusters:
- cluster:
    certificate-authority-data : CA.crt (Base64)
    server: https://192.168.0.30:6443
  name: kubernetes

contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes

users:
- name: kubernetes-admin
  user:
    client-certificate-data: Client.crt (Base64)
    client-key-data: Client.key (Base64)
```

위에 있는 crt, key 내용은 모두 Base64로 인코딩 되어 있으므로, 다음의 커맨드를 통해 Client crt와 Client key를 디코딩해서 가져온다.

```bash
$ grep 'client-certificate-data' /etc/kubernetes/admin.conf | head -n 1 | awk '{print $2}' | base64 -d

$ grep 'client-key-data' /etc/kubernetes/admin.conf | head -n 1 | awk '{print $2}' | base64 -d
```

위 명령어를 통해 얻은 내용을 각각 client.crt, client,key 파일을 생성하여 붙여넣기 한다.

### Client key, crt 이용하여 요청 보내기

이제 해당 crt, key 파일을 이용해서 쿠버네티스 API 서버에 요청을 보낸다.
노드 정보를 조회하는 endpoint에 요청을 보낸다.

```bash
# case1) postman
https://192.168.56.30:6443/api/v1/nodes

Settings > General > SSL certificate verification > OFF
Settings > Certificates > Client Certificates > Host, CRT file, KEY file

# case2) curl
curl -k --key ./Client.key --cert ./Client.crt https://192.168.56.30:6443/api/v1/nodes
```

인증서가 적용되어 정상적으로 요청에 응답이 온다.

### Proxy 띄우고 요청 보내기

Proxy 서버를 띄우면 http 요청을 보내서 API Server에 접근하는 것이 가능하다.
이를 위해서는 먼저 다음의 커맨드로 kubeadm / kubectl / kubelet 을 설치해야 한다.

```bash
$ yum install -y --disableexcludes=kubernetes kubelet-1.22.0-0.x86_64 kubeadm-1.22.0-0.x86_64 kubectl-1.22.0-0.x86_64
```

이제 설치 과정에서 생성된 admin.conf 인증서를 kubectl config로 옮겨온다.

```bash
# admin.conf 인증서 복사
$ mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

이제 다음의 커맨드로 8001번 포트에 프록시를 띄운다.  
accept-hosts 에 프록시 접근을 허용할 IP를 지정할 수 있다.

```bash
$ nohup kubectl proxy --port=8001 --address=192.168.56.30 --accept-hosts='^*$' >/dev/null 2>&1 &
```

이제 다음의 url로 프록시에 http 요청을 보낼 수 있다.

```bash
# case1) postman
http://192.168.56.30:8001/api/v1/nodes

# case2) curl
curl http://192.168.56.30:8001/api/v1/nodes
```

## kubectl 다중 클러스터 실습

kubectl을 이용해 복수 개의 클러스터에 접근하는 실습을 위해서, 각 클러스터의 CA crt, Client crt, Client key 값을 이용해 하나의 admin.conf 파일로 합쳐야 한다.  
최종적으로 합쳐진 파일 형식은 아래와 같다.

```yaml
apiVersion: v1
clusters:
  - cluster:
      certificate-authority-data: ClusterA.CA.crt(Base64)
      server: https://192.168.0.30:6443
    name: cluster-a
  - cluster:
      certificate-authority-data: ClusterB.CA.crt(Base64)
      server: https://192.168.0.50:6443
    name: cluster-b
contexts:
  - context:
      cluster: cluster-a
      user: admin-a
    name: context-a
  - context:
      cluster: cluster-b
      user: admin-b
    name: context-b
current-context: context-a
kind: Config
preferences: {}
users:
  - name: admin-a
    user:
      client-certificate-data: ClusterA.Client.crt (Base64)
      client-key-data: ClusterA.Client.key (Base64)
  - name: admin-b
    user:
      client-certificate-data: ClusterB.Client.crt (Base64)
      client-key-data: ClusterB.Client.key (Base64)
```

이 상태에서 다음의 명령어로 특정 context로 kubectl config를 수행하면, 해당 context에 연결된 클러스터의 자원에 접근할 수 있다.

```bash
kubectl config use-context context-a
kubectl get nodes
```

## Service Account

먼저 Namespace를 하나 만들고, 함께 만들어진 Service Account의 내용을 조회한다.
default 라는 이름으로 Service Account 가 생성되어 있고, default-token-61krz 이름의 Secret 이 연결되어 있다.

```
$ kubectl create ns nm-01

$ kubectl describe -n nm-01 serviceaccounts
Name: default
Namespace: nm-01
Labels: <none>
Annotations: <none>
Image pull secrets: <none>
Mountable secrets: default-token-61krz
Tokens: default-token-61krz
Events: <none>
```

이제 다음의 커맨드로 Secret을 조회한다.  
내용으로 CA 인증서와 토큰 값이 포함되어 있다.

```
$ kubectl describe -n nm-01 secrets
ca.crt:     1025 bytes
namespace:  5 bytes
token: <Token>
```

여기서 확인된 토큰을 Bearer Token으로 입력하여 요청을 보내면 API Server에 접근하는 것이 가능하다.

```
# case1) http
# [header] Authorization : Bearer TOKEN
https://192.168.56.30:6443/api/v1

# case2) curl
curl -k -H "Authorization: Bearer TOKEN" https://192.168.56.30:6443/api/v1
```

다만 아직 해당 Service Account에 권한이 부여되지 않았기 때문에, 파드 등의 자원에 접근하는 것은 불가능한 상태이다.  
권한을 부여하는 방법을 다음 챕터에서 확인한다.
