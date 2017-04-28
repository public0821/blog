# 试玩image

上一篇介绍了image的格式，这里我们就来用一下hello-world这个image，看怎么输出和```docker run hello-world```同样的内容。

## 相关工具
本文将用到三个工具，分别是[skopeo](https://github.com/projectatomic/skopeo)、[oci-image-tools](https://github.com/opencontainers/image-tools)和[runc](https://github.com/opencontainers/runc)。

* skopeo： 用来从Docker Hub上拉取image，并保存为OCI格式
* oci-image-tools： 包含几个用来操作本地image的工具
* runc： 运行容器

runc可以用docker自带的docker-runc命令替代，效果是一样的，skopeo的安装可以参考[上一篇最后的介绍]()或者[github上的主页](https://github.com/projectatomic/skopeo)，oci-image-tools的安装请参考[github上的主页](https://github.com/opencontainers/image-tools)。

## 获取hello-world的image
利用skopeo获得hello-world的oci格式的image
```bash
dev@debian:~/images$ skopeo copy docker://hello-world oci:hello-world
dev@debian:~/images$ tree hello-world/
hello-world/
├── blobs
│   └── sha256
│       ├── 0a2ad94772e366c2b7f2266ca46daa0c38efe08811cf1c1dee6558fcd7f2b54e
│       ├── 78445dd45222097f5f8d5a16e48dc19c4ca162dcdb80010ab6f1ccfc7e2c0fa3
│       └── 998a60597add14861de504277c0d850e9181b1768011f51c7daaf694dfe975ef
├── oci-layout
└── refs
    └── latest
```

然后我们看看hello-world这个image的文件系统都有些什么文件
```bash
#利用oci-image-tool unpack，将image解压到hello-world-filesystem目录
dev@debian:~/images$ mkdir hello-world-filesystem
dev@debian:~/images$ oci-image-tool unpack --ref latest hello-world hello-world-filesystem
dev@debian:~/images$ tree hello-world-filesystem/
hello-world-filesystem/
└── hello

0 directories, 1 file
dev@debian:~/images$ file hello-world-filesystem/hello
hello-world-filesystem/hello: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, BuildID[sha1]=4999eecfa472a2341b53954c0eca1e893f01305c, stripped
```

从上面的结果可以看出，hello-world这个image就只包含一个名字叫做hello的静态链接的可执行文件。

## 根据image生成runtime的bundle
runtime的bundle就是运行容器时需要的东西的集合。
```bash
dev@debian:~/images$ mkdir hello-world-bundle
dev@debian:~/images$ oci-image-tool create --ref latest hello-world hello-world-bundle
dev@debian:~/images$ tree hello-world-bundle
hello-world-bundle
├── config.json
└── rootfs
    └── hello

1 directory, 2 files
```

从这里生成的bundle可以看出，bundle里面就是一个配置文件加上rootfs，rootfs里面的东西就是image里面的文件系统部分，config.json是对容器的描述，比如rootfs的路径，容器启动后要运行什么命令等，后续介绍runtime标准的时候再详细介绍。

## 使用runc运行该bundle
有了bundle后，就可以用runc来运行该容器了

>这里直接用docker的docker-runc代替runc命令，如果你自己编译了opencontainer的runc，那么用runc命令也是一样的。

```       
dev@debian:~/images$ cd hello-world-bundle/
#oci-image-tool帮我们生成的config文件版本和runc需要的版本不一致，
#所以这里先将它删掉，然后用runc spec命令生成一个默认的config文件
dev@debian:~/images/hello-world-bundle$ rm config.json
dev@debian:~/images/hello-world-bundle$ docker-runc spec

#默认生成的config里面指定容器启动的进程为sh，
#我们需要将它换成我们的hello程序
#这里用vim修改config.json文件，将里面的"args": ["sh"]改成"args": ["/hello"]
dev@debian:~/images/hello-world-bundle$ vim config.json

#然后用runc运行该容器，这里命令行里的hello是给容器取的名字，
#可以是任意不和其它容器冲突的字符串
dev@debian:~/images/hello-world-bundle$ sudo docker-runc run hello
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://cloud.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/

```

## 结束语
该篇展示了如何不通过docker而运行docker的hello-world容器，主要目的为了了解镜像以及runc之间的关系，同时也触发我们思考一个问题，既然我们可以绕过docker运行容器，那我们为什么还要用docker呢？