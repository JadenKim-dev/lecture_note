# Networking 실습

## Pause Container 실습

먼저 다음의 구성 파일로 컨테이너 2개로 이루어진 파드를 생성한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-pause
spec:
  nodeSelector:
    kubernetes.io/hostname: k8s-node1
  containers:
    - name: container1
      image: kubetm/p8000
      ports:
        - containerPort: 8000
    - name: container2
      image: kubetm/p8080
      ports:
        - containerPort: 8080
```

pod-pause라는 이름으로 파드를 생성했다.  
이제 해당 파드와 함께 생성된 pause container를 조회해볼 것이다.  
nodeSelector로 선택한 node1에서 `docker ps`를 통해 컨테이너를 조회해보자.  
`grep pod-pause` 를 통해 방금 생성한 파드에 연결된 컨테이너들만 추려낸다.

```bash
$ docker ps | grep pod-pause

33912ec43b63     kubetm/p8080             "docker-entrypoint.sm"     About a minute ago   Up About a minute    k8s_container2_pod-pause_default_144ef874-4dc2-4c70-8d93-c67f4c6f8865_0
9a5a6Oec8abb     kubetm/p8000             "docker-entrypoint.s_"     About a minute ago   Up About a minute    k8s_container1_pod-pause_default_144ef874-4dc2-4c70-8d93-c67f4c6f8865_0
c2924165ad25     k8s.gcr.io/pause:3.2     "/pause"                   About a minute ago   Up About a minute    k8s_POD_pod-pause_default_144ef874-4dc2-4c70-8d93-c67f4c6f8865_0
```

파드에 생성된 kubetm/p8080, kubetm/p8000 이미지 컨테이너와 함께, Pause Container가 조회되는 것을 확인할 수 있다.

이제 해당 Pause Container에 연결된 Network Namespace를 조회해보자.
먼저 다음 커맨드로 컨테이너의 NetworkSetting 정보를 조회하고, 그 중에서 SandboxKey를 확인해보자.

```bash
$ docker inspect <container-id> -f "{{json .NetworkSettings}}"

{"Bridge":"", ... "SandboxKey":"/var/run/docker/netns/d4fla9a8ea61", ...}
```

SandboxKey를 통해 해당 컨테이너의 Network Namespace를 조회해보자.

```bash
$ sudo ln -s /var/run/docker/netns /var/run/netns
$ ip netns exec /var/run/docker/netns/d4fla9a8ea61 ip a

1: lo: <LOOPBACK,UP,LOWERUP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if16: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default
    link/ether fa:57:ef:7b:d0:d7 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 20.111.156.71/32 brd 20.111.156.71 scope global eth0
        valid lft forever preferred lft forever
```

해당 내용 중에서 `20.111.156.71` 이라는 IP가 할당된 것을 확인할 수 있다.  
이는 방금 생성된 파드의 IP로, Pause Container에 연결된 Network Interface가 파드의 IP를 생성한 것이다.

다음으로 Pause Container의 인터페이스가 Host Network의 가상 인터페이스에 연결되는 것을 확인해 볼 것이다.  
이 때 route 명령어를 사용해야 하기 때문에, 다음의 커맨드로 net-tools를 먼저 설치한다.

```yaml
$ yum -y install net-tools
```

이제 다음의 route 명령으로 가상 인터페이스를 조회한다.  
Calico가 생성한 가상 인터페이스의 이름은 cal로 시작하기 때문에, 다음의 명령어로 조회할 수 있다.

```bash
$ route | grep cal

20.111.156.65   0.0.0.0   255.255.255.255  UH  0     0   0  calib2890b39490
26.111.156.71   0.0.0.0   255.255.255.255  UH  0     0   0  calid73126c1689
link-local      0.0.0.0   255.255.0.0      U   1002  0   0  eth0
link-local      0.0.0.0   255.255.0.0      U   1003  0   0  ethl
```

이 때 `26.111.156.71` 이 방금 생성한 Pod의 IP이고, `calid73126c1689` 이 Pause Container와 1:1 매칭되는 가상 인터페이스의 Id이다.

이제 다음의 명령어로 컨테이너의 세부 정보 중 NetworkMode를 조회해보자.
다음과 같은 형태로 실행하면 된다. - `docker inspect <container-id> -f "{{json .HostConfig.NetworkMode}}"`
파드 내에 생성한 두 컨테이너의 NetworkMode를 조회하면 다음과 같다.

```bash
$ docker inspect 33912ec43b63 -f "{{json .HostConfig.NetworkMode}}"
"container:c2924165ad2510c34a8bf0f2553f7f6fd5ca40b94e4d81c64c7681f5afba2f8e"

