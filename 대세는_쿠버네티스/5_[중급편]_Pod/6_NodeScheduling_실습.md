# Node Scheduling 실습

## NodeAffinity - MatchExpressions 실습

먼저 다음의 커맨드를 통해 노드1에 kr=az-1, 노드2에 us=az-1 라벨을 각각 적용한다.

```bash
kubectl label nodes k8s-node1 kr=az-1
kubectl label nodes k8s-node2 us=az-1
```

### 1. required

그 다음 NodeAffinity가 적용된 파드를 다음과 같이 생성한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-match-expressions1
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - { key: kr, operator: Exists }
  containers:
    - name: container
      image: kubetm/app
  terminationGracePeriodSeconds: 0
```

affinity에 nodeAffinity를 지정하고, required로 표현식을 적용하기 위해 `requiredDuringSchedulingIgnoredDuringExecution` 속성을 적용한다.  
그 하위의 `nodeSelectorTerms: matchExpressions`에 적용하고자 하는 표현식을 정의한다.

위 파드는 kr 키를 가진 노드를 선택하기 때문에, 노드1에 파드가 할당된다.  
해당 구성 파일로 수차례 파드를 생성해도 동일하게 노드1에 할당이 되는 것을 확인할 수 있다.

### 2. preferred

다음으로 preferred를 확인하기 위해, 모든 노드에 매칭되지 않는 matchExpressions를 조건식에 적용한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-preferred
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 1
          preference:
            matchExpressions:
              - { key: ch, operator: Exists }
  containers:
    - name: container
      image: kubetm/app
  terminationGracePeriodSeconds: 0
```

`preferredDuringSchedulingIgnoredDuringExecution` 속성을 적용하고, preference: matchExpressions에 원하는 조건식을 작성했다.  
현재 ch 키를 가진 노드는 존재하지 않기 때문에 만약 required로 만들었다면 파드가 pending 상태에 머물게 된다.  
위 경우에는 preferred로 만들었기 때문에 조건을 만족하지 않는 노드들에도 할당이 가능하고, 위 상태에서는 현재 파드가 없는 노드2에 파드가 할당된다.

## PodAffinity / PodAntiAffinity 실습

### PodAffinity

PodAffinity를 실습해보기 위해, 먼저 다음의 커맨드를 통해 node1에 a-team=1, node2에 a-team=2 라벨을 붙인다.

```bash
kubectl label nodes k8s-node1 a-team=1
kubectl label nodes k8s-node2 a-team=2
```

이제 다음의 구성 파일로 a-team=1 라벨을 가진 노드1에 파드를 생성한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web1
  labels:
    type: web1
spec:
  nodeSelector:
    a-team: "1"
  containers:
    - name: container
      image: kubetm/app
  terminationGracePeriodSeconds: 0
```

이제 위에서 생성한 파드와 동일한 노드에 파드를 생성하기 위해, podAffinity가 적용된 파드를 다음과 같이 생성한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: server1
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - topologyKey: a-team
          labelSelector:
            matchExpressions:
              - { key: type, operator: In, values: [web1] }
  containers:
    - name: container
      image: kubetm/app
  terminationGracePeriodSeconds: 0
```

`labelSelector: matchExpressions` 에 지정된 표현식에 따라 `type: web1` 라벨이 적용된 파드가 위치한 노드를 찾게 된다.  
이 때 topologyKey에 지정한 값에 따라 key가 a-team인 라벨이 적용된 노드 중에서 찾게 된다.  
최종적으로 방금 생성한 파드와 동일한 노드에 스케줄링된다.

만약 podAffinity에 적용한 조건에 맞는 파드가 기존에 존재하지 않는다면, 해당 파드는 pending 상태에 머물게 된다.  
이 때 조건에 맞는 파드를 추후에 생성하면, pending이 걸렸던 파드도 함께 동일한 노드에 생성된다.

### PodAntiAffinity

다음으로 PodAntiAffinity에 대한 실습이다.  
먼저 다음의 구성 파일로 master 파드를 노드1에 생성한다.  
`type: master` 라벨이 적용되어 있고, `nodeSelector: a-team: '1'` 을 통해 node1을 선택하여 파드를 생성하고 있다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: master
  labels:
    type: master
spec:
  nodeSelector:
    a-team: "1"
  containers:
    - name: container
      image: kubetm/app
  terminationGracePeriodSeconds: 0
```

이제 podAntiAffinity 속성을 통해 slave 파드를 master 파드와 다른 노드에 생성한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: slave
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - topologyKey: a-team
          labelSelector:
            matchExpressions:
              - { key: type, operator: In, values: [master] }
  containers:
    - name: container
      image: kubetm/app
  terminationGracePeriodSeconds: 0
```

matchExpressions 의 표현식을 통해 마스터 파드의 라벨을 특정해서, 다른 노드에 슬레이브 파드가 생성되도록 구성했다.

## Taint & Toleration 실습

### NoSchedule

먼저 다음의 커맨드로 node1에 라벨을 달고, Taint를 적용시킨다.

```bash
kubectl label nodes k8s-node1 gpu=no1
kubectl taint nodes k8s-node1 hw=gpu:NoSchedule
```

taint를 적용하는 커맨드에는 해당 taint의 key, value를 지정하고, 그 옆에 effect를 지정한다.  
이제부터는 toleration이 지정되지 않은 상태에서 node1에 파드를 할당하려고 하면 pending이 걸리게 된다.  
다음과 같이 매칭되는 toleration 설정이 파드 구성 파일에 포함되어야 정상적으로 파드가 생성된다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-toleration
spec:
  nodeSelector:
    gpu: no1
  tolerations:
    - effect: NoSchedule
      key: hw
      operator: Equal
      value: gpu
  containers:
    - name: container
      image: kubetm/app
  terminationGracePeriodSeconds: 0
```

Taint 적용시 지정한 key, value 와 effect가 동일하게 지정되어 있기 때문에, 정상적으로 node1에 파드가 생성된다.

### NoExecute

이번엔 NoExecute effect를 적용해 볼 것이다.  
먼저 다음의 구성 파일로 NoExecute toleration이 적용되어 있는 파드를 node2에 생성한다.  
tolerationSeconds는 30으로 적용되어 있다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1-with-no-execute
spec:
  tolerations:
    - effect: NoExecute
      key: out-of-disk
      operator: Exists
      tolerationSeconds: 30
  containers:
    - name: container
      image: kubetm/app
  terminationGracePeriodSeconds: 0
```

이 상태에서 다음의 커맨드로 노드2에 NoExecute Taint를 적용한다.

```bash
kubectl taint nodes k8s-node2 out-of-disk=True:NoExecute
```

이제 노드2에 할당되어 있던 일반 파드들이 모두 종료된다.  
위 파드는 `tolerationSeconds: 30` 으로 지정했기 때문에, 30초의 지연시간 후에 파드가 삭제 된다.  
(tolerationSeconds를 지정한 파드는 해당 초만큼 지연시간 후에 삭제 되고, 그냥 Toleration만 지정한 파드는 계속 노드에 남아 있게 된다.)
