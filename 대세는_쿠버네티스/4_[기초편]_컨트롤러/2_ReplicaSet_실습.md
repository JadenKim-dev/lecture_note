# ReplicaSet 실습

## ReplicaSet 기본 생성 방법

먼저 다음의 구성파일로 파드를 생성한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  labels:
    type: web
spec:
  containers:
    - name: container
      image: kubetm/app:v1
  terminationGracePeriodSeconds: 0
```

`terminationGracePeriodSeconds: 0` 옵션은 파드 삭제 요청시 즉시 객체가 삭제 되도록 하기 위한 옵션이다.  
(기본적으로는 30초의 지연 시간 후에 파드가 삭제된다.)

이제 다음의 구성파일로 파드를 연결한 ReplicaSet을 생성한다.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replica1
spec:
  replicas: 1
  selector:
    matchLabels:
      type: web
  template:
    metadata:
      name: pod1
      labels:
        type: web
    spec:
      containers:
        - name: container
          image: kubetm/app:v1
      terminationGracePeriodSeconds: 0
```

selector에 `type: web` 으로 지정했기 때문에 방금 생성한 파드가 연결된다.

```bash
$ kubectl get replicaset
NAME                  DESIRED   CURRENT   READY   AGE
replica1              1         1         1       8s

$ kubectl describe replicaset replica1
Name:         replica1
Namespace:    default
Selector:     type=web
Replicas:     1 current / 1 desired
Pods Status:  1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  type=web
  Containers:
   container:
    Image:        kubetm/app:v1
    ...
```

이 때 replicas를 2로 변경하여 scale out하면 자동으로 template에 지정한 정보를 바탕으로 파드가 추가 생성되는 것을 확인할 수 있다.

```bash
# replicas 값을 2로 변경
$ kubectl edit replicaset replica1

$ kubectl get po
NAME             READY   STATUS    RESTARTS   AGE
pod1             1/1     Running   0          5m4s
replica1-mh2zn   1/1     Running   0          9s  # 추가 생성된 파드
```

> template의 metadata name을 pod1으로 지정했으나, 추가로 생성되는 파드는 name이 replica1~~~ 로 랜덤 문자열이 붙는다.  
> 하나의 ns에는 동일한 이름의 객체가 여러개 있으면 안 되기 때문에 생긴 조치

## ReplicaSet 삭제

ReplicaSet을 삭제하면 해당 컨트롤러에 연결된 파드가 함께 삭제된다.

파드는 남겨두고 컨트롤러만 삭제하기 위해서는 다음과 같이 옵션을 줘서 커맨드로 삭제해야 한다.

```bash
kubectl delete replicaset replica1 --cascade=false
```

해당 커맨드로 ReplicaSet을 삭제해도 연결되었던 파드는 그대로 남아있는 것을 확인할 수 있다.  
이전과 동일한 구성 파일로 ReplicaSet을 다시 생성하면 selector에 지정한 내용이 따라 파드들을 선택해서, 기존의 파드들이 자동으로 연결된다.

## matchExpressions

ReplicaSet에서 matchExpressions를 사용하는 경우는 거의 없다.  
보통 matchExpressions는 기존에 존재하는 오브젝트들을 라벨을 이용해서 세밀하게 선택할 때 사용한다.  
하지만 ReplicaSet은 template을 이용해서 새롭게 파드를 생성하고 이를 연결하기 때문에, 필요성이 떨어진다.

> 보통 matchExpressions는 파드에서 여러 label들을 이용해서 Node를 스케줄링할 때 많이 사용한다.

matchExpressions를 사용한 ReplicaSet은 다음과 같이 구성할 수 있다.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replica1
spec:
  replicas: 1
  selector:
    matchLabels:
      type: web
      ver: v1
    matchExpressions:
      - { key: type, operator: In, values: [web] }
      - { key: ver, operator: Exists }
  template:
    metadata:
      labels:
        type: web
        ver: v1
        location: dev
    spec:
      containers:
        - name: container
          image: kubetm/app:v1
      terminationGracePeriodSeconds: 0
```

ReplicaSet에서 selector를 지정할 때 주의할 점은, **selector에서 지정한 라벨 조건들이 template의 라벨들에 포함되어야 한다**는 것이다.  
그래야 template으로 생성된 파드가 컨트롤러에 연결된다.  
제대로 지정을 하지 않으면 ReplicaSet 생성이 불가능하다.

matchExpressions에는 여러 가지 조건을 넣는게 가능하다.  
다만 이 경우에도 해당 조건들에 부합하는 라벨이 templete의 라벨로 등록되어 있어야 한다.

출처: [인프런 대세는 쿠버네티스 [초급 ~ 중급]](https://inf.run/yW34)
