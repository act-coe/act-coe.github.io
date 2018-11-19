---
layout: post
title: "[K8S]POD에 대한 활용편 "
date: 2018-11-14 11:27:24 +0900
categories: [Kubernetes in action, Kubernetes, K8S]
tags: [Kubernetes, POD]
---
목표 :
1. POD을 생성, 실행, 삭제한다.
1. Label을 통한 POD과 다른 리소스와의 연결
1. Label을 이용해 관련 POD에 명령 내리기
1. POD을 분리하기 위해 namespace를 사용하기
1. 특정 Work Node에서 POD 배치하기

### POD을 만들어보자

```yaml
apiVersion: v1 #Kubernetes API Version
kind: Pod #Type of resources
metadata:
name: kubia-manual #Pod의 이름
spec:
containers:
- image: russell/kubia #컨테이너 이미지 정보
name: kubia #컨테이너 이름
ports:
- containerPort: 8080 #App의 Listening PORT정보
protocol: TCP
```

이전 내용에서 얻은 POD 보다는 적은 정보를 정의하고 있지만, POD에 대한 필수 적인 정보를 위와 같이 정의한다.
위 예제에서 POD에 정의된 Port정보는 오직 정보성으로 존재하며, 이를 생략한다고 해서 POD에 접속하는데 문제는 없다.
다만 이러한 정보를 적는 이유는 POD을 expose 할 경우에 다른 사람들이 쉽게 인지하기 위함이다.

이러한 Yaml파일에 대한 Field들에 대한 자세한 정보는 http://Kubernetes.io/docs/api 에서 확인을 할 수 있으며,
kubectl(Kubernetes Controller)를 통해서는 아래와 같은 명령으로 자세히 알 수 있다.

```sh
$ kubectl explain pod
KIND: Pod
VERSION: v1

DESCRIPTION:
Pod is a collection of containers that can run on a host. This resource is
created by clients and scheduled onto hosts.
.....

$ kubectl explain pod.metadata
KIND: Pod
VERSION: v1

RESOURCE: metadata <Object>

DESCRIPTION:
Standard object's metadata. More info:
https://git.k8s.io/community/contributors/devel/api-conventions.md#metadata

ObjectMeta is metadata that all persisted resources must have, which
includes all objects users must create.
...
```

#### Kubectl을 이용해서 POD을 만들기

```sh
$ kubectl create -f ./samples/k8s/chapter3/kubia-manual.yml
```

kubectl create -f 명령을 사용하면 Yaml이나 Json에 정의된 대로 어떠한 resource든 만들 수 있다.

Check.1. 만든 POD이 정상동장했는지에 대한 명령은 아래와 같으며, 정상동작시 Status는 Running, 문제가 발생 시는 마지막처럼 상태 값에 이유가 표기되는 것을 볼 수 있다.
```sh
$ kubectl get pods
NAME READY STATUS RESTARTS AGE
kubia-5ndt5 1/1 Running 0 2d
kubia-96gdb 1/1 Running 0 2d
kubia-h867s 1/1 Running 0 2d
kubia-manual 0/1 ImagePullBackOff 0 25m
```

Check.2. 설정한 Detail 한 정보를 알기 위해서는 아래의 명령하면 설정한 값들이 적용되었는지를 확인할 수 있다.

```sh
$ kubectl get pod kubia-manual -o yaml # 결과가 yaml 형식으로 출력된다
$ kubectl get pod kubia-manual -o json # 결과가 json 형식으로 출력된다.
```

#### 생성한 Application Log를 확인하자

컨테이너화 된 Application은 일반적으로 자신의 로그를 File로 쓰는 대신에 Standard output과 error에 남깁니다. 이러한 방법은 사용자가 간단하고 표준적인 방법으로 로그를 볼 수 있게 합니다. Container Runtime(우리의 경우엔 Docker겠죠)은 이러한 스트림을 파일로 남기고 사용자가 runtime log를 볼 수 있도록 해줍니다.

```sh
$ docker logs <container ID>
```

위와 같이 POD 내부에 접속해서 Docker Log를 확인하는 방법도 있겠지만 Kubernetes에서는 아래 명령을 이용해서 조금 더 쉽게 로그를 볼 수 있다.

