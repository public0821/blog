# Linux Namespace系列（08）：user namespace (CLONE_NEWUSER)   (第二部分)

本篇将主要介绍user namespace和其他类型的namespace的关系。

权限涉及的范围非常广，所以导致user namespace比其他的namespace要复杂； 同时权限也是容器安全的基础，所以user namespace非常重要。

>本篇所有例子都在ubuntu-server-x86_64 16.04下执行通过

## 和其他类型的namespace一起使用
除了user namespace外，创建其它类型的namespace都需要CAP_SYS_ADMIN的capability。当新的user namespace创建并映射好uid、gid了之后， 这个user namespace的第一个进程将拥有完整的所有capabilities，意味着它就可以创建新的其它类型namespace

```bash
#先记下默认的user namespace编号
dev@ubuntu:~$ readlink /proc/$$/ns/user
user:[4026531837]

#用非root账号创建新的user namespace
dev@ubuntu:~$ unshare --user -r /bin/bash
root@ubuntu:~# readlink /proc/$$/ns/user
user:[4026532463]

#虽然新user namespace的root账号映射到外面的dev账号
#但还是能创建新的ipc namespace，因为当前bash进程拥有全部的capabilities
root@ubuntu:~# cat /proc/$$/status | egrep 'Cap(Inh|Prm|Eff)'
CapInh: 0000000000000000
CapPrm: 0000003fffffffff
CapEff: 0000003fffffffff
root@ubuntu:~# readlink /proc/$$/ns/ipc
ipc:[4026531839]
root@ubuntu:~# unshare --ipc /bin/bash
root@ubuntu:~# readlink /proc/$$/ns/ipc
ipc:[4026532469]

```

当然我们也可以不用这么一步一步的创建，而是一步到位
```bash
dev@ubuntu:~$ readlink /proc/$$/ns/user
user:[4026531837]
dev@ubuntu:~$ readlink /proc/$$/ns/ipc
ipc:[4026531839]
dev@ubuntu:~$ unshare --user -r --ipc /bin/bash
root@ubuntu:~# readlink /proc/$$/ns/user
user:[4026532463]
root@ubuntu:~# readlink /proc/$$/ns/ipc
ipc:[4026532469]

```

在unshare的实现中，就是传入了CLONE_NEWUSER | CLONE_NEWIPC，大致如下

```c
unshare(CLONE_NEWUSER | CLONE_NEWIPC);
```

在上面这种情况下，内核会保证CLONE_NEWUSER先被执行，然后执行剩下的其他CLONE_NEW*，这样就使得不用root账号而创建新的容器成为可能，这条规则对于clone函数也同样适用。

## 和其他类型namespace的关系
Linux下的每个namespace，都有一个user namespace和他关联，这个user namespace就是创建相应namespace时进程所属的user namespace，相当于每个namespace都有一个owner（user namespace），这样保证对任何namespace的操作都受到user namespace权限的控制。这也是上一篇中为什么sethostname失败的原因，因为要修改的uts namespace属于的父user namespace，而新user namespace的进程没有老user namespace的任何capabilities。

这里可以看看uts namespace的结构体，里面有一个指向user namespace的指针，指向他所属于的user namespace，其他类型的namespace也类似。
```c
struct uts_namespace {
  struct kref kref;
  struct new_utsname name;
  struct user_namespace *user_ns;
  struct ns_common ns;
};
```

## 不和任何user namespace关联的资源

在系统中，有些需要特权操作的资源没有跟任何user namespace关联，比如修改系统时间（需要CAP_SYS_MODULE）、创建设备（需要CAP_MKNOD），这些操作只能由initial user namespace里有相应权限的进程来操作（这里initial user namespace就是系统启动后的默认user namespace）。

