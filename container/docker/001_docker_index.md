# docker系列(01)：开篇

本人docker初学者，边学习边总结，一方面加深自己的理解，另一方面希望对其他想深入了解docker的同学有所帮助。

由于本人缺乏实战经验，错误在所难免，欢迎批评指正，谢谢。

## 包含的内容
本系列主要介绍三个github上的项目：[docker](https://github.com/docker/docker)、[containerd](https://github.com/containerd/containerd)和[runc](https://github.com/opencontainers/runc).

由于只介绍docker核心的东西，所以**不会包含**下面这些项目：

* [compose](https://github.com/docker/compose)：使用Python语言开发，将多个相关的容器配置在一起，从而可以同时创建、启动、停止和监控它们。
* [machine](https://github.com/docker/machine)：帮助安装docker到指定位置，包括本地虚拟机、远端云主机等，同时还管理这些主机的信息，可以很方便的操作安装在不同主机上的这些docker。
* [kitematic](https://github.com/docker/kitematic)：桌面版的docker客户端（图形界面），使用JavaScript基于[electron](https://electron.atom.io/)开发。
* [toolbox](https://github.com/docker/toolbox)：帮助安装docker环境到Windows和Mac平台，包括Docker引擎、Compose、 Machine和 Kitematic，当然docker引擎是安装在虚拟机里面的，本地只有客户端，使用哪个虚拟机依赖于平台，toolbox会帮你搞定这一切。
* [distribution](https://github.com/docker/distribution)：[Registry 2.0](https://github.com/docker/distribution/blob/master/docs/spec/api.md)的实现，主要是管理和分发docker镜像，[Docker Hub](https://hub.docker.com/)背后的技术。
* [swarmkit](https://github.com/docker/swarmkit)：嵌入在docker里面的容器编排系统，可以简单的把它和docker的关系理解成IE浏览器和Windows的关系，捆绑销售。

## 面向读者

本系列**不介绍怎么使用docker，不介绍代码细节**，主要专注docker背后的技术和实现思路。

* 如果你是docker初学者，想了解怎么使用docker，那么本系列不适合你。
* 如果你已经熟悉了基本的操作，想了解下高级点的参数，或者想了解背后到底发生了什么，便于自己更好的使用docker，更好的解决碰到的问题，那么本系列适合你。
* 如果你是一名开发人员，想了解docker的代码实现细节，但又不知道从何处下手，本系列也许会给你一些启发。

希望在结束本系列的时候，你也可以有自己的想法，像[rkt](https://github.com/coreos/rkt)一样，开发出一个可以和docker竞争的产品

## docker版本
自从docker决定将swarm整合进来之后，代码一直在调整，docker的一些目录和程序名称也在发生变化，所以本系列的内容没法覆盖所有docker版本，但尽量会支持1.13及以上版本，如果变化太快的话，这里会更新支持的版本信息，建议大家阅读本系列时，手头的docker版本不低于1.13。docker完整的变更列表请参考[这里](https://github.com/docker/docker/blob/master/CHANGELOG.md)。

## 文章列表
该系列的所有文章都会列在这里，便于大家选择阅读

* docker系列(01)：开篇
* docker系列(02)：hello-world的背后发生了什么？

## 建议阅读
在阅读本系列之前，如果对Linux不是很熟的话，建议先阅读本人的[Linux程序员](https://segmentfault.com/blog/wuyangchun)专栏，里面包含了内存、CPU、文件系统、网络、namespace、cgroup等方面的详细内容，和docker相关的Linux知识还在更新中，敬请关注。


