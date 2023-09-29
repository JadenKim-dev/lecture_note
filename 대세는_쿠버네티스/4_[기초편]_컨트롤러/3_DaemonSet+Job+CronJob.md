# DaemonSet, Job, CronJob

## DaemonSet 개념

### 일반적인 노드 스케줄링

보통의 클러스터 환경에는 여러 노드가 존재하고, 각 노드마다 서로 다른 자원 양을 가지고 있다.  
이 때 ReplicaSet 컨트롤러에 연결된 파드들은 노드 스케줄링에 의해서 자원이 많이 남은 노드에 주로 스케줄링 된다.  
따라서 특정 노드에 많이 쏠리거나, 자원이 없는 노드에는 생성이 되지 않을 수도 있다.

### DaemonSet을 통한 노드 스케줄링

하지만 DaemonSet의 경우 반드시 각 노드 당 하나의 파드를 생성해서 할당한다.  
각 노드마다 하나씩은 들어가야 하는 서비스들에 DaemonSet을 많이 사용한다.

각 노드의 성능 관련 정보를 수집하기 위해 Prometheus를 두거나, 로깅을 하기 위해 fluentd를 둘 수 있다.  
또는 각 노드를 Storage에 활용하기 위해 GlusterFS를 각 노드에 설치해서 Network File System을 구축하는 것도 가능하다.  
또한 쿠버네티스에서는 각 노드에 프록시 역할을 하는 파드를 만드는데, 이 떄에도 DaemonSet이 사용된다.

## Job, CronJob 개념

### Pod 직접 생성 vs ReplicaSet vs Job

Job을 통해서 파드를 생성하는게 가능하다.  
이 때 파드를 직접 생성한 경우, ReplicaSet을 통해 생성한 경우, Job을 통해 생성한 경우에 모두 다르게 동작한다.

### 장애 상황에서의 차이 (Pod vs [ReplicaSet, Job])

만약 파드가 위치해 있는 노드에 문제가 생겨서 다운이 되면, 그 위에 있는 파드들도 더 이상 작동 불가능한 상태가 된다.  
이 때 Pod를 직접 생성한 경우에는 장애가 발생해도 재생성이 이루어지지 않아서 서비스가 그대로 중단되어 버린다.  
이와 달리 ReplicaSet이나 Job을 통해 생성한 경우에는 파드가 Recreate 되어 서비스가 유지될 수 있다.

### 파드 실행 중단 시 차이 (ReplicaSet vs Job)

또한 만약 파드가 중간에 실행을 멈추면 ReplicaSet은 해당 파드를 재시작(Restart)해서 서비스가 계속 유지되게 해준다.  
이와 달리 Job은 파드의 실행이 멈추면 해당 파드가 더 이상 작동되지 않게 하고, 파드 재시작을 하지 않는다.  
사용자는 멈춘 Pod를 통해 수행한 작업의 로그 등 결과를 확인할 수 있다.

### CronJob

CronJob은 일정 시간마다 Job을 생성하는 오브젝트이다.  
주로 데이터를 백업하거나, 업데이트를 체크하거나, 이메일/메시지를 발송하는 등 주기적으로 해야하는 작업을 CronJob으로 등록한다.

## DaemonSet 설명

DaemonSet의 구성 파일 예시는 아래와 같다.

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
      nodeSelector:
        os: centos
      containers:
        - name: container
          image: kubetm/app
          ports:
            - containerPort: 8080
              hostPort: 18080
```

다른 컨트롤러와 동일하게 DaemonSet에도 selector와 template을 지정한다.  
이에 따라 DaemonSet은 각 노드에 template에 정의한대로 파드를 만들어서, selector로 연결한다.  
이 때 일부 노드에만 파드가 생성되도록 하기 위해서 nodeSelector를 통해 선택할 노드의 라벨을 지정할 수 있다.
또한 외부에서 해당 파드에 트래픽을 전달할 수 있도록 template의 컨테이너 정보에서 hostPort를 지정할 수도 있다.  
각 노드에 hostPort 포트로 오는 트래픽의 경우, 해당하는 파드의 컨테이너 포트로 트래픽이 전달된다.

## Job 설명

Job의 구성 파일 예시는 아래와 같다.

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
```

### 기본 옵션

Job을 생성할 때도 마찬가지로 template, selector를 지정할 수 있다.  
하지만 Job에 어떤 Pod가 연결되었는지는 반드시 식별이 되어야 하기 때문에, selector의 경우 지정하지 않아도 자동으로 생성하도록 되어 있다.

### 추가 설정

기본적으로 Job은 하나의 파드만 실행하지만, completions에는 순차적으로 실행할 파드의 개수를 지정할 수 있고, 한 번에 parallelism 값 만큼 파드가 실행된다.  
또한 activeDeadlineSeconds에 지정한 초를 넘어도 파드 실행이 안 될 경우, 전체 Job 실행이 멈추게 된다.  
(행이 걸리는 등 실행에 문제가 있는 경우 더이상 해당 Job에 자원을 할당하지 않기 위한 정책이다.)

### restartPolicy

또한 Job을 만들 때 restartPolicy도 지정해야 하는데, 여기에는 Never와 OnFaliure를 지정할 수 있다.  
이에 대한 자세한 설명은 중급편에서 할 예정이다.

## CronJob 설명

CronJob의 예시 구성 파일은 아래와 같다.

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

### 기본 옵션

schedule에는 Job을 생성할 주기를 Cron 형식에 맞게 작성한다.  
jobTemplate에는 CronJob이 매 시간 간격마다 생성해 낼 Job의 정보를 입력한다.

### restartPolicy

이 때 template에 restartPolicy를 지정할 수 있는데, 각 설정마다 Job과 Pod가 생성되는 방식이 다르다.

- Allow: 매 시간마다 언제나 Job을 생성한다.
- Forbid: 이전 시간 단위에서 실행한 파드가 남아있으면 일단 Job을 생성하지 않는다.  
  실행 중이던 파드가 종료되면 직전에 스케줄되어 있던 Job이 생성되는데, 만약 2개 이상의 Job을 건너 뛴 상태라면 하나를 제외한 나머지 Job들은 skip 된다.
- Replace의 경우 이전 시간 단위에서 실행 중인 파드가 남아있으면 바로 기존의 Job과 Pod를 삭제하고, 새롭게 Job을 생성한다.

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
$ curl 192.168.56.31:18080/hostname
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
nodeSelector에 지정한 라벨을 노드에 추후에 추가해도 해당 노드에도 파드가 생성된다.

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
kubectl patch cronjobs cron-job -p '{"spec" : {"suspend" : true }}'
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
21분 시점에 생성 시도하는 Job의 경우 아직 이전의 Job이 실행 중이기 때문에 concurrencyPolicy: Forbid에 따라서 skip이 된다.  
22분 20초 시점에는 이전 Job이 종료되기 때문에 22분에 스케줄된 파드가 해당 시점에 생성된다.

만약 위와 동일한 구성 파일에서 concurrencyPolicy만 Replace로 바꿔주면, 파드가 다른 방식으로 생성된다.  
이 때 21분이 되면 `concurrencyPolicy: Replace`에 따라 이전에 실행 중이던 Job과 Pod는 삭제되고, 새로운 Job이 생성된다.

출처: [인프런 대세는 쿠버네티스 [초급 ~ 중급]](https://inf.run/yW34)
