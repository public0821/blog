# 走进docker(05)：docker在本地如何管理image（镜像）?

docker里面可以通过```docker pull```、```docker build```、```docker commit```、```docker load```、```docker import```等方式得到一个image，得到image之后docker在本地是怎么存储的呢？本篇将以```git pull```为例，简述image的获取和存储方式。

## 镜像相关的配置
docker里面和image有关的目录为/var/lib/docker，里面存放着image的所有信息，可以通过下面这个dockerd的启动参数来修改这个目录的路径。
```
--graph, -g /var/lib/docker Root of the Docker runtime
```

## 镜像的引用方式
在需要引用image的时候，比如docker pull的时候，或者运行容器的时候，都需要指定一个image名称，引用一个镜像有多种方式，下面以ubuntu为例进行说明.

>由于sha256码太长，所以用bcdef...来表示完整的sha256，节约空间

#### docker hub上的官方镜像

* ubuntu： 官方提供的最新ubuntu镜像，对应的完整名称为docker.io/library/ubuntu:latest
* ubuntu:16.04： 官方提供的ubuntu 16.04镜像，对应的完整名称为docker.io/library/ubuntu:16.04
* ubuntu:@sha256:abcdef...： 官方提供的digest码为sha256:abcdef...的ubuntu镜像，对应的完整名称为docker.io/library/ubuntu@sha256:abcdef...

#### docker hub上的非官方（个人）镜像
引用方式和官方镜像一样，唯一不同的是需要在镜像名称前面带上用户前缀，如：

* user1/ubuntu： 由user1提供的最新ubuntu镜像， 对应的完整名称为docker.io/user1/ubuntu:latest

user1/ubuntu:16.04 和 user1/ubuntu:@sha256:abcdef...这两种方式也是和上面一样，等同于docker.io/user1/ubuntu:16.04和docker.io/user1/ubuntu:@sha256:abcdef...

#### 自己搭建的registry里的镜像
引用方式和docker hub一样，唯一不同的是需要在镜像名称最前面带上地址，如：

* localhost:5000/ubuntu： 本地自己搭建的registry（localhost:5000）里面的官方ubuntu的最新镜像，对应的完整名称为localhost:5000/library/ubuntu:latest
* localhost:5000/user1/ubuntu@sha256:a123def...： 本地自己搭建的registry（localhost:5000）里面由用户user1提供的digest为sha256:a123def的ubuntu镜像

其它的几种情况和上面的类似。

####为什么需要镜像的digest？

对于某些image来说，可能在发布之后还会做一些更新，比如安全方面的，这时虽然镜像的内容变了，但镜像的名称和tag没有变，所以会造成前后两次通过同样的名称和tag从服务器得到不同的两个镜像的问题，于是docker引入了镜像的digest的概念，一个镜像的digest就是镜像的manifes文件的sha256码，当镜像的内容发生变化的时候，即镜像的layer发生变化，从而layer的sha256发生变化，而manifest里面包含了每一个layer的sha256，所以manifest的sha256也会发生变化，即镜像的digest发生变化，这样就保证了digest能唯一的对应一个镜像。

