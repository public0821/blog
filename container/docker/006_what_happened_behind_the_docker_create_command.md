# 走进docker(06)：docker create命令背后发生了什么？

有了image之后，就可以开始创建并启动容器了，平时我们都是用```docker run```命令直接创建并运行一个容器，它的背后其实包含独立的两步，一步是```docker create```创建容器，另一步是```docker start```启动容器，本篇将介绍在```docker create```这一步中，docker做了哪些事情。

简单点来说，dockerd在收到客户端的创建容器请求后，做了两件事情，一是准备容器需要的layer，二是检查客户端传过来的参数，并和image配置文件中的参数进行合并，然后存储成容器的配置文件。

## 创建容器
创建一个容器用于示例
```
#创建一个容器，并取名为docker_test,
#-i是为了让容器能接受用户的输入，-t是指定docker为容器创建一个tty，
#因为ubuntu这个镜像默认启动的进程是bash，而bash需要tty，否则会异常退出
dev@dev:~$ docker create -it --name docker_test ubuntu
967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368

dev@dev:~$ docker ps -a
CONTAINER ID   IMAGE    COMMAND       CREATED         STATUS   PORTS   NAMES
967438113fba   ubuntu   "/bin/bash"   6 seconds ago   Created          docker_test
```

## layer的元数据
创建容器时，docker会为每个容器创建两个新的layer，一个是只读的init layer，里面包含docker为容器准备的一些文件，另一个是容器的可写mount layer，以后在容器里面对rootfs的所有增删改操作的结果都会存在这个layer中。

```
# layer的元数据存储在layerdb/mounts/目录下，目录名称就是容器的ID
# 里面包含了三个文件
root@dev:/var/lib/docker/image/aufs/layerdb/mounts/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368# ls
init-id  mount-id  parent

# mount-id文件包含了mount layer的cacheid
root@dev:/var/lib/docker/image/aufs/layerdb/mounts/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368# cat mount-id
305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281

# init-id文件包含了init layer的cacheid
# init layer的cacheid就是在mount layer的cacheid后面加上了一个“-init”
root@dev:/var/lib/docker/image/aufs/layerdb/mounts/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368# cat init-id
305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281-init

# parent里面包含的是image的最上一层layer的chainid
# 表示这个容器的init layer的父layer是image的最顶层layer
root@dev:/var/lib/docker/image/aufs/layerdb/mounts/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368# cat parent
sha256:9840d207a275f956f3e634148044e63dc78df511fd72e22d8cb3dad57dc49bf6

# 可以根据parent的chainid得到它的diffid，
# 这个diffid对应的确实是ubuntu：latest的最顶层layer
# 如何得到image的layer信息请参考上一篇文章： 
# 走进docker(05)：docker在本地如何管理image（镜像）?
root@dev:/var/lib/docker/image# cat ./aufs/layerdb/sha256/9840d207a275f956f3e634148044e63dc78df511fd72e22d8cb3dad57dc49bf6/diff
sha256:d8b353eb3025c49e029567b2a01e517f7f7d32537ee47e64a7eac19fa68a33f3
```

新加的这两层layer比较特殊，只保存在layerdb/mounts下面，在layerdb/sha256目录下没有相关信息，说明docker将container的layer和image的layer的元数据放在了不同的两个目录中

## layer的数据
container layer的数据和image layer的数据的管理方式是一样的，都存在```/var/lib/docker/<storage-driver>```目录下面。

#### layers目录
该目录下包含了每个layer的祖先layer的cacheid
```
root@dev:/var/lib/docker/aufs# cat ./layers/305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281-init
7938f2b32c53a9e0d3974f9579dd9dbb450202e1e11fe514e31556d4ea808c4e
4c10796e21c796a6f3d83eeb3613c566ca9e0fd0a596f4eddf5234b87955b3c8
fd0ba28a44491fd7559c7ffe0597fb1f95b63207a38a3e2680231fb2f6fe92bd
b656bf5f0688069cd90ab230c029fdfeb852afcfd0d1733d087474c86a117da3
1e83d2ea184e08eed978127311cc96498e319426abe2fb5004d4b1454598bd76

#从这里可以看出mount layer在init layer的上面
root@dev:/var/lib/docker/aufs# cat ./layers/305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281
305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281-init
7938f2b32c53a9e0d3974f9579dd9dbb450202e1e11fe514e31556d4ea808c4e
4c10796e21c796a6f3d83eeb3613c566ca9e0fd0a596f4eddf5234b87955b3c8
fd0ba28a44491fd7559c7ffe0597fb1f95b63207a38a3e2680231fb2f6fe92bd
b656bf5f0688069cd90ab230c029fdfeb852afcfd0d1733d087474c86a117da3
1e83d2ea184e08eed978127311cc96498e319426abe2fb5004d4b1454598bd76

#上面的7938f2...就是ubuntu:latest最上一层layer的cacheid
root@dev:~# find /var/lib/docker/image/ -name cache-id|xargs grep 7938f2b32c53a9e0d3974f9579dd9dbb450202e1e11fe514e31556d4ea808c4e
/var/lib/docker/image/aufs/layerdb/sha256/9840d207a275f956f3e634148044e63dc78df511fd72e22d8cb3dad57dc49bf6/cache-id:7938f2b32c53a9e0d3974f9579dd9dbb450202e1e11fe514e31556d4ea808c4e
#9840d2...是ubuntu:latest最上一层layer的chainid
```

