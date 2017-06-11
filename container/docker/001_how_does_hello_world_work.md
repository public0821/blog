# 走进docker(01)：hello-world的背后发生了什么？

在程序员的世界里，hello world是个很特殊的存在，当我们接触一门新的语言、新的开发库或者框架时，第一时间想了解的一般都是怎么实现一个hello world，然后思考hello world的背后发生了什么，在学习docker的时候，也是同样的思路，本篇将会介绍hello world背后的故事

## 运行hello world

docker的安装不在本篇的介绍范围内，本文假设你已经安装好了17.03版本的docker。先来看看hello world运行的效果：

```bash
dev@dev:~$ docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

    ......
```

hello-world这个容器运行的时候就是打印上面一段话到终端，然后退出。它在输出结果中给我们描述了它运行的大概步骤，下面我们将对这个过程做进一步的分析。

## 关系图
这里是运行hello-world过程中，进程之间的关系
```
                              +------------+
                              |            |
                              | Docker Hub |
                              |            |
                              +------------+
                                    ↑
                                    |
                                  2 | REST
                                    |
                                    ↓
                               +---------+
+--------+       REST          |         |    grpc      +-------------------+
| docker |<------------------->| dockerd |<------------>| docker-containerd |
+--------+         1           |         |      3       +-------------------+
                               +---------+                       ↑
                                                                 |
                                                                 | 4
                                                                 ↓
                                                      +------------------------+  5   +-------------+
                                                      | docker-containerd-shim |<---->| docker-runc |
                                                      +------------------------+      +-------------+
                                                                                             ↑
                                                                                             | 6
                                                                                             ↓
                                                                                         +-------+
                                                                                         | hello |
                                                                                         +-------+

```