## docker pull的大概过程
如果对Image manifest，Image Config和Filesystem Layers等概念不是很了解，请先参考[image(镜像)是什么](https://segmentfault.com/a/1190000009309347)。

取image的大概过程如下：

* docker发送image的名称+tag（或者digest）给registry服务器，服务器根据收到的image的名称+tag（或者digest），找到相应image的manifest，然后将manifest返回给docker
* docker得到manifest后，读取里面image配置文件的digest(sha256)，这个sha256码就是image的ID
* 根据ID在本地找有没有存在同样ID的image，有的话就不用继续下载了
* 如果没有，那么会给registry服务器发请求（里面包含配置文件的sha256和media type），拿到image的配置文件（Image Config）
* 根据配置文件中的diff_ids（每个diffid对应一个layer tar包的sha256，tar包相当于layer的原始格式），在本地找对应的layer是否存在
* 如果layer不存在，则根据manifest里面layer的sha256和media type去服务器拿相应的layer（相当去拿压缩格式的包）。
* 拿到后进行解压，并检查解压后tar包的sha256能否和配置文件（Image Config）中的diff_id对的上，对不上说明有问题，下载失败
* 根据docker所用的后台文件系统类型，解压tar包并放到指定的目录
* 等所有的layer都下载完成后，整个image下载完成，就可以使用了

**注意**： 对于layer来说，config文件中diffid是layer的tar包的sha256，而manifest文件中的digest依赖于media type，比如media type是tar+gzip，那digest就是layer的tar包经过gzip压缩后的内容的sha256，如果media type就是tar的话，diffid和digest就会一样。

>dockerd和registry服务器之间的协议为[Registry HTTP API V2](https://docs.docker.com/registry/spec/api/)。

## image本地存放位置
这里以ubuntu的image为例，展示docker的image存储方式。

先看看debian的image id和digest，然后再分析image数据都存在哪里
```
dev@debian:~$ docker images --digests
REPOSITORY  TAG     DIGEST            MAGE ID       CREATED      SIZE
ubuntu      latest  sha256:ea1d85...  7b9b13f7b9c0  4 weeks ago  118 MB
......
```

>对于本地生成的镜像来说，由于没有上传到registry上去，所以没有digest，因为镜像的manifest由registry生成

### repositories.json
repositories.json中记录了和本地image相关的repository信息，主要是name和image id的对应关系，当image从registry上被pull下来后，就会更新该文件：
```
#这里目录中的aufs为docker后台所采用的存储文件系统名称，
#如果是其他的文件系统的话，名字会是其他的，比如btrfs、overlay2、devicemapper等。
root@debian:~# cat /var/lib/docker/image/aufs/repositories.json|python -m json.tool
{
    "Repositories": {
        "ubuntu": {
            "ubuntu:latest": "sha256:7b9b13f7b9c086adfb6be4d2d264f90f16b4d1d5b3ab9f955caa728c3675c8a2",
            "ubuntu@sha256:ea1d854d38be82f54d39efe2c67000bed1b03348bcc2f3dc094f260855dff368": "sha256:7b9b13f7b9c086adfb6be4d2d264f90f16b4d1d5b3ab9f955caa728c3675c8a2"
        }
        ......
    }
}
```


* ubuntu： repository的名称，前面没有服务器信息的表示这是官方registry(docker hub)里面的repository，里面包含的都是image标识和image ID的对应关系
* ubuntu:latest和debian@sha256:ea1d85...： 他们都指向同一个image（sha256:7b9b13...）

### 配置文件（image config）
docker根据第一步得到的manifest，从registry拿到config文件，然后保存在image/aufs/imagedb/content/sha256/目录下，文件名称就是文件内容的sha256码，即image id：
```
root@debian:~# sha256sum /var/lib/docker/image/aufs/imagedb/content/sha256/7b9b13f7b9c086adfb6be4d2d264f90f16b4d1d5b3ab9f955caa728c3675c8a2
7b9b13f7b9c086adfb6be4d2d264f90f16b4d1d5b3ab9f955caa728c3675c8a2  /var/lib/docker/image/aufs/imagedb/content/sha256/7b9b13f7b9c086adfb6be4d2d264f90f16b4d1d5b3ab9f955caa728c3675c8a2

#这里我们只关注这个image的rootfs，
#从diff_ids里可以看出ubuntu:latest这个image包含了5个layer，
#从上到下依次是从底层到顶层，6a8bf8...是最底层，d8b353...是最顶层
root@debian:~# cat /var/lib/docker/image/aufs/imagedb/content/sha256/7b9b13f7b9c086adfb6be4d2d264f90f16b4d1d5b3ab9f955caa728c3675c8a2|python -m json.tool
......
    "rootfs": {
        "diff_ids": [
            "sha256:6a8bf8c8edbd705f67cdc062eee3911470a38a763258c81c05da1f28d6eec896",
            "sha256:fe9a3f9c4559684b75bba751883fa084d34e418c018d687ddc25c3f23f13f657",
            "sha256:fc9e1e5e38f700997585295bd65a47e58f3da7b2f0e6a971e14a6104f199de1f",
            "sha256:f2e85bc0b7b17ab33f9060d2f24824defe600e189b05a895d60b2e8a6a7bd0d7",
            "sha256:d8b353eb3025c49e029567b2a01e517f7f7d32537ee47e64a7eac19fa68a33f3"
        ],
        "type": "layers"
    }

......
```

### layer的diff_id和digest的对应关系
layer的diff_id存在image的配置文件中，而layer的digest存在image的manifest中，他们的对应关系被存储在了image/aufs/distribution目录下：
```
root@debian:~# tree -d /var/lib/docker/image/aufs/distribution/
/var/lib/docker/image/aufs/distribution/
├── diffid-by-digest
│   └── sha256
└── v2metadata-by-diffid
    └── sha256
```

* diffid-by-digest： 存放digest到diffid的对应关系
* v2metadata-by-diffid： 存放diffid到digest的对应关系

```
#这里以最底层layer(6a8bf8...)为例，查看其digest信息
root@debian:~# cat /var/lib/docker/image/aufs/distribution/v2metadata-by-diffid/sha256/6a8bf8c8edbd705f67cdc062eee3911470a38a763258c81c05da1f28d6eec896|python -m json.tool
[
    {
        "Digest": "sha256:bd97b43c27e332fc4e00edf827bbc26369ad375187ce6eee91c616ad275884b1",
        "HMAC": "",
        "SourceRepository": "docker.io/library/ubuntu"
    }
]

#根据digest得到diffid
root@debian:~# cat /var/lib/docker/image/aufs/distribution/diffid-by-digest/sha256/bd97b43c27e332fc4e00edf827bbc26369ad375187ce6eee91c616ad275884b1
sha256:6a8bf8c8edbd705f67cdc062eee3911470a38a763258c81c05da1f28d6eec896

```

### layer的元数据
layer的属性信息都放在了image/aufs/layerdb目录下，目录名称是layer的chainid，由于最底层的layer的chainid和diffid相同，所以这里我们用第二层（fe9a3f...）作为示例：

>计算chainid时，用到了所有祖先layer的信息，从而能保证根据chainid得到的rootfs是唯一的。比如我在debian和ubuntu的image基础上都添加了一个同样的文件，那么commit之后新增加的这两个layer具有相同的内容，相同的diffid，但由于他们的父layer不一样，所以他们的chainid会不一样，从而根据chainid能找到唯一的rootfs。计算chainid的方法请参考[image spec](https://github.com/opencontainers/image-spec/blob/master/config.md)

```
#计算chainid
#这里fe9a3f...是第二层的diffid，而6a8bf8...是fe9a3f...父层的chainid，
#由于6a8bf8...是最底层，它没有父层，所以6a8bf8...的chainid就是6a8bf8...
dev@debian:~$ echo -n "sha256:6a8bf8c8edbd705f67cdc062eee3911470a38a763258c81c05da1f28d6eec896 sha256:fe9a3f9c4559684b75bba751883fa084d34e418c018d687ddc25c3f23f13f657"|sha256sum -
67b5e1df1b671d4751794767b20f6973d77e91528ab475ee1118ce8dab796193  -

#根据chainid来看看相应目录的内容
root@debian:/var/lib/docker/image/aufs/layerdb/sha256/67b5e1df1b671d4751794767b20f6973d77e91528ab475ee1118ce8dab796193# ls
cache-id  diff  parent  size  tar-split.json.gz
#每个layer都有这样一个对应的文件夹

#cache-id是docker下载layer的时候在本地生成的一个随机uuid，
#指向真正存放layer文件的地方
root@debian:/var/lib/docker/image/aufs/layerdb/sha256/67b5e1df1b671d4751794767b20f6973d77e91528ab475ee1118ce8dab796193# cat cache-id
b656bf5f0688069cd90ab230c029fdfeb852afcfd0d1733d087474c86a117da3

#diff文件存放layer的diffid
root@debian:/var/lib/docker/image/aufs/layerdb/sha256/67b5e1df1b671d4751794767b20f6973d77e91528ab475ee1118ce8dab796193# cat diff
sha256:fe9a3f9c4559684b75bba751883fa084d34e418c018d687ddc25c3f23f13f657

#parent文件存放当前layer的父layer的diffid，
#注意：对于最底层的layer来说，由于没有父layer，所以没有这个文件
root@debian:/var/lib/docker/image/aufs/layerdb/sha256/67b5e1df1b671d4751794767b20f6973d77e91528ab475ee1118ce8dab796193# cat parent
sha256:6a8bf8c8edbd705f67cdc062eee3911470a38a763258c81c05da1f28d6eec896

#当前layer的大小，单位是字节
root@debian:/var/lib/docker/image/aufs/layerdb/sha256/67b5e1df1b671d4751794767b20f6973d77e91528ab475ee1118ce8dab796193# cat size
745

#tar-split.json.gz，layer压缩包的split文件，通过这个文件可以还原layer的tar包，
#在docker save导出image的时候会用到
#详情可参考https://github.com/vbatts/tar-split
root@debian:/var/lib/docker/image/aufs/layerdb/sha256/67b5e1df1b671d4751794767b20f6973d77e91528ab475ee1118ce8dab796193# ls -l tar-split.json.gz
-rw-r--r-- 1 root root 1429 May 28 22:57 tar-split.json.gz
```

## layer数据
docker根据后台所采用的文件系统不同，在/var/lib/docker目录下创建了不同的子目录，对于debian来说，默认文件系统是aufs，所以所有layer的文件都放在了/var/lib/docker/aufs目录下。

```
root@dev:~# tree -d -L 1 /var/lib/docker/aufs
/var/lib/docker/aufs
├── diff
├── layers
└── mnt
```

该目录下有三个子目录，layers里面存放的layer的父子关系，diff目录存放的是每个layer的原始数据，mnt存储的是layer和祖先layer叠加起来的数据。

> 注意：由于docker所采用的文件系统不同，```/var/lib/docker/<storage-driver>```目录下的目录结构及组织方式也会不一样，要具体文件系统具体分析，本文只介绍aufs这种情况。
> 关于aufs和btrfs的相关特性可以参考[Linux文件系统之aufs](https://segmentfault.com/a/1190000008489207)和[Btrfs文件系统之subvolume与snapshot](https://segmentfault.com/a/1190000008605135)

还是以刚才的第二层layer（fe9a3f...）为例，看看实际的数据：

```
#从上面layerdb中，我们已经找到了第二层layer对应的cache id为b656bf...
#先看看layers目录下的这个文件,里面存放的是当前layer的祖先layer的cacheid
root@dev:~# cat /var/lib/docker/aufs/layers/b656bf5f0688069cd90ab230c029fdfeb852afcfd0d1733d087474c86a117da3
1e83d2ea184e08eed978127311cc96498e319426abe2fb5004d4b1454598bd76

#再来看看diff目录下的内容，看这一层包含了哪些文件，从下面的输出可以看出，这一层包含的文件很少
root@dev:~# tree /var/lib/docker/aufs/diff/b656bf5f0688069cd90ab230c029fdfeb852afcfd0d1733d087474c86a117da3/
/var/lib/docker/aufs/diff/b656bf5f0688069cd90ab230c029fdfeb852afcfd0d1733d087474c86a117da3/
├── etc
│   ├── apt
│   │   └── apt.conf.d
│   │       ├── docker-autoremove-suggests
│   │       ├── docker-clean
│   │       ├── docker-gzip-indexes
│   │       └── docker-no-languages
│   └── dpkg
│       └── dpkg.cfg.d
│           └── docker-apt-speedup
├── sbin
│   └── initctl
├── usr
│   └── sbin
│       └── policy-rc.d
└── var
    └── lib
        └── dpkg
            ├── diversions
            └── diversions-old

#再来看看这一层和上一层合并后的文件目录
root@dev:~# tree /var/lib/docker/aufs/mnt/b656bf5f0688069cd90ab230c029fdfeb852afcfd0d1733d087474c86a117da3/
/var/lib/docker/aufs/mnt/b656bf5f0688069cd90ab230c029fdfeb852afcfd0d1733d087474c86a117da3/

0 directories, 0 files
#为什么是空的呢？这和aufs文件系统有关，
#它只有在运行容器的时候，才会将多层合并起来，提供一个统一的视图，
#所以这里看不到这两层合并之后的效果。
```

## manifest文件去哪了？
从前面介绍docker pull的过程中得知，docker是先得到manifest，然后根据manifest得到config文件和layer。

前面已经介绍了config文件和layer的存储位置，但唯独不见manifest，去哪了呢？

manifest里面包含的内容就是对config和layer的sha256 + media type描述，目的就是为了下载config和layer，等image下载完成后，manifest的使命就完成了，里面的信息对于image的本地管理来说没什么用，所以docker在本地没有单独的存储一份manifest文件与之对应。

## 结束语
本篇介绍了image在本地的存储方式，包括了```/var/lib/docker/image```和```/var/lib/docker/aufs```这两个目录，但/var/lib/docker/image下面有两个目录没有涉及：

* /var/lib/docker/image/aufs/imagedb/metadata：里面存放的是本地image的一些信息，从服务器上pull下来的image不会存数据到这个目录，下次有机会再补充这部分内容。
* /var/lib/docker/image/aufs/layerdb/mounts: 创建container时，docker会为每个container在image的基础上创建一层新的layer，里面主要包含/etc/hosts、/etc/hostname、/etc/resolv.conf等文件，创建的这一层layer信息就放在这里，后续在介绍容器的时候，会专门介绍这个目录的内容。

## 参考

* [docker源代码](https://github.com/moby/moby)
