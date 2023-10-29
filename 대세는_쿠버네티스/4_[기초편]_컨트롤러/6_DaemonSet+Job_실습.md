# DaemonSet, Job, CronJob 실습

## DaemonSet 실습

### 기본 구성 방법

다음의 구성 파일로 hostPort를 지정한 DaemonSet을 생성한다.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-1
spec:
  selector:
    matchLabels:
      type: app
  template:
    metadata:
      labels:
        type: app
    spec:
      containers:
        - name: container
          image: kubetm/app
          ports:
            - containerPort: 8080
              hostPort: 18080
```

이러면 각 노드에 파드가 생성되고, 각 노드의 18080번 포트에 각 파드가 연결된다.  
다음의 커맨드로 각 노드의 IP의 18080번 포트로 요청을 보내면 그 안의 파드에 잘 닿는 것을 확인할 수 있다.

```bash
$ curl 10.95.33.92:18080/hostname

Hostname : daemonset-1-n7j7t
```

DaemonSet을 삭제하면 연결된 파드들이 함께 삭제된다.
또한 template의 image를 변경하면 각 노드에 설치된 파드가 업데이트 된다.  
(구성 파일에 업데이트 방식을 지정 가능, 기본 RollingUpdate)

### nodeSelector

nodeSelector를 통해 일부 노드를 선택하여 해당 노드에만 파드를 생성할 수도 있다.  
먼저 다음 명령어로 각 노드에 라벨을 붙인다.

```bash
$ kubectl label nodes k8s-node1 os=centos
$ kubectl label nodes k8s-node2 os=ubuntu
```

이제 `os: centos` 라벨이 붙은 노드를 선택하는 DaemonSet을 다음과 같이 생성한다.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-2
spec:
  selector:
    matchLabels:
      type: app
  template:
    metadata:
      labels:
        type: app
    spec:
      nodeSelector:
        os: centos
      containers:
        - name: container
          image: kubetm/app
          ports:
            - containerPort: 8080
```

이제 해당 라벨이 붙은 노드에만 파드가 생성되는 것을 확인할 수 있다.
DaemonSet을 생성한 이후에도 nodeSelector에 지정한 라벨을 다른 노드에 추가하면 해당 노드에도 파드가 생성된다.

```bash
# 라벨 삭제
kubectl label nodes k8s-node2 os-

# 라벨 추가
kubectl label nodes k8s-node2 os=ubuntu

-> k8s-node2에도 파드가 생성됨
```

## Job 실습

### 기본 구성 방법

다음 구성 파일로 Job을 생성할 수 있다.  
`job start` 로그를 찍고 20초 뒤에 `job end` 로그를 찍은 뒤 종료하는 pod로 template을 지정했다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-1
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: container
          image: kubetm/init
          command: ["sh", "-c", "echo 'job start';sleep 20; echo 'job end'"]
      terminationGracePeriodSeconds: 0
```

Job을 생성하면 파드가 자동으로 생성되어서 지정된 작업을 진행하고, 20초 뒤에 파드가 종료되는 것을 확인할 수 있다.  
종료된 뒤에도 들어가서 로그를 확인할 수 있고, Job을 삭제하면 그 때 파드도 함께 삭제된다.

```bash
$ kubectl get job
NAME    COMPLETIONS   DURATION   AGE
job-1   0/1           3s         3s

$ kubectl get job  # 20초 뒤
NAME    COMPLETIONS   DURATION   AGE
job-1   1/1           24s        26s

$ kubectl get po
NAME                        READY   STATUS      RESTARTS   AGE
job-1-8pdwm                 0/1     Completed   0          66s

$ kubectl logs job-1-8pdwm
job start
job end
```

### completions, parallelism, activeDeadlineSeconds

다음의 구성파일로 옵션이 추가된 Job을 생성한다.  
총 6개의 파드(`completions`)를 한 번에 2개의 파드씩(`parallelism`) 생성하는 Job이다.  
이 때 총 수행 시간이 30초가 되면 Job의 실행을 멈추도록(`activeDeadlineSeconds`) 구성했다.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-2
spec:
  completions: 6
  parallelism: 2
  activeDeadlineSeconds: 30
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: container
          image: kubetm/init
          command: ["sh", "-c", "echo 'job start';sleep 20; echo 'job end'"]
      terminationGracePeriodSeconds: 0
```

이 경우, Job 생성시 동시에 파드가 2개 생성된다.  
20초 뒤에는 파드 실행이 종료되고, 새로운 파드 2개가 새롭게 생성된다.  
이때 30초가 되는 시점에 파드 실행이 모두 종료되고 하나씩 파드가 삭제된다.  
Job의 상세 정보에 들어가면 이러한 실행 및 종료 내역을 확인할 수 있다.

