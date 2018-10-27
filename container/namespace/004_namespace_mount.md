# Linux Namespace系列（04）：mount namespaces (CLONE_NEWNS)

Mount namespace用来隔离文件系统的挂载点, 使得不同的mount namespace拥有自己独立的挂载点信息，不同的namespace之间不会相互影响，这对于构建用户或者容器自己的文件系统目录非常有用。

当前进程所在mount namespace里的所有挂载信息可以在/proc/[pid]/mounts、/proc/[pid]/mountinfo和/proc/[pid]/mountstats里面找到。

Mount namespaces是第一个被加入Linux的namespace，由于当时没想到还会引入其它的namespace，所以取名为CLONE_NEWNS，而没有叫CLONE_NEWMOUNT。

每个mount namespace都拥有一份自己的挂载点列表，当用clone或者unshare函数创建新的mount namespace时，新创建的namespace将拷贝一份老namespace里的挂载点列表，但从这之后，他们就没有关系了，通过mount和umount增加和删除各自namespace里面的挂载点都不会相互影响。

>本篇所有例子都在ubuntu-server-x86_64 16.04下执行通过

## 演示

```bash
#--------------------------第一个shell窗口----------------------
#先准备两个iso文件，用于后面的mount测试
dev@ubuntu:~$ mkdir iso
dev@ubuntu:~$ cd iso/
dev@ubuntu:~/iso$ mkdir -p iso01/subdir01
dev@ubuntu:~/iso$ mkdir -p iso02/subdir02
dev@ubuntu:~/iso$ mkisofs -o ./001.iso ./iso01
dev@ubuntu:~/iso$ mkisofs -o ./002.iso ./iso02
dev@ubuntu:~/iso$ ls
001.iso  002.iso  iso01  iso02
#准备目录用于mount
dev@ubuntu:~/iso$ sudo mkdir /mnt/iso1 /mnt/iso2

#查看当前所在的mount namespace
dev@ubuntu:~/iso$ readlink /proc/$$/ns/mnt
mnt:[4026531840]

#mount 001.iso 到 /mnt/iso1/
dev@ubuntu:~/iso$ sudo mount ./001.iso /mnt/iso1/
mount: /dev/loop1 is write-protected, mounting read-only

#mount成功
dev@ubuntu:~/iso$ mount |grep /001.iso
/home/dev/iso/001.iso on /mnt/iso1 type iso9660 (ro,relatime)

#创建并进入新的mount和uts namespace
dev@ubuntu:~/iso$ sudo unshare --mount --uts /bin/bash
#更改hostname并重新加载bash
root@ubuntu:~/iso# hostname container001
root@ubuntu:~/iso# exec bash
root@container001:~/iso#

#查看新的mount namespace
root@container001:~/iso# readlink /proc/$$/ns/mnt
mnt:[4026532455]

#老namespace里的挂载点的信息已经拷贝到新的namespace里面来了
root@container001:~/iso# mount |grep /001.iso
/home/dev/iso/001.iso on /mnt/iso1 type iso9660 (ro,relatime)

#在新namespace中mount 002.iso
root@container001:~/iso# mount ./002.iso /mnt/iso2/
mount: /dev/loop0 is write-protected, mounting read-only
root@container001:~/iso# mount |grep iso
/home/dev/iso/001.iso on /mnt/iso1 type iso9660 (ro,relatime)
/home/dev/iso/002.iso on /mnt/iso2 type iso9660 (ro,relatime)

#umount 001.iso
root@container001:~/iso# umount /mnt/iso1
root@container001:~/iso# mount |grep iso
/home/dev/iso/002.iso on /mnt/iso2 type iso9660 (ro,relatime)

#/mnt/iso1目录变为空
root@container001:~/iso# ls /mnt/iso1
root@container001:~/iso#

#--------------------------第二个shell窗口----------------------
#打开新的shell窗口，老namespace中001.iso的挂载信息还在
#而在新namespace里面mount的002.iso这里看不到
dev@ubuntu:~$ mount |grep iso
/home/dev/iso/001.iso on /mnt/iso1 type iso9660 (ro,relatime)
#iso1目录里面也有内容
dev@ubuntu:~$ ls /mnt/iso1
subdir01
#说明两个namespace中的mount信息是隔离的
```

## Shared subtrees
在某些情况下，比如系统添加了一个新的硬盘，这个时候如果mount namespace是完全隔离的，想要在各个namespace里面用这个硬盘，就需要在每个namespace里面手动mount这个硬盘，这个是很麻烦的，这时[Shared subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt)就可以帮助我们解决这个问题。

