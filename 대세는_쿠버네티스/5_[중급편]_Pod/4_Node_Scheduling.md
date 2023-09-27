# Node Scheduling

파드는 노드에 스케줄링 되어 사용되는데, 어떤 노드에 할당될지를 지정하거나 특정 노드에는 할당되지 않도록 설정하는 것이 가능하다.

## 노드를 선택하는 속성 - NodeName, NodeSelector, NodeAffinity

`NodeName`, `NodeSelector`, `NodeAffinity`는 어떤 노드에 파드를 할당할지 선택하는 속성들이다.

`NodeName`은 직접적으로 노드 이름을 명시해서 해당 노드에 파드를 할당한다.
다만 노드는 지속적으로 다시 뜰 수 있고 이름도 변경될 수 있기 때문에 NodeName은 잘 사용하지 않는다.

`NodeSelector`의 경우 지정한 라벨에 맞는 노드에 파드를 할당시킨다.  
해당 라벨을 가진 노드가 여러개이면 그 중 자원이 많은 노드를 선택해서 할당하고, 만약 라벨을 가진 노드가 없다면 파드가 어디에도 할당되지 않아서 에러가 발생한다.

`NodeAffinity`는 NodeSelector 보다 좀 더 유연하게 작동한다.  
라벨의 key/value가 matchExpressions에 지정한 조건식에 맞는 노드들을 고르고, 그 중에서 자원이 많은 노드에 파드를 할당한다.  
옵션을 지정하면 조건을 만족하는 노드가 없을 경우 다른 노드에라도 파드를 할당시키도록 설정할 수 있다.

<img src="./images/NodeScheduling1.png" />

## 파드를 노드에 집중시키거나 분산시키는 속성 - PodAffinity, PodAntiAffinity

다음으로 파드를 특정 노드에만 생성되게 하거나, 겹치는 노드 없이 분산시키는 옵션이 있다.  
PodAffinity는 파드를 특정 노드에만 파드를 할당시키는 옵션으로, 예를 들어 hostPath 볼륨을 공유해서 사용해야 하는 등 모든 파드가 동일한 노드에 할당되어야 할 때 사용한다.

먼저 하나의 파드가 다른 노드에 할당되어 pv가 만들어진 상태라고 하자.  
새로운 파드를 생성할 때 PodAffinity 속성을 주고 기존 파드와 동일한 라벨을 삽입해서 생성하면 동일한 노드에 파드가 할당되어, hostPath 볼륨을 함께 사용할 수 있다.

이와 달리 AntiAffinity 속성을 주면 반드시 기존 파드와 다른 노드에 파드가 생성된다.  
보통 Master - Slave 관계에서 하나를 백업 파드로 사용해야 할 때, 다른 노드에 백업 파드를 위치시키기 위해 사용한다.

<img src="./images/NodeScheduling2.png" width=80% />

## 파드를 노드에 집중시키거나 분산시키는 속성 - PodAffinity, PodAntiAffinity
