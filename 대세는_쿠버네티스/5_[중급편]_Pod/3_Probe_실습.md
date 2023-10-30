# Pod - ReadinessProbe, LivenessProbe 실습

## ReadinessProbe 실습

다음의 구성 파일로 ReadinessProbe를 적용한 파드를 생성하고, 여기에 연결할 Service를 생성한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-readiness
spec:
  selector:
    app: readiness
  ports:
    - port: 8080
      targetPort: 8080
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-readiness-exec1
  labels:
    app: readiness
spec:
  containers:
    - name: readiness
      image: kubetm/app
      ports:
        - containerPort: 8080
      readinessProbe:
        exec:
          command: ["cat", "/readiness/ready.txt"]
        initialDelaySeconds: 5
        periodSeconds: 10
        successThreshold: 3
      volumeMounts:
        - name: host-path
          mountPath: /readiness
  volumes:
    - name: host-path
      hostPath:
        path: /tmp/readiness
        type: DirectoryOrCreate
  terminationGracePeriodSeconds: 0
```

container의 속성으로 readinessProbe가 추가되어 있고, 테스트할 커맨드(exec: command)와 여러 옵션들이 설정되어 있다.

### 발생한 event 확인

다음의 커맨드로 pod-readiness-exec1 파드를 생성하면서 발생한 이벤트를 확인할 수 있다.

```bash
$ kubectl get events -w | grep pod-readiness-exec1

45s         Normal    Scheduled   pod/pod-readiness-exec1   Successfully assigned default/pod-readiness-exec1 to k8s-node1
44s         Normal    Pulling     pod/pod-readiness-exec1   Pulling image "kubetm/app"
38s         Normal    Pulled      pod/pod-readiness-exec1   Successfully pulled image "kubetm/app" in 6.197601539s
38s         Normal    Created     pod/pod-readiness-exec1   Created container readiness
38s         Normal    Started     pod/pod-readiness-exec1   Started container readiness
3s          Warning   Unhealthy   pod/pod-readiness-exec1   Readiness probe failed: cat: /readiness/ready.txt: No such file or directory
0s          Warning   Unhealthy   pod/pod-readiness-exec1   Readiness probe failed: cat: /readiness/ready.txt: No such file or directory
```

`cat /readiness/ready.txt` 실행에 실패해서 지속적으로 unhealthy warning이 발생하고 있다.

### 파드의 상태 확인

다음의 커맨드를 이용해서 파드의 상태를 확인한다.

```bash
$ kubectl describe pod pod-readiness-exec1 | grep -A5 Conditions
Conditions:
  Type             Status
  Initialized      True
  Ready            False
  ContainersReady  False
  PodScheduled     True
```

현재 Ready, ContainersReady 값이 False로 설정되어 서비스가 파드가 준비되지 않았음을 확인할 수 있다.

### 서비스의 endpoints 확인

다음의 커맨드를 이용해서 파드를 연결한 서비스의 endpoints 상태를 조회한다.

```yaml
kubectl describe endpoints svc-readiness | grep -A2 Subsets

Subsets:
  Addresses:          <none>
  NotReadyAddresses:  10.240.12.246
```

NotReadyAddresses에 현재 Probe에 실패한 파드의 주소가 저장되어 있다.  
이 상태에서는 해당 파드로 트래픽이 전달되지 않는다.

### 파드가 정상 작동하도록 변경

이제 hostPath 볼륨에 연결된 노드의 /tmp/readiness 경로에 ready.txt 파일을 생성한다.  
파일을 생성하면 event 목록 조회를 해도 추가로 Warning 메시지가 표시되지 않는다.

파드의 상태를 조회하면 Ready, ContainersReady 값이 True가 되어 있다.

```bash
$ kubectl describe pod pod-readiness-exec1 | grep -A5 Conditions
Conditions:
  Type             Status
  Initialized      True
  Ready            True
  ContainersReady  True
  PodScheduled     True
```

서비스의 endpoint 상태를 확인하면 이제 Addresses에 파드의 IP가 추가되어, 정상적으로 트래픽을 전달함을 확인할 수 있다.

```bash
kubectl describe endpoints svc-readiness | grep -A2 Subsets

Subsets:
  Addresses:          10.240.12.246
  NotReadyAddresses:  <none>
```

실제로 다음의 커맨드로 해당 서비스의 외부 IP에 요청을 보내면, ReadinessProbe에 성공한 후에는 연결된 파드에 요청이 잘 닿는 것을 확인할 수 있다.

```bash
while true; do date && curl svc-readiness.default.svc.cluster.local:8080/hostname; sleep 1; done

