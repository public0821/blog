## 代码相关
看代码之前一定要先了解docker背后的相关技术以及思路，文档的更新可能滞后，但代码不会骗人，看代码的目的是为了了解实现细节，以及验证某些想法推测，或者某些部分不太清楚，需要从代码来反推实现思路，对代码熟悉之后，就能更好的处理遇到的docker问题，也可以对docker做定制开发。

反过来如果对docker背后的技术一点都不了解，想完全从代码来反推原理是不太现实的，就像不懂操作系统的相关知识而去直接看Linux的内核代码一样，效率很低不说，有可能还看不明白。

本人刚开始看代码，所以给不出什么好的建议，还是老老实实一个命令一个命令的来跟踪代码吧。

>containerd和runc原来都属于docker项目，只是后来由于某些原因独立了出去

#### docker客户端程序
代码在[docker](https://github.com/docker/docker)里面，main函数在cmd/docker/docker.go，其命令行的处理用的是github.com/spf13/cobra这个库，稍微了解一下该库对看代码有帮助。

如果想直接看命令干了些啥，cli/command下面包含了所有的命令，并且以文件夹进行组织，比如想看看docker run命令都干了些啥，可以直接看cli/command/container/run.go里面的NewRunCommand函数，它里面包含了命令行参数的注册，以及命令的执行函数```RunE: func(cmd *cobra.Command, args []string) error ```，直接看该执行函数的内容就可以了。

#### dockerd服务器端程序
代码也在[docker](https://github.com/docker/docker)里面，main函数在cmd/dockerd/docker.go，通过http对外提供服务，router的初始化在cmd/dockerd/daemon.go的initRouter函数中，具体每个router的定义在api/server/router下面的子目录里面，通过router就能找到具体干活的函数。

#### docker-containerd进程
代码在[containerd](https://github.com/containerd/containerd)里面，main函数在containerd/main.go，对外通过[grpc](http://www.grpc.io/)提供服务，所以了解下grpc框架就能看懂代码是怎么调的了。

如果只想了解一下干了些啥，那么看API接口的定义和实现就可以了，它们分别在api/grpc/types/api.proto(api.pb.go)和api/grpc/server/server.go里面

#### docker-containerd-shim进程
代码也在[containerd](https://github.com/containerd/containerd)里面，当需要创建container或者使用exec命令在指定container里面执行程序时，docker-containerd就会启动一个docker-containerd-shim进程，抛开exec命令不谈，可以理解为一个docker-containerd-shim对应一个container

containerd-shim是一个小程序，负责container运行时这一块的工作，相当于是对runc的一个包装，具体干活还得靠runc，main函数在containerd-shim/main.go。

#### docker-runc进程
代码在[runc](https://github.com/docker/runc.git)里面，docker的runc项目克隆自opencontainer的runc项目，应该只有一些细微差别，所以看哪个都可以。

runc进程由containerd-shim启动，runc启动容器里面的进程后就退出了，由于containerd-shim调用了```syscall.RawSyscall(syscall.SYS_PRCTL, prSetChildSubreaper, uintptr(i), 0)```，所以当runc退出的时候，它的所有子孙进程都变成了containerd-shim的子孙进程，这也是为什么我们通过ps命令看不到runc的原因。从Linux 3.4开始，[prctl](http://man7.org/linux/man-pages/man2/prctl.2.html)增加了对PR_SET_CHILD_SUBREAPER的支持，这样就可以控制孤儿进程可以被谁接管，而不是像以前一样只能由init进程接管。

