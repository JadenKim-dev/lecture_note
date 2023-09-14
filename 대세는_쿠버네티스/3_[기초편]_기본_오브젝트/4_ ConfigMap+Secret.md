# ConfigMap, Secret - Env, Mount

여러 환경에 따라 서로 다른 데이터를 참조해야 하는경우가 많다(개발 환경, 배포 환경).  
이를 위해 해당 데이터만 다르게 정의된 여러 개의 도커 이미지를 사용해야 한다면, 관리해야 하는 이미지가 불필요하게 늘어난다.

환경에 따라 달라지는 값들은 ConfigMap, Secret으로 관리하는게 적절하다.  
이러면 이미지는 모든 환경에서 동일하게 사용하고, 환경에 따라 다른 ConfigMap, Secret을 사용하도록 구성하면 된다.  
일반적인 상수들은 ConfigMap으로 관리하고, Secret에는 보안적인 관리가 필요한 값을 저장한다.

Secret의 경우 Secret 파일을 인메모리 파일시스템(tmpfs)영역에 올려놓고 이를 Pod에 마운트 하는 식으로 작동한다.  
민감한 데이터가 디스크가 아닌 메모리로 관리되기 때문에 보안에 더 유리하다.

## 구성 파일(yaml)로 관리, 환경변수로 주입

ConfigMap, Secret 구성 파일에 값을 명시하고, 환경변수로 값을 삽입하는 방법은 아래와 같다.  
각 구성 파일에 key: value 형태로 값을 적어둔다

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-dev
data:
  SSH: "false"
  User: dev
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sec-dev
data:
  Key: MTIzNA==
```

이 때 Secret의 value는 Base64로 인코딩 된 값이어야 하고, 최대 1Mbyte만 저장할 수 있다.  
파드에 값이 마운팅 될 때에는 Base64로 디코딩하기 때문에, 반드시 인코딩한 값을 삽입해야 한다.

파드에 ConfigMap, Secret의 값을 환경변수로 삽입하는 예시는 아래와 같다.  
각 컨테이너의 envFrom에서 configMapRef, secretRef에 지정하면 된다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-1
spec:
  containers:
    - name: container
      image: kubetm/init
      envFrom:
        - configMapRef:
            name: cm-dev
        - secretRef:
            name: sec-dev
```

## 키만 별도의 파일로 관리, 환경변수로 삽입

ConfigMap, Secret을 파일로 관리하고, 이를 환경변수로 삽입하는 방법도 있다.  
각 쿠버네티스 객체에는 [파일명]:[파일내용] 형태로 값이 저장된다.  
파일을 작성한 후 kubectl 커맨드를 이용해서 등록하면 된다.

다음은 cm.txt 파일을 이용하여 ConfigMap 객체를 생성하는 커맨드이다.

```bash
kubectl create configmap cm-file --from-file=./cm.txt
```

다음은 secret.txt 파일을 이용하여 Secret 객체를 생성하는 커맨드이다.  
(파일을 이용하여 Secret을 생성할 때에는 자동으로 Base64 인코딩을 수행하기 때문에 평문으로 값을 삽입해야 한다.)

```bash
kubectl create secret generic sec-file --from-file=./secret.txt
```

이제 다음의 구성파일로 Pod를 생성해서 컨테이너에 ConfigMap, Secret을 연결한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-file
spec:
  containers:
    - name: container
      image: kubetm/init
      env:
        - name: file-c
          valueFrom:
            configMapKeyRef:
              name: cm-file
              key: cm.txt
        - name: file-s
          valueFrom:
            secretKeyRef:
              name: sec-file
              key: secret.txt
```

위 예시에서는 file-c 환경변수에 cm.txt 파일 내용이 값으로 들어가고, file-s 환경변수에 secret.txt 파일 내용이 값으로 들어간다.

## 컨테이너 내에 파일로 마운트

ConfigMap, Secret을 사용해서 컨테이너 내에 파일을 마운트하는 방법도 있다.  
다음과 같이 파드의 컨테이너에 마운트 설정을 한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-mount
spec:
  containers:
    - name: container
      image: kubetm/init
      volumeMounts:
        - name: file-volume
          mountPath: /mount
  volumes:
    - name: file-volume
      configMap:
        name: cm-file
```

기존의 방식에서는 변경된 ConfigMap에 맞게 환경변수 값을 수정하려면 파드를 다시 띄어야 했다.  
위와 같이 ConfigMap을 파일로 마운트하면 내용을 수정했을 때 파드 내에도 바로 반영된다.
