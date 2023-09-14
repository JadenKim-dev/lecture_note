# Volume - emptyDir, hostPath, PV/PVC

볼륨에는 크게 3가지 종류가 있다 - **emptyDir, hostPath, PVC/PV**

## emptyDir

emptyDir의 경우 파드 안에 위치하는 볼륨으로, 파드 내 컨테이너 간 편리하게 파일을 공유하게 해준다.  
기본적으로 볼륨이 생성될 때 비어 있는 상태로 생성되기 때문에 emptyDir이라는 이름을 가지고 있다.

emptyDir의 경우 파드가 생성될 때 만들어지고 파드가 삭제되면 함께 삭제되기 때문에, 휘발성이여도 상관 없는 데이터만 저장해야 한다.

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

위와 같이 Pod 정의시 함께 volumes에 정의되며, 각 컨테이너에서 mount할 volume을 name으로, 위치를 mountPath로 지정한다.  
이 때 각 컨테이너는 서로 다른 mountPath를 가질 수 있다.

## hostPath

hostPath는 노드 안에 위치한 볼륨이다.  
자신이 속한 노드(호스트)의 path를 하나 할당 받아서, 해당 위치에 볼륨을 위치시키는 것이다.  
hostPath 볼륨은 파드 밖에 존재하기 때문에 파드가 종료되더라도 볼륨이 유지될 수 있다.

이 때 hostPath에 지정할 path는 사전에 node에 만들어져 있어야 한다.  
만약 디렉토리를 자동으로 생성하게 하고 싶다면 DirectoryOrCreate로 타입을 지정하면 된다.

주의해야 할 점은, 파드는 노드 자원 상황에 따라 이전과 다른 노드에 생성될 수 있기 때문에, 다른 노드에 피드가 생성되면 이전에 사용하던 볼륨을 계속해서 사용하지 못한다.  
따라서 보통 hostPath는 자신이 속한 노드의 시스템 파일, 구성 파일 등에 접근할 때만 사용한다.

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

## Persistent Volume, Persistent Volume Claim

영속적으로 데이터를 저장할 때에는 PV(Persistent Volume)를 사용한다.

이 때 쿠버네티스는 Admin 영역과 User 영역을 구분했다.  
Admin 영역에서 관리자는 local이나 볼륨을 제공하는 여러 서비스에 연결하기 위한 PV를 생성해둔다.  
각 볼륨의 속성마다 제공해야 하는 설정이 다르기 때문에, 전문적으로 이를 관리하는 관리 영역을 별도로 분리해두었다.

PV에는 해당 볼륨이 가지는 접근 권한(accessModes), 용량(capacity storage)를 정의할 수 있다.  
아래의 예시에서는 local 타입의 볼륨을 사용했다.  
local 볼륨을 사용할 경우 특정 노드의 path를 사용하여 볼륨이 생성되기 때문에, nodeSelectorTerms를 이용하여 특정 노드에만 파드가 생성되게 설정해두었다.  
(PV는 노드에 상관없이 볼륨을 사용하기 위한 목적이 크기 때문에, local 타입의 PV는 잘 사용하지 않는다.)

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

서비스를 배포하고 운영하는 User는 PV에 연결할 PVC를 만든다.  
PVC에는 어떤 접근 권한이 필요한지(accessModes), 어느 정도의 크기가 필요한지(requests storage)를 정의한다.  
여기에 `storageClassName: ""` 옵션을 넣으면 해당 조건을 만족하는 PV가 자동으로 할당 된다.

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

마지막으로 파드의 volumes에서는 claimName에 방금 생성해 둔 PVC를 지정하여 볼륨을 정의해두고, 컨테이너 정의 부분에서 해당 볼륨을 마운트해서 사용하도록 정의한다.

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

# 실습

## emptyDir

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

container1에서 마운트된 볼륨에 파일을 생성하면 container2에서도 해당 파일을 확인할 수 있다.  
또한 파드를 삭제하면 볼륨에 저장된 내용이 초기화된다.

## hostPath

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

node1에 파드가 생성되도록 구성했고, 노드의 node-v 디렉토리에 볼륨이 연결된다.  
hostPath의 타입을 DirectoryOrCreate로 지정했기 때문에 노드 상에 해당 디렉토리가 없어도 자동으로 생성된다.

위 구성 파일로 여러 개의 파드를 만들고, 한 파드에서 볼륨 내에 파일을 생성하면 파드들 간 해당 파일이 공유되는 것을 확인할 수 있다.  
또한 node1을 직접 실행해서 들어가면 /node-v 경로가 생성되어 있고 그 안에 파일이 생성되어 있는 것을 확인할 수 있다.

파드를 임의로 삭제 후 nodeSelector 부분을 제거한 뒤 재생성하면 다른 노드에 파드가 생성된다.  
해당 파드에도 볼륨이 마운트되긴 하지만, 노드가 달라진 상태이므로 이전에 작업한 내용은 찾을 수 없다.

## PersistentVolume, PersistentVolumeClaim

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