## 和mount namespace的关系
* 当和mount namespace一起用时，不能挂载基于块设备的文件系统，但是可以挂载下面这些文件系统
```
     #摘自user namespaces帮助文件： 
     #http://man7.org/linux/man-pages/man7/user_namespaces.7.html
     * /proc (since Linux 3.8)  
     * /sys (since Linux 3.8) 
     * devpts (since Linux 3.9)
     * tmpfs (since Linux 3.9)
     * ramfs (since Linux 3.9)
     * mqueue (since Linux 3.9)
     * bpf (since Linux 4.4)
```
示例
```bash
#创建新的user和mount namespace
dev@ubuntu:~$ unshare --user -r --mount bash
root@ubuntu:~# mkdir ./mnt
#查找挂载到根目录的设备
root@ubuntu:~#  mount|grep " / "
/dev/mapper/ubuntu--vg-root on / type ext4 (rw,relatime,errors=remount-ro,data=ordered)
#确认这个设备肯定是块设备
root@ubuntu:~# ls -l /dev/mapper/ubuntu--vg-root
lrwxrwxrwx 1 nobody nogroup 7 7月  30 11:22 /dev/mapper/ubuntu--vg-root -> ../dm-0
root@ubuntu:~# file /dev/dm-0
/dev/dm-0: block special (252/0)
#尝试把他挂载到其他目录，结果挂载失败，说明在新的user namespace下没法挂载块设备
root@ubuntu:~# mount /dev/mapper/ubuntu--vg-root ./mnt
mount: /dev/mapper/ubuntu--vg-root is write-protected, mounting read-only
mount: cannot mount /dev/mapper/ubuntu--vg-root read-only
#即使是root账号映射过去也不行（这里mount的错误提示的好像不太准确）
root@ubuntu:~$ exit
exit
dev@ubuntu:~$ sudo unshare --user -r --mount bash
root@ubuntu:~# mount /dev/mapper/ubuntu--vg-root ./mnt
mount: /dev/mapper/ubuntu--vg-root is already mounted or /home/dev/mnt busy
       /dev/mapper/ubuntu--vg-root is already mounted on /

#由于当前pid namespace不属于当前的user namespace，所以挂载/proc失败
root@ubuntu:~# mount -t proc none ./mnt
mount: permission denied
#创建新的pid namespace，然后挂载成功
root@ubuntu:~# unshare --pid --fork bash
root@ubuntu:~# mount -t proc none ./mnt
root@ubuntu:~# mount |grep mnt|grep proc
none on /home/dev/mnt type proc (rw,nodev,relatime)
root@ubuntu:~# exit
exit

#只能通过bind方式挂载devpts，直接mount报错
root@ubuntu:~# mount -t devpts devpts ./mnt
mount: wrong fs type, bad option, bad superblock on devpts,
       missing codepage or helper program, or other error

       In some cases useful info is found in syslog - try
       dmesg | tail or so.
root@ubuntu:~# mount --bind /dev/pts ./mnt
root@ubuntu:~# mount|grep mnt|grep devpts
devpts on /home/dev/mnt type devpts (rw,nosuid,noexec,relatime,mode=600,ptmxmode=000)

#sysfs直接mount和bind mount都不行
root@ubuntu:~# mount -t sysfs sysfs ./mnt
mount: permission denied
root@ubuntu:~# mount --bind /sys ./mnt
mount: wrong fs type, bad option, bad superblock on /sys,
       missing codepage or helper program, or other error

       In some cases useful info is found in syslog - try
       dmesg | tail or so.
#TODO: 对于sysfs和devpts，和帮助文件中描述的对不上，不确定是我理解有问题，还是测试环境有问题，等以后有新的理解后再来更新

# 挂载tmpfs成功
root@ubuntu:~# mount -t tmpfs tmpfs ./mnt
root@ubuntu:~# mount|grep mnt|grep tmpfs
tmpfs on /home/dev/mnt type tmpfs (rw,nodev,relatime,uid=1000,gid=1000)
#ramfs和tmpfs类似，都是内存文件系统，这里就不演示了

#对mqueue和bpf不太熟悉，在这里也不演示了
```

* 当mount namespace和user namespace一起用时，就算老mount namespace中的mount point是shared并且用unshare命令时指定了--propagation shared，新mount namespace里面的挂载点的propagation type还是slave。这样就防止了在新user namespace里面mount的东西被外面父user namespace中的进程看到。
```bash
#准备目录和disk
dev@ubuntu:~$ mkdir -p disks/disk1
dev@ubuntu:~$ dd if=/dev/zero bs=1M count=32 of=./disks/disk1.img
dev@ubuntu:~$ mkfs.ext2 ./disks/disk1.img

#mount好disk，确认是shared
dev@ubuntu:~$ sudo mount /home/dev/disks/disk1.img /home/dev/disks/disk1
dev@ubuntu:~$ cat /proc/self/mountinfo |grep disk| sed 's/ - .*//'
164 24 7:1 / /home/dev/disks/disk1 rw,relatime shared:105

#先不创建user namespace，看看效果，便于和后面的结果比较，
#当不和user namespace一起用时，新mount namespace中的挂载点为shared
dev@ubuntu:~$ sudo unshare --mount --propagation shared /bin/bash
root@ubuntu:~# cat /proc/self/mountinfo |grep disk| sed 's/ - .*//'
220 174 7:1 / /home/dev/disks/disk1 rw,relatime shared:105
root@ubuntu:~# exit
exit

#创建mount namesapce的同时，创建user namespace，
#可以看出，虽然指定的是--propagation shared，但得到的结果还是slave（master:105）
#由于指定了--propagation shared， 系统为我们新创建了一个peer group（shared:154），
#并让新mount namespace中的挂载点属于它，
#这里同时也说明一个挂载点可以属于多个peer group。
dev@ubuntu:~$ unshare --user -r --mount --propagation shared /bin/bash
root@ubuntu:~# cat /proc/self/mountinfo |grep disk| sed 's/ - .*//'
220 174 7:1 / /home/dev/disks/disk1 rw,relatime shared:154 master:105
```

## 其他可以写map文件的情况
1. 在有一种情况下，没有CAP_SETUID的权限也可以写uid_map和gid_map，那就是在父user namespace中用新user namespace的owner来写，但是限制条件是只能在里面映射自己的账号，不能映射其他的账号。

