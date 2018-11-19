--- 
layout: post
title: "[K8S]POD에 대한 이론 편"
date: 2018-11-13 11:27:24 +0900
categories: [Kubernetes in action, Kubernetes, K8S]
tags: [Kubernetes, POD]
---

## 3 Running Containers in Kubernetes

### 3.1 [POD](https://kubernetes.io/docs/concepts/workloads/PODs/POD/)을 다시 소개한다면..
- POD은 함께 배치되는 컨테이너의 그룹으로 Kubernetes에 사용되는 기본적인 block이다.
- POD은 다수의 Containers를 가질 수 있으며, 항상 하나의 Worker Node위에서 동작한다.
- 아래 그림과 같이 POD은 여러 Node에 걸쳐서 작동하지 않는다.
- Node는 여러 개의 독립적인 PODs를 가지고, 개별 POD은 여러 개의 Container를 가질 수 있다는 의미입니다.

![Kubectl](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/images/k8s/chapter3/figure3.1.png)

> Q1. 멀티 Container가 하나의 컨테이너에서 Multi Process를 띄우는 것보다 나을까?

1. 만약 멀티 프로세스가 하나의 컨테이너에 떠 있고, 어떤 두 프로세스가 충돌이 났다고 가정을 한다면, 두 프로세스는 한 컨테이너의 같은 Standard Output을 이용해 log를 남길 것이고, 우리는 그 log의 주인이 누구인지 찾기 위해 많은 시간을 소비할 것이다.
1. Docker에서 설계한 Container는 하나의 Process를 가진다. 즉, Docker Container에 직접 들어가서 보면 1번 Process는 우리가 올린 Container의 실행시킨 처음 명령이다. 물론 실행한 프로세스들의 Children Process들이 더 떠 있을 수는 있으나, 다른 Process들은 관여하지 않는다. 이 개념은 K8S(Kubernetes)에서도 그대로 적용된다.

> Q2. 그렇다면 컨테이너를 직접 이용하지 않고, POD이라는 개념을 왜 만들었을까?

1. POD은 POD 내부에 존재하는 컨테이너가 Running 할 수 있는 동일한 환경을 제공하면서, 독립성을 보존할 수 있는 Container보다 조금 더 상위 레벨의 무언가가 필요했다. 이러한 POD은 Container들에 한꺼번에 명령을 내린다던지, 혹은 POD을 지움으로써 Container를 따로 삭제할 필요하 없는 등의 편의를 제공한다.

