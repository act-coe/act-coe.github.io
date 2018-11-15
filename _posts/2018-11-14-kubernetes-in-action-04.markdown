---
layout: post
title:  "[K8S]POD에 대한 활용편 "
date:   2018-11-14 11:27:24 +0900
tags: [Kubernetes in action]
comments: true
---
목표 :
1. POD을 생성, 실행, 삭제 한다.
1. Label을 통한 POD과 다른 리소스와의 연결
1. Label을 이용해 관련 POD에 명령 내리기
1. POD을 분리하기 위해 namespace를 사용하기
1. 특정 Work Node에서 POD 배치하기

### POD을 만들어보자

```yaml
apiVersion: v1                #Kubernetes API Version
kind: Pod                     #Type of resources
metadata:
  name: kubia-manual          #Pod의 이름
spec:
  containers:
  - image: russell/kubia      #컨테이너 이미지 정보
    name: kubia               #컨테이너 이름
    ports:
    - containerPort: 8080     #App의 Listening PORT정보
      protocol: TCP
```

이전 내용에서 얻은 POD 보다는 적은 정보를 정의하고 있지만, POD에 대한 필수 적인 정보를 위와 같이 정의한다.
위 예제에서 POD에 정의된 Port정보는 오직 정보성으로 존재하며, 이를 생략한다고 해서 POD에 접속하는데 문제는 없다.
다만 이러한 정보를 적는 이유는 POD을 expose 할 경우에 다른사람들이 쉽게 인지하기 위함이다.

이러한 Yaml파일에 대한 Field들에 대한 자세한 정보는 http://Kubernetes.io/docs/api 에서 확인을 할 수 있으며,
kubectl(Kubernetes Controller)를 통해서는 아래와 같은 명령으로 자세히 알수 있다.

```sh
$ kubectl explain pod
KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.
     .....

$ kubectl explain pod.metadata
KIND:     Pod
VERSION:  v1

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
$ kubectl create -f ./samples/chapter3/kubia-manual.yml
```

kubectl create -f 명령을 사용하면 Yaml이나 Json에 정의된 대로 어떠한 resource든 만들 수 있다.

Check.1. 만든 POD이 정상동장했는지에 대한 명령은 아래와 같으며, 정상동작시 Status는 Running, 문제가 발생시는 마지막처럼 상태값에 이유가 표기되는 것을 볼 수 있다.
```sh
$ kubectl get pods
NAME           READY   STATUS             RESTARTS   AGE
kubia-5ndt5    1/1     Running            0          2d
kubia-96gdb    1/1     Running            0          2d
kubia-h867s    1/1     Running            0          2d
kubia-manual   0/1     ImagePullBackOff   0          25m
```

Check.2. 설정한 Detail한 정보를 알기 위해서는 아래의 명령하면 설정한 값들이 적용되었는지를 확인할 수 있다.

```sh
$ kubectl get pod kubia-manual -o yaml  # 결과가 yaml 형식으로 출력된다
$ kubectl get pod kubia-manual -o json  # 결과가 json 형식으로 출력된다.
```

#### 생성한 Application Log를 확인하자

컨테이너화 된 Application은 일반적으로 자신의 로그를 File로 쓰는 대신에 Standard output과 error에 남깁니다. 이러한 방법은 사용자가 간단하고 표준적인 방법으로 로그를 볼 수 있게 합니다. Container Runtime(우리의 경우엔 Docker겠죠)은 이러한 스트림을 파일로 남기고 사용자가 runtime log를 볼 수 있도록 해줍니다.

```sh
$ docker logs <container ID>
```

위와 같이 POD내부에 접속해서 Docker Log를 확인하는 방법도 있겠지만 Kubernetes에서는 아래 명령을 이용해서 조금 더 쉽게 로그를 볼 수있다.

```sh
$ kubectl logs kubia-manual
kubia server starting....
```

만약 하나의 POD에 여러 컨테이너가 있고, 이중에 특정 Container의 로그만을 보고 싶다면 아래 명령처럼 <b>-c</b> 옵션과 <b>  Container Name </b> 를 준다
```#!/bin/sh
$ kubectl logs <Pod name> -c <container name>
```

> 로그 파일은 항상 File이 10MB가 되면, 자동으로 Rotate된다. 따라서 kubectl logs는 마지막 rotation에 전체 로그만을 보여줄 수 있는 한계는 있다. 또한 POD이 삭제되면 LOG 또한 삭제 되므로 중앙집중된 Log 관리가 필요하다. 관련 내용은 이후에 다룰 예정이다.

#### POD에 요청보내기
POD에 어떻게 요청을 보낼까 이전 설명에서, 우리는 kubectl expose 명령을 통해서 service를 만들어 외부에서 접근할 수 있는 방법을 사용했었다. 하지만 이후에도 계속해서 Service를 사용할 예정이기 때문에, 이번에는 port forwarding을 통해서 접근해보기로 하자. 이 방법은 Debugging과 Testing을 목적으로  POD에 접근할 때 주로 사용한다.

```sh
# kubectl port-forward <POD Name> <Local port>:<Pod Port>
$ kubectl port-forward kubia-manual 8888:8080
Forwarding from 127.0.0.1:8888 -> 8080
Forwarding from [::1]:8888 -> 8080
```

위 명령을 이용하게 되면, local의 8888 port를 kube-manual의 8080으로 포워드를 하게 된다.
정상동작을 확인해 보기위해서 Local Terminal에서 port-forwarder로 요청을 아래와 같이 해보자

