# DaemonSet, Job, CronJob

## DaemonSet 개념

보통의 클러스터 환경에는 여러 노드가 존재하고, 각 노드마다 서로 다른 자원 양을 가지고 있다.  
이 때 ReplicaSet 컨트롤러에 연결된 파드들은 노드 스케줄링에 의해서 자원이 많이 남은 노드에 주로 스케줄링 된다.  
따라서 특정 노드에 많이 쏠리거나, 자원이 없는 노드에는 생성이 되지 않을 수도 있다.

하지만 DaemonSet의 경우 반드시 각 노드 당 하나의 파드를 생성해서 할당한다.  
이렇게 각 노드마다 하나씩은 들어가야 하는 서비스들이 있다.  
각 노드의 성능 관련 정보를 수집하기 위해 Prometheus를 두거나, 로깅을 하기 위해 fluentd를 둘 수 있다.  
또는 각 노드를 Storage에 활용하기 위해 GlusterFS를 각 노드에 설치해서 Network File System을 구축하는 것도 가능하다.
또한 쿠버네티스에서는 각 노드에 프록시 역할을 하는 파드를 만드는데, 이 떄에도 DaemonSet이 사용된다.

## Job, CronJob 개념

Job을 통해서 파드를 생성하는게 가능하다.  
이 때 파드를 직접 생성한 경우, ReplicaSet을 통해 생성한 경우, Job을 통해 생성한 경우에 모두 다르게 동작한다.

만약 파드가 위치해 있는 노드에 문제가 생겨서 다운이 되면, 그 위에 있는 파드들도 더이상 작동 불가능한 상태가 된다.  
이 떄 파드를 직접 생성한 경우에는 장애에 따라 재생성이 되거나 하지 않기 때문에 서비스가 중단되어 버린다.  
이와 달리 ReplicaSet이나 Job을 통해 생성한 경우에는 파드가 Recreate 되어 서비스가 유지될 수 있다.

또한 만약 파드가 중간에 실행을 멈추면 ReplicaSet은 해당 파드를 재시작(Restart)해서 서비스가 계속 유지되게 해준다.  
이와 달리 Job은 해당 파드를 자원을 사용하지 않는 상태로 변경해서 더 이상 작동되지 않게 한다.  
사용자는 멈춘 Pod를 이용해서 Job을 통해 수행한 작업의 로그 등의 결과를 확인할 수 있다.

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