Sat Oct 28 01:31:08 UTC 2023  # Ready: False
curl: (7) Failed to connect to svc-readiness.default.svc.cluster.local port 8080: Connection refused
Sat Oct 28 01:31:09 UTC 2023  # Ready: True
Hostname : pod-readiness-exec1
Sat Oct 28 01:31:10 UTC 2023
Hostname : pod-readiness-exec1
```

## LivenessProbe 실습

### Service, Pod 생성

다음의 구성파일로 LivenessProbe를 적용한 파드와, 연결할 Service 객체를 생성한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-liveness
spec:
  selector:
    app: liveness
  ports:
    - port: 8080
      targetPort: 8080
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-httpget1
  labels:
    app: liveness
spec:
  containers:
    - name: liveness
      image: kubetm/app
      ports:
        - containerPort: 8080
      livenessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
        failureThreshold: 3
  terminationGracePeriodSeconds: 0
```

컨테이너의 속성으로 livenessProbe 가 추가되어 있고, http 요청으로 파드 상태를 체크하기 위한 설정들이(path, port) 포함되어 있다.

다음의 커맨드로 지속적으로 서비스에 요청을 보내서 파드의 연결 상태를 확인한다.

```bash
while true; do date && curl svc-liveness.default.svc.cluster.local:8080/health; sleep 1; done

pod-liveness-httpget1 is Running
```

이제 다음의 커맨드로 파드에 발생한 이벤트를 확인할 수 있다.

```bash
kubectl describe pod pod-liveness-httpget1 | grep -A10 Events

Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  58s   default-scheduler  Successfully assigned default/pod-liveness-httpget1 to k8s-node1
  Normal  Pulling    57s   kubelet            Pulling image "kubetm/app"
  Normal  Pulled     56s   kubelet            Successfully pulled image "kubetm/app" in 1.27264105s
  Normal  Created    56s   kubelet            Created container liveness
  Normal  Started    55s   kubelet            Started container liveness
```

현재는 LivenessProbe의 http 요청이 정상적으로 수행되기 때문에 별다른 추가 로그가 남지 않고 있다.  
노드 스케줄링, 이미지 풀, 컨테이너 생성 및 실행 로그만 남아 있다.

### LivenessProbe 실패 확인

이제 해당 앱이 500 에러 응답을 하도록 변경한다.  
/status500 경로로 요청을 보내면 그 후로는 500 에러 응답을 하도록 구현되어 있다.

```bash
$ curl svc-liveness.default.svc.cluster.local:8080/status500
Status Code has Changed to 500

$ while true; do date && curl svc-liveness.default.svc.cluster.local:8080/health; sleep 1; done
Sat Oct 28 01:45:44 UTC 2023
pod-liveness-httpget1 is Running
Sat Oct 28 01:45:45 UTC 2023
pod-liveness-httpget1 is Running
Sat Oct 28 01:45:46 UTC 2023 # /status500 호출 시점
pod-liveness-httpget1 : Internal Server Error
Sat Oct 28 01:45:47 UTC 2023
pod-liveness-httpget1 : Internal Server Error
```

이벤트 목록을 조회하면 다음과 같이 http 응답에 세 처례 실패하고, 파드가 재시작되는 로그가 확인된다.

```bash
kubectl describe pod pod-liveness-httpget1 | grep -A10 Events

Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  9m9s                  default-scheduler  Successfully assigned default/pod-liveness-httpget1 to k8s-node1
  Normal   Pulled     9m7s                  kubelet            Successfully pulled image "kubetm/app" in 1.27264105s
  Warning  Unhealthy  115s (x6 over 3m45s)  kubelet            Liveness probe failed: HTTP probe failed with statuscode: 500
  Normal   Killing    115s (x2 over 3m25s)  kubelet            Container liveness failed liveness probe, will be restarted
  Normal   Pulling    113s (x3 over 9m8s)   kubelet            Pulling image "kubetm/app"
  Normal   Created    111s (x3 over 9m7s)   kubelet            Created container liveness
  Normal   Started    111s (x3 over 9m6s)   kubelet            Started container liveness
```

서비스에 보낸 요청의 응답을 확인해보면, LivenessProbe에 의해 재시작되기 전까지 잠깐의 시간 동안만 에러 응답을 받고 그 이후에는 다시 정상 응답을 받게 된다.

```bash
$ while true; do date && curl svc-liveness.default.svc.cluster.local:8080/health; sleep 1; done
Sat Oct 28 01:45:44 UTC 2023
pod-liveness-httpget1 is Running
Sat Oct 28 01:45:45 UTC 2023
pod-liveness-httpget1 is Running
# /status500 호출 시점
Sat Oct 28 01:45:46 UTC 2023
pod-liveness-httpget1 : Internal Server Error
Sat Oct 28 01:45:47 UTC 2023
pod-liveness-httpget1 : Internal Server Error
# 파드 재시작
Sat Oct 28 01:45:48 UTC 2023
curl: (7) Failed to connect to svc-liveness.default.svc.cluster.local port 8080: Connection refused
Sat Oct 28 01:45:49 UTC 2023
curl: (7) Failed to connect to svc-liveness.default.svc.cluster.local port 8080: Connection refused
# 파드 실행 완료
Sat Oct 28 01:45:50 UTC 2023
pod2 is Running
```

이와 같이 LivenessProbe는 장애 상황을 빠르게 탐지하고, 파드를 재시작하여 서비스가 안정적으로 운영될 수 있게 한다.
