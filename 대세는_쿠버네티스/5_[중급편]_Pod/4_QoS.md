# QoS (Quality of Service)

## 필요한 상황

특정 노드에 파드 3개가 떠서 균등하게 자원을 분배해서 여유분 없이 사용하고 있다고 하자.  
만약 이 때 특정 파드에서 자원을 추가로 요구하면 해당 파드를 다운 시킬지 아니면 다른 파드를 다운 시켜서 자원을 할당할지를 결정해야 한다.

<img src="./images/QoS1.png" width=30% />

## QoS Class 정책

쿠버네티스에서는 QoS(Quality of Service)를 통해 각 파드에 우선 순위를 부여하여 관리한다.  
각 파드는 `Guaranted`, `Burstable`, `BestEffort` 로 나뉘는데, Guaranted 파드는 최우선적으로 자원을 할당받는 파드이다.  
위 상황에서 자원을 추가로 요구한 파드가 Guaranted라면 BestEffort로 할당된 파드를 다운시키고 Guaranted 파드를 유지한다.  
만약 추가 요구한 파드가 Burstable이라면 자신이 다운된다.

<img src="./images/QoS2.png" width=50% />

## QoS Class 결정 방법

QoS를 관리하는 별도의 객체가 있는 것은 아니고, 파드의 requests와 limits에 지정한 값에 따라 QoS Class가 결정된다.

만약 파드 내의 모든 컨테이너에 requests, limits가 설정되어 있고, 각 컨테이너에서 requests와 limits에 지정한 값이 동일하다면 `Guaranted`가 된다.  
만약 파드 내의 모든 컨테이너에 requests와 limits가 누락되었다면 `BestEffort`가 된다.  
그 외의 경우들은 `Burstable`로 간주된다.  
requests와 limits에 지정한 값이 다르거나, requests만 지정되거나, 일부 컨테이너에 requests/limits가 지정되지 않은 등의 경우가 포함된다.

<img src="./images/QoS3.png" width=80% />

## Burstable 클래스의 OOM Score

Burstable 파드 중에는 어떤 파드가 먼저 종료될지에 대한 우선 순위를 `OOM(Out Of Memory) Score`를 기준으로 판단한다.  
OOM은 requests에 지정된 메모리를 기준으로 몇 퍼센트만큼 메모리를 사용하고 있는지 계산한 값으로, 이 점수가 더 높은 파드가 먼저 삭제된다.  
예를 들어 A, B 두 파드에서 동일하게 4G의 메모리를 사용하고 있을 때 A의 request 메모리가 5G이고 B가 8G였다고 하자.  
이 때 A의 OOM은 75%이고 B의 OOM은 50%이므로, OOM이 더 높은 A 파드가 먼저 다운된다.

<img src="./images/QoS4.png" width=80% />

출처: [인프런 대세는 쿠버네티스 [초급 ~ 중급]](https://inf.run/yW34)