#### POD에 있는 Container들은 부분적으로 독립적인 이유
위와 같이 이야기할 수 있는 이유는 POD 내부의 모든 Container는 동일한 Network, UTS namespace(Linux namespace를 말한다), IPC(Inter-Process Communication) namespace 아래에서 동작하며, 각 POD들을 hostname과 network interface를 공유한다. IPC를 통해서 통신을 하게 된다. 최근 Docker와 K8S에서는 PID namespace까지 공유한다고 한다.(Default 아니라고 하는데... 확인해 봐야 할 듯하다.)
하지만 FileSystem(FS)의 경우는 조금 다르다. 한 Container의 FileSystem은 Container image에서 만들어 지기 때문에 Container별로 완벽하게 분리되어 있다. 물론 K8S에서는 [Volume](https://kubernetes.io/docs/concepts/storage/volumes/)이라는 개념을 이용해서 별도의 Shared File System을 만들 수 있지만, 이는 이후에 설명하기로 한다.

### POD의 IP와 PORT를 공유하는 방법
앞서 말한 바와 같이 POD의 Container들은 같은 Network Namespace를 공유하고 있다. 즉, 동일한 IP주소와 Port Space를 가지고 있다는 말이다. 다시 한번 즉, 우리는 하나의 POD에서 Container를 띄울 때 PORT가 충돌 나지 않도록 조심해야 한다는 것이다. 물론 그래서 동일한 POD 내부의 Container들은 localhost를 이용해서 통신이 가능한 것도 같은 이유이다.

### FLAT INNER-POD NETWORK - POD 간의 통신은 평평하다? => POD 간의 통신은 단조롭게 이뤄진다.

아래 그림처럼 K8S에 있는 POD들은 단순하고, 공유 가능한 네트워크 Address 주소 값을 가진다. 즉, 이 말은 각각의 POD은 각각의 IP주소 값을 가지고 있으며, 이 IP를 이용해서 통신을 하기 때문에, NAT(Network Address Translation) gateway 같은 장비 없이- 마치 Local Area Network(LAN)처럼 - 통신이 가능하다.

![Flat Inner-POD Network](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/images/k8s/chapter3/figure3.2.png)

### 전체 POD에 적절한 Container 구성

#### Multi-Tier App은 Multi POD으로 분리되어야 한다.
만약에 frontend Server와 database를 Container화 해서 하나의 POD에 넣는다고 한다면, 이를 막을 수는 없지만 적절한 조치는 아닐 것이다. 왜냐하면, 이러한 구성은 리소스(CPU와 메모리) 활용에 있어서 적절하지 않을 것이기 때문이다. 차라리 이러한 경우에는 Frontend를 한 POD(Node)에 backend를 다른 한 POD(Node)에 구성하는 것이 인프라를 활용함에 있어서 좀 더 효과적인 방법일 것이다.

#### POD은 개별적으로 scaling 될 수 있도록 분리되어야 한다.
위 예제에서 분리해서 구성해야 하는 다른 이유는 Scaling이다. 왜냐하면, POD은 Scaling의 기본 단위이며, K8S에서 수평적인 확장(Scaling)은 할 수 없으며, 하나의 POD 전체를 확장해야 하기 때문이다. 예를 들어 만약에 Frontend와 Backend 컨테이너를 하나의 POD에 구성했고, 이를 Scaling 한다면 결국 POD이 하나 더 생성이 되기 때문에 결국 두 개의 Frontend와 Backend를 가지게 된다. 그런데 Frontend와 Backend의 scaling을 위한 기준은 일반적으로 다르기에 이러한 방법은 좋지 않다. 그러므로 개별 컨테이너가 당신이 원하는 대로 scaling 하기 위해서는 명확히 분리해서 서로 다른 POD에 deploy 되도록 해야 한다.

#### 여러 개의 Container를 한 POD에서 사용할 때 이해해야 하는 점

제목처럼 한 POD에 여러 Container를 구성하는 주된 이유는 메인 프로세스와 보완적인 하나이상의 프로세스들로 구성되어 있을 때이다 - 아래 그림처럼

![Flat Inner-POD Network](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/images/k8s/chapter3/figure3.3.png)

예를 들어보면 메인 Container가 Web Server이고 특정 디렉터리에 파일을 받는다고 생각하고, 다른 Container(a sidecar container)는 임시로 webServer에서 다운로드하는 파일을 저장해 놓는 경우, 게다가 이 sidecar Container가 log rotator나 log collector 혹은 data processor, communication adapter 같은 것들을 포함하고 있다면 더욱더 그럴 것이다.

#### 하나의 POD에 여러 Container를 띄우기를 결정해야 할 때

두 개의 Container를 하나의 POD에 넣을지 혹은 두 개의 POD에 나눠서 넣을지 고민이 된다면, 아래와 같은 질문들을 스스로 해보기를 바란다.

> Do they need to be run together or can they run on different hosts?

> Do they represent a single whole or are they independent components?

> Must they be caled together or individually?

개인적인 의견을 말씀드린다면 가능하다면 POD:Container = 1:1을 유지하는 것이 좋아 보인다.

### POD 생성하기

기본적으로 Kubernetes에서는 POD을 포함한 다른 Resource는 JSON이나 YAML을 이용해서 만들 수 있다.

#### Examining a YAML descriptor of an existing POD
먼저 이전에 만든 POD에 대해서 YMAL을 볼 수 있다. 명령은 아래와 같다.

```sh
$ kubectl get pod kubia-96gdb -o yaml
apiVersion: v1
kind: Pod <-- Type Of Kubernetes Object/Resource
metadata:
creationTimestamp: 2018-11-12T08:27:58Z
generateName: kubia-
labels:
run: kubia
name: kubia-96gdb
namespace: default
ownerReferences:
- apiVersion: v1
blockOwnerDeletion: true
controller: true
kind: ReplicationController
name: kubia
uid: cea03641-e648-11e8-8d80-080027bf22d8
resourceVersion: "14609"
selfLink: /api/v1/namespaces/default/pods/kubia-96gdb
uid: dcaffc4e-e654-11e8-8d80-080027bf22d8
spec:
containers:
- image: imageurl/russell/kubia
imagePullPolicy: Always
name: kubia
ports:
- containerPort: 8080
protocol: TCP
resources: {}
terminationMessagePath: /dev/termination-log
terminationMessagePolicy: File
volumeMounts:
- mountPath: /var/run/secrets/kubernetes.io/serviceaccount
name: default-token-tr9zh
readOnly: true
dnsPolicy: ClusterFirst
imagePullSecrets:
- name: coe-registry-key
nodeName: minikube
restartPolicy: Always
schedulerName: default-scheduler
securityContext: {}
serviceAccount: default
serviceAccountName: default
terminationGracePeriodSeconds: 30
tolerations:
- effect: NoExecute
key: node.kubernetes.io/not-ready
operator: Exists
tolerationSeconds: 300
- effect: NoExecute
key: node.kubernetes.io/unreachable
operator: Exists
tolerationSeconds: 300
volumes:
- name: default-token-tr9zh
secret:
defaultMode: 420
secretName: default-token-tr9zh
status:
conditions:
- lastProbeTime: null
lastTransitionTime: 2018-11-12T08:27:58Z
status: "True"
type: Initialized
- lastProbeTime: null
lastTransitionTime: 2018-11-12T08:28:01Z
status: "True"
type: Ready
- lastProbeTime: null
lastTransitionTime: 2018-11-12T08:27:58Z
status: "True"
type: PodScheduled
containerStatuses:
- containerID: docker://dockerContainerID
image: imageurl/russell/kubia:latest
imageID: docker-pullable://imageurl/russell/kubia@sha256:encryptedPassword
lastState: {}
name: kubia
ready: true
restartCount: 0
state:
running:
startedAt: 2018-11-12T08:28:01Z
hostIP: 10.0.2.15
phase: Running
podIP: 172.17.0.5
qosClass: BestEffort
startTime: 2018-11-12T08:27:58Z
```

위의 샘플 정보를 보면 POD을 정의하는 구성에 대해서 알아보자.
- 먼저 두 줄에서 아래의 항목을에 대해서 정의한다.
1. Kubernetes API Version
2. 리소스의 타입
- Metadata는 Pod에 대한 name, namespace, labels 등의 정보를 정의한다. 이중 name은 꼭 있어야 하는 정보이다.
- Spec은 Pod의 Container, Volume 등의 Pod의 Contents들에 대한 실직적인 정보를 제공한다.
- Status는 Pod의 상태, 각 Container의 상태, Pod 내부의 IP와 다른 기본적인 정보를 제공한다. Status는 Read-only runtime data를 포함하고 있으며, 명령을 날리는 시점의 상태 정보이다. 따라서 POD을 생성할 때, Status에 대한 별도의 정의는 절대 작성할 필요가 없다.

@ 'Kubernetes in action', All Images Copy Rights Reserved.