```sh
$ kubectl logs kubia-manual
kubia server starting....
```

만약 하나의 POD에 여러 컨테이너가 있고, 이중에 특정 Container의 로그만을 보고 싶다면 아래 명령처럼 <b>-c</b> 옵션과 <b> Container Name </b> 를 준다
```#!/bin/sh
$ kubectl logs <Pod name> -c <container name>
```

> 로그 파일은 항상 File이 10MB가 되면, 자동으로 Rotate 된다. 따라서 kubectl logs는 마지막 rotation에 전체 로그만을 보여줄 수 있는 한계는 있다. 또한 POD이 삭제되면 LOG 또한 삭제되므로 중앙 집중된 Log 관리가 필요하다. 관련 내용은 이후에 다룰 예정이다.

#### POD에 요청 보내기
POD에 어떻게 요청을 보낼까 이전 설명에서, 우리는 kubectl expose 명령을 통해서 service를 만들어 외부에서 접근할 수 있는 방법을 사용했었다. 하지만 이후에도 계속해서 Service를 사용할 예정이기 때문에, 이번에는 port forwarding을 통해서 접근해보기로 하자. 이 방법은 Debugging과 Testing을 목적으로 POD에 접근할 때 주로 사용한다.

```sh
# kubectl port-forward <POD Name> <Local port>:<Pod Port>
$ kubectl port-forward kubia-manual 8888:8080
Forwarding from 127.0.0.1:8888 -> 8080
Forwarding from [::1]:8888 -> 8080
```

위 명령을 이용하게 되면, local의 8888 port를 kube-manual의 8080으로 포워드를 하게 된다.
정상 동작을 확인해 보기 위해서 Local Terminal에서 port-forwarder로 요청을 아래와 같이 해보자

```sh
$ curl localhost:8888
You've hit kubia-manual # Server 응답 값
```

```sh
kubectl port-forward kubia-manual 8888:8080
Forwarding from 127.0.0.1:8888 -> 8080
Forwarding from [::1]:8888 -> 8080
Handling connection for 8888 #추가됨: forward
```

![Port-Forwarding](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/assets/images/k8s/chapter3/figure3.4.png)

#### Label을 통한 POD의 조직하기

마이크로 서비스(MSA)를 하다 보면 수십 개 혹은 수백 개의 서비스가 생겨나게 되는데, 이러한 서비스가 늘어날수록 관리가 힘들어진다. 이럴 때 활용할 수 있는 것이 Label이다. Label은 Key-Value구조로 되어 있으며, Resource를 생성할 때, MetaData에 정의할 수 있다. 이렇게 붙은 Label은 Label-selector를 이용해서 특정 POD을 선택할 때 활용한다. 또한 정의된 Label은 POD혹은 다른 Resource를 재생성하지 않아도 언제든지 변경이나 추가가 가능하다.

아래 두 그림을 보면 앞선 이야기가 쉽게 이해될 것이다.

![Label Before](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/assets/images/k8s/chapter3/figure3.5.png)

![Label After](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/assets/images/k8s/chapter3/figure3.6.png)

먼저 그림에서는 분산되어 있는 MSA 환경을 표현한 것이다. 같은 POD 혹은 Container들이 여러 곳에 흩어져 있는 것을 볼 수 있다.
두 번째 그림에서는, 여기에 app={type정보}, rel={Version정보} 이 두 라벨을 이용해서 각 POD을 구분할 수 있다는 것을 보여준다.

#### Label 적용

> Label 생성 시 적용하기

먼저 만들었던 kubia-manual에서 MetaData하위에 labels라고 정의하고 (Key: value) 형태로 아래와 같이 Label을 추가한다.
```Yaml
apiVersion: v1 #Kubernetes API Version
kind: Pod #Type of resources
metadata:
name: kubia-manual-v2 #Pod의 이름
labels:
creation_method: manual
env: prod
spec:
containers:
- image: sdzeus/kubia #컨테이너 이미지 정보
name: kubia #컨테이너 이름
ports:
- containerPort: 8080 #App의 Listening PORT정보
protocol: TCP
```
해당 파일은 /samples/k8s/chapter3/kubia-manual-with-label.yml 이다.

