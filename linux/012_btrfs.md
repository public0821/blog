#Btrfs文件系统之subvolume与snapshot

对于大部分文件系统来说，在磁盘上创建好文件系统，然后再挂载到系统中去就完事了。但对于Btrfs来说，除了在格式化和挂载的时候指定不同的参数外，还支持很多其他的功能，比如管理多块硬盘，支持LVM和RAID等，具体的可以参考它的[官方文档](https://btrfs.wiki.kernel.org/index.php/Main_Page)或者[Linux下常见文件系统对比](https://segmentfault.com/a/1190000008481493)

[Btrfs](https://btrfs.wiki.kernel.org/index.php/Main_Page)是Linux下大家公认的将会替代ext4的下一代文件系统，功能非常强大。本篇不会介绍Btrfs的原理，也不会介绍Btrfs的所有功能，只是挑了其中的subvolume和snapshot这两个特性来进行介绍

>本篇所有例子都在ubuntu-server-x86_64 16.04下执行通过

##准备环境

先创建一个虚拟的硬盘，然后将它格式化成Btrfs，最后将它挂载到目录/mnt/btrfs
```bash

#为了简单起见，这里只使用一块硬盘来做测试（Btrfs可以管理多块硬盘或分区）。
#新建一个文件，用来虚拟一块硬盘
dev@ubuntu:~$ fallocate -l 512M /tmp/btrfs.img

#在上面创建Btrfs文件系统
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
可以把subvolume理解为一个虚拟的设备，由Btrfs管理，创建好了之后就自动挂载到了Btrfs文件系统的一个目录上，所以我们在文件系统里面看到的subvolume就是一个目录，但它是一个特殊的目录，具有挂载点的一些属性。

新创建的Btrfs文件系统会创建一个路径为“/”的默认subvolume，即root subvolume，其ID为5（别名为0），这是一个ID和目录都预设好的subvolume。
```bash
#这里从mount的参数“subvolid=5,subvol=/”就可以看出来，
#默认的root subvolume的id为5，路径为“/”
dev@debian:/mnt/btrfs$ mount|grep btrfs
/dev/loop1 on /mnt/btrfs type btrfs (rw,relatime,space_cache,subvolid=5,subvol=/)
```

###创建subvolume

这里我们将会利用Btrfs提供的工具创建两个新subvolume和两个文件夹，来看看他们之间的差别
```bash
dev@ubuntu:~$ cd /mnt/btrfs
#btrfs命令是Btrfs提供的应用层工具，可以用来管理Btrfs

#这里依次创建两个subvolume，创建完成之后会自动在当前目录下生成两个目录
dev@ubuntu:/mnt/btrfs$ btrfs subvolume create sub1
Create subvolume './sub1'
dev@ubuntu:/mnt/btrfs$ btrfs subvolume create sub2
Create subvolume './sub2'
#创建两个文件夹
dev@ubuntu:/mnt/btrfs$ mkdir dir1 dir2

#在sub1、sub2和dir1中分别创建一个文件
dev@ubuntu:/mnt/btrfs$ touch dir1/dir1-01.txt
dev@ubuntu:/mnt/btrfs$ touch sub1/sub1-01.txt
dev@ubuntu:/mnt/btrfs$ touch sub2/sub2-01.txt

#最后看看目录结构，是不是看起来sub1和dir1没什么区别？
dev@ubuntu:/mnt/btrfs$ tree
.
├── dir1
│   └── dir1-01.txt
├── dir2
├── sub1
│   └── sub1-01.txt
└── sub2
    └── sub2-01.txt
```
不过由于每个subvolume都是一个单独的虚拟设备，所以无法跨subvolume建立硬链接

```bash
#虽然sub1和sub2属于相同的Btrfs文件系统，并且在一块物理硬盘上
#但由于他们属于不同的subvolume，所以在它们之间建立硬链接失败
dev@ubuntu:/mnt/btrfs$ ln ./sub1/sub1-01.txt ./sub2/
ln: failed to create hard link './sub2/sub1-01.txt' => './sub1/sub1-01.txt': Invalid cross-device link
```

##删除subvolume
subvolume不能用rm来删除，只能通过btrfs命令来删除
```bash
#普通的目录通过rm就可以被删除
dev@ubuntu:/mnt/btrfs$ rm -r dir2

#通过rm命令删除subvolume失败
dev@ubuntu:/mnt/btrfs$ sudo rm -r sub2
rm: cannot remove 'sub2': Operation not permitted

#需要通过btrfs命令才能删除
#删除sub2成功（就算subvolume里面有文件也能被删除）
dev@ubuntu:/mnt/btrfs$ sudo btrfs subvolume del sub2
Delete subvolume (no-commit): '/mnt/btrfs/sub2'
dev@ubuntu:/mnt/btrfs$ tree
.
├── dir1
│   └── dir1-01.txt
└── sub1
    └── sub1-01.txt

```
上面删除的时候可以看到这样的提示： Delete subvolume (no-commit)，表示subvolume被删除了，但没有提交，意思是在内存里面生效了，但磁盘上的内容还没删，意味着如果这个时候系统crash掉，这个subvolume有可能还会回来。btrfs这样做的好处是删除速度很快，不会影响使用，缺点是有可能在后台commit的过程中系统挂掉，导致commit失败。

为了确保subvolume里的数据被真正的从磁盘上移除掉，可以在删除subvolume的时候指定-c参数，这样btrfs命令会等提交完成之后再返回
```bash
dev@ubuntu:/mnt/btrfs$ sudo btrfs subvolume del -c sub2
Delete subvolume (commit): '/mnt/btrfs/sub2'
```

##挂载subvolume
subvolume可以直接通过mount命令挂载，和挂载其它设备没什么区别，具体的挂载参数请参考[文档](https://btrfs.wiki.kernel.org/index.php/Mount_options)

```bash
#创建一个用于挂载点的目录
dev@ubuntu:/mnt/btrfs$ sudo mkdir /mnt/sub1

#先查看待挂载的subvolume的id
dev@debian:/mnt/btrfs$ sudo btrfs subvolume list /mnt/btrfs/
ID 256 gen 9 top level 5 path sub1

#通过-o参数来指定要挂载的subvolume的ID
#通过路径来挂载也是一样的效果：sudo mount -o subvol=/sub1 /tmp/btrfs.img /mnt/sub1/
dev@debian:/mnt/btrfs$ sudo mount -o subvolid=256 /tmp/btrfs.img /mnt/sub1/
dev@debian:/mnt/btrfs$ tree /mnt/sub1/
/mnt/sub1/
└── sub1-01.txt
```

##设置subvolume只读
subvolume可以被设置成只读状态
```bash
#通过btrfs property可以查看和修改subvolume的只读状态
#默认情况下，subvolume的只读属性为false，即允许写
dev@ubuntu:/mnt/btrfs$ btrfs property get -ts ./sub1/
ro=false

#将sub1的只读属性设置成true
dev@ubuntu:/mnt/btrfs$ btrfs property set -ts ./sub1/ ro true
dev@ubuntu:/mnt/btrfs$ btrfs property get -ts ./sub1
ro=true

#写文件失败，提示文件系统只读
dev@ubuntu:/mnt/btrfs$ touch ./sub1/sub1-02.txt
touch: cannot touch './sub1/sub1-02.txt': Read-only file system

#将sub1的状态改回去，以免影响后续测试
dev@ubuntu:/mnt/btrfs$ btrfs property set -ts ./sub1/ ro false
```
##snapshot
可以在subvolume的基础上制作快照，几点需要注意：

* 默认情况下subvolume的快照是可写的
* 快照是特殊的subvolume，具有subvolume的属性。所以快照也可以通过mount挂载，也可以通过btrfs property命令设置只读属性
* 由于快照的本质就是一个subvolume，所以可以在快照上面再做快照

在subvolume上做了快照后，subvolume和快照就会共享所有的文件，只有当文件更新的时候，才会触发COW（copy on write），所以创建快照很快，基本不花时间，并且Btrfs的COW机制很高效，就算多个快照共享一个文件，更新这个文件也和更新一个普通文件差不多的速度。

如果用过git的话，就能很容易理解Btrfs里的快照，可以把subvolume理解为git里面的master分支，而快照就是从master checkout出来的新分支，于是快照跟git里的分支有类似的特点：

 * 创建快照几乎没有开销
 * 可以在快照的基础上再创建快照
 * 当前快照里面的修改不会影响其它快照
 * 快照可以被删除

当然subvolume也可以像git里的master一样被删除。

###创建快照
```bash
#在root subvolume的基础上创建一个快照
#默认情况下快照是可写的，如果要创建只读快照，需要加上-r参数
dev@debian:/mnt/btrfs$ sudo btrfs subvolume snapshot ./ ./snap-root
Create a snapshot of './' in './snap-root'

#创建完成后，可以看到我们已经有了两个subvolume
dev@debian:/mnt/btrfs$ sudo btrfs subvolume list ./
ID 256 gen 11 top level 5 path sub1
ID 257 gen 13 top level 5 path snap-root

#我们可以通过指定-s参数来只列出快照
dev@debian:/mnt/btrfs$ sudo btrfs subvolume list -s ./
ID 257 gen 10 cgen 10 top level 5 otime 2017-03-05 21:46:03 path snap-root

#再来看看快照snap-root中的文件，可以看到有dir1及下面的文件，
#但看不到sub1下的文件，那是因为sub1是一个subvolume，
#在做一个subvolume的快照的时候，不会将它里面的subvolume也做快照
dev@debian:/mnt/btrfs$ tree ./snap-root
./snap-root
├── dir1
│   └── dir1-01.txt
└── sub1

#创建sub1的一个快照，可以看到sub1里面的文件出现在了快照里面
dev@debian:/mnt/btrfs$ sudo btrfs subvolume snapshot ./sub1/ ./snap-sub1
Create a snapshot of './sub1/' in './snap-sub1'
#然后在sub1和它的快照snap-sub1下面各自创建一个文件，
#会发现它们之间不受影响
dev@debian:/mnt/btrfs$ touch snap-sub1/snap-sub1-01.txt
dev@debian:/mnt/btrfs$ touch sub1/sub1-02.txt
dev@debian:/mnt/btrfs$ tree
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
删除快照和删除subvolume是一样的，没有区别
```
dev@debian:/mnt/btrfs$ sudo btrfs subvolume del snap-root
Delete subvolume (no-commit): '/mnt/btrfs/snap-root'
dev@debian:/mnt/btrfs$ sudo btrfs subvolume del snap-sub1
Delete subvolume (no-commit): '/mnt/btrfs/snap-sub1'
dev@debian:/mnt/btrfs$ tree
.
├── dir1
│   └── dir1-01.txt
└── sub1
    ├── sub1-01.txt
    └── sub1-02.txt
```
##default subvolume
可以设置Btrfs分区的默认subvolume，即在挂载磁盘的时候，可以只让分区中的指定subvolume对用户可见。看下面的例子：

```bash
#查看sub1的ID
dev@debian:/mnt/btrfs$ sudo btrfs subvolume list ./
ID 256 gen 14 top level 5 path sub1

#将sub1设置为当前Btrfs文件系统的默认subvolume
dev@debian:/mnt/btrfs$ sudo btrfs subvolume set-default 256 /mnt/btrfs/

#重新将虚拟硬盘挂载到一个新目录
dev@debian:/mnt/btrfs$ sudo mkdir /mnt/btrfs1
dev@debian:/mnt/btrfs$ sudo mount /tmp/btrfs.img /mnt/btrfs1/
#这里将只能看到sub1下的文件
dev@debian:/mnt/btrfs$ tree /mnt/btrfs1
/mnt/btrfs1
├── sub1-01.txt
└── sub1-02.txt

#由于Btrfs原来的默认subvolume是root subvolume，
#其ID是5（也可以通过0来标识），
#所以我们可以通过同样的命令将默认subvolume再改回去
dev@debian:/mnt/btrfs$ sudo btrfs subvolume set-default 0 /mnt/btrfs/
```

####default subvolume有什么用呢？

利用snapshot和default subvolume，可以很方便的实现不同系统版本的切换，比如将系统安装在一个subvolume下面，当要做什么危险操作的时候，先在subvolume的基础上做一个快照A，如果操作成功，那么什么都不用做（或者把A删掉），继续用原来的subvolume，A不被删掉也没关系，多一个快照在那里也不占空间，如果操作失败，那么可以将A设置成default subvolume，并将原来的subvolume删除，这样就相当于系统回滚。

有了这样的功能后，Linux的每次操作都能回滚，养成在修改操作前做snapshot的习惯，就再也不用担心rm误删文件了。

现在有些发行版已经有了类似的功能，如ubuntu，将安装工具（apt）和Btrfs结合，自动的在安装软件之前打一个snapshot，然后安装软件，如果成功，删除新的snapshot，如果失败，修改default subvolume为新的snapshot，删除掉原来的snapshot，这样对系统没有任何影响，并且所有操作对用户是透明的。

随着Btrfs的成熟和普及，相信会改变一些我们使用Linux的习惯。

##结束语
Btrfs的功能太多，需要在使用的过程中去熟悉，本文只是粗略的介绍了一下subvolume和snapshot，关于subvolume的增量备份和磁盘限额都没有涉及到，下次有时间再继续这部分内容。

##参考
* [新一代 Linux 文件系统 btrfs 简介](https://www.ibm.com/developerworks/cn/linux/l-cn-btrfs/)
* [Btrfs Main Page](https://btrfs.wiki.kernel.org/index.php/Main_Page)
* [Btrfs: Subvolumes and snapshots](https://lwn.net/Articles/579009/)