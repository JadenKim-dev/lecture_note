# Namespace Resource 실습

## Namespace 기본 실습

먼저 다음의 구성 파일로 네임스페이스를 생성한다.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nm-1
```

그리고 해당 네임스페이스 안에 Pod, Service를 만든다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-1
  namespace: nm-1
  labels:
    app: pod
spec:
  containers:
    - name: container
      image: kubetm/app
      ports:
        - containerPort: 8080
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-1
  namespace: nm-1
spec:
  selector:
    app: pod
  ports:
    - port: 9000
      targetPort: 8080
```

해당 네임스페이스에 생성된 객체들을 다음과 같이 조회할 수 있다.

```bash
$ kubectl get all --namespace=nm-1

NAME                            READY   STATUS    RESTARTS   AGE
pod/pod-1                       1/1     Running   0          77s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/svc-1        ClusterIP   10.231.20.229   <none>        9000/TCP   70s
```

이 상태에서 동일한 이름의 파드를 한 번 더 생성하려고 하면 다음과 같은 에러가 발생한다.

```bash
$ kubectl create -f pod.yaml
Error from server (AlreadyExists): error when creating "pod.yaml": pods "pod-1" already exists
```

또한 Service에서 다른 네임스페이스에 있는 Pod를 selector로 선택했을 때, 연결이 되지 않는 것을 확인할 수 있다.

다만 다른 네임스페이스의 파드끼리의 IP 트래픽이 기본적으로 막혀있지 않다.(서로의 ip에 curl 요청이 가능)  
다른 네임스페이스의 파드나 서비스의 IP로 요청을 보내도 요청이 잘 닿는 것을 확인할 수 있다.

네임스페이스와 무관하게 생성되는 객체들도 있다.  
노드의 특정 포트를 할당받는 NodePort 타입의 Service 객체는 다른 네임스페이스이더라도 nodePort를 중복해서 생성하면 에러가 발생한다.

```bash
$ kubectl create -f service.yaml
The Service "svc-2" is invalid: spec.ports[0].nodePort: Invalid value: 30000: provided port is already allocated
```

또한 노드의 특정 디렉토리를 공유하는 hostPath 타입의 Volume을 사용하면, 다른 네임스페이스의 파드에서 해당 디렉토리에 hostPath Volume을 연결할 경우 공유해서 사용하게 된다.  
이는 기본적으로 Pod에 root 권한을 부여하기 때문이다.  
Pod의 Security 정책을 정의하여 제한된 user 권한을 부여하는 식으로 조치해야 한다.

## ResourceQuata

다음의 구성 파일로 ResourceQuata를 생성한다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq-1
  namespace: nm-3
spec:
  hard:
    requests.memory: 1Gi
    limits.memory: 1Gi
```

다음의 명령어로 네임스페이스에 생성한 ResourceQuata의 상세 정보를 확인할 수 있다.

```bash
kubectl describe resourcequotas --namespace=nm-3
Name:            rq-1
Namespace:       nm-3
Resource         Used  Hard
--------         ----  ----
limits.memory    0     1Gi
requests.memory  0     1Gi
```

이렇게 ResourceQuota가 적용된 상태에서 파드를 생성할 때에는, resource 제약 사항을 적지 않고 생성하면 에러가 발생한다.

```bash
$ kubectl apply -f pod.yaml
Error from server (Forbidden): error when creating "pod.yaml": pods "pod-2" is forbidden: failed quota: rq-1: must sp
ecify limits.memory,requests.memory
```

또한 requests 및 limits 제약 사항을 넘어서는 파드를 생성하려고 하면 에러가 발생한다.

```bash
$ kubectl apply -f pod.yaml
Error from server (Forbidden): error when creating "pod.yaml": pods "pod-4" is forbidden: exceeded quota: rq-1, requested: limits.memory=858993459200m,requests.memory=858993459200m, used: limits.memory=512Mi,requests.memory=512Mi, limited: limits.memory=1Gi,requests.memory=1Gi
```

다음과 같이 ResourceQuota를 통해 파드 개수에 제약을 거는 것도 가능하다.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq-1
  namespace: default
spec:
  hard:
    pods: 2
```

```bash
kubectl apply -f pod.yaml
Error from server (Forbidden): error when creating "pod.yaml": pods "pod-4" is forbidden: exceeded quota: rq-1, requested : pods=1, used: pods=2, limited: pods=2
```

다만 이미 Pod가 생성되어 있는 상태에서 ResourceQuata를 적용하면 기존에 연결되어 있던 파드들은 제약 사항들에 영향을 받지 않는다.
따라서 파드가 없는 상태에서 ResourceQuata를 적용해야 한다.

## LimitRange

다음의 구성 파일로 LimitRange를 생성한다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: lr-1
  namespace: nm-5
spec:
  limits:
    - type: Container
      min:
        memory: 0.1Gi
      max:
        memory: 0.4Gi
      maxLimitRequestRatio:
        memory: 3
      defaultRequest:
        memory: 0.1Gi
      default:
        memory: 0.2Gi
```

LimitRange의 상세 정보를 다음의 명령어로 확인할 수 있다.

```bash
kubectl describe limitranges --namespace=nm-5
Name:       lr-1
Namespace:  nm-5
Type        Resource  Min            Max            Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---            ---            ---------------  -------------  -----------------------
Container   memory    107374182400m  429496729600m  107374182400m    214748364800m  3
```

이 때 파드 생성 시 min, max, maxLimitRequestRatio, defaultRequest, default 설정 값이 잘 적용되는 것을 확인할 수 있다.

```bash
$ kubectl apply -f pod.yaml
error when creating "pod.yaml": pods "pod-4" is forbidden: [maximum memory usage per Container is 429496729600m, but limit is 858993459200m, memory max limit to request ratio per Container is 3, but provided ratio is 8.000000]
```

이 때 주의할 점은 하나의 ns에 여러개의 LimitRange를 적용할 경우 값에 충돌이 발생해서 예상치 못한 동작을 할 수 있다는 점이다.  
가능하면 하나의 LimitRange만 적용해서 사용하는게 적절하다.

출처: [인프런 대세는 쿠버네티스 [초급 ~ 중급]](https://inf.run/yW34)
