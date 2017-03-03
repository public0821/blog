# Linux Cgroup系列（01）：Cgroup概述

cgroup和namespace类似，也是将进程进行分组，但它的目的和namespace不一样，namespace是为了隔离进程组之间的资源，而cgroup是为了对一组进程进行统一的资源监控和限制。

cgroup分[v1](https://www.kernel.org/doc/Documentation/cgroup-v1)和[v2](https://www.kernel.org/doc/Documentation/cgroup-v2.txt)两个版本，v1实现较早，功能比较多，但是由于它里面的功能都是零零散散的实现的，所以规划的不是很好，导致了一些使用和维护上的不便，v2的出现就是为了解决v1中这方面的问题，在最新的4.5内核中，cgroup v2声称已经可以用于生产环境了，但它所支持的功能还很有限，随着v2一起引入内核的还有cgroup namespace。v1和v2可以混合使用，但是这样会更复杂，所以一般没人会这样用。

本系列只介绍v1，因为这是目前大家正在用的版本，包括systemd，docker等。如果对v1比较熟悉的话，适应v2也不是问题。

>本篇所有例子都在ubuntu-server-x86_64 16.04下执行通过

##为什么需要cgroup
在Linux里，一直以来就有对进程进行分组的概念和需求，比如[session group， progress group](https://www.win.tue.nl/~aeb/linux/lk/lk-10.html)等，后来随着人们对这方面的需求越来越多，比如需要追踪一组进程的内存和IO使用情况等，于是出现了cgroup，用来统一将进程进行分组，并在分组的基础上对进程进行监控和资源控制管理等。

##什么是cgroup
术语cgroup在不同的上下文中代表不同的意思，可以指整个Linux的cgroup技术，也可以指一个具体进程组。

cgroup是Linux下的一种将进程按组进行管理的机制，在用户层看来，cgroup技术就是把系统中的所有进程组织成一颗一颗独立的树，每棵树都包含系统的所有进程，树的每个节点是一个进程组，而每颗树又和一个或者多个subsystem关联，树的作用是将进程分组，而subsystem的作用就是对这些组进行操作。cgroup主要包括下面两部分：

* **subsystem** 一个subsystem就是一个内核模块，他被关联到一颗cgroup树之后，就会在树的每个节点（进程组）上做具体的操作。subsystem经常被称作"resource controller"，因为它主要被用来调度或者限制每个进程组的资源，但是这个说法不完全准确，因为有时我们将进程分组只是为了做一些监控，观察一下他们的状态，比如perf_event subsystem。到目前为止，Linux支持12种subsystem，比如限制CPU的使用时间，限制使用的内存，统计CPU的使用情况，冻结和恢复一组进程等，后续会对它们一一进行介绍。

* **hierarchy** 一个hierarchy可以理解为一棵cgroup树，树的每个节点就是一个进程组，每棵树都会与零到多个subsystem关联。在一颗树里面，会包含Linux系统中的所有进程，但每个进程只能属于一个节点（进程组）。系统中可以有很多颗cgroup树，每棵树都和不同的subsystem关联，一个进程可以属于多颗树，即一个进程可以属于多个进程组，只是这些进程组和不同的subsystem关联。目前Linux支持12种subsystem，如果不考虑不与任何subsystem关联的情况（systemd就属于这种情况），Linux里面最多可以建12颗cgroup树，每棵树关联一个subsystem，当然也可以只建一棵树，然后让这棵树关联所有的subsystem。当一颗cgroup树不和任何subsystem关联的时候，意味着这棵树只是将进程进行分组，至于要在分组的基础上做些什么，将由应用程序自己决定，systemd就是一个这样的例子。

##如何查看当前系统支持哪些subsystem
可以通过查看/proc/cgroups(since Linux 2.6.24)知道当前系统支持哪些subsystem，下面是一个例子

```
#subsys_name    hierarchy       num_cgroups     enabled
cpuset          11              1               1
cpu             3               64              1
cpuacct         3               64              1
blkio           8               64              1
memory          9               104             1
devices         5               64              1
freezer         10              4               1
net_cls         6               1               1
perf_event      7               1               1
net_prio        6               1               1
hugetlb         4               1               1
pids            2               68              1
```

从左到右，字段的含义分别是：

1. subsystem的名字

2. subsystem所关联到的cgroup树的ID，如果多个subsystem关联到同一颗cgroup树，那么他们的这个字段将一样，比如这里的cpu和cpuacct就一样，表示他们绑定到了同一颗树。如果出现下面的情况，这个字段将为0：
   * 当前subsystem没有和任何cgroup树绑定
   * 当前subsystem已经和cgroup v2的树绑定
   * 当前subsystem没有被内核开启

3. subsystem所关联的cgroup树中进程组的个数，也即树上节点的个数

4. 1表示开启，0表示没有被开启(可以通过设置内核的启动参数“cgroup_disable”来控制subsystem的开启).

##如何使用cgroup
cgroup相关的所有操作都是基于内核中的cgroup virtual filesystem，使用cgroup很简单，挂载这个文件系统就可以了。一般情况下都是挂载到/sys/fs/cgroup目录下，当然挂载到其它任何目录都没关系。

这里假设目录/sys/fs/cgroup已经存在，下面用到的xxx为任意字符串，取一个有意义的名字就可以了，当用mount命令查看的时候，xxx会显示在第一列

* 挂载一颗和所有subsystem关联的cgroup树到/sys/fs/cgroup
    ```
    mount -t cgroup xxx /sys/fs/cgroup
    ```

* 挂载一颗和cpuset subsystem关联的cgroup树到/sys/fs/cgroup/cpuset
    ```
    mkdir /sys/fs/cgroup/cpuset
    mount -t cgroup -o cpuset xxx /sys/fs/cgroup/cpuset
    ```

* 挂载一颗与cpu和cpuacct subsystem关联的cgroup树到/sys/fs/cgroup/cpu,cpuacct
    ```
    mkdir /sys/fs/cgroup/cpu,cpuacct
    mount -t cgroup -o cpu,cpuacct xxx /sys/fs/cgroup/cpu,cpuacct
    ```

* 挂载一棵cgroup树，但不关联任何subsystem，下面就是systemd所用到的方式
    ```
    mkdir /sys/fs/cgroup/systemd
    mount -t cgroup -o none,name=systemd xxx /sys/fs/cgroup/systemd
    ```

在很多使用systemd的系统中，比如ubuntu 16.04，systemd已经帮我们将各个subsystem和cgroup树关联并挂载好了
```bash
dev@ubuntu:~$ mount|grep cgroup
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
```


创建并挂载好一颗cgroup树之后，就有了树的根节点，也即根cgroup，这时候就可以通过创建文件夹的方式创建子cgroup，然后再往每个子cgroup中添加进程。在后续介绍具体的subsystem的时候会详细介绍如何操作cgroup。

**注意**

* 第一次挂载一颗和指定subsystem关联的cgroup树时，会创建一颗新的cgroup树，当再一次用同样的参数挂载时，会重用现有的cgroup树，也即两个挂载点看到的内容是一样的。
    ```bash
    #在ubuntu 16.04中，systemd已经将和cpu,cpuacct绑定的cgroup树挂载到了/sys/fs/cgroup/cpu,cpuacct
    dev@ubuntu:~$ mount|grep /sys/fs/cgroup/cpu,cpuacct
    cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct,nsroot=/)

    #创建一个子目录，用于后面的测试
    dev@ubuntu:~$ sudo mkdir /sys/fs/cgroup/cpu,cpuacct/test
    dev@ubuntu:~$ ls -l /sys/fs/cgroup/cpu,cpuacct/|grep test
    drwxr-xr-x  2 root root 0 Oct  9 02:27 test

    #将和cpu,cpuacct关联的cgroup树重新mount到另外一个目录
    dev@ubuntu:~$ mkdir -p ./cgroup/cpu,cpuacct && cd ./cgroup/
    dev@ubuntu:~/cgroup$ sudo mount -t cgroup -o cpu,cpuacct new-cpu-cpuacct ./cpu,cpuacct

    #在新目录中看到的内容和/sys/fs/cgroup/cpu,cpuacct的一样，
    #说明我们将同一颗cgroup树mount到了系统中的不同两个目录，
    #这颗cgroup树和subsystem的关联关系不变，
    #这点类似于mount同一块硬盘到多个目录
    dev@ubuntu:~/cgroup$ ls -l ./cpu,cpuacct/ |grep test
    drwxr-xr-x  2 root root 0 Oct  9 02:27 test

    #清理
    dev@ubuntu:~/cgroup$ sudo umount new-cpu-cpuacct
    ```

* 挂载一颗cgroup树时，可以指定多个subsystem与之关联，但一个subsystem只能关联到一颗cgroup树，一旦关联并在这颗树上创建了子cgroup，subsystems和这棵cgroup树就成了一个整体，不能再重新组合。以上面ubuntu 16.04为例，由于已经将cpu,cpuacct和一颗cgroup树关联并且他们下面有子cgroup了，所以就不能单独的将cpu和另一颗cgroup树关联。
    ```bash
    #尝试将cpu subsystem重新关联一颗cgroup树并且将这棵树mount到./cpu目录
    dev@ubuntu:~/cgroup$ mkdir cpu
    dev@ubuntu:~/cgroup$ sudo mount -t cgroup -o cpu new-cpu ./cpu
    mount: new-cpu is already mounted or /home/dev/cgroup/cpu busy
    #由于cpu和cpuacct已经和一颗cgroup树关联了，所以这里mount失败

    #尝试将devices和pids关联到同一颗树上，由于他们各自已经关联到了不同的cgroup树，所以mount失败
    dev@ubuntu:~/cgroup$ mkdir devices,pids
    dev@ubuntu:~/cgroup$ sudo mount -t cgroup -o devices,pids new-devices-pids ./devices,pids
    mount: new-devices-pids is already mounted or /home/dev/cgroup/devices,pids busy
    ```
但由于/sys/fs/cgroup/hugetlb和/sys/fs/cgroup/perf_event下没有子cgroup，我们可以将他们重新组合。一般情况下不会用到这个功能，一但最开始关联好了之后，就不会去重新修改它，也即我们一般不会去修改systemd给我们设置好的subsystem和cgroup树的关联关系。
    ```bash
    #/sys/fs/cgroup/hugetlb和/sys/fs/cgroup/perf_event里面没有子目录，说明没有子cgroup
    dev@ubuntu:~$ ls -l /sys/fs/cgroup/hugetlb|grep ^d
    dev@ubuntu:~$ ls -l /sys/fs/cgroup/perf_event|grep ^d

    #直接mount不行，因为perf_event,hugetlb已经被系统单独mount过了
    dev@ubuntu:~$ sudo mount -t cgroup -operf_event,hugetlb xxx /mnt
    mount: xxx is already mounted or /mnt busy

    #先umount
    dev@ubuntu:~$ sudo umount /sys/fs/cgroup/perf_event
    dev@ubuntu:~$ sudo umount /sys/fs/cgroup/hugetlb
    #如果系统默认安装了lxcfs的话，lxcfs会将它们挂载在自己的目录，
    #所以需要umount lxcfs及下面这两个目录，否则就没有真正的umount掉perf_event和hugetlb
    dev@ubuntu:~$ sudo umount lxcfs
    dev@ubuntu:~$ sudo umount /run/lxcfs/controllers/hugetlb
    dev@ubuntu:~$ sudo umount /run/lxcfs/controllers/perf_event

    #再mount，成功
    dev@ubuntu:~$ sudo mount -t cgroup -operf_event,hugetlb xxx /mnt
    dev@ubuntu:~$ ls /mnt/
    cgroup.clone_children  cgroup.sane_behavior  hugetlb.2MB.limit_in_bytes      hugetlb.2MB.usage_in_bytes  release_agent
    cgroup.procs           hugetlb.2MB.failcnt   hugetlb.2MB.max_usage_in_bytes  notify_on_release           tasks

    #清理
    dev@ubuntu:~$ sudo reboot
    ```

* 可以创建任意多个不和任何subsystem关联的cgroup树，name是这棵树的唯一标记，当name指定的是一个新的名字时，将创建一颗新的cgroup树，但如果内核中已经存在一颗一样name的cgroup树，那么将mount已存在的这颗cgroup树
    ```bash
    #由于name=test的cgroup树在系统中不存在，所以这里会创建一颗新的name=test的cgroup树
    dev@ubuntu:~$ mkdir -p cgroup/test && cd cgroup
    dev@ubuntu:~/cgroup$ sudo mount -t cgroup -o none,name=test test ./test
    #系统为新创建的cgroup树的root cgroup生成了默认文件
    dev@ubuntu:~/cgroup$ ls ./test/
    cgroup.clone_children  cgroup.procs  cgroup.sane_behavior  notify_on_release  release_agent  tasks
    #新创建的cgroup树的root cgroup里包含系统中的所有进程
    dev@ubuntu:~/cgroup$ wc -l ./test/cgroup.procs
    131 ./test/cgroup.procs

    #创建子cgroup
    dev@ubuntu:~/cgroup$ cd test && sudo mkdir aaaa
    #系统已经为新的子cgroup生成了默认文件
    dev@ubuntu:~/cgroup/test$ ls aaaa
    cgroup.clone_children  cgroup.procs  notify_on_release  tasks
    #新创建的子cgroup中没有任何进程
    dev@ubuntu:~/cgroup/test$ wc -l aaaa/cgroup.procs
    0 aaaa/cgroup.procs

    #重新挂载这棵树到test1，由于mount的时候指定的name=test，所以和上面挂载的是同一颗cgroup树，于是test1目录下的内容和test目录下的内容一样
    dev@ubuntu:~/cgroup/test$ cd .. && mkdir test1
    dev@ubuntu:~/cgroup$ sudo mount -t cgroup -o none,name=test test ./test1
    dev@ubuntu:~/cgroup$ ls ./test1
    aaaa  cgroup.clone_children  cgroup.procs  cgroup.sane_behavior  notify_on_release  release_agent  tasks

    #清理
    dev@ubuntu:~/cgroup$ sudo umount ./test1
    dev@ubuntu:~/cgroup$ sudo umount ./test
    dev@ubuntu:~/cgroup$ cd .. && rm -r ./cgroup
    ```

##如何查看当前进程属于哪些cgroup
可以通过查看/proc/[pid]/cgroup(since Linux 2.6.24)知道指定进程属于哪些cgroup。
```bash
dev@ubuntu:~$ cat /proc/777/cgroup
11:cpuset:/
10:freezer:/
9:memory:/system.slice/cron.service
8:blkio:/system.slice/cron.service
7:perf_event:/
6:net_cls,net_prio:/
5:devices:/system.slice/cron.service
4:hugetlb:/
3:cpu,cpuacct:/system.slice/cron.service
2:pids:/system.slice/cron.service
1:name=systemd:/system.slice/cron.service
```
每一行包含用冒号隔开的三列，他们的意思分别是

1. cgroup树的ID， 和/proc/cgroups文件中的ID一一对应。

2. 和cgroup树绑定的所有subsystem，多个subsystem之间用逗号隔开。这里name=systemd表示没有和任何subsystem绑定，只是给他起了个名字叫systemd。

3. 进程在cgroup树中的路径，即进程所属的cgroup，这个路径是相对于挂载点的相对路径。

##所有的subsystems
目前Linux支持下面12种subsystem

* [cpu](https://www.kernel.org/doc/Documentation/scheduler/sched-bwc.txt) (since Linux 2.6.24; CONFIG_CGROUP_SCHED)
用来限制cgroup的CPU使用率。              

* [cpuacct](https://www.kernel.org/doc/Documentation/cgroup-v1/cpuacct.txt) (since Linux 2.6.24; CONFIG_CGROUP_CPUACCT)
统计cgroup的CPU的使用率。

* [cpuset](https://www.kernel.org/doc/Documentation/cgroup-v1/cpusets.txt) (since Linux 2.6.24; CONFIG_CPUSETS)
绑定cgroup到指定CPUs和NUMA节点。

* [memory](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt) (since Linux 2.6.25; CONFIG_MEMCG)
统计和限制cgroup的内存的使用率，包括process memory, kernel memory, 和swap。

* [devices](https://www.kernel.org/doc/Documentation/cgroup-v1/devices.txt) (since Linux 2.6.26; CONFIG_CGROUP_DEVICE)
限制cgroup创建(mknod)和访问设备的权限。

* [freezer](https://www.kernel.org/doc/Documentation/cgroup-v1/freezer-subsystem.txt) (since Linux 2.6.28; CONFIG_CGROUP_FREEZER)
suspend和restore一个cgroup中的所有进程。

* [net_cls](https://www.kernel.org/doc/Documentation/cgroup-v1/net_cls.txt) (since Linux 2.6.29; CONFIG_CGROUP_NET_CLASSID)
将一个cgroup中进程创建的所有网络包加上一个classid标记，用于[tc](http://man7.org/linux/man-pages/man8/tc.8.html)和iptables。 只对发出去的网络包生效，对收到的网络包不起作用。

* [blkio](https://www.kernel.org/doc/Documentation/cgroup-v1/blkio-controller.txt) (since Linux 2.6.33; CONFIG_BLK_CGROUP)
限制cgroup访问块设备的IO速度。

* [perf_event](https://www.kernel.org/doc/Documentation/perf-record.txt) (since Linux 2.6.39; CONFIG_CGROUP_PERF)
对cgroup进行性能监控

* [net_prio](https://www.kernel.org/doc/Documentation/cgroup-v1/net_prio.txt) (since Linux 3.3; CONFIG_CGROUP_NET_PRIO)
针对每个网络接口设置cgroup的访问优先级。

* [hugetlb](https://www.kernel.org/doc/Documentation/cgroup-v1/hugetlb.txt) (since Linux 3.5; CONFIG_CGROUP_HUGETLB)
限制cgroup的huge pages的使用量。

* [pids](https://www.kernel.org/doc/Documentation/cgroup-v1/pids.txt) (since Linux 4.3; CONFIG_CGROUP_PIDS)
限制一个cgroup及其子孙cgroup中的总进程数。

上面这些subsystem，有些需要做资源统计，有些需要做资源控制，有些即不统计也不控制。对于cgroup树来说，有些subsystem严重依赖继承关系，有些subsystem完全用不到继承关系，而有些对继承关系没有严格要求。

不同subsystem的工作方式可能差别较大，对系统性能的影响也不一样，本人不是这方面的专家，后续文章中只会从功能的角度来介绍不同的subsystem，不会涉及到他们内部的实现。

##结束语
本文介绍了cgroup的一些概念，包括subsystem和hierarchy，然后介绍了怎么挂载cgroup文件系统以及12个subsystem的功能。从下一篇开始，将介绍cgroup具体的用法和不同的subsystem。


##参考
[cgroups man page](http://man7.org/linux/man-pages/man7/cgroups.7.html)
[CGROUPS v1](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)
[CGROUPS v2](https://www.kernel.org/doc/Documentation/cgroup-v2.txt)
[Control groups series by Neil Brown](https://lwn.net/Articles/604609/)