# Deployment 실습

먼저 파드들에 요청을 보낼 때 사용할 Service를 다음의 구성 파일로 생성한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-1
spec:
  selector:
    type: app
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
```

## Recreate 실습

### Deployment 생성

이제 다음의 구성 파일로 Deployment 객체를 생성한다.  
Recreate 방식의 배포를 진행하기 위해 `strategy: type: Recreate`로 설정한다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-1
spec:
  selector:
    matchLabels:
      type: app
  replicas: 2
  strategy:
    type: Recreate
  revisionHistoryLimit: 1
  template:
    metadata:
      labels:
        type: app
    spec:
      containers:
        - name: container
          image: kubetm/app:v1
      terminationGracePeriodSeconds: 10
```

replicas가 2로 지정 되어 있기 때문에 template에 지정한 내용을 바탕으로 파드가 2개 생성된다.  
이 때 파드들에는 pod-template-hash 라벨이 적용되어, 이를 통해 Deployment의 ReplicaSet에 연결되어 있다.

```bash
$ kubectl get po
NAME                            READY   STATUS    RESTARTS   AGE
deployment-1-6cd446c788-hxsvw   1/1     Running   0          17s
deployment-1-6cd446c788-rjwxs   1/1     Running   0          17s

$ kubectl describe po deployment-1-6cd446c788-hxsvw
Name:             deployment-1-6cd446c788-hxsvw
Namespace:        default
Labels:           pod-template-hash=6cd446c788
                  type=app
```

### Deployment를 통한 파드 업데이트

이 때 Deployment 객체를 수정하여 `template: containers: image` 내용을 변경하면, 새로운 이미지를 담은 파드로 업데이트할 수 있다.  
현재 배포 방식을 Recreate로 지정했기 때문에 잠시 동안 요청이 불가능한 downtime이 존재한다.  
다음의 커맨드로 1초 주기로 요청을 보내서 파드의 버전이 변경되는 것을 확인할 수 있다.

```bash
$ while true; do curl svc-1.default.svc.cluster.local:8080/version; sleep 1; done

Version : v1
Version : v1
Version : v1
curl: (7) Failed to connect to svc-1.default.svc.cluster.local port 8080: Connection refused  # downtime
curl: (7) Failed to connect to svc-1.default.svc.cluster.local port 8080: Connection refused
curl: (7) Failed to connect to svc-1.default.svc.cluster.local port 8080: Connection refused
Version : v2
Version : v2
Version : v2
```

### 이전 버전으로 롤백

Deployment를 사용하면 손쉽게 롤백도 가능하다.  
kubectl 커맨드로 롤백을 진행할 경우, 먼저 다음의 커맨드로 현재 존재하는 revision을 조회할 수 있다.

```bash
$ kubectl rollout history deployment deployment-1
deployment.extensions/deployment-1
REVISION  CHANGE-CAUSE
1         <none>
2         <none>

```

rollback해서 돌아가고 싶은 revision을 선택하고, 다음의 커맨드로 롤백을 수행한다.

```bash
$ kubectl rollout undo deployment deployment-1 --to-revision=2
deployment.extensions/deployment-1 rolled back
```

(Deployment 객체의 template의 image 값을 변경해서 rollback을 진행할 수도 있다.)

## RollingUpdate 실습

### Deployment 생성

RollingUpdate 방식의 배포를 진행하기 위해 먼저 다음과 같이 Deployment 객체를 생성한다.  
배포 전략을 RollingUpdate로 하기 위해 `strategy: type: RollingUpdate`로 속성을 지정한다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-2
spec:
  selector:
    matchLabels:
      type: app2
  replicas: 2
  strategy:
    type: RollingUpdate
  minReadySeconds: 10
  template:
    metadata:
      labels:
        type: app
    spec:
      containers:
        - name: container
          image: kubetm/app:v1
      terminationGracePeriodSeconds: 0
```

### Deployment 업데이트

앞서 했던 것과 동일한 방식으로 업데이트를 진행할 수 있다.  
RollingUpdate 수행 시 중간에 v1, v2가 섞여서 요청이 처리되다가, v1 파드들이 모두 종료되면서 v2로 완전히 변경되는 식으로 작동한다.

```bash
$ while true; do curl svc-1.default.svc.cluster.local:8080/version; sleep 1; done

Version : v1
Version : v1
Version : v1
Version : v2 // RollingUpdate 시작
Version : v1
Version : v2
Version : v1
Version : v2
Version : v2 // 업데이트 완료
Version : v2
```

## Blue/Green

Blue/Green 방식의 배포를 하고 싶다면, 새로운 버전의 파드를 연결한 Deployment 객체를 새롭게 하나 더 생성해야 한다.  
먼저 v1의 파드들을 연결할 Deployment를 생성한다.  
해당 파드들에는 `ver: v1` 라벨이 달려 있어 해당 라벨로 Deployment에 의해 선택된다.  
또한 `type: app` 라벨이 함께 달려 있다.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replica1
spec:
  replicas: 2
  selector:
    matchLabels:
      ver: v1
  template:
    metadata:
      name: pod1
      labels:
        type: app
        ver: v1
    spec:
      containers:
        - name: container
          image: kubetm/app:v1
      terminationGracePeriodSeconds: 0
```

이번에는 다음과 같이 selector, label에 새로운 버전이 연결된 Deployment를 생성한다.  
해당 파드들은 `ver: v2` 라벨로 Deployment에 의해 선택되고, type: app` 라벨이 함께 달려 있다.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replica1
spec:
  replicas: 2
  selector:
    matchLabels:
      ver: v2
  template:
    metadata:
      name: pod1
      labels:
        type: app
        ver: v2
    spec:
      containers:
        - name: container
          image: kubetm/app:v2
      terminationGracePeriodSeconds: 0
```

위와 같이 생성한 후, 파드 생성까지 모두 완료되면 v1, v2 파드들 모두에 달려 있는 `type: app` 라벨을 선택하는 Service 객체를 생성한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-3
spec:
  selector:
    type: app
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
```

이제 v1, v2 파드들이 모두 연결되어 트래픽을 받게 된다.  
새로운 버전의 파드의 정상 동작을 테스트한 후에는 예전 버전의 Deployment의 replicas를 0으로 만들어서 연결한 파드를 삭제하고, 롤백용으로 Deployment 객체는 유지할 수 있다.  
롤백용이 필요없다면 Deployment 객체를 삭제해서 연결된 파드들까지 한 번에 삭제할 수 있다.

출처: [인프런 대세는 쿠버네티스 [초급 ~ 중급]](https://inf.run/yW34)
