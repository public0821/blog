# Linux Cgroup系列（05）：限制cgroup的CPU使用（subsystem之cpu）

在cgroup里面，跟CPU相关的子系统有[cpusets](https://www.kernel.org/doc/Documentation/cgroup-v1/cpusets.txt)、[cpuacct](https://www.kernel.org/doc/Documentation/cgroup-v1/cpuacct.txt)和[cpu](https://www.kernel.org/doc/Documentation/scheduler/sched-bwc.txt)。

其中cpuset主要用于设置CPU的亲和性，可以限制cgroup中的进程只能在指定的CPU上运行，或者不能在指定的CPU上运行，同时cpuset还能设置内存的亲和性。设置亲和性一般只在比较特殊的情况才用得着，所以这里不做介绍。

cpuacct包含当前cgroup所使用的CPU的统计信息，信息量较少，有兴趣可以去看看它的文档，这里不做介绍。

本篇只介绍cpu子系统，包括怎么限制cgroup的CPU使用上限及相对于其它cgroup的相对值。

>本篇所有例子都在ubuntu-server-x86_64 16.04下执行通过

##创建子cgroup
在ubuntu下，systemd已经帮我们mount好了cpu子系统，我们只需要在相应的目录下创建子目录就可以了

```bash
#从这里的输出可以看到，cpuset被挂载在了/sys/fs/cgroup/cpuset，
#而cpu和cpuacct一起挂载到了/sys/fs/cgroup/cpu,cpuacct下面
dev@ubuntu:~$ mount|grep cpu
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)

#进入/sys/fs/cgroup/cpu,cpuacct并创建子cgroup
dev@ubuntu:~$ cd /sys/fs/cgroup/cpu,cpuacct
dev@ubuntu:/sys/fs/cgroup/cpu,cpuacct$ sudo mkdir test
dev@ubuntu:/sys/fs/cgroup/cpu,cpuacct$ cd test
dev@ubuntu:/sys/fs/cgroup/cpu,cpuacct/test$ ls
cgroup.clone_children  cpuacct.stat   cpuacct.usage_percpu  cpu.cfs_quota_us  cpu.stat           tasks
cgroup.procs           cpuacct.usage  cpu.cfs_period_us     cpu.shares        notify_on_release
```

除了cgroup里面通用的cgroup.clone_children、tasks、cgroup.procs、notify_on_release这几个文件外，以cpuacct.开头的文件跟cpuacct子系统有关，我们这里只需要关注cpu.开头的文件。

#### cpu.cfs_period_us & cpu.cfs_quota_us
cfs_period_us用来配置时间周期长度，cfs_quota_us用来配置当前cgroup在设置的周期长度内所能使用的CPU时间数，两个文件配合起来设置CPU的使用上限。两个文件的单位都是微秒（us），cfs_period_us的取值范围为1毫秒（ms）到1秒（s），cfs_quota_us的取值大于1ms即可，如果cfs_quota_us的值为-1（默认值），表示不受cpu时间的限制。下面是几个例子：
```
1.限制只能使用1个CPU（每250ms能使用250ms的CPU时间）
    # echo 250000 > cpu.cfs_quota_us /* quota = 250ms */
    # echo 250000 > cpu.cfs_period_us /* period = 250ms */

2.限制使用2个CPU（内核）（每500ms能使用1000ms的CPU时间，即使用两个内核）
    # echo 1000000 > cpu.cfs_quota_us /* quota = 1000ms */
    # echo 500000 > cpu.cfs_period_us /* period = 500ms */

3.限制使用1个CPU的20%（每50ms能使用10ms的CPU时间，即使用一个CPU核心的20%）
    # echo 10000 > cpu.cfs_quota_us /* quota = 10ms */
    # echo 50000 > cpu.cfs_period_us /* period = 50ms */
```

#### cpu.shares
shares用来设置CPU的相对值，并且是针对所有的CPU（内核），默认值是1024，假如系统中有两个cgroup，分别是A和B，A的shares值是1024，B的shares值是512，那么A将获得1024/(1204+512)=66%的CPU资源，而B将获得33%的CPU资源。shares有两个特点：

* 如果A不忙，没有使用到66%的CPU时间，那么剩余的CPU时间将会被系统分配给B，即B的CPU使用率可以超过33%
* 如果添加了一个新的cgroup C，且它的shares值是1024，那么A的限额变成了1024/(1204+512+1024)=40%，B的变成了20%

从上面两个特点可以看出：

* 在闲的时候，shares基本上不起作用，只有在CPU忙的时候起作用，这是一个优点。
* 由于shares是一个绝对值，需要和其它cgroup的值进行比较才能得到自己的相对限额，而在一个部署很多容器的机器上，cgroup的数量是变化的，所以这个限额也是变化的，自己设置了一个高的值，但别人可能设置了一个更高的值，所以这个功能没法精确的控制CPU使用率。

#### cpu.stat
包含了下面三项统计结果

* nr_periods： 表示过去了多少个cpu.cfs_period_us里面配置的时间周期
* nr_throttled： 在上面的这些周期中，有多少次是受到了限制（即cgroup中的进程在指定的时间周期中用光了它的配额）
* throttled_time: cgroup中的进程被限制使用CPU持续了多长时间(纳秒)

##示例
这里以cfs_period_us & cfs_quota_us为例，演示一下如何控制CPU的使用率。
```bash
#继续使用上面创建的子cgroup： test
#设置只能使用1个cpu的20%的时间
dev@ubuntu:/sys/fs/cgroup/cpu,cpuacct/test$ sudo sh -c "echo 50000 > cpu.cfs_period_us"
dev@ubuntu:/sys/fs/cgroup/cpu,cpuacct/test$ sudo sh -c "echo 10000 > cpu.cfs_quota_us"

#将当前bash加入到该cgroup
dev@ubuntu:/sys/fs/cgroup/cpu,cpuacct/test$ echo $$
5456
dev@ubuntu:/sys/fs/cgroup/cpu,cpuacct/test$ sudo sh -c "echo 5456 > cgroup.procs"

#在bash中启动一个死循环来消耗cpu，正常情况下应该使用100%的cpu（即消耗一个内核）
dev@ubuntu:/sys/fs/cgroup/cpu,cpuacct/test$ while :; do echo test > /dev/null; done

#--------------------------重新打开一个shell窗口----------------------
#通过top命令可以看到5456的CPU使用率为20%左右，说明被限制住了
#不过这时系统的%us+%sy在10%左右，那是因为我测试的机器上cpu是双核的，
#所以系统整体的cpu使用率为10%左右
dev@ubuntu:~$ top
Tasks: 139 total,   2 running, 137 sleeping,   0 stopped,   0 zombie
%Cpu(s):  5.6 us,  6.2 sy,  0.0 ni, 88.2 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :   499984 total,    15472 free,    81488 used,   403024 buff/cache
KiB Swap:        0 total,        0 free,        0 used.   383332 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 5456 dev       20   0   22640   5472   3524 R  20.3  1.1   0:04.62 bash

#这时可以看到被限制的统计结果
dev@ubuntu:~$ cat /sys/fs/cgroup/cpu,cpuacct/test/cpu.stat
nr_periods 1436
nr_throttled 1304
throttled_time 51542291833
```

##结束语
使用cgroup限制CPU的使用率比较纠结，用cfs_period_us & cfs_quota_us吧，限制死了，没法充分利用空闲的CPU，用shares吧，又没法配置百分比，极其难控制。总之，使用cgroup的cpu子系统需谨慎。

##参考
* [CFS Bandwidth Control](https://www.kernel.org/doc/Documentation/scheduler/sched-bwc.txt)
* [cpu](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/sec-cpu.html)