---
layout: post
title: "Kubernetes Cluster"
date: 2018-11-12 11:27:24 +0900
categories: [Kubernetes in action, Kubernetes, K8S]
tags: [Kubernetes, Cluster]
--- 
### 2.2 Kubernetes cluster

Kubernetes Cluster를 세팅하는 방법은 3가지 정도가 있다.

1. Minikube를 이용해서 Local Development PC(1개)에 Setup 하는 것
2. GCE, AWS 등의 Cloud환경에서 사용하는 것 ([kcop](https://github.com/kubernetes/kops)을 활용할 수 있다.)
3. Kubeadm을 이용해서 3개의 노드에 설치하는 방법

#### 2.2.1 [Minikube](https://github.com/kubernetes/minikube)를 이용하여 Local Single-node kubernetes를 실행한다.

- 설치방법

MACOS
```sh
$ brew cask install minikube
```

Linux
```sh
$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

window의 경우 - Window 10 이상 지원한다고 한다. 또한 choco실행 시, 꼭 관리자 모드로 실행할 것
```sh
$ choco install minikube
$ choco install kubernetes-cli
```
- 실행방법
```sh
$ minikube start
```
결과
아래와 같은 결과를 볼 수 있는데 VM을 실행하기 때문에 미리 Virtual Box를 설치해 놓는 것이 좋다.
```sh
$ minikube start
Starting local Kubernetes v1.10.0 cluster...
Starting VM...
Downloading Minikube ISO
170.78 MB / 170.78 MB [============================================] 100.00% 0s
Getting VM IP address...
Moving files into cluster...
Downloading kubeadm v1.10.0
Downloading kubelet v1.10.0
Finished Downloading kubelet v1.10.0
Finished Downloading kubeadm v1.10.0
Setting up certs...
Connecting to cluster...
Setting up kubeconfig...
Starting cluster components...
Kubectl is now configured to use the cluster.
Loading cached images from config file.
```
- Kubernetes client(KUBECTL) [설치하기](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl)

K8S를 작동하기 위해서는 Kubernetes client를 설치해야 한다. 여기서는 curl을 통해서 설치하는 방법이지만, https://kubernetes.io에서는 tool을 이용하는 방법을 소개하고 있기에
설명은 링크로 대체한다. 위에 '설치하기'를 클릭하셔서 확인하시기 바랍니다.

- 동작 확인

```sh
$ kubectl cluster-info
Kubernetes master is running at https://192.168.99.100:8443
CoreDNS is running at https://192.168.99.100:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

#### 2.2.2 Google Kubernetes Engine에서 실행하기
[Google Quickstart](https://cloud.google.com/kubernetes-engine/docs/quickstart)를 이용하는 방법이 더 나을 것 같아 자세한 설명은 링크된 사이트를 참고한다.

#### 2.2.3 시작하기 전에 설정

- 책에서는 시작하기 전에 Alias설정과 Command-line Complete를 설명하고 있는데, 개인적으로는 이 설정보다 Zshell의 plugin을 설정하는 것이 더 나아 보여 해당 내용을 작성한다.

- 홈에 .zshrc파일을 열면 plugin항목이 있는데 그곳에 kubectl을 추가해주면 기본적인 Alias는 볼 수 있다.
```sh
# Which plugins would you like to load? (plugins can be found in ~/.oh-my-zsh/plugins/*)
# Custom plugins may be added to ~/.oh-my-zsh/custom/plugins/
# Example format: plugins=(rails git textmate ruby lighthouse)
# Add wisely, as too many plugins slow down shell startup.
plugins=(git kubectl)
```

### 2.3 Kubernetes에 내 앱을 올리기

```
$ kubectl run kubia --image=russell/kubia --port=8080
```

- "--image": 사용하고자 하는 컨테이너 이미지
- "--port": app에서 열어놓는 listening port
- ps: kubectl run --generator=run/v1 is DEPRECATED and will be removed in a future version. Use kubectl create instead. ReplicationController/kubia created

#### 2.3.1 PODS
! [POD](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/assets/images/k8s/chapter2/figure2.5.png)
- 위의 그림과 같이 각각의 POD은 개별 IP, Hostname, Process를 가지고, 1개 이상의 Container를 구동시킨다.
- POD List를 확인하기
```sh
$ kubectl get PODs
```

- POD에 대한 더 자세한 정보를 보기 위해서
```sh
$ kubectl describe PODs
```

- 지금까지 한 내용을 그림으로 보자면 아래와 같다

![Kubectl](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/assets/images/k8s/chapter2/figure2.6.png)

#### 2.3.2 Application에 접근하기

- POD이 가진 개별 IP는 클러스터 내부 IP이기 때문에 외부에서 접근할 수 없다. 그래서 Service Object를 만들어 외부에서 접근하도록 한다. 자! 이제 Service를 만들어보자 근데, 그냥 만들면 Type이 Cluster IP인 Service가 생성된다. 이건 POD처럼 클러스터 안쪽에서만 접근할 수 있기 때문에 Type을 꼭 loadBalancer로 설정해야 한다. 실행 명령은 아래와 같다.
```sh
$ kubectl expose rc kubia --type=LoadBalancer --name kubia-http
# 만약 위에서 deployment를 이용할 경우 아래 명령을 이용한다.
# 내용은 참조하는 정보를 ReplicationController에서 가져오는가? 혹은 Deployment에서 가져오는 가의 차이다.
$ kubectl expose deployment kubia --type=LoadBalancer --name kubia-http
```

- Listing Service
POD과 마찬가지로 정상적으로 동작하는지 보기 위해서 kubernetes get services(svc) 명령을 하게 되면 아래와 같이 서비스 리스트들을 확인할 수 있다.
```sh
$ kubernetes get svc
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
kubernetes ClusterIP 10.96.0.1 <none> 443/TCP 1h
kubia-http LoadBalancer 10.105.20.128 <pending> 8080:32619/TCP 28s
```
위에서 보면 Extenal_ip가 pending 상태인 것을 볼 수 있는데, 이는 시간이 지나면 loadBalancer에 IP가 할당된다... 시간이 필요하다.. 음.. 그런데 Minikube에서는 loadBalancer를 지원하지 않는다. 그래서 Minikube에서는 절대 EXTERNAL_IP가 할당되는 모습을 볼 수 없다.
만약 external ip가 할당되었다면 해당 exeternal ip와 32617 포트를 이용해서 호출해보면, 정상적인 응답을 받을 수 있을 것이다.
(minikube에서는 dashboard ip와 32619 포트를 이용하면 확인은 가능하다.)
- 왜 서비스가 필요할까? 만약에 누군가가 POD를 지운다고 생각해보자 그렇다면, ReplicationController(현재는 ReplicaSet)이 새롭게 POD을 생성할 것이다. 그렇다면 해당 POD은 새로운 IP가 발급이 된다. IP주소가 계속 변경될 것이다. 또한 같은 POD이 여러 개 있을 때를 가정해보자. 어떻게 개별 POD으로 가는 것을 결정할까? 그래서 계속 변경되는 POD의 IP 주소, 여러 POD 주소들에 대한 묶음으로 이 문제를 해결하기 서비스가 필요한 것이다. Service는 생성시점에 고정된 IP를 가지고 있으며, 해당 서비스를 삭제하기 전까지 IP는 절대 변경되지 않는다. 또한 POD들에 정보를 가지고 있기 때문에 Client가 Service의 Ip를 통해서 요청하게 되면 POD들로 요청을 전달하게 된다.

#### 2.3.4 수평적 확장
- kubernetes에서는 쉽게 수평적 확장이 가능하다. 이유는 이미 기존에 설정한 ReplicationController(현재는 Deployment와 ReplicaSet을 권장한다.)을 이용해서 POD의 삭제, 교체, 개수 관리를 하기 때문이다.
```sh
$ kubectl get replicationcontrollers
NAME DESIRED CURRENT READY AGE
kubia 1 1 1 26m
```
- 위의 커맨드를 실행하게 되면 DESIRED, CURRENT, READY 등이 나오는데, DESIRED는 우리가 요청하는 POD의 수, Current는 현재 구동 중인 POD의 수, Ready는 현재 완전히 구동이 완료된 것으로 구분된다.
- 여기서 다시 한번 ReplicationController의 수를 변경하는 명령을 요청해보자.
```sh
$ kubectl scale rc kubia --replicas=3
replicationcontroller/kubia scaled
$ kubectl get rc
NAME DESIRED CURRENT READY AGE
kubia 3 3 2 1h
```
- 위에서 설명한 항목(DESIRED, CURRENT, READY)등의 값이 변경된 것을 확인할 수 있다.
```sh
$ kubernetes get PODs -o wide
NAME READY STATUS RESTARTS AGE IP NODE
kubia-5ndt5 1/1 Running 0 1h 172.17.0.4 minikube
kubia-96gdb 1/1 Running 0 2m 172.17.0.5 minikube
kubia-h867s 1/1 Running 0 2m 172.17.0.6 minikube
```
- POD의 개수도 정상적으로 늘어난 것을 확인할 수 있다.

![Kubectl](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/assets/images/k8s/chapter2/figure2.8.png)

#### 2.3.5 Kubernetes Dashboard

- Kubernetes에서는 위에서 만든 PODs, Services, ReplicationController 등을 확인할 수 있는 화면을 제공한다. 이를 Kubernetes Dashboard라고 한다.
- minikube에서는 아래의 명령으로 자동으로 브라우저를 통해 볼 수 있다.
```sh
$ minikube dashboard
```
- Google Kubernetes Engine에서는 아래 명령어를 통해서 접속 URL을 확인할 수 있으며,
```sh
$ kubectl cluster-info grep dashboard
```
접속 시 ID/PW를 요청할 때는 아래의 명령으로 확인할 수 있다.
```sh
$ gcloud Container clusters describe kubiagrep -E "(usernamepassword):"
```
- Local 혹은 별도 환경을 구성한 경우에는 별도 설치가 필요하다. 자세한 사항은 [링크](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)를 참조한다.
```sh
$ kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

![kubernetes Dashboard](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/assets/images/k8s/chapter2/figure2.9.png)

@ 'Kubernetes in action', All Images Copy Rights Reserved.