#### diff目录
该目录下存放着每个layer所包含的数据
```
#mount layer是新建的供容器写数据layer，
#由于容器还没有运行，所以这里没有任何数据
root@dev:/var/lib/docker/aufs/diff# tree 305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281
305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281

0 directories, 0 files

#init layer包含了docker为每个容器所预先准备的文件
root@dev:/var/lib/docker/aufs/diff# tree 305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281-init/
305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281-init/
├── dev
│   └── console
└── etc
    ├── hostname
    ├── hosts
    ├── mtab -> /proc/mounts
    └── resolv.conf

2 directories, 5 files
```

init layer里面的文件有什么作用呢？从下面的结果可以看出，除了mtab文件是指向/proc/mounts的软连接之外，其他的都是空的普通文件。

```
root@dev:/var/lib/docker/aufs/diff/305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281-init# ls -l dev
total 0
-rwxr-xr-x 1 root root 0 Jun 25 11:25 console
root@dev:/var/lib/docker/aufs/diff/305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281-init# ls -l etc
total 0
-rwxr-xr-x 1 root root  0 Jun 25 11:25 hostname
-rwxr-xr-x 1 root root  0 Jun 25 11:25 hosts
lrwxrwxrwx 1 root root 12 Jun 25 11:25 mtab -> /proc/mounts
-rwxr-xr-x 1 root root  0 Jun 25 11:25 resolv.conf
```

这几个文件都是Linux运行时必须的文件，如果缺少的话会导致某些程序或者库出现异常，所以docker需要为容器准备好这些文件：

* /dev/console: 该文件指向Linux主机的当前控制台，可以通过该文件往控制台打印数据，在容器启动的时候，docker会通过bind mount的方式将主机的/dev/console文件绑定到容器里面的/dev/console上，所以这里需要创建一个空文件，相当于占个坑，作为后续bind mount的目的路径
* hostname，hosts，resolv.conf：对于每个容器来说，容器内的这几个文件内容都有可能不一样，这里和console一样，也是占个坑，等着docker在外面生成这几个文件，然后通过bind mount的方式将这些文件绑定到容器中的这些位置，即这些文件都会被宿主机中的文件覆盖掉。
* /etc/mtab：这个文件在新的Linux发行版中都指向/proc/mounts，里面包含了当前mount namespace中的所有挂载信息，很多程序和库会依赖这个文件。

>注意： 这里mtab指向的路径是固定的，但内容是变化的，取决于你从哪里打开这个文件，当在宿主机上打开时，是宿主机上/proc/mounts的内容，当启动并进入容器后，在容器中打开看到的就是容器中/proc/mounts的内容。

#### mnt目录
里面存放的是经过aufs文件系统mount之后的layer数据，即当前layer和所有的下层layer合并之后的数据，对于aufs文件系统来说，只有在运行容器的时候才会被docker所mount，所以容器没启动的时候，这里看不到任何文件。
```
root@dev:/var/lib/docker/aufs/mnt# tree 305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281
305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281

0 directories, 0 files
root@dev:/var/lib/docker/aufs/mnt# tree 305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281-init/
305226f2e0755956ada28b3baf39b18fa328f1a59fd90e0b759a239773db2281-init/

0 directories, 0 files
```

## 配置文件
docker将用户指定的参数和image配置文件中的部分参数进行合并，然后将合并后生成的容器的配置文件放在/var/lib/docker/containers/下面，目录名称就是容器的ID
```
root@dev:/var/lib/docker/containers/967438113fba0b7a3005bcb6efae6a77055d6be53945f30389888802ea8b0368# tree
.
├── checkpoints
├── config.v2.json
└── hostconfig.json

1 directory, 2 files
```

* config.v2.json: 通用的配置，如容器名称，要执行的命令等
* hostconfig.json： 主机相关的配置，跟操作系统平台有关，如cgroup的配置
* checkpoints： 容器的checkpoint这个功能在当前版本还是experimental状态。

这里不详细介绍配置项的内容，后续介绍某些具体配置项的时候再来看这些文件。

>checkpoints这个功能很强大，可以在当前node做一个checkpoint，然后再到另一个node上继续运行，相当于无缝的将一个正在运行的进程先暂停，然后迁移到另一个node上并继续运行。

## 结束语
docker create命令干的活比较少，主要是准备container的layer和配置文件，配置文件中的项比较多，后续会挑一些常用的项进行专门介绍。

