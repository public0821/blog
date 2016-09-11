# mount namespaces (CLONE_NEWNS)
Mount namespace隔离了文件系统的挂载点, 使得不同的mount namespace拥有自己独立的挂载点信息，不同的namespace之间不会相互影响，这对于构建用户或者容器自己的文件系统目录非常有用。

当前进程所在mount namespace里的所有挂载信息可以在/proc/[pid]/mounts、/proc/[pid]/mountinfo和/proc/[pid]/mountstats里面找到。

Mount namespaces是第一个被加入Linux的namespace，由于当时没想到还会引入其它的namespace，所以取名为CLONE_NEWNS，而没有叫CLONE_NEWMOUNT。

每个mount namespace都拥有一份自己的挂载点列表，当用clone或者unshare函数创建新的mount namespace时，新创建的namespace将拷贝一份老namespace里的挂载点列表，但从这之后，他们就没有关系了，通过mount和umount增加和删除各自namespace里面的挂载点都不会相互影响。

###演示

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

#查看mount namespace
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

#mount 002.iso
root@container001:~/iso# mount ./002.iso /mnt/iso2/
mount: /dev/loop0 is write-protected, mounting read-only
root@container001:~/iso# mount |grep iso
/home/dev/iso/001.iso on /mnt/iso1 type iso9660 (ro,relatime)
/home/dev/iso/002.iso on /mnt/iso2 type iso9660 (ro,relatime)

#umount 001.iso
root@container001:~/iso# umount /mnt/iso1
root@container001:~/iso# mount |grep iso
/home/dev/iso/002.iso on /mnt/iso2 type iso9660 (ro,relatime)

#/mnt/iso1目录为空
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


###Shared subtrees
在某些情况下，比如系统添加了一个新的硬盘，这个时候如果mount namespace是完全隔离的，想要在各个namespace里面用这个硬盘，就需要在每个namespace里面手动mount这个硬盘，这个是很麻烦的，[Shared subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt)可以帮助我们解决这个问题。

有了shared subtrees这个feature后，每个挂载点都有一个传播类型（propagation type）标志, 由它来决定当一个挂载点的下面创建和移除挂载点的时候，是否会传播到属于相同peer group的其他挂载点下去，也即同一个peer group里的其他的挂载点下面是不是也会创建和移除相应的挂载点.现在有4种不同类型的传播类型：

* MS_SHARED: 从名字就可以看出，挂载信息会在同一个peer group的不同挂载点之间共享传播. 当一个挂载点下面添加或者删除挂载点的时候，同一个peer group里的其他挂载点下面也会挂载和卸载同样的挂载点

* MS_PRIVATE: 跟上面的刚好相反，挂载信息根本就共享，也即private的挂载点不会属于任何peer group

* MS_SLAVE: 跟名字一样，信息的传播是单向的，在同一个peer group里面，master的挂载点下面发生变化的时候，slave的挂载点下面也跟着变化，但反之则不然，slave下发生变化的时候不会通知master，master不会发生变化。

