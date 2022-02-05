## 도커 네트워크
## 쿠버네티스 네트워크
## 쿠버네티스 CNI(Container Network Interface)
[공식문서 - 클러스터 네트워킹](https://kubernetes.io/ko/docs/concepts/cluster-administration/networking/)
쿠버네티스 네트워크 환경을 구성해주는 것을 CNI add on이라고 한다. 공식적으로 k8s CNI는 k8s의 core component에 속하지 않기에 add on으로 분류되며, CNI는 컨테이너의 네트워크 문제에만 관여하기 때문에 그 외의 요소에는 별도의 제한이 없다. 때문에 공식 문서만 해도 엄청나게 많은 CNI 3rd-party 프로젝트를 소개하고 있으며 이들 간의 특징이 각각 다르다.

CNI는 컨테이너의 netns를 세팅하고, 호스트의 bridge와 컨테이너 사이에 veth를 연결하여 각 네트워크 인터페이스마다 대역에 맞는 IP를 할당하는 작업을 수행한다. 이 내용이 1주차에 수행했던 실습 내용인데, 결국 CNI는 반드시 k8s가 아닌 다른 런타임에서도 동일하게 동작할 수 있다는 이야기가 된다.

*찾아보니 CNI는 애초에 쿠버네티스에 종속된 플러그인이 아니라 CNCF(Cloud Native Computing Foundation) 프로젝트임.

k8s CNI의 공식 명세에 따르면 k8s CNI는 4가지 네트워크 문제를 해결해야 한다.

### k8s CNI가 해결해야 하는 4가지 네트워크 문제
1. 파드<-> 파드 간 NAT 없이 통신 가능해야 한다.
2. 파드<->서비스 간 통신 가능해야 한다.
3. 외부<->서비스 간 통신 가능해야 한다.
4.  파드 내 여러 컨테이너가 동작 시, pause 컨테이너의 netns를 공유하여 컨테이너간 통신이 localhost로 가능해야 한다.

### k8s 네트워크의 동작 기준
1. 노드가 달라도 Pod to Pod간 NAT없이 통신 가능해야 한다.
2. 노드의 에이전트(system daemons, kubelet)는 해당 노드의 모든 파드와 통신 가능해야 한다.
3. 노드의 host network에 있는 파드는 NAT 없이 모든 노드의 파드와 통신 가능해야 한다.
4. 서비스 클러스터 IP 대역은 각 노드에 할당된 파드의 IP 대역과 겹치지 않아야 한다.(IPM)

## flannel
 
 여러개의 워커노드의 파드간 NAT 없는 통신을 위한 overlay 네트워크 기법 중 하나를 VXLAN을 사용
 UDP 8472 사용
 Vtep
 
 flannel cni가 구성되면 flannel cni 파드가 목적지에 해당하는 노드의 iptables rule을 변경하는데 관여하고, 통신이 가능하게 됨.
 
 ## pause 컨테이너
 파드 안의 컨테이너 간에는 net ns와 ipc ns를 공유함
 
pause 컨테이너(인프라 컨테이너라고도 불림)
파드를 생성하고 나면 docker ps로 확인해봤을 때 동일 파드 내 pause라는 컨테이너가 추가로 배포됨
kubectl describe pod로 확인해보면 정작 보이진 않음
  
파드 안에 여러 컨테이너가 있으면, 각 컨테이너의 ip는 동일함.
![[Pasted image 20220203003452.png]]
어떻게 이게 가능하냐? 노드에서 보면 두 컨테이너의 net ns와 ipc ns가 같음.
하나의 파드 안에 존재하는 컨테이너들은 net ns와 ipc ns를 공유한다. 그러나 pid ns는 다르다. 이게 pause 컨테이너와 관련있음.
![[Pasted image 20220203003846.png]]
kubectl로는 조회가 잘 안되므로 노드에서 docker inspect 명령으로 pause 컨테이너에 접근해서 확인해보면,
1번 컨테이너의 network mode와, 2번 컨테이너의 network mode는 다른 어떤 컨테이너의  network mode를 공유해서 사용함. 그 컨테이너가 pause임.

즉, pasue 컨테이너가 net ns와 ipc ns를 생성하고 해당 파드 안의 다른 컨테이너들은 pause에서 만들어준 net ns와 ipc ns를를 공유해서 사용한다.
굳이 왜 이렇게 만들었냐? 컨테이너 1,2,3번이 있으면 1번이 ns 만들고 그거 공유해서 쓰면 되는거 아니냐? -> 만약 1번 컨테이너가 삭제되면 그 ns들도 날아감. 그래서 pause가 먼저 생성되어서 이걸 잡고 있어야 함. 그래서 인프라 컨테이너라고도 하는 것. 영향도를 최소화 하기 위해서.

동일 파드에 nginx 컨테이너와 netshoot 컨테이너를 띄워놓고, netshoot 컨테이너에서 curl localhost 찍어보면 nginx 응답이 온다. 그런데 ps로 확인해보면 웹서버 데몬이 돌고 있지는 않음. 응답이 nginx 컨테이너에서 왔다는건데, 이게 가능한 이유?
ss -nltp 쳐보면 80번 포트에서 리스닝중인걸 확인할 수 있음. pid ns는 격리되어있기 때문에 nginx 프로세스는 보이지 않지만, net ns는 공유되어 있기 때문에 리스닝중인 포트 정보가 보이는 것.

*yaml파일에서 옵션값으로 pid ns도 공유시킬 수 있기는 함. 기본값은 격리임.

*만약 1pod-multi container 구조를 사용한다면, 레거시에서 넘어올 때 모니터링 방식을 바꿔야 함. 동일한  ip를 여러 컨테이너가 공유해서 쓰기 때문에.

 *debug pod??

하나의 파드의 특정 컨테이너에 침해사고 발생시 어떻게 조치?




 
 

![[Pasted image 20220202171112.png]]
출처 : https://coffeewhale.com/packet-network1

즉, 사용자로부터 pod 생성 요청이 들어오면 kubelet 위에 정리된 작업을 CNI가 해주는 것.





es 권고는 local nvme 사용. nas 사용은 속도저하가 심해서 권장 안한다고 함. -> 이 사유로 k8s에서 온프렘으로 deprecate 한 사례도 있음

https://coffeewhale.com/packet-network1


https://coffeewhale.com/k8s/network/2019/05/30/k8s-network-03/