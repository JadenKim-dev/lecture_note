# Pod - Container, Label, NodeSchedule

하나의 파드에는 여러개의 컨테이너를 둘 수 있다.  
이 때 파드 내의 컨테이너들은 하나의 호스트를 공유하는 식으로 작동하며, 그 내에서 localhost:[port] 로 서로 통신이 가능하다.  
(따라서 파드 내 컨테이너들은 서로 다른 포트를 가지고 있어야 한다.)

각각의 파드들은 라벨을 가질 수 있다.
라벨은 키/밸류 형식으로 할당되며, 서비스의 selector를 이용해서 원하는 파드만 골라서 서비스에 연결하는 것이 기능하다.

파드는 자신이 실행될 노드를 골라야 하는데, 이 때 nodeSelector를 통해 라벨로 원하는 노드를 선택하거나, 자동으로 할당을 받게 된다.  
자동 할당 시 자신이 지정해 둔 메모리/CPU 자원에 따라 적절한 유휴 자원이 있는 노드에 할당된다.

각 파드 정의 시 리소스의 requests(최소 요구량), limits(최대 요구량)을 지정할 수 있다.  
무한정 리소스를 가져가는 것을 막기 위한 옵션이다.

노드의 전체 부하 상태가 OverCommit되면 자원 사용량을 적정 수준으로 낮추기 위한 조치를 취하게 된다.  
메모리의 경우 limits를 초과한 파드가 즉시 종료되고, cpu는 requests 수준으로 떨어질 때까지 기다린다.  
이는 메모리 사용량이 초과될 경우 프로그램 간 충돌이 나는 등 중대한 문제가 생길 수 있기 때문에 정해진 정책이다.

## 실습

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

파드가 생성되면 ip를 하나 할당 받고, 해당 ip로 클러스터 내에서 접근이 가능하다.  
실제로 마스터 노드에서 해당 ip의 port에 curl로 요청을 보내면 요청이 닿는다.

파드의 복제본 관리를 해주는 컨트롤러 객체를 다음과 같이 정의할 수 있다.

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

이로써 해당 컨트롤러에 연결된 파드는 언제나 한 개가 유지되도록 관리된다.  
실제로 파드 하나를 임의로 삭제하면 자동으로 파드가 새롭게 생성된다.  
이 때 파드에 할당된 ip는 변경된다.

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

위 파드에는 type: web 과 lo: dev 라벨을 붙였다.  
type: web인 파드만 선택하는 서비스 객체를 다음과 같이 정의할 수 있다.

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

파드를 정의할 때 자신이 실행될 노드를 선택하는 nodeSelector는 다음과 같이 정의할 수 있다.

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

위에서는 kubernetes.io/hostname: k8s-node1 라벨이 붙은 노드를 선택하고 있다.  
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
