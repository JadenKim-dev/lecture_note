# Autoscaler 실습

## Metrics Server 설치

Autoscaler를 실습하기 위해서는 먼저 Metrics Server를 설치해야 한다.  
다음의 커맨드로 git으로 부터 Metrics Server 설치를 위한 구성 파일들을 가져와서 설치한다.

```bash
$ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.5.0/components.yaml
```

이 때 deplyment 객체를 통해서 metrics-server가 생성되는데, kubelet 통신을 insecure로 변경하기 위해 containers의 argument에 kubelet-insecure-tls를 추가해야 한다.

```bash
$ kubectl edit deployment metrics-server -n kube-system

#spec:
#  containers:
#  - args:
#    - --cert-dir=/tmp
#    - --secure-port=443
#    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
#    - --kubelet-use-node-status-port
#    - --metric-resolution=15s
#    - --kubelet-insecure-tls  <-- 추가
```

이제 metrics-server가 정상적으로 동작하는지 확인해보자.

```bash
$ kubectl get apiservices | egrep metrics

------------------------
v1beta1.metrics.k8s.io                     kube-system/metrics-server   True        28m
------------------------
```

그리고 설치된 지 1-2분이 지나면 `kubectl top` 명령어를 통해 metric 정보를 확인하는 것이 가능하다.  
CPU, Memory 정보가 표시된다.

```bash
$ kubectl top node

------------------------
NAME             CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-master       485m         9%     4852Mi          32%
k8s-node1        413m         8%     4929Mi          33%
k8s-node2        554m         11%    4672Mi          31%
------------------------
```

## HPA 실습

이제 본격적으로 HPA를 실습해보자.  
먼저 HPA를 적용할 Deployment를 생성한다.
request cpu가 10m, limits cpu가 100m인 파드를 2개 생성하도록 구성했다.  
또한 30001번의 NodePort를 할당한 서비스를 생성해서 해당 파드에 트래픽을 줄 수 있도록 구성했다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stateless-cpu1
spec:
  selector:
    matchLabels:
      resource: cpu
  replicas: 2
  template:
    metadata:
      labels:
        resource: cpu
    spec:
      containers:
        - name: container
          image: kubetm/app:v1
          resources:
            requests:
              cpu: 10m
            limits:
              cpu: 20m
---
apiVersion: v1
kind: Service
metadata:
  name: stateless-svc1
spec:
  selector:
    resource: cpu
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30001
  type: NodePort
```

이제 해당 Deployment를 스케일링할 HPA를 생성해보자.  
`maxReplicas: 10`, `minReplicas: 2`로 최대/최수 파드 개수를 정의했다.  
그리고 metrics에 `type: Resource`로 정의하고, resource에 `name: cpu`로 지정해서 cpu를 기반으로 스케일링을 수행하는 HPA를 생성한다.  
target에는 `type: Utilization`, `averageUtilization: 50` 으로 지정했다.  
`averageUtilization: 50` 이기 때문에 CPU의 평균 사용량이 `10m * 0.5 = 5m`이 되면 스케일링이 시작된다.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-resource-cpu
spec:
  maxReplicas: 10
  minReplicas: 2
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: stateless-cpu1
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

이제 다음의 명령어로 HPA의 상태를 조회하는데, -w 옵션을 줘서 실시간으로 조회하도록 한다.  
이러면 HPA 정보가 변경되는 것을 실시간으로 확인할 수 있다.  
실시간 리소스 현황이 변경되거나 스케일링이 이루어지면 HPA의 상태가 추가로 확인된다.

```bash
$ kubectl get hpa -w

NAME              REFERENCE                   TARGETS    MINPODS  MAXPODS  REPLICAS  AGE
hpa-resource-cpu  Deployment/deployment-cpu1  0%/50%     2        10       2         15s
```

이제 다음의 명령어로 0.03초마다 계속 요청을 보내서 파드에 부하를 준다.

```bash
while true;do curl 192.168.56.30:30001/hostname; sleep 0.01; done
```

이 때 `kubectl get hpa -w`로 확인되는 HPA의 상태값에서 리소스 사용량이 150% 수준으로 증가하고, 이에 따라서 replicas가 10개까지 증가하는 것을 확인할 수 있다.

```bash
$ kubectl get hpa -w

