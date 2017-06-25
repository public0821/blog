# 走进docker(07)：docker start命令背后发生了什么？

在[上一篇](https://segmentfault.com/a/1190000009769352)介绍过了docker create之后，这篇来看看docker start是怎么根据create之后的结果运行容器的。

## 启动容器
在这里我们启动[上一篇](https://segmentfault.com/a/1190000009769352)中创建的那个容器，然后看看docker都干了些什么。
```
#根据容器名称启动容器（也可以根据容器ID来启动）
root@dev:~# docker start docker_test
docker_test

#可以看出容器正在后台运行bash
root@dev:~# docker ps
CONTAINER ID   IMAGE    COMMAND       CREATED          STATUS         PORTS   NAMES
967438113fba   ubuntu   "/bin/bash"   38 minutes ago   Up 8 seconds           docker_test
```

## start的大概流程

* docker（client）发送启动容器命令给dockerd
* dockerd收到请求后，准备好rootfs，以及一些其它的配置文件，然后通过grpc的方式通知containerd启动容器
* containerd根据收到的请求以及配置文件位置，创建容器运行时需要的bundle，然后启动shim进程，让它来启动容器
* shim进程启动后，做一些准备工作，然后调用runc启动容器

下面就来详细的解释一下每一步都干了些什么。

## dockerd
我们先来看看dockerd收到客户端的启动容器请求后，做了些什么。

#### 准备rootfs
dockerd做的第一件事情就是准备好容器运行时需要的rootfs，由于在docker create创建容器的时候，容器的所有layer都已经准备好了，现在就差一步将他们合并起来了，对于aufs来说，需要通过mount的方式将所有的layer合并起来，对于其他的文件系统来说，有些可能不需要这一步，/var/lib/docker/aufs/mnt下面已经是合并好的rootfs了。

下面来看看这个容器启动之后/var/lib/docker/aufs/mnt下的内容。
```
#init目录下没有文件
root@dev:/var/lib/docker/aufs/mnt# tree 305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281-init/
305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281-init/

0 directories, 0 files

#305226f...目录下面的内容就是rootfs的内容，包含了大量的文件
root@dev:/var/lib/docker/aufs/mnt# tree 305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281 | tail
    │   ├── lastlog
    │   └── wtmp
    ├── mail
    ├── opt
    ├── run -> /run
    ├── spool
    │   └── mail -> ../mail
    └── tmp

692 directories, 4804 files

#虽然在容器中，/dev/console，/etc/hosts，/etc/hostname，
#/etc/resolv.conf这几个文件都有内容，
#但从外面主机的mount namespace中来看的话，还是空的，
#因为bind mount发生在容器中的mount namespace中，所以外面根本就看不到
root@dev:/var/lib/docker/aufs/mnt/305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281# ls -l ./dev/console ./etc/hosts ./etc/hostname ./etc/resolv.conf
-rwxr-xr-x 1 root root 0 Jun 25 11:25 ./dev/console
-rwxr-xr-x 1 root root 0 Jun 25 11:25 ./etc/hostname
-rwxr-xr-x 1 root root 0 Jun 25 11:25 ./etc/hosts
-rwxr-xr-x 1 root root 0 Jun 25 11:25 ./etc/resolv.conf
```

和上一篇中create之后的内容相比，唯一的差别就是305226f...目录下有了内容，而init目录下还是空的，说明对于aufs文件系统来说，它只需要构造好最上面的一层就可以了，不需要init层和它下面所有层合并之后的结果，大家有兴趣的话可以检查一下/var/lib/docker/aufs/mnt目录下的其它目录的内容，会发现其它层的文件夹也全是空的，因为aufs只在运行的时候动态的将容器的最上面一层和下面的所有层进行合并，合并的过程等同于下面的命令：

```
root@dev:/var/lib/docker/aufs/diff# mkdir /tmp/rootfs
root@dev:/var/lib/docker/aufs/diff# mount -t aufs -o br=./305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281=rw:./305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281-init=ro:./7938f2b32c53a9e0d3974f9579dd9dbb450202e1e11fe514e31556d4ea808c4e=ro:./4c10796e21c796a6f3d83eeb3613c566ca9e0fd0a596f4eddf5234b87955b3c8=ro:./fd0ba28a44491fd7559c7ffe0597fb1f95b63207a38a3e2680231fb2f6fe92bd=ro:./b656bf5f0688069cd90ab230c029fdfeb852afcfd0d1733d087474c86a117da3=ro:./1e83d2ea184e08eed978127311cc96498e319426abe2fb5004d4b1454598bd76=ro none /tmp/rootfs

root@dev:/var/lib/docker/aufs/diff# tree /tmp/rootfs/ | tail
    │   ├── lastlog
    │   └── wtmp
    ├── mail
    ├── opt
    ├── run -> /run
    ├── spool
    │   └── mail -> ../mail
    └── tmp

693 directories, 4820 files

#这里mount后的文件夹和文件数量要多于上面的/var/lib/docker/aufs/mnt/305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281，可能跟mount时使用的参数有关，具体情况我没有仔细研究，有兴趣的话可以参考源代码docker/daemon/graphdriver/aufs/aufs.go中的aufsMount函数。
```

>关于aufs文件系统的使用可以参考：[Linux文件系统之aufs](https://segmentfault.com/a/1190000008489207)

#### 准备容器内部需要的文件
先看看container目录下的变化：
```
root@dev:/var/lib/docker/containers# tree 967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368/
967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368/
├── 967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368-json.log
├── checkpoints
├── config.v2.json
├── hostconfig.json
├── hostname
├── hosts
├── resolv.conf
├── resolv.conf.hash
└── shm

2 directories, 7 files
```
容器启动后，多了下面这几个文件，这几个文件都是docker动态生成的：

* 967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368-json.log：容器的日志文件，后续容器的stdout和stderr都会输出到这个目录
* hostname：里面是容器的主机名，来自于config.v2.json，由docker create命令的-h参数指定，如果没指定的话，就是容器ID的前12位，这里即为967438113fba
* resolv.conf：里面包含了DNS服务器的IP，来自于hostconfig.json，由docker create命令的--dns参数指定，没有指定的话，docker会根据容器的网络类型生成一个默认的，一般是主机配置的DNS服务器或者是docker bridge的IP。
* resolv.conf.hash：resolv.conf文件的校验码
* shm：为容器分配的一个内存文件系统，后面会绑定到容器中的/dev/shm目录，可以由docker create的参数--shm-size控制其大小，默认是64M，其本质上就是一个挂载到/dev/shm的tmpfs，由于这个目录的内容是放在内存中的，所以读写速度快，有些程序会利用这个特点而用到这个目录，所以docker事先为容器准备好这个目录。

>注意：除了日志文件外，其它文件在每次容器启动的时候都会自动生成，所以修改他们的内容后只会在当前容器运行的时候生效，容器重启后，配置又都会恢复到默认的状态

#### 准备OCI需要的bundle
在[什么是容器的runtime?](https://segmentfault.com/a/1190000009583199)中，介绍过bundle的概念，它主要包含一个名字叫做config.json的配置文件。

dockerd在生成这个文件前，要做一些准备工作，比如创建好cgroup的相关目录，准备网络相关的配置等，然后才生成config.json文件。

> cgroup的相关目录可以直接通过命令```find /sys/fs/cgroup/ -name 967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368```找到.
> 网络相关的内容这里不介绍，后续会有专门的文章进行介绍。

bundle被dockerd放在了目录/run/docker/libcontainerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368下，我们这里主要看一下生成的config.json文件中一些比较重要且易懂的字段。

```
#这里的只截取了部分输出，仅供参考
root@dev:/run/docker/libcontainerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368# cat config.json |python -m json.tool
{
    "hostname": "967438113fba",
    "linux": {
        "cgroupsPath": "/docker/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368",
        "namespaces": [
            {
                "type": "mount"
            },
            {
                "type": "network"
            },
            {
                "type": "uts"
            },
            {
                "type": "pid"
            },
            {
                "type": "ipc"
            }
        ]
    },
    "mounts": [
        {
            "destination": "/etc/resolv.conf",
            "options": [
                "rbind",
                "rprivate"
            ],
            "source": "/var/lib/docker/containers/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368/resolv.conf",
            "type": "bind"
        },        
        {
            "destination": "/etc/hostname",
            "options": [
                "rbind",
                "rprivate"
            ],
            "source": "/var/lib/docker/containers/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368/hostname",
            "type": "bind"
        },
        {
            "destination": "/etc/hosts",
            "options": [
                "rbind",
                "rprivate"
            ],
            "source": "/var/lib/docker/containers/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368/hosts",
            "type": "bind"
        },
        {
            "destination": "/dev/shm",
            "options": [
                "rbind",
                "rprivate"
            ],
            "source": "/var/lib/docker/containers/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368/shm",
            "type": "bind"
        }
    ],
    "process": {
        "args": [
            "/bin/bash"
        ],
        "cwd": "/",
        "env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "HOSTNAME=967438113fba",
            "TERM=xterm"
        ],
        "terminal": true,
        "user": {
            "gid": 0,
            "uid": 0
        }
    },
    "root": {
        "path": "/var/lib/docker/aufs/mnt/305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281"
    }
}
```

#### 准备IO文件
在bundle目录里面，除了上面介绍的容器配置文件之外，dockerd还创建了一些跟io相关的命名管道，主要用来和容器之间进行通信，比如这里的init-stdin文件用来向容器的stdin中写数据，init-stdout用来接收容器的stdout输出。这里不详细介绍标准输入输出相关的内容，后续文章中会专门介绍。

```
#bundle目录里面除了config.json之外，还有两个文件
root@dev:/run/docker/libcontainerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368# tree
.
├── config.json
├── init-stdin
└── init-stdout

0 directories, 3 files

#这两个文件是命名管道文件
root@dev:/run/docker/libcontainerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368# file init-stdin init-stdout
init-stdin:  fifo (named pipe)
init-stdout: fifo (named pipe)

#它们被dockerd和docker-containerd-shim两个进程所打开
root@dev:/run/docker/libcontainerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368# lsof *
COMMAND    PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
dockerd   1218 root   18u  FIFO   0,18      0t0  640 init-stdin
dockerd   1218 root   21u  FIFO   0,18      0t0  641 init-stdout
dockerd   1218 root   24w  FIFO   0,18      0t0  640 init-stdin
dockerd   1218 root   25r  FIFO   0,18      0t0  641 init-stdout
docker-co 7971 root    7u  FIFO   0,18      0t0  640 init-stdin
docker-co 7971 root    9u  FIFO   0,18      0t0  640 init-stdin
docker-co 7971 root   10r  FIFO   0,18      0t0  640 init-stdin
docker-co 7971 root   12u  FIFO   0,18      0t0  641 init-stdout
docker-co 7971 root   13w  FIFO   0,18      0t0  641 init-stdout
docker-co 7971 root   14u  FIFO   0,18      0t0  641 init-stdout
docker-co 7971 root   15r  FIFO   0,18      0t0  641 init-stdout
docker-co 7971 root   16w  FIFO   0,18      0t0  640 init-stdin

#7971是容器进程docker-containerd-shim 
root@dev:/run/docker/libcontainerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368# ps -ef|grep 7971|grep docker
root      7971  1311  0 17:43 ?        00:00:00 docker-containerd-shim 967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368 /var/run/docker/libcontainerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368 docker-runc
```

待上面的文件都准备好了之后，通过grpc的方式给containerd发送请求，通知containerd启动容器。

## containerd
## 结束语
docker start命令干的活很多，有些在这里并没有涉及到，比如存储插件和网络，以及容器直接网络link之类的，后续在专门介绍相关部分的时候再详细介绍。