关于Shared subtrees的详细介绍请参考[Linux mount (第二部分)](https://segmentfault.com/a/1190000006899213)，里面有他的详细介绍以及bind nount的例子。

### 演示
对Shared subtrees而言，mount namespace和bind mount的情况差不多，这里就简单演示一下shared和private两种类型
```bash
#--------------------------第一个shell窗口----------------------
#准备4个虚拟的disk，并在上面创建ext2文件系统，用于后续的mount测试
dev@ubuntu:~/iso$ cd && mkdir disks && cd disks
dev@ubuntu:~/disks$ dd if=/dev/zero bs=1M count=32 of=./disk1.img
dev@ubuntu:~/disks$ dd if=/dev/zero bs=1M count=32 of=./disk2.img
dev@ubuntu:~/disks$ dd if=/dev/zero bs=1M count=32 of=./disk3.img
dev@ubuntu:~/disks$ dd if=/dev/zero bs=1M count=32 of=./disk4.img
dev@ubuntu:~/disks$ mkfs.ext2 ./disk1.img
dev@ubuntu:~/disks$ mkfs.ext2 ./disk2.img
dev@ubuntu:~/disks$ mkfs.ext2 ./disk3.img
dev@ubuntu:~/disks$ mkfs.ext2 ./disk4.img
#准备两个目录用于挂载上面创建的disk
dev@ubuntu:~/disks$ mkdir disk1 disk2
dev@ubuntu:~/disks$ ls
disk1  disk1.img  disk2  disk2.img  disk3.img  disk4.img


#显式的分别以shared和private方式挂载disk1和disk2
dev@ubuntu:~/disks$ sudo mount --make-shared ./disk1.img ./disk1
dev@ubuntu:~/disks$ sudo mount --make-private ./disk2.img ./disk2
dev@ubuntu:~/disks$ cat /proc/self/mountinfo |grep disk| sed 's/ - .*//'
164 24 7:1 / /home/dev/disks/disk1 rw,relatime shared:105
173 24 7:2 / /home/dev/disks/disk2 rw,relatime

#查看mount namespace编号
dev@ubuntu:~/disks$ readlink /proc/$$/ns/mnt
mnt:[4026531840]

#--------------------------第二个shell窗口----------------------
#重新打开一个新的shell窗口
dev@ubuntu:~$ cd ./disks
#创建新的mount namespace
#默认情况下，unshare会将新namespace里面的所有挂载点的类型设置成private，
#所以这里用到了参数--propagation unchanged，
#让新namespace里的挂载点的类型和老namespace里保持一致。
#--propagation参数还支持private|shared|slave类型，
#和mount命令的那些--make-private参数一样，
#他们的背后都是通过调用mount（...）函数传入不同的参数实现的
dev@ubuntu:~/disks$ sudo unshare --mount --uts --propagation unchanged /bin/bash
root@ubuntu:~/disks# hostname container001
root@ubuntu:~/disks# exec bash
root@container001:~/disks# 

#确认已经是在新的mount namespace里面了
root@container001:~/disks# readlink /proc/$$/ns/mnt
mnt:[4026532463]

#由于前面指定了--propagation unchanged，
#所以新namespace里面的/home/dev/disks/disk1也是shared，
#且和老namespace里面的/home/dev/disks/disk1属于同一个peer group 105
#因为在不同的namespace里面，所以这里挂载点的ID和原来namespace里的不一样了
root@container001:~/disks# cat /proc/self/mountinfo |grep disk| sed 's/ - .*//'
221 177 7:1 / /home/dev/disks/disk1 rw,relatime shared:105
222 177 7:2 / /home/dev/disks/disk2 rw,relatime

#分别在disk1和disk2目录下创建disk3和disk4，然后挂载disk3，disk4到这两个目录
root@container001:~/disks# mkdir ./disk1/disk3 ./disk2/disk4
root@container001:~/disks# mount ./disk3.img ./disk1/disk3/
root@container001:~/disks# mount ./disk4.img ./disk2/disk4/
root@container001:~/disks# cat /proc/self/mountinfo |grep disk| sed 's/ - .*//'
221 177 7:1 / /home/dev/disks/disk1 rw,relatime shared:105
222 177 7:2 / /home/dev/disks/disk2 rw,relatime
223 221 7:3 / /home/dev/disks/disk1/disk3 rw,relatime shared:107
227 222 7:4 / /home/dev/disks/disk2/disk4 rw,relatime

#--------------------------第一个shell窗口----------------------
#回到第一个shell窗口

#可以看出由于/home/dev/disks/disk1是shared，且两个namespace里的这个挂载点都属于peer group 105，
#所以在新namespace里面挂载的disk3，在老的namespace里面也看的到
#但是看不到disk4的挂载信息，那是因为/home/dev/disks/disk2是private的
dev@ubuntu:~/disks$ cat /proc/self/mountinfo |grep disk| sed 's/ - .*//'
164 24 7:1 / /home/dev/disks/disk1 rw,relatime shared:105
173 24 7:2 / /home/dev/disks/disk2 rw,relatime
224 164 7:3 / /home/dev/disks/disk1/disk3 rw,relatime shared:107

#我们可以随时修改挂载点的propagation type
#这里我们通过mount命令将disk3改成了private类型
dev@ubuntu:~/disks$ sudo mount --make-private /home/dev/disks/disk1/disk3
dev@ubuntu:~/disks$ cat /proc/self/mountinfo |grep disk3| sed 's/ - .*//'
224 164 7:3 / /home/dev/disks/disk1/disk3 rw,relatime

#--------------------------第二个shell窗口----------------------
#回到第二个shell窗口，disk3的propagation type还是shared，
#表明在老的namespace里面对propagation type的修改不会影响新namespace里面的挂载点
root@container001:~/disks# cat /proc/self/mountinfo |grep disk3| sed 's/ - .*//'
223 221 7:3 / /home/dev/disks/disk1/disk3 rw,relatime shared:107

```


关于mount命令和mount namespace的配合，里面有很多技巧，后面如果需要用到更复杂的用法，会再做详细的介绍。

## 参考
* [kernel:Shared Subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt)
* [lwn:Shared subtrees](https://lwn.net/Articles/159077/)
* [lwn:Mount namespaces, mount propagation, and unbindable mounts](https://lwn.net/Articles/690679/)
* [lwn:Mount namespaces and shared subtrees](https://lwn.net/Articles/689856/)