NAME              REFERENCE                   TARGETS    MINPODS  MAXPODS  REPLICAS  AGE
hpa-resource-cpu  Deployment/deployment-cpu1  0%/50%     2        10       2         15s
hpa-resource-cpu  Deployment/deployment-cpu1  150%/50%   2        10       2         15s
hpa-resource-cpu  Deployment/deployment-cpu1  150%/50%   2        10       4         15s
hpa-resource-cpu  Deployment/deployment-cpu1  150%/50%   2        10       6         15s
hpa-resource-cpu  Deployment/deployment-cpu1  57%/50%    2        10       6         15s
hpa-resource-cpu  Deployment/deployment-cpu1  60%/50%    2        10       8         15s
hpa-resource-cpu  Deployment/deployment-cpu1  60%/50%    2        10       8         15s
hpa-resource-cpu  Deployment/deployment-cpu1  50%/50%    2        10       10        15s
```

이제 부하를 주기 위해 실행하던 명령어를 종료해서 부하를 감소시켜 보자.  
scale out 되어 replicas가 최소 개수인 2까지 감소하는 것을 확인할 수 있다.

```bash
$ kubectl get hpa -w

NAME              REFERENCE                   TARGETS    MINPODS  MAXPODS  REPLICAS  AGE
hpa-resource-cpu  Deployment/deployment-cpu1  50%/50%    2        10       10        15s
hpa-resource-cpu  Deployment/deployment-cpu1  10%/50%    2        10       10        15s
hpa-resource-cpu  Deployment/deployment-cpu1  3%/50%     2        10       10        15s
hpa-resource-cpu  Deployment/deployment-cpu1  2%/50%     2        10       10        15s
hpa-resource-cpu  Deployment/deployment-cpu1  0%/50%     2        10       10        15s
hpa-resource-cpu  Deployment/deployment-cpu1  0%/50%     2        10       2         15s
```

HPA를 삭제해서 autoscaling을 종료시키기 위해서는 다음의 명령어를 사용한다.

```bash
kubectl delete horizontalpodautoscalers.autoscaling hpa-resource-cpu
```

## memory의 AverageValue 기반 스케일링 및 사용

이번엔 파드의 memory 사용량의 AverageValue 기준으로 스케일링하도록 HPA를 구성해보자.  
먼저 다음의 구성 파일로 `requests: memory: 10Mi`와 `limits: memory: 20Mi`로 설정된 template을 가진 Deployment를 생성한다.  
Service에는 30002번의 NodePort를 할당했다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stateless-memory1
spec:
  selector:
    matchLabels:
      resource: memory
  replicas: 2
  template:
    metadata:
      labels:
        resource: memory
    spec:
      containers:
        - name: container
          image: kubetm/app:v1
          resources:
            requests:
              memory: 10Mi
            limits:
              memory: 20Mi
---
apiVersion: v1
kind: Service
metadata:
  name: stateless-svc2
spec:
  selector:
    resource: memory
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30002
  type: NodePort
```

이제 metrics에 `type: Resource`로 정의하고, resource에 `name: memory`로 지정해서 memory를 기반으로 스케일링을 수행하는 HPA를 생성한다.  
target에는 `type: AverageValue`, `averageValue: 5Mi`로 지정했다.  
이에 따라 메모리 사용량의 평균값이 5Mi 이상이 되면 스케일 아웃이 시작된다.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-resource-memory
spec:
  maxReplicas: 10
  minReplicas: 2
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: stateless-memory1
  metrics:
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 5Mi
```

이제 아까와 동일하게 파드에 부하를 주면서 HPA의 상태를 확인해보자.

```bash
$ while true;do curl 192.168.56.30:30002/hostname; sleep 0.01; done
```

```bash
$ kubectl get hpa -w

NAME              REFERENCE                     TARGETS       MINPODS  MAXPODS  REPLICAS  AGE
hpa-resource-cpu  Deployment/stateless-memory1  4169728/5Mi   2        10       2         15s
hpa-resource-cpu  Deployment/stateless-memory1  7976960/5Mi   2        10       2         15s
hpa-resource-cpu  Deployment/stateless-memory1  7976960/5Mi   2        10       4         15s
hpa-resource-cpu  Deployment/stateless-memory1  7652352/5Mi   2        10       4         15s
hpa-resource-cpu  Deployment/stateless-memory1  7652352/5Mi   2        10       6         15s
hpa-resource-cpu  Deployment/stateless-memory1  7968768/5Mi   2        10       10        15s
```