$ docker inspect 9a5a6Oec8abb -f "{{json .HostConfig.NetworkMode}} "container:c2924165ad2510c34a8bf0f2553f7f6fd5ca40b94e4d81c64c7681f5afba2f8e"
```

출력된 값에서 `c2924165ad25` 부분은 아까 조회했던 Pause Container의 container id 값이다.  
두 컨테이너의 NetworkMode 값이 동일한 것을 확인할 수 있다.  
이를 통해 해당 컨테이너들이 파드 내의 Pause Container와 네트워크를 공유하게 되고, 두 컨테이너 모두 Pause Container에 연결되어 있는 가상 인터페이스를 통해 외부와 통신할 수 있게 되는 것이다.

## Pod Network 실습

이번엔 서로 다른 노드에 있는 파드 간의 통신을 실습하면서, 패킷이 어떻게 변화하는지 확인해보자.  
먼저 다음의 구성 파일로 Source 파드와 Target 파드를 생성한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-src
  labels:
    type: src
spec:
  nodeSelector:
    kubernetes.io/hostname: k8s-node2
  containers:
    - name: container
      image: kubetm/init
      ports:
        - containerPort: 8080
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-dest
  labels:
    type: dest
spec:
  nodeSelector:
    kubernetes.io/hostname: k8s-node1
  containers:
    - name: container
      image: kubetm/app
      ports:
        - containerPort: 8080
```

이 상태에서 Overlay Network를 먼저 확인해보자.

```bash
$ kubectl describe IPPool
...
Spec:
  Ipipmode:   Always
  Vxlanmode:  Never
  Cidr: 20.96.0.0/12
```

Ipipmode와 Vxlanmode에 대한 설정을 확인할 수 있다.  
또한 Overlay Network에서 20.96.0.0의 CIDR을 사용하고 있는 것을 확인 가능하다.  
다음의 커맨드로 확인한 Pod Network CIDR과 동일한 대역이다.

```bash
$ kubectl cluster-info dump | grep -m 1 cluster-cidr
```

이제 인터페이스의 트래픽을 확인해보자.
먼저 트래픽을 확인하기 위해 다음의 커맨드로 `tcpdump`를 설치한다.

```bash
$ yum -y install tcpdump
```

먼저 각각의 가상 인터페이스의 id를 확인해야 한다.  
route 커맨드를 통해 node1, node2에서 각각 가상 인터페이스의 id를 확인한다.

```bash
# node 2
$ route | grep cal
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
20.109.131.5    0.0.0.0         255.255.255.255 UH    0      0      0   calia5814ebbae9

# node 1
$ route | grep cal
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
20.111.156.72   0.0.0.0         255.255.255.255 UH    0      0      0   calib8c6d5407e3
```

node2에서 조회된 `20.109.131.5`는 Source Pod의 IP이고, node1에서 조회된 `20.111.156.72`는 Target Pod의 IP이다.  
먼저 Source Pod의 가상 인터페이스에 tcpdump를 걸어두고, Source Pod에서 Target Pod로 요청을 보내면 다음과 같이 확인된다.

```bash
# tcpdump를 걸어둔 후, Source Pod에서 curl 20.111.156.72:8080/hostname 실행
$ tcpdump -i calia5814ebbae9

06:23:19.857336 IP 20.109.131.5.35096 > 20.111.156.72.webcache: Flags [P.], seq 1:91, ack 1, win 219, options [nop,nop,TS val 20
11667352 ecr 2011729227], length 90: HTTP: GET /hostname HTTP/1.1
```

`20.109.131.5` IP에서 `20.111.156.72` IP로 요청을 보낸 내역이 확인된다.  
`HTTP: GET /hostname HTTP/1.1` 으로 요청을 보낸 것으로 표시된다.

이번엔 node2의 Host Interface의 가상 인터페이스에서 트래픽을 확인해보자.  
다음의 명령어로 Host Interface의 가상 인터페이스를 조회한다.

```bash
$ ip addr

3: eth1: <BROADCAST,MULTICAST,UP,LOWER UP> mtu 1500 gdisc pfifo fast state UP group default glen 1000 link/ether 08:00:27:c0:02:d9 brd ff:ff:ff:ff:ff:ff inet 192.168.219.23/24 brd 192.168.219.255 scope global eth1 valid lft forever preferred_lft forever inet6 fe80::a00:27ff:fec0:2d9/64 scope link valid lft forever preferred lft forever
```

여러 호스트 네트워크 목록이 나오는데, 192.168.219.23이 node2의 host에 대한 IP이므로 해당 부분을 확인한다.  
여기에서는 eth1이 가상 인터페이스의 이름이다.  
tcpdump로 해당 인터페이스의 트래픽을 확인해보자.

```bash
$ tcpdump -i eth1

06:30:39.390535 IP k8s-node2 > k8s-node1: IP k8s-node2.35552 > 20.111.156.72.webcache: Flags [.], ack 141, win 228, options [nop,nop,TS val 2012106885 ecr 2012168761], length 0 (ipip-proto-4)
```

여기서는 `k8s-node2 > k8s-node1` 부분에서 k8s-node2로부터 k8s-node1으로 트래픽이 흐르고 있음을 알 수 있다.  
k8s-node1으로 트래픽이 흐르도록 패킷이 incapsulate 되어있는 것이다.  
또한 `k8s-node2.35552 > 20.111.156.72.webcache` 부분에서 Target Pod의 IP인 20.111.156.72를 확인할 수 있다.  
그리고 `ipip-proto-4`로 되어있는 부분에서 ipip 모드의 Overlay Network를 사용하고 있음을 알 수 있다.

