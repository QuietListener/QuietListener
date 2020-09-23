---
layout: post
title: 容器化扫盲(docker k8s)
date: 2020-9-9 14:32:00
categories:  java maven
---

1. 10分钟看懂docker和k8s
https://zhuanlan.zhihu.com/p/53260098


2. Docker 和 Kubernetes 从听过到略懂：给程序员的旋风教程
https://juejin.im/post/6844903650611953677


# 1. docker
## docker引擎
docker引擎包括: docker客户端，docker守护进程, containerd和runc。
 ![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/docker-engine-1.png) 

1. containerd用于容器生命周期管理, start stop pause rm
2. runc 实现了开放容器计划(OCI)用来创建容器
3. 实现api，镜像管理，省份认证，安全特性，核心网络等 

 ![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/docker-start-1.png) 

## docker镜像
1. docker镜像就像是类Class
2. docker需要先从镜像仓库拉(docker hub)去镜像，然后启动一个多者多个容器。
3. 镜像由多个层组成，每层叠加后，从外部看是一个独立的对象。镜像内是一个精简的操作系统，同事包括了运行需要的文件和依赖包。
4. 镜像可以看成是一个构建时结构，容器可以看成是一个运行时结构
5. 在容器启动后，如果不停止或者销毁容器，删除对应的对象会报错。


6. 镜像很小，需要裁减掉不必要的部分，例如可以连shell都不要
7. 镜像不包含内核，内核是所有docker共享的。例如Ubuntu的镜像只有110M.

拉取镜像到本地

```shell
➜  ~ docker image pull ubuntu:latest
latest: Pulling from library/ubuntu
e6ca3592b144: Pull complete 
534a5505201d: Pull complete 
990916bd23bb: Pull complete 
Digest: sha256:cbcf86d7781dbb3a6aa2bcea25403f6b0b443e20b9959165cf52d2cc9608e4b9
Status: Downloaded newer image for ubuntu:latest
docker.io/library/ubuntu:latest
```

查看当前有哪些镜像
```shell
➜  ~ docker image ls                
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
ubuntu                      latest              bb0eaf4eee00        6 days ago          72.9MB
ubuntu                      <none>              ccc6e87d482b        8 months ago        64.2MB
swaggerapi/swagger-editor   latest              ee350c7d4db2        10 months ago       41.9MB
```

Docker Hub是官方的镜像仓库，这里的镜像质量是很高和安全的。


8. 镜像命名和标签

从官方仓库拉去镜像:
docker image pull <repository>:<tag>
从非官方仓库拉去镜像:
docker image pull <dockerhub 用户名或者组织名>/<repository>:<tag>
例如 docker image pull microsoft/powershell:nanoserver

9. 使用cli方式搜索docker hub

```shell
 docker search redis     
NAME                             DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
redis                            Redis is an open source key-value store that…   8591                [OK]                
bitnami/redis                    Bitnami Redis Docker Image                      161                                     [OK]
sameersbn/redis                                                                  81                                      [OK]

```


10. 镜像和分层
镜像是由一些只读镜像层组成
![部署](https://raw.githubusercontent.com/QuietListener/quietlistener.github.io/master/images/docker-image-layers.png) 

例如 下面每一个complete都代表一个镜像层。 一共三个镜像层。
```shell
➜  ~ docker image pull ubuntu:latest
latest: Pulling from library/ubuntu
e6ca3592b144: Pull complete 
534a5505201d: Pull complete 
990916bd23bb: Pull complete 
Digest: sha256:cbcf86d7781dbb3a6aa2bcea25403f6b0b443e20b9959165cf52d2cc9608e4b9
Status: Downloaded newer image for ubuntu:latest
docker.io/library/ubuntu:latest
```

使用 **docker image inspect ubuntu:latest** 可以查看镜像层

镜像层举例：
基于ubuntu 16.04创建了一个镜像层，这就是第一层，如果在该镜像添加python包，这就是第二层；如果继续添加安全补丁，这既是低三层。如果需要修改第二层的一些配置文件，可以加上第四层。

11. 共享镜像层
多个镜像之间可以共享镜像层。
docker可以识别要拉取镜像中，哪些层已经在本地了，如果存在就不用拉取了。

12. 根据摘要来去镜像
再要就就是镜像内容的一个hash值

13. 多层架构镜像
一个镜像标签可以支持多个架构的镜像:比如arm，windows，x64。镜像仓库支持**Manifest列表**和**Manifest**.
Manifest列表中包含了支持的所有架构。如果docker 跑在arm架构上，那拉取的时候去Manifest列表中找到arm对应的Manifest。然后拉取

14. 删除镜像
docker image rm 
```shell
 docker image ls                   
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
ubuntu                      latest              bb0eaf4eee00        6 days ago          72.9MB
ubuntu                      <none>              ccc6e87d482b        8 months ago        64.2MB
swaggerapi/swagger-editor   latest              ee350c7d4db2        10 months ago       41.9MB
➜  ~ docker image rm ccc6e87d482b
Error response from daemon: conflict: unable to delete ccc6e87d482b (must be forced) - image is being used by stopped container d01e42d5d77d
➜  ~ docker image rm ee350c7d4db2            
Untagged: swaggerapi/swagger-editor:latest
Untagged: swaggerapi/swagger-editor@sha256:b582504a6007c7026811c978b73b9f97df501fd35e0e38dba37a48d0fd2b8486
Deleted: sha256:ee350c7d4db2fe22b7ff04bf75b80d5e11edbd14e797fddd3f0141d99443bd3b
Deleted: sha256:ae9dbf7b44330a115cba9640e30c26b132160de7ec4594445c78f8b5b07565bf
Deleted: sha256:c6bfdd5bfba8b0085856caae86f1099d2602305b262c6fe29af7328c055bb1ea
Deleted: sha256:df3b1523fe1af01173f3f9f3575fe49f60ae699c96a7f58dc572ca61ef7393da
Deleted: sha256:e71f768138aec5153f12d460fb95f0a4cd305ff908092a0694cc6113bb578caa
Deleted: sha256:9b7ff37239933b76c80e5a4b08aea0721ec9328126c7091b58a020c4cf5afa48
Deleted: sha256:ae951b0c75e3db102b9907d4858bd04a133a7bf277f54e70b0307a7dc0584deb
Deleted: sha256:9ecd7491292f7bb06bd8f11ac3caf90a03061d602b7290dc6b4688691e67a0f4
Deleted: sha256:29cb1a997992dd43d829ecd13600fdbb34ba744bc21da5eb2c95251e15256c98
Deleted: sha256:eed045915edcdb3084891b25476e19991af54cc0d4ddfe97a301c6ea4fad812c
Deleted: sha256:d978755e04392d047b21ef378936b6cddda7b34fc05bf442a74d4a2f1df4b2f5
Deleted: sha256:77cae8ab23bf486355d1b3191259705374f4a11d483b24964d2f729dd8c076a0
➜  ~ docker image ls             
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              latest              bb0eaf4eee00        6 days ago          72.9MB
ubuntu              <none>              ccc6e87d482b        8 months ago        64.2MB
➜  ~ 
```


