# 走进docker(07)：docker start命令背后发生了什么？

在[上一篇](https://segmentfault.com/a/1190000009769352)介绍过了```docker create```之后，这篇来看看```docker start```是怎么根据create之后的结果运行容器的。

## 启动容器
在这里我们先启动[上一篇](https://segmentfault.com/a/1190000009769352)中创建的那个容器，然后看看docker都干了些什么。
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

下面就来详细的了解一下每一步都干了些什么。

## dockerd
首先来看看dockerd收到客户端的启动容器请求后，做了些什么。

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

#这里mount后的文件夹和文件数量要多于上面的/var/lib/docker/aufs/mnt/305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281，
#可能跟mount时使用的参数有关，具体情况我没有仔细研究，
#有兴趣的话可以参考源代码docker/daemon/graphdriver/aufs/aufs.go中的aufsMount函数。
```

>关于aufs文件系统的使用可以参考：[Linux文件系统之aufs](https://segmentfault.com/a/1190000008489207)

#### 准备容器内部需要的文件
rootfs准备好了之后，dockerd接着就会准备一些容器里面需要用到的配置文件，先看看container目录下的变化：
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

* 967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368-json.log：容器的日志文件，后续容器的stdout和stderr都会输出到这个目录。当然如果配置了其它的日志插件的话，日志就会写到别的地方。
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

bundle被dockerd放在了目录/run/docker/libcontainerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368下，我们这里主要看一下生成的config.json文件中一些比较常见且易懂的字段。

>只有当容器在运行的时候，目录/run/docker/libcontainerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368才存在，容器停止执行后该目录会被删除掉，下一次启动的时候会再次被创建。

```
#这里的只截取了部分输出，仅供参考
root@dev:/run/docker/libcontainerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368# cat config.json |python -m json.tool
{
    "hostname": "967438113fba",     #主机名
    "linux": {
        "cgroupsPath": "/docker/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368",        #cgroup路径
        "namespaces": [           #需要加入的namespace，只有type没有值表示创建并加入一个新的namespace，这里没看到user namespace，说明docker默认情况下是不开启user namespace的。
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
    "mounts": [        #需要mount到容器中的文件或者目录，这里列出来的的几个文件就是上面介绍的由dockerd进程生成的那几个文件，它们将通过bind的方式mount到容器中
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
    "process": {      #这里/bin/bash就是进程启动后要运行的程序，
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
    "root": {       #rootfs的路径
        "path": "/var/lib/docker/aufs/mnt/305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281"
    }
}
```

#### 准备IO文件
在bundle目录里面，除了上面介绍的容器配置文件之外，dockerd还创建了一些跟io相关的命名管道，用来和容器之间进行通信，比如这里的init-stdin文件用来向容器的stdin中写数据，init-stdout用来接收容器的stdout输出。

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

>上面只有init-stdin和init-stdout，没有init-stderr，那是因为我们创建容器的时候指定了-t参数，意思是让docker为容器创建一个tty（虚拟的），在这种情况下，stdout和stderr将采用同样的通道，即容器中进程往stderr中输出数据时，会写到init-stdout中。

待上面的文件都准备好了之后，通过grpc的方式给containerd发送请求，通知containerd启动容器。

## containerd
containerd主要功能是启动并管理运行时的所有contianer。

#### 准备相关文件
containerd会创建目录/run/docker/libcontainerd/containerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368/init并将相关文件放到这里。

>只有当容器在运行的时候，目录/run/docker/libcontainerd/containerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368才存在，容器停止执行后该目录会被删除掉，下一次启动的时候会再次被创建。

```
root@dev:/run/docker/libcontainerd/containerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368/init# file *
control:       fifo (named pipe)
exit:          fifo (named pipe)
log.json:      empty
pid:           ASCII text, with no line terminators
process.json:  ASCII text, with very long lines
shim-log.json: empty
starttime:     ASCII text, with no line terminators
```

* control: 用来往shim发送控制命令，包括关闭stdin和调整终端的窗口大小。
* exit：shim进程退出的时候，会关闭该管道，然后containerd就会收到通知，做一些清理工作。
* process.json：包含容器中进程相关的一些属性信息，后续在这个容器上执行docker exec命令时会用到这个文件。
* log.json: runc如果运行失败的话，会写日志到这个文件
* shim-log.json：shim进程执行失败的话，会写日志到这个文件
* pid：容器启动后，runc会将容器中第一个进程的pid写到这个文件中（外面pid namespace中的pid）
* starttime：记录容器的启动时间

#### 启动过程

1. contianerd收到启动容器请求后，就会创建control、exit、process.json这三个文件
2. 然后启动shim进程，等着runc创建容器并将容器里第一个进程的pid写入pid文件
3. 如果containerd读取pid文件失败，则读取shim-log.json和log.json，看出了什么异常
4. 如果读取pid文件成功，说明容器创建成功，则将当前时间作为容器的启动时间写入starttime文件
5. 调用runc的start命令启动容器

#### 监听容器
待容器启动之后，containerd还需要监听容器的OOM事件和容器退出事件，以便及时作出响应，OOM事件通过[cgroup的内存限制](https://segmentfault.com/a/1190000008125359)机制进行监听（通过group.event_control），而容器退出事件通过exit这个命名pipe来实现。

>按道理来说如果容器里面的所有进程属于一个pid namespace的话，id为1的进程退出后，容器也就退出了，调用wait函数并传入容器里第一个进程的pid也能知道容器是否退出，不确定为什么containerd一定要弄个exit来监听容器的退出，我没有继续深入研究，可能是因为pipe的fd可以通过epool来统一监听并且是异步，处理起来方便。

## shim
shim进程被containerd启动之后，第一步是设置子孙进程成为孤儿进程后由shim进程接管，即shim将变成孤儿进程的父进程，这样就保证容器里的第一个进程不会因为runc进程的退出而被init进程接管。

>从Linux 3.4开始，[prctl](http://man7.org/linux/man-pages/man2/prctl.2.html)增加了对PR_SET_CHILD_SUBREAPER的支持，这样就可以控制孤儿进程可以被谁接管，而不是像以前一样只能由init进程接管。

接着根据传入的参数设置好要启动进程的stdin,stdout,stderr（来自于上面的init-stdin，init-stdout，init-stderr），然后调用```runc create```命令创建容器，容器创建成功后，runc会将容器的第一个进程的pid写入上面containerd目录下的pid文件中，这样containerd进程就知道容器创建成功了，于是containerd接着就会调用```runc start```启动容器。

## runc
runc会被调用两次，第一次是shim调用```runc create```创建容器，第二次是containerd调用```runc start```启动容器。

#### 创建容器
runc会根据参数中传入的bundle目录名称以及容器ID，创建容器.

创建容器就是启动进程```/proc/self/exe init```，由于/proc/self/exe指向的是自己，所以相当于fork了一个新进程，并且新进程启动的参数是init，相当于运行了```runc init```，```runc init```会根据配置创建好相应的namespace，同时创建一个叫exec.fifo的临时文件，等待其它进程打开这个文件，如果有其它进程打开这个文件，则启动容器。

#### 启动容器
启动容器就是运行```runc start```，它会打开并读一下文件exec.fifo，这样就会触发```runc init```进程启动容器，如果```runc start```读取该文件没有异常，将会删掉文件exec.fifo，所以一般情况下我们看不到文件exec.fifo。

runc创建的容器都会在在/run/runc下有一个目录，里面有一个state.json文件（上面说到的exec.fifo这个临时文件也在这里），包含当前容器详细的配置及状态信息。对于本文中的这个容器，相应的目录为/run/runc/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368。
```
root@dev:/run/runc/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368# ls
state.json

#通过runc state命令，可以查到指定容器的相关信息
root@dev:/run/runc/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368# docker-runc state 967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368
{
  "ociVersion": "1.0.0-rc2-dev",
  "id": "967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368",
  "pid": 8001,
  "status": "running",     #刚创建时这里的状态是created，只有运行runc start之后这里才变成running
  "bundle": "/run/docker/libcontainerd/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368",
  "rootfs": "/var/lib/docker/aufs/mnt/305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281",
  "created": "2017-06-25T04:04:18.830443417Z"
}
```

> 如果我们平时单独的调用runc命令的话，可以将创建容器和启动容器这两步合并成一步，那就是```runc run```，具体启动方法可参考[“走进docker(03)：如何绕过docker运行hello-world？”](https://segmentfault.com/a/1190000009309378)中关于runc运行bundle的介绍。


## 结束语
docker start命令干的活很多，这里只是介绍了大概的流程和涉及的进程和文件，还有一些其他东西并没有涉及到，比如存储插件和网络，后续在专门介绍相关部分的时候再详细介绍。

