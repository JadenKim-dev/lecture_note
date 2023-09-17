## Recreate 실습

다음의 구성 파일로 Deployment 객체를 생성한다.

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

이제 해당 파드들을 연결한 Service를 다음의 구성 파일로 생성한다.

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

이 때 Deployment 객체를 수정하여 template의 containers: image 내용을 변경하면, 새로운 이미지를 담은 파드로 업데이트할 수 있다.  
현재 배포 방식(strategy:type)을 Recreate로 지정했기 때문에 잠시 동안 요청이 불가능한 downtime이 존재한다.  
다음의 커맨드로 1초 주기로 요청을 보내서 파드의 버전이 변경되는 것을 확인할 수 있다.

```bash
$ while true; do curl 10.99.5.3:8080/version; sleep 1; done

Version : v1
Version : v1
Version : v1
curl: (7) Failed connect to 10.99.5.3:8080; // downtime
curl: (7) Failed connect to 10.99.5.3:8080;
curl: (7) Failed connect to 10.99.5.3:8080;
Version : v2
Version : v2
Version : v2
```

Deployment를 사용하면 손쉽게 롤백도 가능하다.  
kubectl 커맨드로 롤백을 진행할 경우, 먼저 다음의 커맨드로 현재 존재하는 revision을 조회할 수 있다.

```bash
$ kubectl rollout history deployment deployment-1
deployment.extensions/deployment-1
REVISION CHANGE-CAUSE
2        <none>
3        <none>

```

rollback해서 돌아가고 싶은 revision을 선택하고, 다음의 커맨드로 롤백을 수행한다.

```bash
$ kubectl rollout undo deployment deployment-1 --to-revision=2
deployment.extensions/deployment-1 rolled back
```

(Deployment 객체의 template의 image 값을 변경해서 rollback을 진행할 수도 있다.)

## RollingUpdate

RollingUpdate 방식의 배포를 진행하기 위해 다음과 같이 Deployment 객체를 생성할 수 있다.

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
        type: app2
    spec:
      containers:
        - name: container
          image: kubetm/app:v1
      terminationGracePeriodSeconds: 0
```

앞서 했던 것과 동일한 방식으로 업데이트를 진행할 수 있다.  
RollingUpdate 수행 시 중간에 v1, v2가 섞여서 요청이 처리되다가, v1 파드들이 모두 종료되면서 v2로 완전히 변경되는 식으로 작동한다.

```bash
$ while true; do curl 10.99.5.3:8080/version; sleep 1; done

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

Blue/Green 방식의 배포를 하고 싶다면, 새로운 버전 정보를 담고 있는 Deployment 객체를 새롭게 하나 더 생성해야 한다.
다음과 같이 selector, label에 새로운 버전이 연결된 Deployment를 생성한다.

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
        ver: v2
    spec:
      containers:
        - name: container
          image: kubetm/app:v1
      terminationGracePeriodSeconds: 0
```

위와 같이 생성한 후, 파드 생성까지 모두 완료되면 Service 객체를 해당 Deployment 객체에 연결시킨다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-3
spec:
  selector:
    ver: v2
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
```

새로운 버전에 연결한 뒤 예전 버전의 Deployment 객체는 삭제하거나, 롤백용으로 replicas를 0으로 만들고 유지할 수 있다.
