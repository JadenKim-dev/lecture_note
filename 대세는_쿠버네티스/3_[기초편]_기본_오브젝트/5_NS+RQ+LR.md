# Namespace, ResourceQuota, LimitRange

클러스터가 가진 cpu, 메모리 등의 자원이 있으면, 이를 각각의 네임스페이스가 공유해서 사용한다.  
만약 한 네임스페이스에서 지나치게 많은 자원을 사용한다면, 다른 클러스터의 파드들은 정상 동작하지 못할 수 있다.  
이를 방지하기 위해 각 네임스페이스가 사용할 수 있는 최대 자원 양인 ResourceQuata를 지정할 수 있다.

또한 각 네임스페이스 안에서도 특정 파드가 지나치게 많은 자원을 사용한다면 다른 파드들이 생성되지 못할 수 있다.  
이를 방지하기 위해 하나의 파드가 사용할 수 있는 최대 자원양인 LimitRange를 지정할 수 있다.

ResourceQuata와 LimitRange는 각 네임스페이스 뿐만 아니라 클러스터 전체에 대해서도 적용할 수 있다.

## Namespace

네임스페이스 내의 오브젝트는 서로 다른 이름을 가지고 있어야 한다.  
따라서 네임 스페이스의 한 리소스 내에서는 이름을 식별자로 사용할 수 있다.

네임스페이스 내의 객체들은 서로 별도로 관리된다.  
PV, Node 처럼 네임스페이스 밖에 존재하여 공유되는 자원도 있지만, 안에 있는 객체들은 명확히 분리된다.

예를 들어 Service 객체에서 다른 ns의 파드를 연결하는 것은 불가능하다.  
다만 다른 ns에 있는 파드들 간 ip로 통신하는 것은 Network Policy에 따라 금지될 수도, 허용될 수도 있다.  
(아무 것도 지정하지 않으면 허용됨)

한 ns를 삭제하면 그 안에 존재하는 자원들도 함께 삭제된다.

네임 스페이스는 다음과 같이 간단히 구성할 수 있다.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nm-1
```

각 객체를 생성할 때 metadata에 어떤 ns에 생성될지 지정할 수 있다.

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

## ResourceQuata

ResourceQuata는 Namespace의 총 자원 스펙을 정의하는 객체이다.  
특정 Namespace에 ResourceQuata를 붙였으면 해당 ns에 생성되는 모든 객체들은 requests, limits 등의 스펙을 명시해야 한다.  
또한 자원 사용량의 합계가 ResourceQuata에서 지정한 requests, limits 값을 초과시키는 객체는 추가가 불가능하다.  
ResourceQuata는 원할 경우 클러스터에 붙이는 것도 가능하다

ResourceQuata를 정의할 때에는 다음과 같이 연결할 ns의 이름을 metadata에 지정하고, 자원 스펙을 spec hard에 명시한다.

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

## LimitRange

LimitRange는 해당 ns에 들어올 수 있는 객체의 스펙을 제한하는 객체이다.  
min에는 최소 스펙, max에는 최대 스펙, maxLimitRequestRatio에는 requests - limits의 최대 비율을 지정할 수 있다.  
해당 조건에 맞지 않는 객체들은 ns에 생성될 수 없다.  
또한 defaultRequest, default 값을 설정하면 스펙을 지정하지 않은 객체들은 해당 스펙으로 객체를 생성한다.

LimitRange를 정의할 때에는 어떤 타입의 객체에 대한 것인지를 명시한다.  
각 객체마다 정의할 수 있는 항목이 다르므로 확인이 필요하다.  
LimitRange는 Namespace에만 연결할 수 있으며, metadata에 연결할 ns를 명시한다.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: lr-5
spec:
  limits:
    - type: Container
      min:
        memory: 0.1Gi
      max:
        memory: 0.5Gi
      maxLimitRequestRatio:
        memory: 1
      defaultRequest:
        memory: 0.5Gi
      default:
        memory: 0.5Gi
```

# 실습

## Namespace

Pod와 Service는 서로 같은 Namespace 내에 있을 때 연결될 수 있다.  
다만 다른 ns의 파드끼리의 IP 트래픽이 기본적으로 막혀있지 않다.(서로의 ip에 curl 요청이 가능)  
이를 막기 위해서는 NetworkPolicy 정의가 필요하다.

ns와 무관하게 생성되는 오브젝트들도 있다.  
Node의 특정 port를 할당받는 nodePort 타입의 Service 객체는 port를 중복해서 생성이 불가능하다.  
또한 노드의 특정 디렉토리를 공유하는 hostPath 타입의 Volume을 사용하면, 다른 ns의 파드에서 해당 디렉토리에 hostPath Volume을 연결할 경우 공유해서 사용하게 된다.  
이는 기본적으로 Pod에 root 권한을 부여하기 때문이다.  
추후 Pod의 SecretPolicy를 정의하여 제한된 user 권한을 부여하는 식으로 조치해야 한다.

## ResourceQuata

ResourceQuata를 ns에 적용한 후에는 다음의 명령어로 확인할 수 있다.

```bash
kubectl describe resourcequotas --namespace=nm-3
```

requests 및 limit 제약과 pods(파드 개수) 제약을 넘어서서 객체를 생성하려고 하면 막히는 것을 확인할 수 있다.

다만 이미 Pod가 생성되어 있는 상태에서 ResourceQuata를 적용하면 기존에 연결되어 있던 파드들은 제약 사항들에 영향을 받지 않는다.  
따라서 파드가 없는 상태에서 ResourceQuata를 적용해야 한다.

## LimitRange

LimitRange를 ns에 적용한 후에는 다음의 명령어로 확인할 수 있다.

```bash
kubectl describe limitranges --namespace=nm-5
```

이 때 파드 생성 시 min, max, maxLimitRequestRatio, defaultRequest, default 설정 값이 잘 적용되는 것을 확인할 수 있다.  
이 때 주의할 점은 하나의 ns에 여러개의 LimitRange를 적용할 경우 값에 충돌이 발생해서 예상치 못한 동작을 할 수 있다는 점이다.  
가능하면 하나의 LimitRange만 적용해서 사용하는게 적절하다

출처: [인프런 대세는 쿠버네티스 [초급 ~ 중급]](https://inf.run/yW34)
