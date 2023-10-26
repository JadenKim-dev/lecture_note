# Service - Headless, Endpoint, ExternalName 실습

## ClusterIP 서비스 객체를 통한 DNS Server 테스트

먼저 다음과 같은 ClusterIP Service 객체를 생성한다

```yaml
apiVersion: v1
kind: Service
metadata:
  name: clusterip1
spec:
  selector:
    svc: clusterip
  ports:
    - port: 80
      targetPort: 8080
```

이제 클러스터 내에서 다음의 커맨드를 통해 service 이름으로 IP를 조회할 수 있다.

```bash
$ nslookup clusterip1
Server:   10.96.0.10
Address:  10.96.0.10#53

Name:    clusteripl.default.svc.cluster.local
Address: 10.100.176.157
```

FQDN의 전체 domain name으로 질의를 해도 정상적으로 IP를 받아오게 된다.

```bash
$ nslookup clusterip1.default.svc.cluster.local
Server:   10.96.0.10
Address:  10.96.0.10#53

Name:    clusterip1.default.svc.cluster.local
Address: 10.100.176.157
```

## Headless Service

다음의 구성 파일로 headless service 객체를 생성한다.  
`clusterIP: None` 으로 지정되어 있기 때문에 headless로 생성된다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless1
spec:
  selector:
    svc: headless
  ports:
    - port: 80
      targetPort: 8080
  clusterIP: None
```

이제 다음의 구성 파일로 headless 서비스에 연결한 파드를 생성한다.  
hostname에 pod name과 다른 값을 지정했고, subdomain에 headless service의 이름을 지정했다.  
(여기서 지정한 hostname으로 DNS Server에 등록된다.)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod4
  labels:
    svc: headless
spec:
  hostname: pod-a
  subdomain: headless1
  containers:
    - name: container
      image: kubetm/app
```

이제 `[Pod hostname].[Service 이름]`의 간단한 domain name으로 질의하여 간편하게 파드의 IP 주소를 받아올 수 있다.

```bash
$ nslookup pod-a.headless1
Server:   10.96.0.10
Address:  10.96.0.10#53

Name:    pod-a.headless1.default.svc.cluster.local
Address: 20.111.156.70
```

해당 domain name을 통해 간편하게 파드에 요청을 보낼 수 있다.

```bash
$ curl pod-a.headless1:8080/hostname
Hostname: pod-a
```

headless service의 이름으로 질의를 하면 등록된 전체 파드들의 IP를 얻을 수 있다.

```bash
$ nslookup headless1
Server:   10.96.0.10
Address:  10.96.0.10#53

Name:    headless1.default.svc.cluster.local
Address: 20.111.156.70
Name:    headless1.default.svc.cluster.local
Address: 20.109.131.11
```

## Endpoint

### selector - label 매칭으로 생성

먼저 selector - label로 매칭되는 Service 와 Pod 를 생성한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: endpoint1
spec:
  selector:
    svc: endpoint
  ports:
    - port: 8080
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod7
  labels:
    svc: endpoint
spec:
  containers:
    - name: container
      image: kubetm/app
```

`svc: endpoint` 라벨로 파드가 선택되어 있다.  
이제 다음의 명령어로 endpoint 정보를 조회한다.

```yaml
$ kubectl describe endpoints endpoint1
Name:         endpoint1
Namespace:    default
Labels:       <none>
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2023-09-29T21:37:14Z
Subsets:
  Addresses:          20.109.131.12
  NotReadyAddresses:  <none>
  Ports:
    Name     Port     Protocol
    <unset>  8080     TCP
Events: <none>
```

서비스 이름과 동일한 endpoint1 이라는 이름으로 Endpoint가 생성된 것을 확인할 수 있다.  
이 때 Addresses에는 파드의 IP 가 담겨있고, Ports에는 포트 번호가 담겨있다.

### 구성 파일로 직접 생성

만약 label - selector를 이용하지 않고 직접 위 Endpoint를 생성한다면 다음의 구성파일로 생성할 수 있다.

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: endpoint1
subsets:
  - addresses:
      - ip: 20.109.131.12
    ports:
      - port: 8080
```

위와 같이 endpoint를 생성한 후 요청을 보내면 파드에 요청이 닿는 것을 확인할 수 있다.

```bash
$ curl endpoint1:8080/hostname
Hostname: pod7
```

### 외부 서비스에 연결한 Endpoint 생성

이번엔 외부 서비스에 연결한 Endpoint를 생성해보자.  
먼저 다음의 커맨드로 github 사이트의 IP 주소를 얻어 온다.

```bash
$ nslookup https://www.github.com
Name:   github.github.io
Address: 185.199.108.153
```

이제 해당 IP 주소를 담고 있는 Endpoint를 다음의 구성파일로 생성한다.

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: endpoint3
subsets:
  - addresses:
      - ip: 185.199.108.153
    ports:
      - port: 80
```

이제 endpoint3 도메인 이름으로 요청을 보내도 github에 잘 닿는 것을 확인할 수 있다.

```bash
# endpoint3가 github.com의 도메인과 동일하게 쓰임
$ curl -O endpoint3/kubetm/kubetm.github.io/blob/master/sample/practice/intermediate/service-sample.md
```

## ExternalName

서비스의 externalName 속성값에 외부 서비스의 도메인 이름을 지정하여 간단하게 서비스를 외부 도메인에 연결할 수 있다.  
먼저 다음의 구성 파일로 서비스를 생성한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: externalname1
spec:
  type: ExternalName
  externalName: github.github.io
```

이제 해당 서비스의 name으로 외부 서비스에 요청을 보낼 수 있다.

```bash
$ curl -O externalname1/kubetm/kubetm.github.io/blob/master/sample/practice/intermediate/service-sample.md
```

외부 서비스의 url을 바꾸고 싶다면, 간단하게 Service 객체를 수정하는 것으로 대응할 수 있다.