* MS_UNBINDABLE: 这个和MS_PRIVATE相同，只是这种类型的挂载点不能作为bind mount的源，主要用来防止递归情况的出现。在这里不会详细介绍这种类型，有兴趣的同学请参考[这里的例子](https://lwn.net/Articles/690679/)。

看完上面的文字估计有些人都晕了，下面先澄清一些概念

* 传播类型（propagation type）是挂载点的属性，每个挂载点都是独立的
* 挂载点是有父子关系的，比如挂载点/和/mnt/cdrom，/mnt/cdrom就是/的子挂载点，/是/mnt/cdrom的父挂载点
* 默认情况下，如果父挂载点是MS_SHARED，那么子挂载点也是MS_SHARED的，否则子挂载点将会是MS_PRIVATE，跟爷爷挂载点没有关系

####peer group
peer group就是一个或多个挂载点的集合，他们之间可以共享挂载信息。目前在下面两种情况下会使两个挂载点属于一个peer group

*  当创建新的mount namespace时，新namespace会拷贝一份老namespace的挂载点信息，于是新的和老的namespace里面的相同挂载点就会属于同一个peer group。
*  利用mount --bind命令， 将会使源和目的属于同一个peer group

这里的前提条件是挂载点的propagation type是shared

####mount --bind 示例
这里不会详细介绍mount命令的用法，如果对bind mount不太了解，请先参考[man mount](http://man7.org/linux/man-pages/man8/mount.8.html)

```bash
#准备4个虚拟的disk，并在上面创建ext2文件系统，用于后续的mount测试
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

#显式的以shared方式挂载disk1
dev@ubuntu:~/disks$ sudo mount --make-shared ./disk1.img ./disk1
#显式的以private方式挂载disk1
dev@ubuntu:~/disks$ sudo mount --make-private ./disk2.img ./disk2

#mountinfo比mounts文件包含有更多的关于挂载点的信息
#这里sed主要用来过滤掉跟当前主题无关的信息
#shared:105表示挂载点/home/dev/disks/disk1是以shared方式挂载，
#且peer group id为105
#而挂载点/home/dev/disks/disk2没有相关信息，表示是private方式挂载
dev@ubuntu:~/disks$ cat /proc/self/mountinfo |grep disk | sed 's/ - .*//'
164 24 7:1 / /home/dev/disks/disk1 rw,relatime shared:105
173 24 7:2 / /home/dev/disks/disk2 rw,relatime

#分别在disk1和disk2目录下创建disk3和disk4，然后挂载disk3，disk4到这两个目录
dev@ubuntu:~/disks$ sudo mkdir ./disk1/disk3 ./disk2/disk4
dev@ubuntu:~/disks$ sudo mount ./disk3.img ./disk1/disk3
dev@ubuntu:~/disks$ sudo mount ./disk4.img ./disk2/disk4

#查看挂载信息，第一列的数字是挂载点ID，第二例是父挂载点ID
#从结果来看，果然在默认mount的情况下，子挂载点会继承父挂载点的propagation type
dev@ubuntu:~/disks$ cat /proc/self/mountinfo |grep disk| sed 's/ - .*//'
164 24 7:1 / /home/dev/disks/disk1 rw,relatime shared:105 
173 24 7:2 / /home/dev/disks/disk2 rw,relatime 
176 164 7:3 / /home/dev/disks/disk1/disk3 rw,relatime shared:107 
179 173 7:4 / /home/dev/disks/disk2/disk4 rw,relatime 

#umount掉disk3和disk4，创建两个新的目录bind1和bind2用于bind测试
dev@ubuntu:~/disks$ sudo umount /home/dev/disks/disk1/disk3
dev@ubuntu:~/disks$ sudo umount /home/dev/disks/disk2/disk4
dev@ubuntu:~/disks$ mkdir bind1 bind2

#bind的方式挂载disk1到bind1，disk2到bind2
dev@ubuntu:~/disks$ sudo mount --bind ./disk1 ./bind1
dev@ubuntu:~/disks$ sudo mount --bind ./disk2 ./bind2

#查看挂载信息，显然默认情况下bind1和bind2的propagation type继承自父挂载点24
#因为164和176都是shared类型且是通过bind方式mount在一起的，所以他们属于同一个peer group 105
dev@ubuntu:~/disks$ cat /proc/self/mountinfo |grep disk| sed 's/ - .*//'
164 24 7:1 / /home/dev/disks/disk1 rw,relatime shared:105 
173 24 7:2 / /home/dev/disks/disk2 rw,relatime 
176 24 7:1 / /home/dev/disks/bind1 rw,relatime shared:105 
179 24 7:2 / /home/dev/disks/bind2 rw,relatime shared:109 
#上面ID为24的挂载点为根目录的挂载点
dev@ubuntu:~/disks$ cat /proc/self/mountinfo |grep ^24| sed 's/ - .*//'
24 0 252:0 / / rw,relatime shared:1

#这时disk3和disk4目录都是空的
dev@ubuntu:~/disks$ ls bind1/disk3/
dev@ubuntu:~/disks$ ls bind2/disk4/
dev@ubuntu:~/disks$ ls disk1/disk3/
dev@ubuntu:~/disks$ ls disk2/disk4/

#重新挂载disk3和disk4
dev@ubuntu:~/disks$ sudo mount ./disk3.img ./disk1/disk3
dev@ubuntu:~/disks$ sudo mount ./disk4.img ./disk2/disk4

#由于disk1/和bind1/属于同一个peer group，
#所以在挂载了disk3后，在两个目录下都能看到disk3下的内容
dev@ubuntu:~/disks$ ls disk1/disk3/
lost+found
dev@ubuntu:~/disks$ ls bind1/disk3/
lost+found
#而disk2/是private类型的，所以在他下面挂载disk4不会通知到bind2，
#于是bind2下的disk4目录是空的
dev@ubuntu:~/disks$ ls disk2/disk4/
lost+found
dev@ubuntu:~/disks$ ls bind2/disk4/
dev@ubuntu:~/disks$

#再看看disk3和disk4的挂载信息
#虽然182和183的父挂载点不一样，但由于他们父挂载点属于同一个peer group，
#且disk3是以默认方式挂载的，所以他们的peer group一样
dev@ubuntu:~/disks$ cat /proc/self/mountinfo |egrep "disk3|disk4"| sed 's/ - .*//'
182 164 7:3 / /home/dev/disks/disk1/disk3 rw,relatime shared:111 
183 176 7:3 / /home/dev/disks/bind1/disk3 rw,relatime shared:111 
188 173 7:4 / /home/dev/disks/disk2/disk4 rw,relatime 

#umount bind1/disk3后，disk1/disk3也相应的自动umount掉了
dev@ubuntu:~/disks$ sudo umount bind1/disk3
dev@ubuntu:~/disks$ cat /proc/self/mountinfo |grep disk3
dev@ubuntu:~/disks$

#umount除disk1的所有其他挂载点
dev@ubuntu:~/disks$ sudo umount ./disk2/disk4
dev@ubuntu:~/disks$ sudo umount /home/dev/disks/bind1
dev@ubuntu:~/disks$ sudo umount /home/dev/disks/bind2
dev@ubuntu:~/disks$ sudo umount /home/dev/disks/disk2
#确认只剩disk1
dev@ubuntu:~/disks$ cat /proc/self/mountinfo |grep disk| sed 's/ - .*//'
164 24 7:1 / /home/dev/disks/disk1 rw,relatime shared:105 

#分别显式的用shared和slave的方式bind disk1
dev@ubuntu:~/disks$ sudo mount --bind --make-shared ./disk1 ./bind1
dev@ubuntu:~/disks$ sudo mount --bind --make-slave ./bind1 ./bind2
#164和173属于同一个peer group，
#master:105表示/home/dev/disks/bind2是peer group 105的slave
dev@ubuntu:~/disks$ cat /proc/self/mountinfo |grep disk| sed 's/ - .*//'
164 24 7:1 / /home/dev/disks/disk1 rw,relatime shared:105 
173 24 7:1 / /home/dev/disks/bind1 rw,relatime shared:105 
176 24 7:1 / /home/dev/disks/bind2 rw,relatime master:105 

#mount disk3到disk1目录下
dev@ubuntu:~/disks$ sudo mount ./disk3.img ./disk1/disk3/
#其他两个目录bin1和bind2里面也挂载成功，说明master发生变化的时候，slave会跟着变化
dev@ubuntu:~/disks$ cat /proc/self/mountinfo |grep disk| sed 's/ - .*//'
164 24 7:1 / /home/dev/disks/disk1 rw,relatime shared:105 
173 24 7:1 / /home/dev/disks/bind1 rw,relatime shared:105 
176 24 7:1 / /home/dev/disks/bind2 rw,relatime master:105 
179 164 7:2 / /home/dev/disks/disk1/disk3 rw,relatime shared:109 
181 176 7:2 / /home/dev/disks/bind2/disk3 rw,relatime master:109 
180 173 7:2 / /home/dev/disks/bind1/disk3 rw,relatime shared:109 

#umount disk3，然后mount disk3到bind2目录下
dev@ubuntu:~/disks$ sudo umount ./disk1/disk3/
dev@ubuntu:~/disks$ sudo mount ./disk3.img ./bind2/disk3/
#由于bind2的propagation type是slave，所以disk1和bind1两个挂载点下面不会挂载disk3
#从179的类型可以看出，当父挂载点176是slave类型时，默认情况下子节点是private类型
dev@ubuntu:~/disks$ cat /proc/self/mountinfo |grep disk| sed 's/ - .*//'
164 24 7:1 / /home/dev/disks/disk1 rw,relatime shared:105 
173 24 7:1 / /home/dev/disks/bind1 rw,relatime shared:105 
176 24 7:1 / /home/dev/disks/bind2 rw,relatime master:105 
179 176 7:2 / /home/dev/disks/bind2/disk3 rw,relatime -

#mount还有很多跟propagation type相关的参数来指定目的挂载点的propagation type
#详情请参考mount的帮助文件

```

####mount namespace演示
mount namespace和bind的情况差不多，这里就简单演示一下
```bash
#--------------------------第一个shell窗口----------------------
#先umount上面例子中的所有挂载点，确保这里的测试不受上面影响
dev@ubuntu:~/disks$ sudo umount /home/dev/disks/disk1
dev@ubuntu:~/disks$ sudo umount /home/dev/disks/bind1
dev@ubuntu:~/disks$ sudo umount -R /home/dev/disks/bind2
#确保没有disk相关的mount信息
dev@ubuntu:~/disks$ cat /proc/self/mountinfo |grep disk| sed 's/ - .*//'
dev@ubuntu:~/disks$

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

#创建新的mount namespace
#默认情况下，unshare会将新namespace里面的所有挂载点的类型设置成private，
#所以这里用到了参数--propagation unchaged，
#让新namespace里的挂载点的类型和老namespace里保持一致。
#--propagation参数还支持private|shared|slave类型，
#和mount命令的那些--make-private参数一样，
#他们的背后都是通过调用mount（...）函数传入不同的参数实现的
dev@ubuntu:~/disks$ sudo unshare --mount --uts --propagation unchaged /bin/bash
root@ubuntu:~/disks# hostname container001
root@ubuntu:~/disks# exec bash
root@container001:~/disks# 

#确认已经是在新的mount namespace里面了
root@container001:~/disks# readlink /proc/$$/ns/mnt
mnt:[4026532463]

#由于前面指定了--propagation unchaged，
#所以新namespace里面的/home/dev/disks/disk1也是shared，
#且和老namespace里面的/home/dev/disks/disk1属于同一个peer group 105
#在上面bind的例子中我们也看到了105这个ID，说明peer group ID也是不断的回收再利用的
#这里挂载点的ID和原来namespace里的也已经不一样了
root@container001:~/disks# cat /proc/self/mountinfo |grep disk| sed 's/ - .*//'
221 177 7:1 / /home/dev/disks/disk1 rw,relatime shared:105
222 177 7:2 / /home/dev/disks/disk2 rw,relatime

#mount disk3
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

#我们也可以随时修改挂载点的propagation type
#这里我们通过mount命令将disk3改成了private类型
dev@ubuntu:~/disks$ sudo mount --make-private /home/dev/disks/disk1/disk3
dev@ubuntu:~/disks$ cat /proc/self/mountinfo |grep disk3| sed 's/ - .*//'
224 164 7:3 / /home/dev/disks/disk1/disk3 rw,relatime

#--------------------------第二个shell窗口----------------------
#回到第二个shell窗口

#结果表明在老的namespace里面对propagation type的修改不会影响新namespace里面的挂载点
root@container001:~/disks# cat /proc/self/mountinfo |grep disk3| sed 's/ - .*//'
223 221 7:3 / /home/dev/disks/disk1/disk3 rw,relatime shared:107

```


关于mount的用法，尤其是bind mount，以及mount命令和mount namespace的配合，里面有很多技巧，这里只是做个简单介绍，后面如果需要用到更复杂的用法，会再做详细的介绍。

###参考
[kernel:Shared Subtrees](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt)
[lwn:Shared subtrees](https://lwn.net/Articles/159077/)
[lwn:Mount namespaces, mount propagation, and unbindable mounts](https://lwn.net/Articles/690679/)
[lwn:Mount namespaces and shared subtrees](https://lwn.net/Articles/689856/)