```bash
$ kubectl get job
NAME    COMPLETIONS   DURATION   AGE
job-2   2/6           83s        83s

$ kubectl describe job job-2
Name:                     job-2
Namespace:                default
Selector:                 controller-uid=eb3c28ab-a71e-46dd-98b9-f91d99984030
Labels:                   controller-uid=eb3c28ab-a71e-46dd-98b9-f91d99984030
                          job-name=job-2
Parallelism:              2
Completions:              6
Active Deadline Seconds:  30s
Pods Statuses:            0 Active / 2 Succeeded / 2 Failed
Events:
  Type     Reason            Age   From            Message
  ----     ------            ----  ----            -------
  Normal   SuccessfulCreate  2m7s  job-controller  Created pod: job-2-mrlhw
  Normal   SuccessfulCreate  2m7s  job-controller  Created pod: job-2-v5qmj
  Normal   SuccessfulCreate  103s  job-controller  Created pod: job-2-8rrql
  Normal   SuccessfulCreate  103s  job-controller  Created pod: job-2-z56kk
  Normal   SuccessfulDelete  97s   job-controller  Deleted pod: job-2-8rrql
  Normal   SuccessfulDelete  97s   job-controller  Deleted pod: job-2-z56kk
  Warning  DeadlineExceeded  97s   job-controller  Job was active longer than specified deadline

```

## CronJob 실습

### 기본 구성 방법

다음의 구성 파일로 CronJob을 생성한다.  
1분에 한 개씩 Job을 생성하도록 설정되어 있다.(`spec : schedule`)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cron-job
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: container
              image: kubetm/init
              command: ["sh", "-c", "echo 'job start';sleep 20; echo 'job end'"]
          terminationGracePeriodSeconds: 0
```

위와 같이 생성시 정상적으로 1분에 1개씩 Job이 생성되는 것을 확인할 수 있다.  
해당 CronJob을 trigger 하거나 kubectl 커맨드를 이용해서 수동으로 Job을 생성할 수도 있다.

```bash
kubectl create job --from=cronjob/cron-job cron-job-manual-001
```

CronJob을 삭제하면 주기마다 생성한 Job들과 수동 생성한 Job들이 함께 삭제된다.

### suspend

CronJob을 잠시동안 멈추게 하기 위해 suspend를 할 수도 있다.  
구성 파일을 수정하거나, 다음의 커맨드를 사용하여 spec : suspend 값을 true로 바꾸면 된다.

```bash
$ kubectl patch cronjobs cron-job -p '{"spec" : {"suspend" : true }}'

$ kubectl get cronjob
NAME       SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cron-job   */1 * * * *   True      0        <none>          4s
```

suspend 해제를 위해서는 해당 값을 false로 바꾸면 된다.

### concurrencyPolicy

이번엔 concurrencyPolicy: Forbid 로 지정한 CronJob을 생성해보자

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: cron-job-2
spec:
  schedule: "20,21,22 * * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: container
              image: kubetm/init
              command:
                ["sh", "-c", "echo 'job start';sleep 140; echo 'job end'"]
          terminationGracePeriodSeconds: 0
```

20분, 21분, 22분에 Job이 생성되도록 스케줄된 크론잡이다.  
컨테이너는 2분 20초 wait 후에 종료되도록 커맨드가 구성되어 있다.

```bash
# 6초가 지난 시점에 49분이 되어 스케줄된 Job이 생성된다.
$ kubectl get cronjob
NAME         SCHEDULE           SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cron-job-2   20,21,22 * * * *   False     1        6s              22s
```

21분 시점에 생성 시도하는 Job의 경우 아직 이전의 Job이 실행 중이기 때문에 concurrencyPolicy: Forbid에 따라서 skip이 된다.  
22분 20초 시점에는 이전 Job이 종료되기 때문에 22분에 스케줄된 파드가 해당 시점에 생성된다.

```bash
$ kubectl get job
NAME                    COMPLETIONS   DURATION   AGE
cron-job-2-1698407340   1/1           2m24s      3m17s # 20분에 생성
cron-job-2-1698407460   0/1           47s        47s # 22분 20초에 생성
```

concurrencyPolicy를 Replace로 바꾸면 파드가 다른 방식으로 생성된다.  
21분이 되면 `concurrencyPolicy: Replace`에 따라 이전에 실행 중이던 Job과 Pod는 삭제되고, 새로운 Job이 생성된다.

출처: [인프런 대세는 쿠버네티스 [초급 ~ 중급]](https://inf.run/yW34)