> 생성

```sh
$ kubectl create -f kubia-manual-with-label.yml
```


> 확인

생성된 POD에 Label이 잘 적용되었는지 확인하기 위해서는 POD List를 가져오는 명령에서 --show-labels 옵션을 이용하면 된다.

```sh
$ kubectl get pods --show-labels
NAME READY STATUS RESTARTS AGE LABELS
kubia-manual 1/1 Running 0 1h <none>
kubia-manual-v2 1/1 Running 0 2m creation_method=manual,env=prod
```

> 특정 Label만 보기

Option 중 -L을 이용하면 아래와 같이 Column으로 볼 수 있다.

```sh
$ kubectl get pods -L creation_method,env
NAME READY STATUS RESTARTS AGE CREATION_METHOD ENV
kubia-manual 1/1 Running 0 1h
kubia-manual-v2 1/1 Running 0 23m manual prod
```

> Label 수정하기

POD에 Label을 추가하기 위해서는 kubectl label 명령을 이용한다.

```sh
$ kubectl label po kubia-manual creation_method=debug
pod/kubia-manual labeled
```

POD에 Label을 Update 하기 위해서는 --overwrite option을 이용한다.

```sh
$ kubectl label po kubia-manual-v2 creation_method=debug --overwrite
pod/kubia-manual-v2 labeled
```

> 확인

```sh
$ kubectl get pods -L creation_method,env
NAME READY STATUS RESTARTS AGE CREATION_METHOD ENV
kubia-manual 1/1 Running 0 1h debug
kubia-manual-v2 1/1 Running 0 27m debug prod
```

Label selector를 이용해서 Pod의 부분집합을 만들 때 유용하게 사용되니 꼭 알고 넘어가자

#### Label Selector을 이용해 해당하는 POD 목록 찾기

> creation_method가 manual인 모든 POD을 찾을 때

```sh
$kubectl get po -l creation_method=manual
NAME READY STATUS RESTARTS AGE
kubia-manual 1/1 Running 0 2h
kubia-manual-v2 1/1 Running 0 35m
```

> env label을 포함하고 있는 POD을 찾을 때

```sh
$kubectl get po -l env
NAME READY STATUS RESTARTS AGE
kubia-manual-v2 1/1 Running 0 37m
```

> env label이 없는 POD을 찾을 때

아래 명령을 사용할 때 single quotes를 유의하자. double quotes사용 시 bash shell에서는! 를 인지하지 못한다.
```sh
$kubectl get po -l '!env'
NAME READY STATUS RESTARTS AGE
kubia-manual 1/1 Running 0 2h
```

이밖에도 아래와 같은 표현을 사용할 수 있다.
```sh
$kubectl get po -l 'env in (prod, debug)'
$kubectl get po -l 'env notin (prod, debug)'
$kubectl get po -l creation_method!=manual
$kubectl get po -l creation_method=manual,env=debug # 다수의 조건을 줄 때는 Comma를 이용한다.
```
### POD Scheduling을 위해서 Label과 Selector를 이용하기

지금까지 생성한(Scheduling) POD들은 Working Node 중에 임의의 곳에 설치가 되었습니다. 이는 Kubernetes가 하나의, 대규모 배포 시스템을 제공하기 때문에 어떤 Node에 POD이 설치가 되었는지 크게 문제가 되지 않습니다. 각 POD은 정확하게 계산된 Resource(CPU, Memory 등등)를 가져올 수 있고, POD들 간의 접근성은 앞서 언급한 FLAT INNER NETWORK를 통해서 접근이 가능하기 때문입니다. 물론 일반적으로는 이렇습니다.

하지만 때로는 특정 Node에 POD이 생성이 필요할 때가 있습니다. 가장 좋은 예가, 하드웨어 인프라가 동일하지 않을 경우입니다. 어떤 NODE는 hardware를 돌리고, 다른 NODE가 SSD인 경우이거나 혹은 이번에 생성하고자 하는 POD이 GPU 가속이 필요할 경우를 생각해보면 쉽게 이해가 될 것이다.

