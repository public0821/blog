# 走进docker系列(04)：什么是容器的runtime?

我们都知道runc是容器runtime的一个实现，那到底什么是runtime？包含了哪些内容？

容器的runtime和image一样，也有规范，也由[Open Containers Initiative](https://www.opencontainers.org/)(OCI)负责维护，地址为[Runtime Specification](https://github.com/opencontainers/runtime-spec/blob/master/spec.md)，本文将对该标准做一个简单的解释。

## 规范内容
在Linux平台上，跟runtime有关的规范主要有四个，分别是[Runtime and Lifecycle (runtime.md)](https://github.com/opencontainers/runtime-spec/blob/master/runtime.md)、[Container Configuration file (config.md)](https://github.com/opencontainers/runtime-spec/blob/master/config.md)、[Linux Container Configuration (config-linux.md)](https://github.com/opencontainers/runtime-spec/blob/master/config-linux.md)和[Linux Runtime (runtime-linux.md)](https://github.com/opencontainers/runtime-spec/blob/master/runtime-linux.md) .

除了这四个之外，还有一个[Filesystem Bundle (bundle.md)](https://github.com/opencontainers/runtime-spec/blob/master/bundle.md)。

下面分别介绍这些规范。

### [Filesystem Bundle](https://github.com/opencontainers/runtime-spec/blob/master/bundle.md)
在[上一篇](https://segmentfault.com/a/1190000009309378)中，已经用过bundle了，hello-world的bundle看起来是这个样子的：
```bash
dev@debian:~/images$ tree hello-world-bundle
hello-world-bundle
├── config.json
└── rootfs
    └── hello

1 directory, 2 files
```

bundle中包含了需要运行容器的所有信息，有了这个bundle后，符合runtime标准的程序（比如runc）就可以根据bundle启动容器了。

bundle包含一个config.json文件和容器的根文件系统目录，config.json就是后面要介绍的[Container Configuration file](https://github.com/opencontainers/runtime-spec/blob/master/config.md)，标准要求该配置文件必须叫这个名字，不过对容器的根文件系统目录没有要求，只要在config.json里面将路径配置正确就可以了，不过一般约定俗成都叫rootfs。

实际使用过程中，根文件系统目录可能在其它的地方，只要config.json里面配置正确的路径就可以了，但如果bundle需要打包和其它人分享的话，必须将根文件系统和config.json打包在一起，并且不包含外层的文件夹。

### [Container Configuration file](https://github.com/opencontainers/runtime-spec/blob/master/config.md)
该规范定义了上面介绍的config.json里面应该包含哪些内容，字段很多，这里不一一介绍，只简单说明一下，完整的示例请参考[这里](https://github.com/opencontainers/runtime-spec/blob/master/config.md#configuration-schema-example)：

* ociVersion（必须）：对应的OCI标准版本
* root（必须）：根文件系统的位置
* mounts：需要挂载哪些目录到容器里面。如果是Linux平台，这里面必须要包含/proc、/sys，/dev/pts，/dev/shm这四个目录
* process：容器启动后执行什么命令
* hostname：容器的主机名，相关原理可参考[UTS namespace (CLONE_NEWUTS)](https://segmentfault.com/a/1190000006908598)
* platform（必须）：平台信息，如 amd64 + Linux
* linux：Linux平台的特殊配置，这里包含下面要介绍的[Linux Container Configuration](https://github.com/opencontainers/runtime-spec/blob/master/config-linux.md)里面的内容
* hooks：配置容器运行生命周期中会调用的hooks，包括prestart、poststart和poststop，容器的生命周期见后面[Runtime and Lifecycle](https://github.com/opencontainers/runtime-spec/blob/master/runtime.md)介绍。
* annotations：相当于容器的属性信息，key:value格式

### [Linux Container Configuration](https://github.com/opencontainers/runtime-spec/blob/master/config-linux.md)
该规范是Linux平台上对[Container Configuration file](https://github.com/opencontainers/runtime-spec/blob/master/config.md)的补充，这部分的内容也包含在上面的config.json文件中。

* namespaces: namespace相关的配置，相关原理可参考[Namespace概述](https://segmentfault.com/a/1190000006908272)及这些namespace（[UTS](https://segmentfault.com/a/1190000006908598)、[IPC](https://segmentfault.com/a/1190000006908729)、[mount](https://segmentfault.com/a/1190000006912742)、[pid](https://segmentfault.com/a/1190000006912878)、[network](https://segmentfault.com/a/1190000006912930)、[user 1](https://segmentfault.com/a/1190000006913195)、[user 2](https://segmentfault.com/a/1190000006913499)）。
* uidMappings，gidMappings：配置主机和容器用户/组之间的对应关系，原理可参考[user namespace](https://segmentfault.com/a/1190000006913195)
* devices：设置哪些设备可以在容器内被访问到。除了这里指定的设备外，/dev/null、/dev/zero、/dev/full、/dev/random、/dev/urandom、/dev/tty、/dev/console（如果在process的配置里面启动terminal的话）和/dev/ptmx这些设备默认就能在容器内访问到，即runtime的实现需要默认将这些设备bind到容器内，dev/tty和/dev/ptmx的原理可以参考[TTY/PTS概述](https://segmentfault.com/a/1190000009082089)
* cgroupsPath：cgroup的路径，可参考[Cgroup概述](https://segmentfault.com/a/1190000006917884)
* resources：Cgroup中具体子项的配置，包括[device whitelist](https://www.kernel.org/doc/Documentation/cgroup-v1/devices.txt), [memory](https://segmentfault.com/a/1190000008125359), [cpu](https://segmentfault.com/a/1190000008323952), [blockIO](https://www.kernel.org/doc/Documentation/cgroup-v1/blkio-controller.txt), [hugepageLimits](https://www.kernel.org/doc/Documentation/cgroup-v1/hugetlb.txt), network([net_cls cgroup](https://www.kernel.org/doc/Documentation/cgroup-v1/net_cls.txt)和[net_prio cgroup](https://www.kernel.org/doc/Documentation/cgroup-v1/net_prio.txt)), [pids](https://segmentfault.com/a/1190000007468509)
* intelRdt：和[Intel Resource Director Technology](https://www.kernel.org/doc/Documentation/x86/intel_rdt_ui.txt)有关
* sysctl：调整容器运行时的kernel参数，主要是一些网络参数，因为每个network namespace都有自己的协议栈，所以可以修改自己协议栈的参数而不影响别人
* seccomp：和安全相关的配置，见[Seccomp ](https://www.kernel.org/doc/Documentation/prctl/seccomp_filter.txt)
* rootfsPropagation：设置Propagation类型。可以参考[Shared subtrees](https://segmentfault.com/a/1190000006899213)
* maskedPaths：设置容器内的哪些目录对用户不可见
* readonlyPaths：设置容器内的哪些目录是只读的
* mountLabel：和Selinux有关。

### [Runtime and Lifecycle](https://github.com/opencontainers/runtime-spec/blob/master/runtime.md)
该规范主要定义了根容器相关的三部分内容，容器相关的操作、生命周期以及容器的状态。
#### 容器相关的操作
该部分定义了一个符合runtime标准的实现（如runc）至少需要实现下面这些命令：

* Create： 
* Start： 
* Kill：
* Delete：
#### 生命周期
1. 首先

### [Linux Runtime](https://github.com/opencontainers/runtime-spec/blob/master/runtime-linux.md)
该规范是Linux平台上对[Runtime and Lifecycle](https://github.com/opencontainers/runtime-spec/blob/master/runtime.md)的补充，目前该规范很简单，只要求容器运行起来后，里面必须建立下面这些软连接：
```
# ls -l /dev/fd /dev/std*
lrwxrwxrwx    1 root     root            13 May  4 12:32 /dev/fd -> /proc/self/fd
lrwxrwxrwx    1 root     root            15 May  4 12:32 /dev/stderr -> /proc/self/fd/2
lrwxrwxrwx    1 root     root            15 May  4 12:32 /dev/stdin -> /proc/self/fd/0
lrwxrwxrwx    1 root     root            15 May  4 12:32 /dev/stdout -> /proc/self/fd/1
```

## 结束语
理解runtime后，就大概知道了docker和runc分别负责什么内容，那就是docker负责准备runtime的bundle，而runc负责运行该bundle，这为进一步详细的了解它们做好了准备。

## 参考

* [Open Container Initiative Runtime Specification](https://github.com/opencontainers/runtime-spec)
