# Service 실습

## ClusterIP

먼저 서비스에 연결할 파드를 생성한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-1
  labels:
    app: pod
spec:
  nodeSelector:
    kubernetes.io/hostname: k8s-node1
  containers:
    - name: container
      image: kubetm/app
      ports:
        - containerPort: 8080
```

ClusterIP 타입의 서비스 객체는 다음과 같이 구성할 수 있다.  
(type을 지정하지 않으면 자동으로 ClusterIP로 구성된다)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-1
spec:
  selector:
    app: pod
  ports:
    - port: 9000
      targetPort: 8080
```

이 경우 `app: pod` 라벨을 적용한 파드에 서비스가 연결된다.  
클러스터 내에서 서비스에 할당된 IP에 curl 요청을 보내면 잘 닿지만, 클러스터 밖의 로컬 컴퓨터에서 요청을 보내면 닿지 않는 것을 확인할 수 있다.

```bash
$ kubectl get svc

NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
svc-1        ClusterIP   10.231.58.59    <none>        9000/TCP   47m

# 클러스터 내에서 요청
$ curl 10.231.58.59:9000/hostname

Hostname : pod-1

# 클러스터 밖에서 요청
$ curl 10.231.58.59:9000/hostname

curl: (7) Failed to connect to 10.231.58.59 port 9000: Connection refused
```

파드 삭제 후 재생성해도 서비스 객체에 파드가 잘 연결되고, 서비스의 동일한 IP로 파드에 요청을 보낼 수 있다.

## NodePort

NodePort 타입의 서비스 객체는 다음과 같이 구성할 수 있다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-2
spec:
  selector:
    app: pod
  ports:
    - port: 9000
      targetPort: 8080
      nodePort: 30000
  type: NodePort
```

위와 같이 구성 시 클러스터 내에서 접근할 수 있는 ClusterIP + 포트와 외부에서 접근할 수 있는 노드 포트가 둘 다 생성된다.

```bash
$ kubectl get svc -o=wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE    SELECTOR
svc-2        NodePort    10.231.33.34    <none>        9000:30000/TCP   109s   app=pod

$ kubectl describe svc svc-2
IP:                       10.231.33.34
Port:                     <unset>  9000/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  30000/TCP
```

노드의 internal IP를 이용해서 30000번 포트로 요청을 보내면 파드에 정상적으로 접근되는 것을 확인할 수 있다.

```bash
$ kubectl get node -o=wide
NAME            STATUS   ROLES                     AGE    VERSION    INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-node1       Ready    worker                    45d    v1.19.16   10.95.33.92   <none>        Ubuntu 20.04.4 LTS   5.4.0-121-generic   docker://19.3.14
k8s-node2       Ready    worker                    45d    v1.19.16   10.95.33.99   <none>        Ubuntu 20.04.4 LTS   5.4.0-121-generic   docker://19.3.14

# 내부망에서 요청
$ curl 10.95.33.92:30000/hostname
Hostname : pod-1
```

### NodePort의 트래픽 분산

또한 NodePort 서비스에 파드를 여러 개 연결한 후, 해당 NodePort로 수차례 요청을 보내면 파드들에 트래픽이 분산되어 전달되는 것을 확인할 수 있다.  
어떤 노드에 요청을 보냈는지와 관계 없이 랜덤하게 요청이 전달된다.

```yaml
$ curl 10.95.33.92:30000/hostname
Hostname : pod-1
$ curl 10.95.33.92:30000/hostname
Hostname : pod-2
$ curl 10.95.33.92:30000/hostname
Hostname : pod-2
$ curl 10.95.33.92:30000/hostname
Hostname : pod-1
```

### externalTrafficPolicy: Local 옵션 설정

NodePort 서비스에 `externalTrafficPolicy: Local` 옵션을 추가하면 요청한 노드에 존재하는 파드로만 요청이 전달된다.  
node1에 요청을 보내면 pod1에, node2에 요청을 보내면 pod2에 요청이 전달되는 것이다.  
만약 해당 노드에 서비스에 연결된 파드가 없으면 요청이 닿지 않는다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-2
spec:
  selector:
    app: pod
  ports:
    - port: 9000
      targetPort: 8080
      nodePort: 30000
  type: NodePort
  externalTrafficPolicy: Local
```

```bash
# node1의 IP로 요청
$ curl 10.95.33.92:30000/hostname
Hostname : pod-1
# node2의 IP로 요청
$ curl 10.95.33.99:30000/hostname
Hostname : pod-2
```

## LoadBalancer

LoadBalancer 타입의 서비스는 아래와 같이 구성할 수 있다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-3
spec:
  selector:
    app: pod
  ports:
    - port: 9000
      targetPort: 8080
  type: LoadBalancer
```

이 경우 자동으로 클러스터 IP와 포트, 노드 포트가 할당 되는 것을 확인할 수 있다.  
다만 외부 라이브러리가 연결되지 않아서 External IP는 발급 받지 못한 상태이다.

```bash
kubectl get svc

NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
svc-3        LoadBalancer   10.231.6.174    <pending>     9000:30605/TCP   4s
```

출처: [인프런 대세는 쿠버네티스 [초급 ~ 중급]](https://inf.run/yW34)
