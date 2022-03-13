AWS EKS에 Cilium CNI 설치하기

# 1. Cilium CNI?
---
iptables를 기반으로 하는 전통적인 네트워크 트래픽 제어는 현재까지도 널리 사용되고 있으며, Kubernetes의 CNI도 iptables를 기반으로 동작합니다. 그러나 개인적으로 동적으로 변화하는 MSA 환경에서 iptables 기반으로 IP/port를 관리하는 방식의 효율성에 대해서는 의문이 듭니다. 물론 대부분의 상황에는 CNI가 문제없이 처리해주고 있지만 만약 예외적인 장애상황이 발생했을 때, 겨우 파드<-> 파드간 통신에 있어 그 사이에 여러 홉이 존재하며, 그들 간의 네트워크를 이해해야 한다는 사실은 네트워크 엔지니어가 아닌 저에게 있어 여전히 부담스럽습니다.

Cilium은 여기에 대해서 다른 방식의 접근을 제시하는 CNI입니다. BPF가 DNAT를 미리 수행하여 (사용자가 보는 입장에서)네트워크를 단순화하고 가시성을 좋게 하며, 성능적으로도 보다 나은 효율을 낸다는 점에서 관심을 끌었습니다. KANS 스터디의 졸업과제를 겸하여 AWS EKS에 Cilium CNI를 적용하는 과정을 소개하는 이번 글을 시작으로 Cilium CNI에 대해서 자세히 알아보고자 합니다.

# 2. EKS 클러스터 구성
---
## 2-1. 주의사항

## 2-2. Terraform 이용한 구성
![[Pasted image 20220312114930.png]]

실습 환경의 아키텍처는 위와 같습니다. 간단한 실습을 목적으로 구성한 데모환경이므로 그리 복잡하지 않습니다.

> github 링크 추가 예정

실습 환경은 테라폼으로 구성하였으나, eksctl로 별도 구성해도 관계 없음.

## 2-3. eksctl 사용
```bash
$ curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

$ sudo mv /tmp/eksctl /usr/local/bin

$ which aws-iam-authenticator

$ eksctl create cluster --name test-cluster --without-nodegroup

```
# 3. Prerequisites
---
## 3-1. 주의사항
- EKS Fargate를 사용하는 경우 AWS VPC CNI외의 CNI 플러그인은 사용할 수 없습니다.
- 공식적으로 VPC CNI 외의 플러그인은 AWS의 지원범위에 해당하지 않습니다.

> 확인중인 내용
	- 기존에 iptables에 정의한 정책이 있는 경우, iptabels rule을 Calico의 정책에 추가하면 기존 정책이 Calico에 의해 덮어씌워지는 것을 방지할 수 있습니다.(Cilium 확인 필요)
	- pod 레벨의 Security Group 사용 시 Security Group만 적용되며 트래픽은 Calico에 의해 제어되지 않습니다.(Cilium 확인 필요)

## 3-2. (옵션)Lens 설치
CLI로 클러스터를 제어하기 어려운 경우 Lens를 사용하여 보다 편리하게 쿠버네티스 클러스터를 관리할 수 있습니다.
> Lens 설치

## 3-3. AWS VPC CNI 제거
Cilium은 VPC CNI를 대신해서 ENI(Elastic Network Interface)를 관리하게 됩니다. AWS VPC CNI는 EKS 배포시 기본적으로 적용되는 CNI로, EKS 클러스터를 생성하면 DaemonSet에 의해 각 노드별로 aws-node 라는 이름을 가진 파드가 배포됩니다. VPC CNI를 제거하기 위해서 Daemonset을 제거합니다. 일반적인 경우에는 AWS VPC CNI를 사용하는 것으로 충분하지만, 간혹 별도의 CNI Plugin을 필요로 하는 상황이 있는데 이런 경우에는 지금과 같이 VPC CNI를 제거한 뒤 새로운 CNI를 구성해야 합니다.

```bash
$ kubectl get daemonset aws-node -n kube-system
NAME       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE

aws-node   3         3         3       3            3           <none>          2d3

$ kubectl delete daemonset aws-node -n kube-system
daemonset.apps "aws-node" deleted
```


