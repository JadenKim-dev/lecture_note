# Replication Controller, ReplicaSet - Template, Replicas, Selector

컨트롤러는 **서비스의 운영을 도와주는 여러 기능을 제공**한다.  
**Auto Healing**은 파드 또는 파드가 스케줄된 노드에 문제가 생겼을 때, 이를 자동으로 감지해서 다른 노드에 새로운 파드를 띄우는 식으로 자동 복구를 해주는 기능이다.  
**Auto Scaling**은 특정 파드의 자원 사용량이 limit에 가까워질 때, 자동으로 새로운 파드를 띄워서 부하를 분담시키는 기능이다.  
또한 서비스의 업데이트가 필요할 때 업데이트 대상이 되는 파드를 손쉽게 변경해주며, 이 과정에서 문제가 발생했을 때 롤백을 하는 기능을 제공한다.  
**Job**의 경우 특정 작업을 수행해야 할 때 일시적으로 자원을 할당해서 작업을 진행하고, 작업이 종료되면 자동으로 자원을 반납하는 기능이다.

Controller 중 **Replication Controller**는 현재 deprecated된 객체이고, 여기에 몇몇 기능이 추가되어 **ReplicaSet**이 만들어졌다.  
Replication Controller도 **template**, **replicas** 기능을 제공하지만, **selector** 기능은 ReplicaSet에서만 지원한다.

## template

먼저 template 기능에 대한 설명이다.  
Controller는 selector를 통해 연결할 파드의 Label을 지정한다.

이 때 `template: spec`에는 유사시에 컨트롤러에서 생성할 파드의 정보를 입력해둔다.  
`containers`에 해당 파드에 생성할 컨테이너의 정보 (name, image)를 지정하면, 해당 컨테이너들을 담은 파드를 생성하는 식이다.  
만약 Replication Controller에 연결한 파드에 문제가 발생하면, 컨트롤러에서는 template에 지정한 정보를 바탕으로 파드를 새롭게 생성하여 연결한다.

이러한 특성을 이용해서 서비스의 업데이트를 진행하는 것도 가능하다.  
template에 파드 정보를 새롭게 갱신하고, 기존 파드를 임의로 정지시키면 새로운 버전의 파드를 생성해서 연결한다.

다음의 구성 파일로 type: web 라벨을 가진 파드를 연결한 ReplicaSet을 생성할 수 있다.

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
```

## replicas

**replicas**를 통해 생성해 둘 파드의 개수를 지정할 수 있다.  
ReplicaSet은 현재 존재하는 파드의 개수가 replicas에 미치지 못하면, 자동으로 template에 지정한 파드를 필요한 개수만큼 생성한다.
예를 들어 replicas를 3으로 지정한 상태로 ReplicaSet을 생성한 뒤 연결된 파드 중 2개를 정지시키면, 자동으로 2개가 새롭게 생성되어 replicas 개수만큼 채우게 된다.

replicas와 template을 지정하면 따로 파드 객체를 생성하지 않아도 된다.  
ReplicaSet만 생성하면 연결된 파드 개수가 0개 임을 인식하여 자동으로 replicas 값만큼 파드를 생성한다.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replica2
spec:
  replicas: 3
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
```

## selector

Replication Controller와 달리 ReplicaSet에는 selector 기능이 존재한다.  
**matchLabels**의 경우 key:value 형식으로 명시하면, 정확히 key, value가 모두 일치하는 파드를 찾는다.  
이와 달리 **matchExpressions**에는 Exists, DoesNotExists, In, NotIn 등의 표현식을 사용해서 파드 선택 규칙을 지정할 수 있다.  
이렇게 되면 특정 키가 존재하는지, 값이 특정 후보들에 해당하는지 등을 기준으로 파드를 선택할 수 있다.

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
    matchExpressions:
      - { key: ver, operator: Exists }
  template:
    metadata:
      labels:
        type: web
```

# 실습

## ReplicaSet 기본 생성 방법

먼저 다음의 구성파일로 파드를 생성한다

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
이 때 replicas를 2로 변경하여 scale out하면 자동으로 template에 지정한 정보를 바탕으로 파드가 추가 생성되는 것을 확인할 수 있다.

> template의 metadata name을 pod1으로 지정했으나, 추가로 생성되는 파드는 name이 replica1~~~ 로 랜덤 문자열이 붙는다. 하나의 ns에는 동일한 이름의 객체가 여러개 있으면 안 되기 때문에 생긴 조치

## ReplicaSet 삭제

ReplicaSet을 삭제하면 해당 컨트롤러에 연결된 파드가 함께 삭제된다.  
컨트롤러만 삭제하기 위해서는 다음과 같이 옵션을 줘서 커맨드로 삭제해야 한다.

```
kubectl delete replicationcontrollers replication1 --cascade=false

```

해당 커맨드로 ReplicaSet을 삭제해도 연결되었던 파드는 그대로 남아있는 것을 확인할 수 있다.  
위 구성 파일로 ReplicaSet을 다시 생성하면 selector에 지정한 내용이 따라 파드들을 선택해서, 기존의 파드들이 자동으로 연결된다.

## matchExpressions

ReplicaSet에서 matchExpressions를 사용하는 경우는 거의 없다.  
보통 matchExpressions는 기존에 존재하는 오브젝트들을 라벨을 이용해서 세밀하게 선택할 때 사용한다.  
하지만 ReplicaSet은 template을 이용해서 새롭게 파드를 생성하고 이를 연결하기 때문에, 필요성이 떨어진다.

보통은 다음과 같이 파드에서 여러 label들을 이용해서 Node를 스케줄링할 때 많이 사용한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-node-affinity1
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIngnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
  	       - {key: AZ-01, operator: Exists}
  containers:
  - name: container
    image: kubetm/init
```

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
