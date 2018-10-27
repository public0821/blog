# Linux Namespace系列（01）：Namespace概述

Namespace是对全局系统资源的一种封装隔离，使得处于不同namespace的进程拥有独立的全局系统资源，改变一个namespace中的系统资源只会影响当前namespace里的进程，对其他namespace中的进程没有影响。

>下面的所有例子都在ubuntu-server-x86_64 16.04下执行通过

## Linux内核支持的namespaces

目前，Linux内核里面实现了7种不同类型的namespace。

```
名称        宏定义             隔离内容
Cgroup      CLONE_NEWCGROUP   Cgroup root directory (since Linux 4.6)
IPC         CLONE_NEWIPC      System V IPC, POSIX message queues (since Linux 2.6.19)
Network     CLONE_NEWNET      Network devices, stacks, ports, etc. (since Linux 2.6.24)
Mount       CLONE_NEWNS       Mount points (since Linux 2.4.19)
PID         CLONE_NEWPID      Process IDs (since Linux 2.6.24)
User        CLONE_NEWUSER     User and group IDs (started in Linux 2.6.23 and completed in Linux 3.8)
UTS         CLONE_NEWUTS      Hostname and NIS domain name (since Linux 2.6.19)
```

>**注意：** 由于Cgroup namespace在4.6的内核中才实现，并且和cgroup v2关系密切，现在普及程度还不高，比如docker现在就还没有用它，所以在namespace这个系列中不会介绍Cgroup namespace。

## 查看进程所属的namespaces
系统中的每个进程都有/proc/[pid]/ns/这样一个目录，里面包含了这个进程所属namespace的信息，里面每个文件的描述符都可以用来作为setns函数(后面会介绍)的参数。

```bash
#查看当前bash进程所属的namespace
dev@ubuntu:~$ ls -l /proc/$$/ns     
total 0
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 cgroup -> cgroup:[4026531835] #(since Linux 4.6)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 ipc -> ipc:[4026531839]       #(since Linux 3.0)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 mnt -> mnt:[4026531840]       #(since Linux 3.8)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 net -> net:[4026531957]       #(since Linux 3.0)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 pid -> pid:[4026531836]       #(since Linux 3.8)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 user -> user:[4026531837]     #(since Linux 3.8)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 uts -> uts:[4026531838]       #(since Linux 3.0)
```

* 上面每种类型的namespace都是在不同的Linux版本被加入到/proc/[pid]/ns/目录里去的，比如pid namespace是在Linux 3.8才被加入到/proc/[pid]/ns/里面，但这并不是说到3.8才支持pid namespace，其实pid namespace在2.6.24的时候就已经加入到内核了，在那个时候就可以用pid namespace了，只是有了/proc/[pid]/ns/pid之后，使得操作pid namespace更方便了

* 虽然说cgroup是在Linux 4.6版本才被加入内核，可是在Ubuntu 16.04上，尽管内核版本才4.4，但也支持cgroup  namespace，估计应该是Ubuntu将4.6的cgroup namespace这部分代码patch到了他们的4.4内核上。

* 以ipc:[4026531839]为例，ipc是namespace的类型，4026531839是inode number，如果两个进程的ipc namespace的inode number一样，说明他们属于同一个namespace。这条规则对其他类型的namespace也同样适用。

* 从上面的输出可以看出，对于每种类型的namespace，进程都会与一个namespace ID关联。


## 跟namespace相关的API
和namespace相关的函数只有三个，这里简单的看一下，后面介绍[UTS namespace](https://segmentfault.com/a/1190000006908598)的时候会有详细的示例

### [clone](http://man7.org/linux/man-pages/man2/clone.2.html)： 创建一个新的进程并把他放到新的namespace中
```c
int clone(int (*child_func)(void *), void *child_stack
            , int flags, void *arg);

flags： 
    指定一个或者多个上面的CLONE_NEW*（当然也可以包含跟namespace无关的flags）， 
    这样就会创建一个或多个新的不同类型的namespace， 
    并把新创建的子进程加入新创建的这些namespace中。
```

### [setns](http://man7.org/linux/man-pages/man2/setns.2.html)： 将当前进程加入到已有的namespace中
```c
int setns(int fd, int nstype);

fd： 
    指向/proc/[pid]/ns/目录里相应namespace对应的文件，
    表示要加入哪个namespace

nstype：
    指定namespace的类型（上面的任意一个CLONE_NEW*）：
    1. 如果当前进程不能根据fd得到它的类型，如fd由其他进程创建，
    并通过UNIX domain socket传给当前进程，
    那么就需要通过nstype来指定fd指向的namespace的类型
    2. 如果进程能根据fd得到namespace类型，比如这个fd是由当前进程打开的，
    那么nstype设置为0即可
```

### [unshare](http://man7.org/linux/man-pages/man2/unshare.2.html): 使当前进程退出指定类型的namespace，并加入到新创建的namespace（相当于创建并加入新的namespace）
```c 
int unshare(int flags);

flags：
    指定一个或者多个上面的CLONE_NEW*，
    这样当前进程就退出了当前指定类型的namespace并加入到新创建的namespace
```

### clone和unshare的区别
clone和unshare的功能都是创建并加入新的namespace， 他们的区别是：

* unshare是使当前进程加入新的namespace
* clone是创建一个新的子进程，然后让子进程加入新的namespace，而当前进程保持不变

## 其它
当一个namespace中的所有进程都退出时，该namespace将会被销毁。当然还有其他方法让namespace一直存在，假设我们有一个进程号为1000的进程，以ipc namespace为例：

1. 通过mount --bind命令。例如mount --bind /proc/1000/ns/ipc /other/file，就算属于这个ipc namespace的所有进程都退出了，只要/other/file还在，这个ipc namespace就一直存在，其他进程就可以利用/other/file，通过setns函数加入到这个namespace

2. 在其他namespace的进程中打开/proc/1000/ns/ipc文件，并一直持有这个文件描述符不关闭，以后就可以用setns函数加入这个namespace。

## 参考

* [overview of Linux namespaces](http://man7.org/linux/man-pages/man7/namespaces.7.html)
* [Namespaces in operation, part 1: namespaces overview](https://lwn.net/Articles/531114/)
