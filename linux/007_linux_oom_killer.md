#Linux OOM killer

作为Linux下的程序员，有时不得不面对一个问题，那就是系统内存被用光了，这时当进程再向内核申请内存时，内核会怎么办呢？程序里面调用的malloc函数会返回null吗？

为了应对内存不足的情况，Linux内核发明了一种机制，叫OOM killer，通过配置它可以控制内存不足时内核的行为。

##OOM killer
当物理内存和交换空间都被用完时，如果还有进程来申请内存，内核将触发OOM killer，其行为如下：

1.检查文件/proc/sys/vm/panic_on_oom，如果里面的值为2，那么系统将触发panic
2.如果/proc/sys/vm/panic_on_oom的值为1，那么系统有可能触发panic（见后面的介绍）
3.如果/proc/sys/vm/panic_on_oom的值为0，或者上一步没有触发panic，那么内核继续检查文件/proc/sys/vm/oom_kill_allocating_task
3.如果/proc/sys/vm/oom_kill_allocating_task为1，那么内核将kill掉当前申请内存的进程
4.如果/proc/sys/vm/oom_kill_allocating_task为0，内核将检查每个进程的OOM分数，分数最高的进程将被kill掉

进程被kill掉之后，如果/proc/sys/vm/oom_dump_tasks为1，且系统的rlimit中设置了core文件大小，将会由/proc/sys/kernel/core_pattern里面指定的程序生成core dump文件，这个文件里将包含
pid, uid, tgid, vm size, rss, nr_ptes, nr_pmds, swapents, oom_score_adj
score, name等内容，拿到这个core文件之后，可以做一些分析，看为什么这个进程被选中kill掉。

这里可以看看ubuntu默认的配置：
```bash
#OOM后不panic，即不重启
dev@ubuntu:~$ cat /proc/sys/vm/panic_on_oom
0

#OOM后kill掉分数最高的进程
dev@ubuntu:~$ cat /proc/sys/vm/oom_kill_allocating_task
0

#进程由于OOM被kill掉后将生成core dump文件
dev@ubuntu:~$ cat /proc/sys/vm/oom_dump_tasks
1

#默认max core file size是0， 所以系统不会生成core文件
dev@ubuntu:~$ prlimit|grep CORE
CORE max core file size 0 unlimited blocks

#core dump文件的生成交给了apport，如果上面都设置了，在/var/crash目录下还是没有core文件，
#请确定apport是否已经安装，默认ubuntu不会安装这个软件，需要手动apt去安装
dev@ubuntu:~$ cat /proc/sys/kernel/core_pattern
|/usr/share/apport/apport %p %s %c %P
```
参考：[apport](https://wiki.ubuntu.com/Apport)

###panic_on_oom
正如上面所介绍的那样，该文件的值可以取0/1/2，0是不触发panlic，2是一定触发panlic，如果为1的话

panic后内核的默认行为是死在那里，目的是给开发人员一个连上去debug的机会。但对于大多数应用层开发人员来说没啥用，倒是希望它赶紧重启。为了让内核panic后重启，可以修改文件/proc/sys/kernel/panic，里面表示的是panic多少秒后系统将重启，这个文件的默认值是0，表示永远不重启。
```bash
#设置panic后3秒重启系统
dev@ubuntu:~$ sudo sh -c "echo 3 > /proc/sys/kernel/panic"
sudo sh -c "echo 2 > /proc/sys/vm/panic_on_oom"
sudo prlimit --core=unlimited:unlimited --pid $$
```

经过实际测试发现，在ubuntu 16.04里面，就算panic_on_oom的值是2，也不是每次都panic，

###调整分数

###修改配置
上面的这些文件可以通过下面三种方式来修改：
* 直接写文件（重启后失效）
```bash
dev@ubuntu:~$ sudo sh -c "echo 1 > /proc/sys/vm/panic_on_oom"
```
* 通过控制命令（重启后失效）
* 修改配置文件（重启后继续生效）

##
[ulimt](http://man7.org/linux/man-pages/man3/ulimit.3.html)
##cgroup的OOM killer

##
[ulimt](http://man7.org/linux/man-pages/man3/ulimit.3.html)
我们之所以关心内存用光时内核的行为，是因为我们想知道内核的行为会对我们的进程造成什么影响。在讨论内核的行为之前，我们先来看看对进程来说，分配内存时受到哪些因素的限制，然后看看达到相应的限制之后会有什么后果。
写C代码时，总是担心内存不够时调用malloc函数会返回null，所以编程时需要考虑malloc函数返回null的情况，在Linux系统下，我们也要这么办吗？

深入了解了Linux才知道，原来Linux内核提供了一种叫做OOM killer的机制，当系统内存不足时，如果进程继续向内核分配内存，那么就会触发OOM killer，对于当前申请内存的进程来说，只有两种可能后果，一种是进程被kill掉，另一种是得到分配好的内存，所以就不存在malloc函数返回null的情况，当然为了程序有较好的移植性，还是需要考虑malloc返回null的情况，因为其它非Linux系统可能没有类似的OOM killer机制。

ENOMEM Out of memory. Possibly, the application hit the RLIMIT_AS or RLIMIT_DATA limit described in getrlimit(2).

This functionality is needed if we want to do anything other than simply kill the process on the machine that will end up freeing the most memory — the only thing the OOM killer is guaranteed to do. Some influence on that heuristic is available through /proc/<pid>/oom_score_adj, which either biases or discounts an amount of memory for a process, but we can't do anything else and we can't possibly implement all practical OOM-kill responses into the kernel itself.

[sysctl/vm.txt](https://www.kernel.org/doc/Documentation/sysctl/vm.txt)
[How to Configure the Linux Out-of-Memory Killer](http://www.oracle.com/technetwork/articles/servers-storage-dev/oom-killer-1911807.html)