#btrfs文件系统

aufs的全称是advanced multi-layered unification filesystem，主要功能是把多个文件夹的内容合并到一起，提供一个统一的视图，一看后面的例子就明白了。

据说由于aufs代码的可维护性不好（代码可读性和注释不太好），所以一直没有被合并到Linux内核的主线中去，不过有些发行版的kernel里面维护的有该文件系统，比如在ubuntu 16.04的内核代码中，就有该文件系统。

>本篇所有例子都在ubuntu-server-x86_64 16.04下执行通过

```
dev@ubuntu:~$ fallocate -l 512M /tmp/btrfs.img
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

dev@ubuntu:~$ mkdir /mnt/btrfs
dev@ubuntu:~$ sudo mkdir /mnt/btrfs
dev@ubuntu:~$ sudo mount /tmp/btrfs.img /mnt/btrfs
dev@ubuntu:/mnt/btrfs$ sudo chmod 777 /mnt/btrfs
dev@ubuntu:/mnt/btrfs$ btrfs subvolume create sub1
Create subvolume './sub1'
dev@ubuntu:/mnt/btrfs$ btrfs subvolume create sub2
Create subvolume './sub2'

dev@ubuntu:/mnt/btrfs$ touch dir1/dir1-01.txt
dev@ubuntu:/mnt/btrfs$ touch sub1/sub1-01.txt
dev@ubuntu:/mnt/btrfs$ tree
.
├── dir1
│   └── dir1-01.txt
├── dir2
├── sub1
│   └── sub1-01.txt
└── sub2

dev@ubuntu:/mnt/btrfs$ rm -r dir2
dev@ubuntu:/mnt/btrfs$ rm -r sub2
rm: cannot remove 'sub2': Operation not permitted
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

##snapshot

##参考
[新一代 Linux 文件系统 btrfs 简介](https://www.ibm.com/developerworks/cn/linux/l-cn-btrfs/)
[Btrfs Main Page](https://btrfs.wiki.kernel.org/index.php/Main_Page)
[Btrfs Quota](https://btrfs.wiki.kernel.org/index.php/Quota_support)
[Btrfs: Subvolumes and snapshots](https://lwn.net/Articles/579009/)