위에서 말한 예의 상황이 벌어지더라도, 사용자는 특정 POD이 특정 NODE에 설치되어야 된다는 이야기는 하고 싶지 않을 것이다. 왜냐하면, 그것은 Application을 인프라와 결합시키게 되는 것이며, 이는 Kubernetes에서 실체 인프라를 Application에게서 숨기고 하자는 콘셉트와도 맞지 않을 것이기 때문이다. 그래서 위의 예제와 같은 경우에 특정 Node에 설치하라는 명령을 내리는 대신에 Label Selector와 Label을 활용해서 -인프라가 어떤 것들이 있는지는 모르겠지만- 조건에 맞는 Node에 설치하도록 명령을 내릴 수 있다.

#### Label을 사용해서 working node 분류하기

이미 알고 있다시피, Label은 어느 Kubernetes Object(POD, NODE 등등)든 붙일 수 있다. 보통 ops팀은 Cluster에 새로운 Node를 붙일 때 이미 Label을 정의해서 붙이겠지만 여기서는 Label을 추가하는 과정부터 시작해 보도록 하자

```sh
$kubectl get nodes # Node 정볼르 가져올 수 있다.
NAME STATUS ROLES AGE VERSION
minikube Ready master 4d v1.10.0

$kubectl label node <NODE Name> gpu=true # 특정 Node에 gpu=true라는 label을 정의한다.
node/<NODE Name> labeled

$kubectl get nodes -l gpu=true # 확인

```

#### 특정 Node에 POD을 설치하기
NODE에 Label을 지정을 마쳤으니, 이제 우리가 새로운 POD을 GPU가 있는 NODE에 배포하고 싶다고 가정해보자. 그렇다면 아래 예제와 같이 POD의 YAML을 작성할 때 Node Selector를 추가해서 주어진 조건이 만족하는 POD을 찾아볼 수 있다.

```yaml
# file path: assets/samples/k8s/chapter3/kubia-gpu.yml
apiVersion: v1 #Kubernetes API Version
kind: Pod #Type of resources
metadata:
name: kubia-gpu #Pod의 이름
spec:
nodeSelector:
gpr: "true"
containers:
- image: sdzeus/kubia #컨테이너 이미지 정보
name: kubia #컨테이너 이름
ports:
- containerPort: 8080 #App의 Listening PORT정보
protocol: TCP
```
위의 예제를 보면 spec 항목 하위에 nodeSelector 필드를 추가하였고, 찾고자 하는 Label값을 정의해줬다. 이렇게 Yaml값을 정의하면, Label을 가지고 있는 Node들 중에 POD을 생성할 수 있다. 이 외에 특정 Node에 배포할 수 있는 방법이 있다. 왜냐하면, 각 Node들은 따로 설정을 하지 않더라도 kubernetes.io/hostname이라는 Label을 가지고 있어서 이것을 이용하면 된다.

### PODS에 주석달기

Object들은 Label과 비슷한 annotations란 속성을 가지고 있다. Annotation 또한 Label과 같은 Key-Value로 되어 있다. annotation은 Kubernetes에서 자동으로 추가되거나, 혹은 사용자가 직접 추가할 수도 있다. 그러나 Annotation은 Label과 달리 Selector가 없으며, 주로 무언가를 설명하고자 할때 사용된다. 최고의 사용법은 POD이나 다른 API Object에 설명을 추가할 때 사용된다. 아직은 그리 중요한 요소로 판단되지 않아서 간단히 추가하는 방법만 설명하고 넘어가도록 하겠다.

> 확인하기

metadata에 annotations를 확인할 수 있을 것이다. 없는 경우도 있으니 없다고 크게 신경을 쓸 부분은 아니다.

```sh
$kubectl get po kubia-XXXX -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    mycompany.com/someannotation: foo bar
    ...
```

> 추가하기

```sh
$kubectl annotate pod kubia-manual mycompany.com/someannotation="foo bar"
pod/kubia-manual annotated
```

### Namespace 사용하기
우리는 이미 POD이나 다른 Object들을 Label을 통해서 그룹화 하는 것을 봤다. 그러나 Object들이 여러가지 Label을 가질 수 있기 때문에 Label을 통한 Group화는 여러 그룹에 POD들이 중복해서 들어갈 수 있다. 게다가 Label을 정확히 모르는데 특정 POD을 찾아야 할 경우에, 모든 POD을 봐야하는 단점도 있다.