```sh
$ curl localhost:8888
You've hit kubia-manual   # Server 응답 값
```

```sh
kubectl port-forward kubia-manual 8888:8080
Forwarding from 127.0.0.1:8888 -> 8080
Forwarding from [::1]:8888 -> 8080
Handling connection for 8888    #추가됨: forward
```

![Port-Forwarding](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/images/chapter3/figure3.4.png)

#### Label을 통한 POD의 조직하기

마이크로서비스(MSA)를 하다보면 수십개 혹은 수백개의 서비스가 생겨나게 되는데, 이러한 서비스가 늘어날 수록 관리가 힘들어진다. 이럴때 활용할 수 있는 것이 Label이다. Label은 Key-Value구조로 되어 있으며, Resource를 생성할때, MetaData에 정의할 수 있다. 이렇게 붙은 Label은 Label-selector를 이용해서 특정 POD을 선택할때 활용한다. 또한 정의된 Label은 POD혹은 다른 Resource를 재생성하지 않아도 언제든지 변경이나 추가가 가능하다.

아래 두 그림을 보면 앞선 이야기가 쉽게 이해 될 것이다.

![Label Before](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/images/chapter3/figure3.5.png)

![Label After](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/images/chapter3/figure3.6.png)

먼저 그림에서는 분산되어 있는 MSA 환경을 표현한 것이다. 같은 POD 혹은 Container들이 여러곳에 흩어져 있는 것을 볼수 있다.
두번째 그림에서는, 여기에 app={type정보}, rel={Version정보} 이 두 라벨을 이용해서 각 POD을 구분할 수 있다는 것을 보여준다.

#### Label 적용

> Label 생성시 적용하기

먼저 만들었던 kubia-manual에서 MetaData하위에 labels라고 정의하고 (Key: value) 형태로 아래와 같이 Label을 추가한다.
```Yaml
apiVersion: v1                #Kubernetes API Version
kind: Pod                     #Type of resources
metadata:
  name: kubia-manual-v2         #Pod의 이름
  labels:
    creation_method: manual
    env: prod
spec:
  containers:
  - image: sdzeus/kubia      #컨테이너 이미지 정보
    name: kubia               #컨테이너 이름
    ports:
    - containerPort: 8080     #App의 Listening PORT정보
      protocol: TCP
```
해당 파일은 /samples/chapter3/kubia-manual-with-lable.yml 이다.

> 생성

```sh
$  kubectl create -f kubia-manual-with-label.yml
```


> 확인

생성된 POD에 Label이 잘 적용되었는지 확인하기 위해서는 POD List를 가져오는 명령에서 --show-lables 옵션을 이용하면 된다.

```sh
$ kubectl get pods --show-labels
NAME              READY   STATUS    RESTARTS   AGE   LABELS
kubia-manual      1/1     Running   0          1h    <none>
kubia-manual-v2   1/1     Running   0          2m    creation_method=manual,env=prod
```

> 특정 Label만 보기

Option중 -L을 이용하면 아래와 같이 Column으로 볼 수 있다.

```sh
$ kubectl get pods -L creation_method,env
NAME              READY   STATUS    RESTARTS   AGE   CREATION_METHOD   ENV
kubia-manual      1/1     Running   0          1h
kubia-manual-v2   1/1     Running   0          23m   manual            prod
```

> Label 수정하기

POD에 Label을 추가하기 위해서는 kubectl label 명령을 이용한다.

```sh
$ kubectl label po kubia-manual creation_method=debug
pod/kubia-manual labeled
```

POD에 Label을 Update하기 위해서는 --overwrite option을 이용한다.

```sh
$ kubectl label po kubia-manual-v2 creation_method=debug --overwrite
pod/kubia-manual-v2 labeled
```

>확인

```sh
$ kubectl get pods -L creation_method,env
NAME              READY   STATUS    RESTARTS   AGE   CREATION_METHOD   ENV
kubia-manual      1/1     Running   0          1h    debug
kubia-manual-v2   1/1     Running   0          27m   debug             prod
```

Label selector를 이용해서 Pod의 부분집합을 만들때 유용하게 사용되니 꼭 알고 넘어가자

#### Label Selector을 이용해 해당하는 POD 목록 찾기

> creation_method가 manual인 모든 POD을 찾을때

```sh
$kubectl get po -l creation_method=manual
NAME              READY   STATUS    RESTARTS   AGE
kubia-manual      1/1     Running   0          2h
kubia-manual-v2   1/1     Running   0          35m
```

> env label을 포함하고 있는 POD을 찾을때

```sh
$kubectl get po -l env
NAME              READY   STATUS    RESTARTS   AGE
kubia-manual-v2   1/1     Running   0          37m
```

> env label이 없는 POD을 찾을 때

아래 명령을 사용할때 single quotes 를 유의하자. double quotes사용시 bash shell에서는 ! 를 인지하지 못한다.
```sh
$kubectl get po -l '!env'
NAME           READY   STATUS    RESTARTS   AGE
kubia-manual   1/1     Running   0          2h
```

이밖에도 아래와 같은 표현을 사용할 수 있다.
```sh
$kubectl get po -l 'env in (prod, debug)'
$kubectl get po -l 'env notin (prod, debug)'
$kubectl get po -l creation_method!=manual
$kubectl get po -l creation_method=manual,env=debug # 다수의 조건을 줄떄는 Comma를 이용한다.
```
### POD Scheduling을 위해서 Label 과 Selector를 이용하기
