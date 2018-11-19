---
layout: post
title: "Docker를 이해하기"
date: 2018-11-05 11:27:24 +0900
categories: [Kubernetes in action, Kubernetes, K8S]
tags: [Kubernetes, Docker, dockerfile]
---
## 1. Introducing Kubernetes

1장에 대해서는 별도의 정리를 하지 않는다.
1장의 내용을 요약하자면, MSA라는 시대적 흐름과 Docker라는 Container화 기술의 만남으로 Kubernetes가 떠오르게 되었다. 그러므로 Kubernetes를 적용하기 위한 환경은 당연히 MSA여야 하고, Docker에 대해서 알아야 한다.

## 2. Docker

### 2.1 Docker Run

```sh
docker run busybox echo "Hello World"
```

> Docker Run이 실행되는 순서

![Docker Run](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/assets/images/k8s/chapter2/figure2.1.png)

### 2.1.2 Docker Build

> app.js

```js
const http = require('http')
const os = require('os')

console.log("kubia server starting....")

var handler = function(request, response){
console.log("Recieved request from " + request.connection.remoteAddress);
response.writeHead(200);
response.end("You've hit " + os.hostname()+ "\n");
};

var www = http.createServer(handler);
www.listen(8080);
```

> Dockerfile

```
From node:7
ADD app.js /app.js
ENTRYPOINT ['node', 'app.js']
```
- From: Base Image를 지정한다
- ADD : Container에 포함시킬 파일을 지정한다.
- ENTRYPOINT: 이미지를 실행시키는 시점에 수행할 명령어를 지정한다.

![Docker Build](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/assets/images/k8s/chapter2/figure2.2.png)
```sh
docker build -t kubia . !!! 뒤에 붙는 .을 꼭 적는다.
```
- 빌드 프로세스는 Docker Client에 의해서 실행되지 않는다. 전체 소스는 Docker Daemon에 올라가게 되고, 거기서 이미지 빌드가 이뤄진다.
- 따라서, Docker Client와 Docker Daemon이 한 개의 OS에 있을 필요는 없다.

> Docker Image Layer (FS-생략한다)

![Docker Image Layer](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/assets/images/k8s/chapter2/figure2.3.png)


### 2.1.3 Running Container image

docker 실행 명령은 아래와 같다.

```sh
$ docker run --name kubia-Container -p 8080:8080 -d kubia
```
option description
---- ----
-name Container의 명칭
-d detached from the console
-p local machine의 8080 Port를 Container의 8080 포트에 mapping 시킬 수 있다. localhost:8080으로 추후에 확인이 가능하다

- docker 실행 확인하기 위해서는 아래의 명령을 확인한다.

```sh
$ docker ps
```

- docker Container 정보를 상세 확인하기 위해서는 다음과 같다.

```sh
$ docker inspect kubia-Container
```

- 실행되고 있는 docker Container를 탐험하기

```sh
// -i : STDIN을 열어 놓는다는 의미이다.
// -t : pseudo terminal(TTY)를 허락한다는 의미이다.
$ docker exec -it kubia-Container bash
```
- Conainer를 멈추고 삭제하기

```sh
$ docker stop kubia-Container // 멈추기
$ docker rm kubia-Container // 삭제하기
```

### 2.1.4 Image Registry에 push 하기

1. build 된 이미지에 Tag달기

이미지에 kubia라는 tag를 다는 예제이다.
```sh
$ docker tag kubia (docker에 사용자 이름)/kubia
```
위의 명령어를 실행하게 되면 아래와 같이 이미지 태그가 설정이 된다.

![Docker Tag](https://raw.githubusercontent.com/act-coe/act-coe.github.io/master/assets/images/k8s/chapter2/figure2.4.png)

1. push

만약 아래 명령어가 정상 동작하지 않는다면, 원하는 registry에 docker login을 통해서 먼저 login을 해야 한다.

```sh
$ docker push russell/kubia
```

@ 'Kubernetes in action', All Images Copy Rights Reserved.