만약에 우리가 중복되지 않은 Object Group을 가지려면 어떻게 해야할까? 바로 Object를 Namespace로 그룹화 하면 됩니다. 이 Namespace는 이전에 언급한 Linux Namespace는 아니며, Kubernetes에서 정의한 Namespace이다. Namespace는 모든 resource에 사용이 가능하다.

#### namespace 확인하기

아래 명령을 통해서 현재 cluster의 모든 namespace를 찾을 수 있다.   
```sh
$kubectl get namespace  # namespace는 줄여서 ns라고 표기 할 수도 있다.
NAME          STATUS   AGE
default       Active   4d
kube-public   Active   4d
kube-system   Active   4d
```

지금까지 우리는 default namespace에서만 실행을 했다. 아마 바꾸지 않았다면 그럴것이다. 물론, 다른 namespace의 POD들도 아래와 같이 --namespace(혹은 -n)을 통해서 확인이 가능하다.

```sh
$kubectl get pods --namespace kube-system  #kubectl get pdos -n kube-system
NAME                                    READY   STATUS    RESTARTS   AGE
etcd-minikube                           1/1     Running   0          4d
kube-addon-manager-minikube             1/1     Running   1          4d
kube-apiserver-minikube                 1/1     Running   0          4d
kube-controller-manager-minikube        1/1     Running   0          4d
kube-dns-86f4d74b45-rfn5w               3/3     Running   4          4d
kube-proxy-zjdq4                        1/1     Running   0          4d
kube-scheduler-minikube                 1/1     Running   0          4d
kubernetes-dashboard-5498ccf677-n9929   1/1     Running   3          4d
storage-provisioner                     1/1     Running   3          4d
```
위와 같이 리소스(POD, ReplicaSet등등)를 namespace를 이용하면 중복되는 경우 없이 관리할 수 있다. 또한 Namespace를 이용하면 특정 리소스에 사용자가 Access할 수 있는지 여부에 대한 권한을 설정할 수도 있다.

#### Namespace를 만들어보자
Namespace도 Kubernetes의 다른 Resource처럼 yaml파일을 작성하고 Kubernetes API Server에 요청을 보내어 만들 수 있다.

```YAML
# File path : ./assets/samples/k8s/chapter3/custom-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: custom-namespace
```

```sh
$kubectl create -f custom-namespace.yaml
namespace/custom-namespace created
```

Namespace는 YAML 파일을 이용하는 대신에 kubectl command를 통해서도 생성이 가능하며, 방법은 아래와 같다.

```sh
$kubectl create namespace custom-namespace
namespace/custom-namespace created
```

#### 다른 namespace에 Object 관리하기
> 다른 Namespace에 pod 생성하기

```sh
$kubectl create -f ./assets/samples/k8s/chapter3/kubia-manual.yml -n custom-namespace
pod/kubia-manual created
```

위와 같이 --namespace(-n) 옵션을 주고 POD을 생성하면, 지정한 Namespace에 POD을 생성하게되며, 지금 생성한 POD에 특정 명령(수정,삭제 등등)을 내리기 위해서는 --namespace(-n) 옵션을 적고 command를 내려야한다. 그렇지 않을 경우 현재 Context에 설정되어 있는 기본 namespace에서 작업을 수행한다.

현재 context에서 기본 Namespace의 변경은 kubectl config명령을 통해서 가능하다. 아래 alias를 이용하면 쉽게 변경이 가능하시 참고 하기 바란다.
```sh
#사용법 kcd some-namespace
$alias kcd='kubectl config set-context $(kubectl config current-context) --namespace '
```

#### namespace를 통한 isolation(분리)에 대한 이해

비록 namespace가 각기 다른 그룹으로 Object를 구분할수 있게 해주지만, 이것은 특정 Namespace의 운영-Command 같은-에 대해서만 적용될 뿐, 실제 동작하고 있는 Object의 기능에 대한 구분을 보장하는 것은 아니다. 즉,