## 3-4. helm 설치
1.  환경에 맞는 버전 다운로드 [링크](https://github.com/helm/helm/releases)
2. 압축 해제  `tar -zxvf helm-v3.8.0....tar.gz`
3. 압축 해제한 디렉토리에서 helm 실행파일을 이동 `mv linux-amd64/helm /usr/local/bin/helm`


# 4. Cilium CNI 구성
---
## 4-1. Cilium 설치
```bash
$ helm repo add cilium https://helm.cilium.io/

$ helm install cilium cilium/cilium --version 1.11.2 \
  --namespace kube-system \
  --set eni.enabled=true \
  --set ipam.mode=eni \
  --set egressMasqueradeInterfaces=eth0 \
  --set tunnel=disabled
```

`eni=true`, `--set ipam.mode=eni`,  `tunnel=disabled` 를 설치시 옵션으로 주면 각각의 pod에 IP를 할당할 때 Cilium에서 라우팅 할 수 있는 IP를 할당하게 되고 이는 AWS ENI를 타게 됩니다.Cilium의 동작을 위해서 overlay 라우팅 모드로 설치됩니다.

```bash
$ kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOSTNETWORK:.spec.hostNetwork --no-headers=true | grep '<none>' | awk '{print "-n "$1" "$2}' | xargs -L 1 -r kubectl delete pod
```

Cilium 설치 이후에 Cilium에 의해 관리되고 있지 않던 pod들은 restart가 필요합니다. 재시작 이후에는 Cilium에 의해 network 통신이 관리되고, network policy 또한 Cilium에 의해 제어받습니다. 만약 운영중인 쿠버네티스 클러스터에 Cilium을 설치하고 적용하고자 한다면 이 부분에서 주의가 필요할 것 같습니다.

## 4-2. Cilium 설치 확인
```bash
$ kubectl -n kube-system get pods --watch

NAME                                            READY   STATUS    RESTARTS   AGE
cilium-4496d                                    1/1     Running     0          37m
cilium-5zkb5                                    1/1     Running     0          37m
cilium-9wnj6                                    1/1     Running     0          37m
cilium-operator-58b56cb859-p5ntn                1/1     Running     0          37m
cilium-operator-58b56cb859-s2b6q                1/1     Running     0          37m
coredns-6dbb778559-7vm98                        1/1     Running     0          35m
coredns-6dbb778559-psdgr                        1/1     Running     0          36m
hubble-generate-certs-26mgp                     0/1     Completed   0          21m
hubble-relay-6fcbfb58f9-g9zrr                   1/1     Running     0          21m
hubble-ui-5f7cdc86c7-rwkm5                      3/3     Running     0          21m

...

```

저의 테스트 환경에서는 3대의 워커 노드를 사용하고 있으므로 cilium 파드가 3개 생성되었습니다.

```bash
kubectl -n kube-system get pods --watch
NAME                               READY   STATUS    RESTARTS   AGE
cilium-bfhwk                       1/1     Running   0          92s
cilium-operator-58b56cb859-hdh5q   0/1     Pending   0          92s
cilium-operator-58b56cb859-t6lw7   1/1     Running   0          92s
coredns-6dbb778559-2dq29           1/1     Running   0          86s
coredns-6dbb778559-hglhd           1/1     Running   0          84s
```

만약 단일 노드로 구성된 클러스터에 배포하는 경우, 위와 같이 1개의 Operagor는 Pending상태를 유지하게 됩니다. cilium-operator의 deployment가 정의될 때 pod antiaffinity와 replicas=2가 설정되어 있기 때문이므로 그대로 진행하시면 됩니다. 단, 아래 connectivity test 수행 시 multi-node에 대한 통신을 체크하는 pod가 배포되어도 pending 상태를 유지하기 때문에 multi-node 환경에 대한 체크가 이루이지지 못 합니다.

## 4-3. Cilium cli 설치
```bash
$ curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum

$ sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin

$ rm cilium-linux-amd64.tar.gz{,.sha256sum}
```

## 4-4. 설치 확인
```bash
$ cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:         OK
 \__/¯¯\__/    Operator:       OK
 /¯¯\__/¯¯\    Hubble:         disabled
 \__/¯¯\__/    ClusterMesh:    disabled
    \__/

Deployment        cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
DaemonSet         cilium             Desired: 3, Ready: 3/3, Available: 3/3
Containers:       cilium-operator    Running: 2
                  cilium             Running: 3
Cluster Pods:     13/13 managed by Cilium
Image versions    cilium-operator    quay.io/cilium/operator-aws:v1.11.2@sha256:abb7af69d6679e64dab9d7a87eae73377b3e9b880ff90ab8689ad1bf9e6ff3cd: 2
                  cilium             quay.io/cilium/cilium:v1.11.2@sha256:4332428fbb528bda32fffe124454458c9b716c86211266d1a03c4ddf695d7f60: 3
```

# 5. 통신 테스트
---
## 5-1. cilium connectivity test
```bash
$ kubectl create ns cilium-test

$ kubectl apply -n cilium-test -f https://raw.githubusercontent.com/cilium/cilium/v1.9/examples/kubernetes/connectivity-check/connectivity-check.yaml

$ cilium connectivity test
ℹ️  Monitor aggregation detected, will skip some flow validation steps
⌛ ...


🏃 Running tests...

[=] Test [no-policies]
....................................
[=] Test [allow-all-except-world]
..............
[=] Test [client-ingress]
..
[=] Test [echo-ingress]
....
[=] Test [client-egress]
....
[=] Test [to-entities-world]
......
[=] Test [to-cidr-1111]
....
[=] Test [echo-ingress-l7]
....
[=] Test [client-egress-l7]
..........
[=] Test [dns-only]
..........
[=] Test [to-fqdns]
......
✅ All 11 tests (100 actions) successful, 0 tests skipped, 0 scenarios skipped.

```

Cilium cli에서 지원하는 기능을 이용하여 통신 테스트를 수행할 수 있습니다.

## 5-2.  Cilium-starwars-demo
테스트 시나리오는 star wars에서 따온 것으로 보입니다. deathstar는 deployment와 service로 80 포트가 오픈되어 있으며, tiefighter 파드와 xwing 파드 모두 deathstar의 웹서비스에 접근할 수 있는 상태입니다. deathstar에서는network policy를 세워 tiefighter 파드만 접근 가능하도록 제어해야 합니다. 위와 같은 시나리오를 통해 Cilium이 정상 동작함을 검증할 수 있습니다. 

저는 스타워즈 시나리오를 잘 모르지만, tiefighter 가 제국군, xwing이 aliance 진영인 것으로 설명되어 있네요. 많은 demo application에서 star wars를 모티브로 하는 경우를 보았는데, 외국에서는 star wars가 대표적인 밈 격으로 사용되나 봅니다.


```bash
$ kubectl create -f https://raw.githubusercontent.com/cilium/cilium/v1.11/examples/minikube/http-sw-app.yaml
service/deathstar created
deployment.extensions/deathstar created
pod/tiefighter created
pod/xwing created

$ kubectl get all
NAME                             READY   STATUS    RESTARTS   AGE
pod/deathstar-6f87496b94-85l2w   1/1     Running   0          6m20s
pod/deathstar-6f87496b94-b4ccm   1/1     Running   0          6m20s
pod/tiefighter                   1/1     Running   0          6m20s
pod/xwing                        1/1     Running   0          6m20s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/deathstar    ClusterIP   10.100.119.27   <none>        80/TCP    6m21s
service/kubernetes   ClusterIP   10.100.0.1      <none>        443/TCP   46m

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/deathstar   2/2     2            2           6m21s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/deathstar-6f87496b94   2         2         2       6m21s

```

테스트 환경을 배포합니다.

```bash

# 연합군 우주선 착륙 시도
$ kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed

# 제국군 우주선 착륙 시도
$ kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed

```

두 진영의 우주선이 모두 착륙(서비스 접근) 가능함을 확인하였습니다.

### L3/L4 network policy test
![L4 policy](https://docs.cilium.io/en/v1.11/_images/cilium_http_l3_l4_gsg.png)
Cilium 에서 network policy를 정의할 때, 적용 대상의 endpoint의 ip 주소는 network policy와 관련이 없습니다. pod의 label을 이용하여 적용 범위가 결정되기 때문입니다. 

```bash
$ kubectl create -f https://raw.githubusercontent.com/cilium/cilium/v1.11/examples/minikube/sw_l3_l4_policy.yaml
```

L3/L4 network policy를 적용합니다.

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L3-L4 policy to restrict deathstar access to empire ships only"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
```
적용되는 yaml 파일을 살펴보면, label 을 통해 empire 진영(tiefighter)의 ingerss를 TCP 80포트 레벨에서 **화이트리스트**로 허용하고 있음을 확인할 수 있습니다. 시나리오로 보자면, `org=empire` 라는 표식을 가진 우주선만 착륙 가능한 것입니다.

```bash
$ kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed

$ kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
time out 발생
```

실제로 xwing 파드에서 접근시 time out이 발생하는 것을 확인할 수 있습니다.

### L7 network policy test
![L7policy](https://docs.cilium.io/en/v1.11/_images/cilium_http_l3_l4_l7_gsg.png)
MSA 구조에서 마이크로 서비스 간 제한은 HTTP 레벨에서 동작해야 합니다. 위 시나리오에서는 deathstar 라는 서비스에서는 같은 제국군의 pod라 할 지라도 임의의 tiefighter 파드에서 접근해서는 안되는 API(자폭 버튼=관리자 페이지라던가..)가 있을 수 있습니다.

```bash
$ kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Panic: deathstar exploded

goroutine 1 [running]:
main.HandleGarbage(0x2080c3f50, 0x2, 0x4, 0x425c0, 0x5, 0xa)
        /code/src/github.com/empire/deathstar/
        temp/main.go:9 +0x64
main.main()
        /code/src/github.com/empire/deathstar/
        temp/main.go:5 +0x85
```

시나리오에서 테스트 해보면 현재는 모든 tiefighter 파드에서 자폭 버튼 api에 접근 가능합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.11/examples/minikube/sw_l3_l4_l7_policy.yaml
```

위 policy의 yaml은 다음과 같습니다.
```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L7 policy to restrict access to specific HTTP call"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "POST"
          path: "/v1/request-landing"
```

L7 레벨에서 경로 기반의 정책이 추가되어 tiefighter는 POST method로 /v1/request-landing에만 접근 가능함을 확인할 수 있습니다. 적용 이후 다시 접근을 시도해봅니다.

```bash
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed

kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
Access denied
```

기존과 같이 /v1/request-landing은 접근 가능하나, /v1/exhaust-port는 접근 권한이 없어 Access denied가 리턴되는 것을 확인할 수 있습니다.

이렇게 CiliumNetworkPolicy를 이용하여 L3/L4, L7 단에서 네트워크 트래픽 제어가 가능하며, Cilium이 정상 동작하고 있음을 확인하였습니다.

# 부록. Hubble 설치(모니터링)
```bash
$ helm upgrade cilium cilium/cilium --version 1.9.13 \
   --namespace $CILIUM_NAMESPACE \
   --reuse-values \
   --set hubble.listenAddress=":4244" \
   --set hubble.relay.enabled=true \
   --set hubble.ui.enabled=true
   
$ kubectl port-forward -n $CILIUM_NAMESPACE svc/hubble-ui --address 0.0.0.0 --address :: 12000:80

$ export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)

$ curl -LO "https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz"

$ curl -LO "https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz.sha256sum"

$ sha256sum --check hubble-linux-amd64.tar.gz.sha256sum

$ tar zxf hubble-linux-amd64.tar.gz

$ sudo mv hubble /usr/local/bin

```

5-1에서 진행한 connectivity test의 flow를 hubble cli 또는 hubble ui를 이용하여 확인할 수 있습니다.

![[Pasted image 20220313153510.png]]