## 步骤过程
### 1. docker <--> dockerd
docker进程是docker客户端，dockerd进程是docker服务器端，它们的代码都在[moby](https://github.com/moby/moby)项目里面。

当在shell里面运行```docker run hello-world```后，docker程序被启动，这个程序就是docker的客户端，它的任务就是解析命令行参数，然后构造相应的启动容器请求，通过rest的方式发给dockerd。

Engine API里描述了dockerd支持的所有请求，docker v17.03.0对应的API版本为[v1.26](https://docs.docker.com/engine/api/v1.26/)，而docker v17.03.1对应的版本为[v1.27](https://docs.docker.com/engine/api/v1.27/)，API版本之间的差别可以参考[version-history](https://docs.docker.com/engine/api/version-history/)。

### 2. dockerd <--> "docker hub"
当dockerd收到客户端的运行容器请求后，发现本地没有相应的镜像（image），就会从[docker hub](https://hub.docker.com/)取相应image。（实际过程要比这个步骤多，这里为了简单直观，省略掉了其它的步骤，后面有详细的说明）

docker hub是docker官方存放镜像（image）的服务器，dockerd和它之间也是使用rest接口，协议为[Registry HTTP API V2](https://docs.docker.com/registry/spec/api/)。

取image的大概过程如下：

* 首先获取image的manifests，manifests里面包含两部分内容，一是image的配置文件的digest(sha256)，另一个是image包含的所有filesystem layer的digest(sha256)
* 根据上一步得到的image的配置文件的digest，在本地找是否已经存在对应的image，如果已经存在的话，就不用再往下走了，用现成的就可以了，如果没有，则继续
* 遍历manifests里面的所有layer，根据其digest在本地找，如果找到对应的layer，则跳过当前layer，否则从服务器取相应layer的压缩包
* 等上面的所有步骤完成后，就会拼出完整的image

> 这里的每个layer都是相对于上一个文件系统layer的变化情况。
> 从上面的过程可以看出，docker只会拉取本地没有的layer。

### 3. dockerd <--> docker-containerd
dockerd拿到image后，就会在本地创建相应的容器，然后再做一些初始化工作后，最后通过grpc的方式通知docker-containerd进程启动指定的容器

docker-containerd是和dockerd一起启动的后台进程，他们之间使用unix socket通信，协议是[grpc](http://www.grpc.io/).

### 4. docker-containerd <--> docker-containerd-shim
docker-containerd和docker-containerd-shim都属于[containerd](https://github.com/containerd/containerd)项目，当docker-containerd收到dockerd的启动容器请求之后，会做一些初始化工作，然后启动docker-containerd-shim进程，并将相关配置所在的目录作为参数传给它。

可以简单的理解成docker-containerd管理所有本机正在运行的容器，而docker-containerd-shim只负责管理一个运行的容器，相当于是对runc的一个包装，充当containerd和runc之间的桥梁，runc能干的就交给runc来做，runc做不了的就放到这里来做。

### 5. docker-containerd-shim <--> docker-runc
docker-containerd-shim进程启动后，就会按照runtime的标准准备好相关运行时环境，然后启动docker-runc进程。

>docker-runc就是runc程序的重命名，它们是相等的，若无特殊情况，后面介绍中不区分docker-runc和runc

[image](https://github.com/opencontainers/image-spec)和[runtime](https://github.com/opencontainers/runtime-spec)标准都由[Open Container Initiative](https://www.opencontainers.org)（OCI）负责定义维护，而[runc](https://github.com/opencontainers/runc)就是docker贡献给OCI的一个标准runtime实现。

如何启动runc是公开的标准，大概过程就是准备好rootfs和配置文件，然后使用合适的参数启动runc进程就可以了。

### 6. docker-runc <--> hello
runc进程打开容器的配置文件，找到rootfs的位置，并启动配置文件中指定的相应进程，在hello-world的这个例子中，runc会启动容器中的hello程序。

到此为止，容器启动成功。

## 进程间的关系
等runc将容器启动起来后，runc进程就退出了，于是容器里面的第一个进程（hello）的父进程就变成了docker-containerd-shim，在pstree的输出里面，进程树的关系大概如下：
```
systemd───dockerd───docker-containerd───docker-containerd-shim───hello
```

>实际操作过程中可能看不到这样的输出，因为hello很快就运行退出了，接着docker-containerd-shim也退出了。

其中dockerd和docker-containerd是后台常驻进程，而docker-containerd-shim则由docker-containerd按需启动。

>runc退出后其子进程hello不是应该由init进程接管吗？怎么就变成了docker-containerd-shim的子进程了呢？这是因为从Linux 3.4开始，[prctl](http://man7.org/linux/man-pages/man2/prctl.2.html)增加了对PR_SET_CHILD_SUBREAPER的支持，这样就可以控制孤儿进程可以被谁接管，而不是像以前一样只能由init进程接管。

## 输出
hello进程启动之后，往标准输出打印一段话后就退出了，那这个标准输出输出到哪里去了呢？docker客户端是怎么得到这段话的呢？这就取决于docker将这个标准输出重定向到哪里去了，以及它是怎么管理容器的这些输出的，这涉及到docker的日志管理方式，该部分内容会在后续做详细介绍，这里只需要知道容器的标准输出的内容能被docker的这些进程一层一层的转发给客户端就行了。

## 详细步骤
上面介绍的是一个精简版的hello world运行步骤，实际过程要多几个回合，下面用curl命令来模拟一下实际的操作流程：
```bash
#这里假设已经配置了dockerd监听本地tcp端口2375

#请求dockerd创建容器，但由于dockerd在本地找不到相应的image，于是返回失败
dev@debian:~$ curl 127.0.0.1:2375/v1.27/containers/create  -X POST -H "Content-Type: application/json" -d '{"Image": "hello-world"}'
{"message":"No such image: hello-world:latest"}

#请求dockerd去找registry服务器拿image
#这里的http应答中的内容包含了很多跟进度条相关的内容
dev@debian:~$ curl '127.0.0.1:2375/v1.27/images/create?fromImage=hello-world&tag=latest' -X POST
{"status":"Pulling from library/hello-world","id":"latest"}
{"status":"Pulling fs layer","progressDetail":{},"id":"78445dd45222"}
{"status":"Downloading","progressDetail":{"current":971,"total":971},"progress":"[==================================================\u003e]    971 B/971 B","id":"78445dd45222"}
{"status":"Verifying Checksum","progressDetail":{},"id":"78445dd45222"}
{"status":"Download complete","progressDetail":{},"id":"78445dd45222"}
{"status":"Extracting","progressDetail":{"current":971,"total":971},"progress":"[==================================================\u003e]    971 B/971 B","id":"78445dd45222"}
{"status":"Extracting","progressDetail":{"current":971,"total":971},"progress":"[==================================================\u003e]    971 B/971 B","id":"78445dd45222"}
{"status":"Pull complete","progressDetail":{},"id":"78445dd45222"}
{"status":"Digest: sha256:c5515758d4c5e1e838e9cd307f6c6a0d620b5e07e6f927b07d05f6d12a1ac8d7"}
{"status":"Status: Downloaded newer image for hello-world:latest"}

#再次创建容器成功，得到容器ID
dev@debian:~$ curl 127.0.0.1:2375/v1.27/containers/create  -X POST -H "Content-Type: application/json" -d '{"Image": "hello-world"}'
{"Id":"2a4717ffb830bf4cff12ef6e6f1e93129970df273387797fd023e10292e3e928","Warnings":null}

#attach到容器的标准输出，curl程序会暂停在这里，等待容器的输出
dev@debian:~$ curl '127.0.0.1:2375/v1.27/containers/2a4717ffb830bf4cff12ef6e6f1e93129970df273387797fd023e10292e3e928/attach?stderr=1&stdout=1&stream=1' -d '{"Connection": "Upgrade", "Upgrade":"tcp"}'
#等下一步容器启动后，在这里可以看到容器的输出

#另外打开一个shell窗口，启动容器
dev@debian:~$ curl 127.0.0.1:2375/v1.27/containers/2a4717ffb830bf4cff12ef6e6f1e93129970df273387797fd023e10292e3e928/start -X POST 
```

从上面的curl命令可以看出，在hello-world这个场景中，docker客户端主要发送了四个请求给dockerd，首先创建image，然后创建容器，接着attach标准输出，最后启动容器。（attach涉及到标准输入输出重定向，这里不细说）

这里是它们之间的交互流程：
```
                               +---------+
+--------+                     |         |
|        | 1.create container  |         |
|        |-------------------->|         |
|        |                     |         |
|        | 2.image not found   |         |
|        |<--------------------|         |
|        |                     |         |
|        | 3.create image      |         |
|        |-------------------->|         |
|        |                     |         |
|        | 4.image created     |         |
|        |<--------------------|         |
| docker |                     | dockerd |
|        | 5.create container  |         |
|        |-------------------->|         |
|        |                     |         |
|        | 6.container id      |         |
|        |<--------------------|         |
|        |                     |         |
|        | 7.start container   |         |
|        |-------------------->|         |
|        |                     |         |
|        | 8.container started |         |
|        |<--------------------|         |
+--------+                     |         |
                               +---------+
```

流程对应的文字描述如下：

* 客户端发送创建容器请求给dockerd，dockerd收到请求后，发现本地没有相应的额image，于是返回失败。
* 客户端收到失败的响应后，立即发送创建image的请求过来，dockerd收到后，会去docker hub上拿相应的image，拿到后返回成功。
* 客户端再次发送创建容器请求给dockerd，dockerd收到会根据拿到的image创建一个新容器，并初始化容器运行时要用到的相关目录和配置文件，里面就包含了rootfs，容器创建完成后，dockerd返回容器的ID给客户端。
* 客户端发启动容器请求给dockerd，dockerd收到请求后，会通知docker-containerd启动容器，启动成功后返回成功给客户端。

>dockerd去docker hub上取image发生在这里的3~4步之间，而docker-containerd───docker-containerd-shim───hello这些事发生在这里的7~8步之间。

## 结束语
本文大概的介绍了一下hello-world是如何工作的，以及涉及了docker的哪些进程，里面还有大量的细节没有涉及，留给后续文章做进一步介绍。

## 参考

* [DOCKER 1.11: THE FIRST RUNTIME BUILT ON CONTAINERD AND BASED ON OCI TECHNOLOGY](https://blog.docker.com/2016/04/docker-engine-1-11-runc/)