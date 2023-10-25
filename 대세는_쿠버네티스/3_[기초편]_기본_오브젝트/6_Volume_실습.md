# Volume 실습

## emptyDir 볼륨 실습

다음의 구성 파일로 emptyDir 볼륨을 포함한 파드를 생성한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-volume-1
spec:
  containers:
    - name: container1
      image: kubetm/init
      volumeMounts:
        - name: empty-dir
          mountPath: /mount1
    - name: container2
      image: kubetm/init
      volumeMounts:
        - name: empty-dir
          mountPath: /mount2
  volumes:
    - name: empty-dir
      emptyDir: {}
```

컨테이너에 직접 들어가 보면 container1의 mount1, container2의 mount2 폴더에 볼륨이 마운트 된 것을 확인할 수 있다.  
`mount | grep mount1` 커맨드로 현재 마운트 된 내역을 확인할 수 있다.

```bash
$ kubectl get po
NAME                        READY   STATUS    RESTARTS   AGE
pod-volume-1                2/2     Running   0          5s

$ kubectl exec -it pod-volume-1 -c container1 -- /bin/bash

[root@pod-volume-1 /]# mount | grep mount1
/dev/vda1 on /mount1 type ext4 (rw,relatime)
```

container1에서 마운트된 볼륨에 파일을 생성하면 container2에서도 해당 파일을 확인할 수 있다.

```bash
# pod-volume-1 파드의 container1에서 파일 생성
$ kubectl exec -it pod-volume-1 -c container1 -- /bin/bash

[root@pod-volume-1 /]# touch mount1/file.txt

# pod-volume-1 파드의 container2에서 파일 확인
$ kubectl exec -it pod-volume-1 -c container2 -- /bin/bash

[root@pod-volume-1 /]# ls mount2
file.txt
```

또한 파드를 삭제하면 볼륨에 저장된 내용이 초기화된다.

## hostPath 볼륨 실습

다음의 구성 파일로 hostPath 볼륨이 연결된 파드를 생성한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-volume-3
spec:
  nodeSelector:
    kubernetes.io/hostname: k8s-node1
  containers:
    - name: container
      image: kubetm/init
      volumeMounts:
        - name: host-path
          mountPath: /mount1
  volumes:
    - name: host-path
      hostPath:
        path: /node-v
        type: DirectoryOrCreate
```

k8s-node1에 파드가 생성되도록 구성했고, 노드의 node-v 디렉토리의 볼륨이 연결된다.  
hostPath의 타입을 DirectoryOrCreate로 지정했기 때문에 노드 상에 해당 디렉토리가 없어도 자동으로 생성된다.

위 구성 파일로 여러 개의 파드를 만들고, 한 파드에서 볼륨 내에 파일을 생성하면 파드들 간 해당 파일이 공유되는 것을 확인할 수 있다.

```bash
$ kubectl get po
NAME                        READY   STATUS    RESTARTS   AGE
pod-volume-3                1/1     Running   0          39s
pod-volume-4                1/1     Running   0          4s

# pod-volume-3 에서 hostPath 볼륨에 파일 생성
$ kubectl exec -it pod-volume-3 -- /bin/bash

[root@pod-volume-3 /]# touch mount1/file.txt

# pod-volume-4 에서 hostPath 볼륨의 파일 확인
$ kubectl exec -it pod-volume-4 -- /bin/bash

[root@pod-volume-4 /]# ls mount1
file.txt
```

또한 node1을 직접 실행해서 들어가면 /node-v 경로가 생성되어 있고 그 안에 파일이 생성되어 있는 것을 확인할 수 있다.

파드를 임의로 삭제한 후 nodeSelector 부분을 제거한 뒤 재생성하면 다른 노드에 파드가 생성된다.  
해당 파드에도 볼륨이 마운트되긴 하지만, 노드가 달라진 상태이므로 이전에 작업한 내용은 찾을 수 없다.

## PV, PVC 실습

다음의 구성 파일로 물리적 저장소 역할을 하는 Persistent Volume을 생성한다.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-03
spec:
  capacity:
    storage: 2G
  accessModes:
    - ReadWriteOnce
  local:
    path: /node-v
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - { key: kubernetes.io/hostname, operator: In, values: [k8s-node1] }
```

ReadWriteOnce 접근 모드를 가진 2GB 크기의 PV를 생성했다.  
local 타입으로 node1 내의 /node-v 경로에 볼륨을 연결한 상태이다(hostPath와 유사).  
위 구성과 함께 ReadOnlyMany 접근 모드의 PV, 3GB 크기의 PV를 추가로 생성했다.

(참고 : local 타입의 PV를 생성할 때 hostPath 처럼 자동으로 디렉토리를 만들어주는 옵션이 없다.  
따라서 노드에는 미리 해당 디렉토리가 만들어져 있어야 한다. 만들지 않으면 파드에서 마운트할 때 문제가 발생한다.)

이제 다음의 구성파일로 PersistentVolumeClaim을 생성한다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-01
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1G
  storageClassName: ""
```

accessModes와 필요 용량을 지정하는 형태이다.  
storageClassName: "" 로 지정되어 있으므로 조건에 맞는 적절한 PV를 자동으로 찾아서 연결해준다.

각각의 PV는 하나의 PVC만 연결 가능하다.  
만약 남아 있는 PV 중 접근 모드와 필요 용량에 맞는 게 없다면 PVC 생성에 Pending이 걸리게 된다.

이제 다음과 같이 PVC를 마운트 한 파드를 생성할 수 있다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-volume-3
spec:
  containers:
    - name: container
      image: kubetm/init
      volumeMounts:
        - name: pvc-pv
          mountPath: /mount3
  volumes:
    - name: pvc-pv
      persistentVolumeClaim:
        claimName: pvc-01
```

claimName에 원하는 PVC의 이름을 지정해서 볼륨을 정의한다.  
volumeMounts에 해당 볼륨을 지정하면 컨테이너의 mountPath에 볼륨이 마운트된다.

출처: [인프런 대세는 쿠버네티스 [초급 ~ 중급]](https://inf.run/yW34)