이번엔 Target Pod가 위치해있는 k8s-node1의 가상 인터페이스에서 트래픽을 확인해보자.
같은 방식으로 트래픽을 확인한다.

```bash
# node1
$ tcpdump -i eth1

06:37:19.484724 IP k8s-node2 > k8s-node1: IP 20.109.131.0.35968 > 20.111.156.72.webcache: Flags [.], ack 141, win 228, options [nop,nop,TS val 2012506979 ecr 2012568854], length 0 (ipip-proto-4)
```

node2에서 node1으로 트래픽이 흘렀다는 것과, Source Pod의 IP 및 Target Pod의 IP를 확인할 수 있다.  
해당 지점까지 패킷은 여전히 incapsulate 되어있다.

이제 Target Pod의 가상 인터페이스에서 트래픽을 확인해보자.

```bash
# node1
$ tcpdump -i calib8c6d5407e3
06:38:59.730182 IP k8s-node1.36072 > 20.111.156.72.webcache: Flags [P.], seq 1:91, ack 1, win 219, options [nop,nop,TS vat 2012607224 ecr 2012669099], length 90: HTTP: GET /hostname HTTP/1.1
```

해당 지점에서는 패킷이 decapsulate 되어 있는 것을 확인할 수 있다.

## ClusterIP Service Network 실습

이번에는 Service Network를 실습해보자.  
다음의 구성 파일로 Source Pod에 ClusterIp 타입의 서비스 객체를 붙인다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-clusterip
spec:
  selector:
    type: dest
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
```

생성된 서비스의 IP는 `10.107.94.97`이다.  
Source Pod로부터 해당 서비스 IP로 요청을 보내고, Source Pod 단에 붙어있는 가상 인터페이스에서 트래픽을 확인해보자.

```bash
# tcpdump를 걸어둔 후, Source Pod에서 curl 10.107.94.97:8080/hostname 실행
$ tcpdump -i calia5814ebbae9

06:52:00.852144 IP 20.109.131.5.54176 > 10.107.94.97.webcache: Flags [P.], seq 1:90, ack 1, win 219, options [nop,nop,TS val 201 3388347 ecr 2013450222], length 89: HTTP: GET /hostname HTTP/1.1
```

이 단계까지는 Service의 IP인 `10.107.94.97`이 확인되고 있다.

이번엔 Host Interface단의 인터페이스에서 확인해보자.  
이 지점에서는 Route의 NAT 기능을 통해 Service IP가 Pod IP로 변환된 상태이다.

```bash
$ tcpdump -i eth1

06:53:20.017454 IP k8s-node2 > k8s-node1: IP k8s-node2.54262 > 20.111.156.72.webcache: Flags [.], ack 140, win 228, options [nop ,nop,TS val 2013467512 ecr 2013529387], length 0 (ipip-proto-4)
```

k8s-node2 > k8s-node1 으로 incapsulate 되어 있다.  
또한 아까까진 Service IP로 표시되었던 부분이 Pod IP인 `20.111.156.72` 으로 변환되어 있는 것을 확인할 수 있다.  
이후 단계에서는 Pod Network에서 설명한 것과 동일하게 동작하게 된다.

## NodePort Service Network 실습

이번에는 NodePort 서비스를 이용해서 Service Network를 실습해보자.  
다음의 구성 파일로 Source Pod에 NodePort 타입의 서비스 객체를 붙인다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-nodeport
spec:
  selector:
    type: dest
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 31080
  type: NodePort
```

31080번의 NodePort가 연결된다.  
먼저 다음의 커맨드로 NodePort가 제대로 연결되어있는지 확인한다.

```bash
$ netstat -anp | grep 31080
tcp    0  0 0.0.0.0:31080       0.0.0.0:*    LISTEN     3426/kube-proxy
```

31080번 포트로 Listen 중인 상태이며, kube-proxy에 의해 해당 프로세스가 생성되었음을 확인할 수 있다.

이제 master node에서 node2의 NodePort로 트래픽을 보내보자.  
node2의 Host Interface의 가상 인터페이스에서 trafficdump를 하면 다음과 같다.

```bash
# tcpdump를 걸어둔 후, Master Node에서 curl 20.111.156.72:8080/hostname 실행
$ tcpdump -i eth1

07:01:36.005836 IP k8s-master.43758 > k8s-node2.31080: Flags [.], ack 141, win 237, options [nop,nop,TS val 2014129977 ecr 20140 25375], length 0

07:01:36.005894 IP k8s-node2 > k8s-node1: IP k8s-node2.43371 > 20.111.156.72.webcache: Flags [.], ack 141, win 237, options [nop ,nop,TS val 2014129977 ecr 2014025375], length 0 (ipip-proto-4)
```

먼저 master node로부터 node2로 향하는 트래픽이 확인된다.  
내부적으로는 이후에 iptables를 거쳐서 Router의 NAT 기능에 의해 Pod의 IP로 변환되어 흐르게 된다.  
실제로 직후에 찍힌 트래픽에서는 파드의 IP인 20.111.156.72으로 향하는 것으로 확인된다.
