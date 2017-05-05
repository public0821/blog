# 走进docker系列(04)：什么是容器的runtime?

我们都知道runc是容器runtime的一个实现，那到底什么是runtime？包含了哪些内容？

容器的runtime和image一样，也有规范，也由[Open Containers Initiative](https://www.opencontainers.org/)(OCI)负责维护，地址为[Runtime Specification](https://github.com/opencontainers/runtime-spec/blob/master/spec.md)，本文将对该标准做一个简单的解释。

## 规范内容
在Linux平台上，跟runtime有关的规范主要有四个，分别是[Runtime and Lifecycle (runtime.md)](https://github.com/opencontainers/runtime-spec/blob/master/runtime.md)、[Container Configuration file (config.md)](https://github.com/opencontainers/runtime-spec/blob/master/config.md)、[Linux Container Configuration (config-linux.md)](https://github.com/opencontainers/runtime-spec/blob/master/config-linux.md)和[Linux Runtime (runtime-linux.md)](https://github.com/opencontainers/runtime-spec/blob/master/runtime-linux.md) .

除了这四个之外，还有一个[Filesystem Bundle (bundle.md)](https://github.com/opencontainers/runtime-spec/blob/master/bundle.md)。

下面分别介绍这些规范。

### [Filesystem Bundle](https://github.com/opencontainers/runtime-spec/blob/master/bundle.md)
其实在[上一篇]()中，已经用过bundle了，hello-world的bundle看起来是这个样子的：
```bash
dev@debian:~/images$ tree hello-world-bundle
hello-world-bundle
├── config.json
└── rootfs
    └── hello

1 directory, 2 files
```

bundle中包含了需要运行容器的所有信息，有了这个bundle后，符合runtime标准的程序（比如runc）就可以根据bundle启动容器了。

bundle包含一个config.json文件和容器的根文件系统目录，config.json就是后面要介绍的[Container Configuration file](https://github.com/opencontainers/runtime-spec/blob/master/config.md)，标准要求该配置文件必须叫这个名字，不过对容器的根文件系统目录没有要求，只要在config.json将路径配置正确就可以了，不过一般约定俗成都叫rootfs。

实际使用过程中，根文件系统目录可能在其它的地方，只要config.json里面配置正确的路径就可以了，但如果bundle需要打包和其它人分享的话，必须将根文件系统和config.json打包在一起，并且不包含外层的文件夹。

### [Container Configuration file](https://github.com/opencontainers/runtime-spec/blob/master/config.md)
该规范定义了上面介绍的config.json里面应该包含哪些内容，字段很多，这里不介绍细节，只列出一些重要的内容。

### [Linux Container Configuration](https://github.com/opencontainers/runtime-spec/blob/master/config-linux.md)
该规范是Linux平台上对[Container Configuration file](https://github.com/opencontainers/runtime-spec/blob/master/config.md)的补充，这部分的内容也包含在上面的config.json文件中。

### [Runtime and Lifecycle](https://github.com/opencontainers/runtime-spec/blob/master/runtime.md)
该规范主要定义了容器的三方面内容，容器相关的操作、生命周期以及容器的状态。
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

## 参考

* [Open Container Initiative Runtime Specification](https://github.com/opencontainers/runtime-spec)
