# 走进docker系列：开篇

本人docker初学者，边学习边总结，一方面加深自己的理解，另一方面希望对其他想深入了解docker的同学有所帮助。

由于本人缺乏实战经验，错误在所难免，欢迎批评指正，谢谢。

## 包含的内容
本系列主要介绍三个github上的项目： **[moby](https://github.com/moby/moby)、[containerd](https://github.com/containerd/containerd)、[runc](https://github.com/opencontainers/runc)**.

由于只介绍docker核心的东西，所以**不会包含**下面这些项目：

* [compose](https://github.com/docker/compose)：使用Python语言开发，将多个相关的容器配置在一起，从而可以同时创建、启动、停止和监控它们。
* [machine](https://github.com/docker/machine)：帮助安装docker到指定位置，包括本地虚拟机、远端云主机等，同时还管理这些主机的信息，可以很方便的操作安装在不同主机上的这些docker。
* [kitematic](https://github.com/docker/kitematic)：桌面版的docker客户端（图形界面），使用JavaScript基于[electron](https://electron.atom.io/)开发。
* [toolbox](https://github.com/docker/toolbox)：帮助安装docker环境到Windows和Mac平台，包括Docker引擎、Compose、 Machine和 Kitematic，当然docker引擎是安装在虚拟机里面的，本地只有客户端，使用哪个虚拟机依赖于平台，toolbox会帮你搞定这一切。
* [distribution](https://github.com/docker/distribution)：[Registry 2.0](https://github.com/docker/distribution/blob/master/docs/spec/api.md)的实现，主要是管理和分发docker镜像，[Docker Hub](https://hub.docker.com/)背后的技术。
* [swarmkit](https://github.com/docker/swarmkit)：嵌入在docker里面的容器编排系统，可以简单的把它和docker的关系理解成IE浏览器和Windows的关系，捆绑销售。

## 面向读者

本系列主要**专注docker背后的技术和实现思路**，**不介绍怎么使用docker，不介绍代码细节**。

* 如果你是docker初学者，想了解怎么使用docker，那么本系列不适合你。
* 如果你已经熟悉了基本的操作，想了解下高级点的参数，或者想了解背后到底发生了什么，便于自己更好的使用docker，更好的解决碰到的问题，那么本系列适合你。
* 如果你是一名开发人员，想了解docker的代码实现细节，但又不知道从何处下手，本系列也许会给你一些启发。

## docker版本
自从docker决定将swarm整合进来弄企业版之后，代码一直在调整，docker的一些目录和程序名称也在发生变化，所以本系列的内容没法覆盖所有docker版本，只能挑其中的一个。

自v17.03开始，docker采用了新的发行方式，版本的发行周期变成了一个月一次，并且也分了企业版和社区版，在本系列中，将以**v17.03社区版**作为参考，建议大家阅读本系列时，手头的docker版本不低于v17.03。

docker完整的变更列表请参考[这里](https://github.com/moby/moby/blob/master/CHANGELOG.md)。

## docker和moby的关系
2017-04-18，在DockerCon 2017上，docker公司正式宣布成立moby项目，同时将github上的docker/docker项目重命名成了moby/moby，虽然会自动重定向，但代码里的相关引用不排除会有问题，需要留意。

这里不评价这次变化，对普通使用者来说，不会发生任何变化，还是熟悉的命令，熟悉的参数，对开发人员来说，代码的位置变了，但代码还是那份代码。

以后moby会变成什么样，现在还不清楚，有可能和docker的关系会变成[blink](https://www.chromium.org/blink)和chrome的关系一样，静观其变，希望不要影响我们学习。

>注意：若没有特别说明，本系列提到的docker源码，都指的是moby的代码

## 文章列表
该系列的所有文章都会列在这里，便于大家选择阅读

>尽量保证每周至少更新一篇，这里列出来但没有链接的表示即将要写的内容，敬请期待

* 走进docker系列(01)：hello-world的背后发生了什么？
* 走进docker系列(02)：image(镜像)是什么？
* 走进docker系列(03)：如何绕过docker运行hello-world？
* 走进docker系列(04)：什么是容器的runtime?
* 走进docker系列(05)：runc能干些啥?
* 走进docker系列(06)：如何给容器配置网络?

## 建议阅读
在阅读本系列之前，如果对Linux不是很熟的话，建议先阅读本人的[Linux程序员](https://segmentfault.com/blog/wuyangchun)专栏，里面包含了内存、CPU、文件系统、网络、namespace、cgroup等方面的详细内容，和docker相关的Linux知识还在更新中，敬请关注。

## 获取docker相关的代码
由于现在docker依赖的containerd和runc是github上两个单独的项目，如果你需要分析docker的代码，请确保containerd和runc的版本和docker的版本是一致的，检查办法如下：

```bash
#假设我们已经将docker的源代码clone到了/home/dev/repos/docker目录下
dev@debian:~/repos/docker$ git branch
* master

#列出17.03相关的tag
dev@debian:~/repos/docker$ git tag|grep 17.03
v17.03.0-ce
v17.03.0-ce-rc1
v17.03.1-ce
v17.03.1-ce-rc1

#取最新的v17.03.1-ce
dev@debian:~/repos/docker$ git checkout -b v17.03.1-ce v17.03.1-ce
Switched to a new branch 'v17.03.1-ce'
dev@debian:~/repos/docker$ git branch
  master
* v17.03.1-ce

#查看docker所用的runc和containerd的commit id
dev@debian:~/repos/docker$ egrep "RUNC_COMMIT|CONTAINERD_COMMIT" ./hack/dockerfile/binaries-commits
# When updating RUNC_COMMIT, also update runc in vendor.conf accordingly
RUNC_COMMIT=54296cf40ad8143b62dbcaa1d90e520a2136ddfe
CONTAINERD_COMMIT=4ab9917febca54791c5f071a9d1f404867857fcc

#查看runc和containerd的库路径
dev@debian:~/repos/docker$ egrep "runc.git|containerd.git" ./hack/dockerfile/install-binaries.sh
        git clone https://github.com/docker/runc.git "$GOPATH/src/github.com/opencontainers/runc"
        git clone https://github.com/docker/containerd.git "$GOPATH/src/github.com/docker/containerd"
```
根据上面的结果，先将runc和containerd克隆下来，然后checkout相应的commit id，这样就可以配合docker的代码一起看了。这里是上面例子中找到的containerd和runc的信息：

* containerd： https://github.com/docker/containerd.git  4ab9917febca54791c5f071a9d1f404867857fcc
* runc： https://github.com/docker/runc.git 54296cf40ad8143b62dbcaa1d90e520a2136ddfe

**注意：**

* 可能是为了方便对runc进行修改，docker将github.com/opencontainers/runc克隆到了github.com/docker/runc，在docker v17.03里面，runc是从github.com/docker/runc.git拉的代码，然后放在本地的opencontainers/runc目录下，假装是opencontainers的runc，这个需要留意，别pull了错误的库。
* 上面显示containerd的地址是https://github.com/docker/containerd.git，这个没有关系，github已经将这个地址重定向到了https://github.com/containerd/containerd.git
