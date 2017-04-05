# hello-world的背后发生了什么？

在程序员的世界里，hello world是个很特殊的存在，当我们接触一门新的语言、新的开发库或者框架时，第一时间想了解的一般都是怎么实现一个hello world，然后思考hello world的背后发生了什么，在使用docker的时候，我们也是同样的思路，本篇将会介绍hello world背后的故事

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

hello-world这个容器运行的时候就是打印上面一段话到终端，然后退出。它背后发生了什么也在上面列出来了，下面我们将会详细的介绍这四步的内容。

## 流程图
这里是比上面四步更细化的流程图
```
                              +------------+
                              |            |
                              | Docker Hub |
                              |            |
                              +------------+
                                    ↑
                                    |
                                  2 | Restful
                                    |
                                    ↓
                               +---------+
+--------+       Restful       |         |    grpc      +-------------------+  4   +------------------------+  5   +-------------+
| docker |<------------------->| dockerd |<------------>| docker-containerd |<---->| docker-containerd-shim |<---->| docker-runc |
+--------+         1           |         |      3       +-------------------+      +------------------------+      +-------------+
                               +---------+
```

#### 1. docker通过发送restful请求给dockerd
当在shell里面运行```docker run hello-world```时，docker程序被启动，这个程序就是docker的客户端，它的任务就是解析命令行参数，然后构造相应的restful请求给dockerd，Engine API里描述了dockerd支持的所有请求，docker v17.03.0对应的API版本为[v1.26](https://docs.docker.com/engine/api/v1.26/)，而docker v17.03.1对应的版本为[v1.27](https://docs.docker.com/engine/api/v1.27/)。API版本之间的差别可以参考[version-history](https://docs.docker.com/engine/api/version-history/)

我们运行hello world就相当于下面的curl命令：
