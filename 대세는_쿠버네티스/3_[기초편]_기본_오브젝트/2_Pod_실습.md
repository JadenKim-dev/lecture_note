# Pod 실습

## Container 실습

먼저 2개의 컨테이너로 구성된 파드를 다음과 같이 구성한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-1
spec:
  containers:
    - name: container1
      image: kubetm/p8000
      ports:
        - containerPort: 8000
    - name: container2
      image: kubetm/p8080
      ports:
        - containerPort: 8080
```

컨테이너가 각각 8000번, 8080번 포트로 실행되었다.  
파드 내에서 해당 컨테이너 간에는 localhost로 요청을 주고받을 수 있다.

```bash
# 파드 내에서 실행
curl http://localhost:8000/name
curl http://localhost:8080/name
```

파드가 생성되면 ip를 하나 할당 받고, 해당 ip로 클러스터 내에서 접근이 가능하다.  
실제로 마스터 노드에서 해당 ip의 port에 curl 요청을 보내면 요청이 닿는다.

```bash
# 마스터 노드에서 실행
curl http://192.168.35.23:8000/name
curl http://192.168.35.23:8080/name
```

### 파드가 재생성되는 경우

파드의 복제본을 관리해주는 컨트롤러 객체를 다음과 같이 정의할 수 있다.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: replication-1
spec:
  replicas: 1
  selector:
    app: rc
  template:
    metadata:
      name: pod-1
      labels:
        app: rc
    spec:
      containers:
        - name: container
          image: kubetm/init
```

`replicas: 1`로 지정했기 때문에 해당 컨트롤러에 연결된 파드는 언제나 한 개가 유지되도록 관리된다.  
이 때 파드 하나를 임의로 삭제하면 자동으로 파드가 새롭게 생성되고, 파드에 할당된 ip가 변경된 것을 확인할 수 있다.

## Label 실습

파드를 정의할 때 다음과 같이 라벨을 붙이는 것이 가능하다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-2
  labels:
    type: web
    lo: dev
spec:
  containers:
    - name: container
      image: kubetm/init
```

위 파드에는 `type: web` 과 `lo: dev` 라벨을 붙였다.

```bash
$ kubectl describe pod pod-2

Labels:   lo=dev
          type=web
```

`type: web`인 파드만 선택하는 서비스 객체를 다음과 같이 정의할 수 있다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-1
spec:
  selector:
    type: web
  ports:
    - port: 8080
```

서비스의 상세 정보를 조회하면 지정된 selector를 확인할 수 있고, 해당 라벨을 가진 파드를 kubectl 명령어를 통해 조회할 수 있다.

```bash
$ kubectl get services -o=wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE    SELECTOR
svc-1        ClusterIP   10.231.58.59    <none>        8080/TCP   12m    type=web

$ kubectl get po --selector=type=web

NAME    READY   STATUS    RESTARTS   AGE
pod-2   1/1     Running   0          14m
```

## NodeSelector 실습

파드를 정의할 때 nodeSelector를 통해 자신이 실행될 노드를 선택할 수 있다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-3
spec:
  nodeSelector:
    kubernetes.io/hostname: k8s-node1
  containers:
    - name: container
      image: kubetm/init
```

위에서는 `kubernetes.io/hostname: k8s-node1` 라벨이 붙은 노드를 선택하고 있다.  
만약 노드를 선택하지 않았다면 requests, limits에 지정한 자원의 크기에 맞게 적절한 노드를 선택하여 할당한다.  
각 노드들 간에는 점수가 매겨져서 만약 여러 노드가 선택 가능하다면 그 중 유휴 자원이 더 많은 노드가 선택된다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-4
spec:
  containers:
    - name: container
      image: kubetm/init
      resources:
        requests:
          memory: 2Gi
        limits:
          memory: 3Gi
```

출처: [인프런 대세는 쿠버네티스 [초급 ~ 중급]](https://inf.run/yW34)
