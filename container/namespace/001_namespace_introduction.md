# Namespace概述

Namespace是对全局系统资源的一种封装隔离，使得处于不同namespace的进程拥有独立的全局系统资源，改变一个namespace中的系统资源只会影响当前namespace里的进程，对其他namespace中的进程没有影响。

目前，Linux内核里面实现了7种不同类型的namespace。

```
Namespace   Constant          Isolates
Cgroup      CLONE_NEWCGROUP   Cgroup root directory (since Linux 4.6)
IPC         CLONE_NEWIPC      System V IPC, POSIX message queues (since Linux 2.6.19)
Network     CLONE_NEWNET      Network devices, stacks, ports, etc. (since Linux 2.6.24)
Mount       CLONE_NEWNS       Mount points (since Linux 2.4.19)
PID         CLONE_NEWPID      Process IDs (since Linux 2.6.24)
User        CLONE_NEWUSER     User and group IDs (started in Linux 2.6.23 and completed in Linux 3.8)
UTS         CLONE_NEWUTS      Hostname and NIS domain name (since Linux 2.6.19)
```

###跟namespace相关的API

* 创建一个新的进程并把他放到新的namespace中
```c
int clone(int (*child_func)(void *), void *child_stack
            , int flags, void *arg);

flags： 
    指定一个或者多个上面的CLONE_NEW*， 
    这样就会创建一个或多个新的不同类型的namespace， 
    并把新创建的子进程加入新创建的这些namespace中。
```

* 将当前进程加入到已有的namespace中
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

* 使当前进程退出指定类型的namespace，并加入到新创建的namespace
```c 
int unshare(int flags);

flags：
    指定一个或者多个上面的CLONE_NEW*，
    这样当前进程就退出了当前指定类型的namespace并加入到新创建的namespace
```

clone和unshare的功能都是创建并加入新的namespace， 就目前我所了解的情况，他们的区别是：

* unshare是使当前进程加入新的namespace
* clone是创建一个新的子进程，然后让子进程加入新的namespace

###/proc/[pid]/ns/ 目录
系统中的每个进程都有/proc/[pid]/ns/这样一个目录，里面包含了这个进程所属namespace的信息，里面每个文件的描述符都可以用来作为setns函数的参数
```bash
#下面所有的命令都是在主机的shell中执行，跟容器没有关系
dev@ubuntu:~$ lsb_release  -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 16.04 LTS
Release:        16.04
Codename:       xenial
dev@ubuntu:~$ uname -a
Linux ubuntu 4.4.0-24-generic #43-Ubuntu SMP Wed Jun 8 19:27:37 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
dev@ubuntu:~$ ls -l /proc/$$/ns     #查看当前进程所属的namespace
total 0
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 cgroup -> cgroup:[4026531835] (since Linux 4.6)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 ipc -> ipc:[4026531839] #(since Linux 3.0)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 mnt -> mnt:[4026531840] #(since Linux 3.8)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 net -> net:[4026531957] #(since Linux 3.0)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 pid -> pid:[4026531836] #(since Linux 3.8)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 user -> user:[4026531837]#(since Linux 3.8)
lrwxrwxrwx 1 dev dev 0 7月 7 17:24 uts -> uts:[4026531838] #(since Linux 3.0)
```

* 上面每种类型的namespace都是在不同的Linux版本才加入到/proc/[pid]/ns/目录里去，比如pid namespace是在Linux 3.8版本才被加入到/proc/[pid]/ns/里面，但这并不是说到3.8才支持pid namespace，其实pid namespace在2.6.24的时候就已经加入到内核了，在那个时候就可以用pid namespace了，只是有了/proc/[pid]/ns/pid之后，使得操作pid namespace更方便了

* 虽然说cgroup是在Linux 4.6版本才被加入内核，可是在Ubuntu 16.04上，尽管内核版本才4.4，但也支持cgroup  namespace，估计应该是Ubuntu将4.6的cgroup namespace这部分代码patch到了他们的4.4内核上。

* 以ipc:[4026531839]为例，ipc是namespace的类型，4026531839是inode number，如果两个进程的ipc namespace的inode number一样，说明他们属于同一个namespace。这条规则对其他类型的namespace也同样适用。

* 从上面的输出可以看出，每个进程都有属于自己的namespace，跟使不使用容器没关系。

**注意： 后续文章里面的所有例子都在ubuntu16.04下执行**

**参考：**

* [overview of Linux namespaces
](http://man7.org/linux/man-pages/man7/namespaces.7.html)
* [Namespaces in operation, part 1: namespaces overview](https://lwn.net/Articles/531114/)