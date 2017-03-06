#btrfs文件系统

对于大部分文件系统来说，在磁盘上创建好文件系统，然后再挂载到系统中去就完事了。但对于btrfs来说，除了在格式化和挂载的时候指定不同的参数外，还支持很多其他的功能，比如管理多块硬盘，支持lvm和raid等，具体的可以参考它的[官方文档](https://btrfs.wiki.kernel.org/index.php/Main_Page)或者[Linux下常见文件系统对比](https://segmentfault.com/a/1190000008481493)

[btrfs](https://btrfs.wiki.kernel.org/index.php/Main_Page)是Linux下大家公认的将会替代ext4的下一代文件系统，功能非常强大。本篇不会介绍btrfs的原理，也不会介绍btrfs的所有功能，只是挑了其中的subvolume和snapshot这两个特性来进行介绍

>本篇所有例子都在ubuntu-server-x86_64 16.04下执行通过

##准备环境

先创建一个虚拟的硬盘，然后将它格式化成btrfs，最后将它挂载到目录/mnt/btrfs
```bash

#为了简单起见，这里只使用一块硬盘来做测试（btrfs可以管理多块硬盘或分区）。
#新建一个文件，用来虚拟一块硬盘
dev@ubuntu:~$ fallocate -l 512M /tmp/btrfs.img

#在上面创建btrfs文件系统
dev@ubuntu:~$ mkfs.btrfs /tmp/btrfs.img
btrfs-progs v4.4
See http://btrfs.wiki.kernel.org for more information.

Label:              (null)
UUID:               fd5efcd3-adc2-406b-a684-e6c87dde99a1
Node size:          16384
Sector size:        4096
Filesystem size:    512.00MiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP              40.00MiB
  System:           DUP              12.00MiB
SSD detected:       no
Incompat features:  extref, skinny-metadata
Number of devices:  1
Devices:
   ID        SIZE  PATH
    1   512.00MiB  /tmp/btrfs.img

#创建文件夹并挂载
dev@ubuntu:~$ sudo mkdir /mnt/btrfs
dev@ubuntu:~$ sudo mount /tmp/btrfs.img /mnt/btrfs

#修改权限，这样后面的部分操作就不再需要sudo
dev@ubuntu:~$ sudo chmod 777 /mnt/btrfs
```

##subvolume
可以把subvolume理解为一个虚拟的设备，由btrfs管理，创建好了之后就自动挂载到了btrfs文件系统的一个目录上，所以我们在文件系统里面看到的subvolume就是一个目录，但它是一个特殊的目录，具有挂载点的一些属性。

新创建的btrfs文件系统会创建一个路径为“/”的默认subvolume，即root subvolume，其ID为5（别名为0），这是一个ID和目录都预设好的subvolume。
```bash
#这里从mount的参数“subvolid=5,subvol=/”就可以看出来，
#默认的root subvolume的id为5，路径为“/”
dev@dev:/mnt/btrfs$ mount|grep btrfs
/dev/loop1 on /mnt/btrfs type btrfs (rw,relatime,space_cache,subvolid=5,subvol=/)
```

###创建subvolume

这里我们将会利用btrfs提供的工具创建两个新subvolume和两个文件夹，来看看他们之间的差别
```bash
dev@ubuntu:~$ cd /mnt/btrfs
#btrfs命令是btrfs提供的应用层工具，可以用来管理btrfs

#这里依次创建两个subvolume，创建完成之后会自动在当前目录下生成两个目录
dev@ubuntu:/mnt/btrfs$ btrfs subvolume create sub1
Create subvolume './sub1'
dev@ubuntu:/mnt/btrfs$ btrfs subvolume create sub2
Create subvolume './sub2'
#创建两个文件夹
dev@ubuntu:/mnt/btrfs$ mkdir dir1 dir2

#在sub1和dir1中分别创建一个文件
dev@ubuntu:/mnt/btrfs$ touch dir1/dir1-01.txt
dev@ubuntu:/mnt/btrfs$ touch sub1/sub1-01.txt

#最后看看目录结构，是不是看起来sub1和dir1没什么区别？
dev@ubuntu:/mnt/btrfs$ tree
.
├── dir1
│   └── dir1-01.txt
├── dir2
├── sub1
│   └── sub1-01.txt
└── sub2

```
不过由于每个subvolume都是一个单独的虚拟设备，所以无法跨subvolume建立硬链接

```bash
#虽然/mnt/btrfs/sub1和/mnt/btrfs属于相同的btrfs文件系统，并且在一块物理硬盘上
#但由于他们属于不同的subvolume（/mnt/btrfs属于默认的subvolume），
#所以在它们之间建立硬链接失败
dev@dev:/mnt/btrfs$ ln ./sub1/sub1-01.txt ./
ln: failed to create hard link './sub1-01.txt' => './sub1/sub1-01.txt': Invalid cross-device link
```

##删除subvolume
subvolume不能用rm来删除，只能通过btrfs命令来删除，并且只有当subvolume下没有文件时才能被删除
```bash
#普通的目录通过rm就可以被删除
dev@ubuntu:/mnt/btrfs$ rm -r dir2

#通过rm命令删除subvolume失败
dev@ubuntu:/mnt/btrfs$ rm -r sub2
rm: cannot remove 'sub2': Operation not permitted

#需要通过btrfs命令才能删除
#由于sub1下有文件，所以不能被删除
dev@dev:/mnt/btrfs$ sudo btrfs subvolume del sub1
Delete subvolume (no-commit): '/mnt/btrfs/sub1'
ERROR: cannot delete '/mnt/btrfs/sub1': Operation not permitted

#删除sub2成功
dev@ubuntu:/mnt/btrfs$ sudo btrfs subvolume del sub2
Delete subvolume (no-commit): '/mnt/btrfs/sub2'
dev@ubuntu:/mnt/btrfs$ tree
.
├── dir1
│   └── dir1-01.txt
└── sub1
    └── sub1-01.txt

```
##挂载subvolume
subvolume可以直接通过mount命令挂载，和挂载其它设备没什么区别，具体的挂载参数请参考[文档](https://btrfs.wiki.kernel.org/index.php/Mount_options)

```bash
#创建一个用于挂载点的目录
dev@ubuntu:/mnt/btrfs$ sudo mkdir /mnt/sub1

#先查看待挂载的subvolume的id
dev@dev:/mnt/btrfs$ sudo btrfs subvolume list /mnt/btrfs/
ID 256 gen 9 top level 5 path sub1

#通过-o参数来指定要挂载的subvolume的ID
#通过路径来挂载也是一样的效果：sudo mount -o subvol=/sub1 /tmp/btrfs.img /mnt/sub1/
dev@dev:/mnt/btrfs$ sudo mount -o subvolid=256 /tmp/btrfs.img /mnt/sub1/
dev@dev:/mnt/btrfs$ tree /mnt/sub1/
/mnt/sub1/
└── sub1-01.txt
```

##挂载subvolume
subvolume可以设置成只读
```bash
#创建一个用于挂载点的目录
dev@ubuntu:/mnt/btrfs$ sudo mkdir /mnt/sub1

#先查看待挂载的subvolume的id
dev@dev:/mnt/btrfs$ sudo btrfs subvolume list /mnt/btrfs/
ID 256 gen 9 top level 5 path sub1

#通过-o参数来指定要挂载的subvolume的ID
#通过路径来挂载也是一样的效果：sudo mount -o subvol=/sub1 /tmp/btrfs.img /mnt/sub1/
dev@dev:/mnt/btrfs$ sudo mount -o subvolid=256 /tmp/btrfs.img /mnt/sub1/
dev@dev:/mnt/btrfs$ tree /mnt/sub1/
/mnt/sub1/
└── sub1-01.txt
```

##snapshot
可以在subvolume的基础上制作快照，几点需要注意：

* 跟我们通常看到的虚拟机的只读快照不同，默认情况下subvolume的快照是可写的
* 快照是特殊的subvolume，具有subvolume的属性
* 由于快照的本质就是一个subvolume，所以可以在快照上面再做快照

###创建快照
```bash
#在默认subvolume的基础上创建一个快照
#默认情况下快照是可写的，如果要创建只读快照，需要加上-r参数
dev@dev:/mnt/btrfs$ sudo btrfs subvolume snapshot ./ ./snap-root
Create a snapshot of './' in './snap-root'

#创建完成后，可以看到我们已经有了两个subvolume
dev@dev:/mnt/btrfs$ sudo btrfs subvolume list ./
ID 256 gen 11 top level 5 path sub1
ID 257 gen 13 top level 5 path snap-root

#我们可以通过指定-s参数来只列出快照
dev@dev:/mnt/btrfs$ sudo btrfs subvolume list -s ./
ID 257 gen 10 cgen 10 top level 5 otime 2017-03-05 21:46:03 path snap-root

#再来看看快照snap-root中的文件，可以看到有dir1及下面的文件，
#但看不到sub1下的文件，那是因为sub1是一个subvolume，
#在做一个subvolume的快照的时候，不会将它里面的subvolume也做快照
dev@dev:/mnt/btrfs$ tree ./snap-root
./snap-root
├── dir1
│   └── dir1-01.txt
└── sub1

#创建sub1的一个快照，可以看到sub1里面的文件出现在了快照里面
dev@dev:/mnt/btrfs$ sudo btrfs subvolume snapshot ./sub1/ ./snap-sub1
Create a snapshot of './sub1/' in './snap-sub1'
dev@dev:/mnt/btrfs$ tree
.
├── dir1
│   └── dir1-01.txt
├── snap-root
│   ├── dir1
│   │   └── dir1-01.txt
│   └── sub1
├── snap-sub1
│   └── sub1-01.txt
└── sub1
    └── sub1-01.txt

#在sub1和它的快照snap-sub1下面各自创建一个文件，
#会发现它们之间不受影响
dev@dev:/mnt/btrfs$ touch snap-sub1/snap-sub1-01.txt
dev@dev:/mnt/btrfs$ touch sub1/sub1-02.txt
dev@dev:/mnt/btrfs$ tree
.
├── dir1
│   └── dir1-01.txt
├── snap-root
│   ├── dir1
│   │   └── dir1-01.txt
│   └── sub1
├── snap-sub1
│   ├── snap-sub1-01.txt
│   └── sub1-01.txt
└── sub1
    ├── sub1-01.txt
    └── sub1-02.txt

```

###删除快照
删除快照和删除subvolume是一样的，不过删除快照不要求快照下为空
```
dev@dev:/mnt/btrfs$ sudo btrfs subvolume del snap-root
Delete subvolume (no-commit): '/mnt/btrfs/snap-root'
dev@dev:/mnt/btrfs$ sudo btrfs subvolume del snap-sub1
Delete subvolume (no-commit): '/mnt/btrfs/snap-sub1'
dev@dev:/mnt/btrfs$ tree
.
├── dir1
│   └── dir1-01.txt
└── sub1
    ├── sub1-01.txt
    └── sub1-02.txt
```

##default subvolume
可以设置btrfs分区的默认subvolume，即在挂载磁盘的时候，可以只让分区中的指定subvolume对用户可见。看下面的例子：

```bash
#查看sub1的ID
dev@dev:/mnt/btrfs$ sudo btrfs subvolume list ./
ID 256 gen 14 top level 5 path sub1

#指定sub1为当前btrfs文件系统的默认subvolume
dev@dev:/mnt/btrfs$ sudo btrfs subvolume set-default 256 /mnt/btrfs/

#重新将虚拟硬盘挂载到一个新目录
dev@dev:/mnt/btrfs$ sudo mkdir /mnt/btrfs1
dev@dev:/mnt/btrfs$ sudo mount /tmp/btrfs.img /mnt/btrfs1/

#可以在挂载点看到原来sub1里面的内容
dev@dev:/mnt/btrfs$ tree /mnt/btrfs1
/mnt/btrfs1
├── sub1-01.txt
└── sub1-02.txt

#由于btrfs原来的默认subvolume ID是0，所以我们可以通过同样的命令将默认subvolume再改回去
dev@dev:/mnt/btrfs$ sudo btrfs subvolume set-default 0 /mnt/btrfs/
```

比如我有一块btrfs的盘，里面有一个default subvolume，名字叫sub1，sub1里面存放的ubuntu的根目录，现在我想装一个软件或者做一系列的操作，但又怕它不稳定，会对系统造成问题，那我可以先在sub1的基础上创建一个快照snap1，然后切换系统根目录到snap1里面，开始安装软件并测试，如果测试不行，可以再切回去，如果测试的可以，那么就可以将snap1设置成default subvolume，这样下次机器重启的时候就会使用新的根目录。

利用snapshot和default subvolume，可以很方便的实现不同系统版本的切换，这样可以让Linux的每次操作都很放心，因为随时可以回滚，再也不用担心rm误删文件了。现在各个发行版已经有了类似的功能，将安装工具和btrfs结合，自动的在安装之前打一个snapshot，然后切换到snapshot里面并安装软件，安装好之后修改default subvolume为新的snapshot，所有操作对用户是透明的，但如果对安装的软件不满意，可以随时回滚。

随着btrfs的成熟，相信会改变我们使用Linux的一些习惯。


##结束语
btrfs的功能太多，需要在使用的过程中去熟悉，本文只是粗略的介绍了subvolume和snapshot，由于本人对btrfs使用的不多，所以有些细节可能没覆盖到，请见谅。

##参考
[新一代 Linux 文件系统 btrfs 简介](https://www.ibm.com/developerworks/cn/linux/l-cn-btrfs/)
[Btrfs Main Page](https://btrfs.wiki.kernel.org/index.php/Main_Page)
[Btrfs Quota](https://btrfs.wiki.kernel.org/index.php/Quota_support)
[Btrfs: Subvolumes and snapshots](https://lwn.net/Articles/579009/)