2. 在新user namespace中用有CAP_SETUID权限的账号可以来写map文件，但跟上面的情况一样，只能映射自己。细心的朋友可能觉察到了，那就是都还没有映射，新user namespace里的账号怎么有CAP_SETUID的权限呢？关于这个问题请参考下一节（创建新user namespace时capabilities的变迁）的内容。

由于演示第二种情况需要写代码（可以参考unshare的实现），这里就只演示第一种情况：
```bash
#--------------------------第一个shell窗口----------------------
#创建新的user namespace并记下当前shell的pid
dev@ubuntu:~$ unshare --user /bin/bash
nobody@ubuntu:~$ echo $$
25430
#--------------------------第二个shell窗口----------------------
#映射多个失败
dev@ubuntu:~$ echo '0 1000 100' > /proc/25430/uid_map
-bash: echo: write error: Operation not permitted
#只映射自己成功，1000是dev账号的ID
dev@ubuntu:~$ echo '0 1000 1' > /proc/25430/uid_map

#设置setgroups为"deny"后，设置gid_map成功
dev@ubuntu:~$ echo '0 1000 1' > /proc/25430/gid_map
-bash: echo: write error: Operation not permitted
dev@ubuntu:~$ cat /proc/25430/setgroups
allow
dev@ubuntu:~$ echo "deny" > /proc/25430/setgroups
dev@ubuntu:~$ cat /proc/25430/setgroups
deny
dev@ubuntu:~$ echo '0 1000 1' > /proc/25430/gid_map
dev@ubuntu:~$

#--------------------------第一个shell窗口----------------------
#回到第一个窗口后重新加载bash，显示当前账号已经是root了
nobody@ubuntu:~$ exec bash
root@ubuntu:~# id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)
```

上面写了"deny"到文件/proc/[pid]/setgroups， 是为了限制在新user namespace里面调用setgroups函数来设置groups，这个主要是基于安全考虑。考虑这样一种情况，一个文件的权限为"rwx---rwx"，意味着other group比自己group有更大的权限，当用setgroups(2)去掉自己所属的相应group后，会获得更大的权限，这个在没有user namespace之前不是个问题，因为调用setgroups需要CAP_SETGID权限，但有了user namespace之后，一个普通的账号在新的user namespace中就有了所有的capabilities，于是他可以通过调用setgroups的方式让自己获得更大的权限。

## 创建新user namespace时capabilities的变迁
```
              clone函数                              unshare函数
+----------------------------------+    +----------------------------------+
|   父进程        |    子进程       |    |              当前进程             |
|----------------------------------+    |----------------------------------+
|   启动                           |    |               启动                |
|    | ①                           |    |                | ①               |
|    ↓                             |    |                ↓                 |
|  clone  ----------→  启动        |    |              unshare             |
|    |                  | ②        |    |                | ②               |
|    |                  ↓          |    |                ↓                 |
|    | ④               exec        |    |               exec               |
|    |                  | ③        |    |                | ③               |
|    ↓                  ↓          |    |                ↓                 |
|   结束               结束         |    |               结束               |
+----------------------------------+    +----------------------------------+

```

上面描述了进程在调用不同函数后所处的不同阶段，clone函数会创建新的进程，unshare不会。

* ①： 处于父user namespace中，进程拥有的capabilities由调用该进程的user决定
* ②： 处于子user namespace中，这个时候进程拥有所在子user namespace的所有capabilities，所以在这里可以写当前进程的uid/gid map文件，但只能映射当前账号，不能映射任意账号。这里就回答了上一节中为什么没有映射账号但在子user namespace有CAP_SETUID权限的问题。
* ③： 处于子user namespace中，调用exec后，由于没有映射，系统会去掉当前进程的所有capabilities（这个是exec的机制），所以到这个位置的时候当前进程已经没有任何capabilities了
* ④： 处于父user namespace中，和①一样。但这里是一个很好的点来设置子user namespace的map文件，如果能在exec执行之前设置好子进程的map文件，exec执行完后当前进程还是有相应的capabilities。但是如果没有在exec执行之前设置好，而是exec之后设置，当前进程还是没有capabilities，就需要再次调用exec后才有。如何在④这个地方设置子进程的map文件需要一点点技巧，可以参考[帮助文件](http://man7.org/linux/man-pages/man7/user_namespaces.7.html)最后面的示例代码。

对于unshare来说，由于没有④，所以没法映射任意账号到子user namespace，这也是为什么unshare命令只能映射当前账号的原因。

## 其它
和pid namespace类似，当在程序中用UNIX domain sokcet将一个user namespace的uid或者gid发送给另一个user namespace中的进程时，内核会自动映射成目的user namespace中对应的uid或者gid。

## 参考
* [user namespaces man page](http://man7.org/linux/man-pages/man7/user_namespaces.7.html)
* [Namespaces in operation, part 5: User namespaces](https://lwn.net/Articles/532593/)
* [Namespaces in operation, part 6: more on user namespaces](https://lwn.net/Articles/540087/)